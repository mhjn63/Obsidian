> HTML Page: [Open HTML Page](HTML%20Pages/Free%20Notes/Cybersecurity/Bug%20Bounty/Bug%20Bounty%20Cheat%20Sheet.html)

🏠 [Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Get the full BUH BOUNTY PACK in PDF [here](https://buymeacoffee.com/notescatalog/e/528934)

**Table of Contents**
- [01 · Setup & Initial Tips](#01%20·%20Setup%20&%20Initial%20Tips)
	- [](#01%20·%20Setup%20&%20Initial%20Tips#01%20·%20Setup%20&%20Initial%20Tips#Burp%20Suite%20Scope%20Regex|Burp%20Suite%20Scope%20Regex)
	- [](#01%20·%20Setup%20&%20Initial%20Tips#01%20·%20Setup%20&%20Initial%20Tips#Pull%20Root%20Subdomains%20from%20a%20Final%20Subdomain%20List|Pull%20Root%20Subdomains%20from%20a%20Final%20Subdomain%20List)
	- [](#01%20·%20Setup%20&%20Initial%20Tips#01%20·%20Setup%20&%20Initial%20Tips#Port%20Scanning%20IP%20Ranges|Port%20Scanning%20IP%20Ranges)
	- [](#01%20·%20Setup%20&%20Initial%20Tips#01%20·%20Setup%20&%20Initial%20Tips#Automation%20Frameworks|Automation%20Frameworks)
- [02 · Subdomain Enumeration](#02%20·%20Subdomain%20Enumeration)
	- [](#02%20·%20Subdomain%20Enumeration#02%20·%20Subdomain%20Enumeration#Subfinder%20(Passive%20Enumeration)|Subfinder%20(Passive%20Enumeration))


> A methodology-driven cheatsheet covering subdomain enumeration, ASN recon, content discovery, JavaScript analysis, fingerprinting, sensitive loot hunting, and exploitation techniques for web application bug bounty programs.

---

## 01 · Setup & Initial Tips
Before diving into subdomain enumeration, a few foundational habits will pay off across an entire engagement. Running recon tasks on a VPS rather than a local workstation offloads CPU-intensive jobs, keeps scans running after you close your laptop, and separates your main manual testing environment from background automation. DigitalOcean and Linode are both popular choices for this — always run long jobs inside a `screen` session and pipe output with `| tee` so nothing is lost if the connection drops.

### Burp Suite Scope Regex

A regex that is easy to forget but essential for keeping Burp's scope tight on wildcard programs:

```
.*\.domain\.com$
```

### Pull Root Subdomains from a Final Subdomain List

When you have a large, flat list of subdomains and want to extract just the root-level third-level domains for further processing:

```bash
cat final | rev | cut -d . -f 1-3 | rev | sort -u | tee root.subdomains
```

### Port Scanning IP Ranges

For large programs covering thousands of IP addresses, use Shodan, Google Dorks, and ASN lookups to find the target CIDR ranges first, then scan with nmap on a VPS. Despite the speed advantages of masscan, it tends to miss ports — VPS + nmap + screen is more reliable. Results can be imported into [IVRE](https://ivre.rocks/) for a visual overview.

### Automation Frameworks

**Sn1per** (`https://github.com/1N3/Sn1per`) automates a large portion of the recon and scanning workflow. A license is available for ~$200 which is quickly recouped from a single bounty payout.

---

## 02 · Subdomain Enumeration

Subdomain enumeration is almost always the first step for any program with a wildcard scope (`*.domain.com`). The goal is to discover the largest possible attack surface before narrowing in on interesting targets. Use multiple passive sources in parallel, then resolve with MassDNS, probe for live HTTP/S services with httprobe, and screenshot with EyeWitness or Aquatone.

### Subfinder (Passive Enumeration)

Populate all API keys before running — Shodan Pro is particularly valuable here.

```bash
subfinder -d domain.com -o Outfile.txt
```
