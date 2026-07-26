> HTML Page: [Open HTML Page](HTML%20Pages/Free%20Notes/Tech/Operating%20Systems/Linux%20OS/Setup%20Guides/Postfix%20Incoming%20Mail%20Setup.html)

[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

**Table of Contents**
- [Purpose of This Setup](#Purpose%20of%20This%20Setup)
- [Protocol Roles](#Protocol%20Roles)
- [Typical Data Flow](#Typical%20Data%20Flow)
- [Objective](#Objective)
- [Required Ports](#Required%20Ports)
- [Analyst Relevance](#Analyst%20Relevance)
- [Install Postfix](#Install%20Postfix)
	- [](#Install%20Postfix#Install%20Postfix#Configuration%20Choice|Configuration%20Choice)
- [Install Mail Utilities](#Install%20Mail%20Utilities)
- [Purpose](#Purpose)
- [Commands](#Commands)
- [Analyst Notes](#Analyst%20Notes)
- [Test SMTP Connectivity](#Test%20SMTP%20Connectivity)
- [If Connection Fails](#If%20Connection%20Fails)
- [Send Manual Test Email](#Send%20Manual%20Test%20Email)
- [Verify Mail Delivery](#Verify%20Mail%20Delivery)
- [Why Maildir Matters](#Why%20Maildir%20Matters)
- [Configure Maildir](#Configure%20Maildir)
- [Test Maildir Access](#Test%20Maildir%20Access)
- [Troubleshooting Maildir Error](#Troubleshooting%20Maildir%20Error)
- [Install Courier IMAP](#Install%20Courier%20IMAP)
- [Enable Authentication Service](#Enable%20Authentication%20Service)
- [Analyst Relevance](#Analyst%20Relevance)
- [Set Local Domains](#Set%20Local%20Domains)
- [Define Trusted Networks](#Define%20Trusted%20Networks)
- [Enable External Interfaces](#Enable%20External%20Interfaces)
- [Configure Address Delimiter](#Configure%20Address%20Delimiter)
- [Restart Postfix](#Restart%20Postfix)
- [Connect to Server](#Connect%20to%20Server)
- [Send External Test Email](#Send%20External%20Test%20Email)
- [Verify Delivery](#Verify%20Delivery)
- [Connect to IMAP](#Connect%20to%20IMAP)
- [Login Test](#Login%20Test)
- [SMTP Failures](#SMTP%20Failures)
	- [](#SMTP%20Failures#SMTP%20Failures#Symptom|Symptom)
- [Recipient Lookup Failure](#Recipient%20Lookup%20Failure)
	- [](#Recipient%20Lookup%20Failure#Recipient%20Lookup%20Failure#Symptom|Symptom)
- [IMAP Authentication Failure](#IMAP%20Authentication%20Failure)
	- [](#IMAP%20Authentication%20Failure#IMAP%20Authentication%20Failure#Symptom|Symptom)
- [Open Relay Prevention](#Open%20Relay%20Prevention)
- [Mail Routing Risks](#Mail%20Routing%20Risks)
- [Local Mail Handling Risks](#Local%20Mail%20Handling%20Risks)
- [Key Log Events](#Key%20Log%20Events)
- [SOC Notification Pipeline](#SOC%20Notification%20Pipeline)
- [Malware Sandbox Alerts](#Malware%20Sandbox%20Alerts)
- [Incident Ticket Automation](#Incident%20Ticket%20Automation)
- [Package Installation](#Package%20Installation)
- [User Setup](#User%20Setup)
- [SMTP Testing](#SMTP%20Testing)
- [Service Management](#Service%20Management)
- [Maildir Configuration](#Maildir%20Configuration)
- [Network Configuration](#Network%20Configuration)

# Linux Email Infrastructure : Postfix Incoming Mail Setup

# 1. Core Architecture Understanding

## Purpose of This Setup

Configure a **Linux-based mail transfer agent (MTA)** using **Postfix** to:

- Receive incoming email via **SMTP (port 25)**
- Store email locally
- Provide IMAP access for email retrieval
- Support automated systems such as:
    - Ticketing systems
    - Git services
    - Alerting pipelines
    - SOC notification workflows

---

## Protocol Roles

|Protocol|Port|Purpose|
|---|---|---|
|SMTP|25|Receive incoming email|
|IMAP|143|Retrieve stored email|
|Maildir|N/A|Mail storage format|

---

## Typical Data Flow

Internet  
   ↓  
SMTP (Port 25)  
   ↓  
Postfix  
   ↓  
Maildir Storage  
   ↓  
IMAP Server  
   ↓  
Client/System Access

---

# 2. Firewall Configuration

## Objective

Allow inbound email traffic.

---

## Required Ports

|Port|Service|Purpose|
|---|---|---|
|25|SMTP|Receive email|
|143|IMAP|Read email|

---

## Analyst Relevance

Firewalls blocking these ports produce:

- SMTP connection failures
- IMAP authentication errors
- Delivery failures

---

# 3. Package Installation

## Install Postfix

sudo apt-get install postfix

### Configuration Choice

Select:

Internet Site

Hostname must match:

mail.example.com

---

## Install Mail Utilities

sudo apt-get install mailutils

Used for:

- Sending test emails
- Verifying local delivery

---

# 4. Create Mail User

## Purpose

Create dedicated mailbox identity.

---

## Commands

sudo useradd -m -s /bin/bash incoming

sudo passwd incoming

---

## Analyst Notes

- User mailbox stored at:

/var/mail/incoming

or:

/home/incoming/Maildir

---

# 5. Service Validation — SMTP Testing

## Test SMTP Connectivity

telnet localhost 25

Expected:

220 mail.example.com ESMTP Postfix

---

## If Connection Fails

Check service:

sudo postfix status

Start service:

sudo postfix start

---

## Send Manual Test Email

SMTP session:

ehlo localhost  
mail from: root@localhost  
rcpt to: incoming@localhost  
data  
Subject: Test  
  
Hello  
.  
quit

---

## Verify Mail Delivery

su - incoming  
mail

Exit:

q

---

# 6. Mail Storage Format — Maildir Configuration

## Why Maildir Matters

Maildir:

- Required by IMAP services
- Stores emails as files
- Improves reliability and concurrency

---

## Configure Maildir

sudo postconf -e "home_mailbox = Maildir/"

Restart Postfix:

sudo /etc/init.d/postfix restart

---

## Test Maildir Access

su - incoming  
MAIL=/home/incoming/Maildir  
mail

---

## Troubleshooting Maildir Error

If:

Maildir: Is a directory

Install:

sudo apt-get install heirloom-mailx

---

# 7. Install IMAP Server

## Install Courier IMAP

sudo apt-get install courier-imap

Start IMAP daemon:

imapd start

---

## Enable Authentication Service

sudo service courier-authdaemon start

Enable auto-start:

sudo systemctl enable courier-authdaemon

---

## Analyst Relevance

Authentication failure usually caused by:

- authdaemon not running
- Password mismatch
- User permission issues

---

# 8. Configure Postfix for Internet Mail

## Set Local Domains

sudo postconf -e "mydestination = mail.example.com, localhost.localdomain, localhost"

---

## Define Trusted Networks

Example:

sudo postconf -e "mynetworks = 127.0.0.0/8, 192.168.1.0/24"

Used to:

- Prevent unauthorized relay
- Define internal hosts

---

## Enable External Interfaces

sudo postconf -e "inet_interfaces = all"

Allows:

Internet email reception

---

## Configure Address Delimiter

sudo postconf -e "recipient_delimiter = +"

Example:

user+alerts@example.com

Useful for:

- Email routing
- Automated parsing

---

## Restart Postfix

sudo service postfix restart

---

# 9. Final SMTP Testing (External)

## Connect to Server

telnet mail.example.com 25

Expected:

220 mail.example.com ESMTP Postfix

---

## Send External Test Email

ehlo mail.example.com  
mail from: root@mail.example.com  
rcpt to: incoming@mail.example.com  
data  
Subject: Test  
  
Hello  
.  
quit

---

## Verify Delivery

su - incoming  
MAIL=/home/incoming/Maildir  
mail

---

# 10. IMAP Authentication Testing

## Connect to IMAP

telnet mail.example.com 143

Expected:

Courier-IMAP ready

---

## Login Test

a login incoming PASSWORD

Expected:

a OK LOGIN Ok.

Logout:

a logout

---

# 11. Troubleshooting Indicators

## SMTP Failures

### Symptom

Connection refused

Likely Causes:

- Firewall blocking port 25
- Postfix not running

---

## Recipient Lookup Failure

### Symptom

Temporary lookup failure

Cause:

Incorrect mynetworks setting

---

## IMAP Authentication Failure

### Symptom

Login failed

Likely Causes:

- courier-authdaemon stopped
- Password mismatch

---

# 12. Security-Relevant Concepts

## Open Relay Prevention

Controlled by:

mynetworks

Improper configuration results in:

Open relay abuse  
Spam relay attacks  
Blacklist inclusion

---

## Mail Routing Risks

Misconfigured:

inet_interfaces = all

Without filtering exposes:

Public SMTP attack surface

---

## Local Mail Handling Risks

Incorrect:

mydestination

Causes:

Mail loops  
Undelivered mail  
Backscatter attacks

---

# 13. Detection and Monitoring Opportunities

Postfix logs typically located:

/var/log/mail.log

or:

/var/log/syslog

---

## Key Log Events

Look for:

connect from  
disconnect from  
NOQUEUE  
status=sent  
status=bounced  
status=deferred

These support:

- Incident response
- Email forensics
- Threat detection

---

# 14. Analyst Operational Use Cases

## SOC Notification Pipeline

Example:

IDS Alert → Email → Postfix → Analyst Inbox

---

## Malware Sandbox Alerts

Email ingestion system:

Malware Detected → Alert Email → Postfix → Processing System

---

## Incident Ticket Automation

Incoming:

alerts@domain.com

Parsed by:

Automation scripts

---

# 15. Hardening Considerations

Essential defensive actions:

- Restrict `mynetworks`
- Monitor logs
- Enable authentication
- Implement TLS (not shown in base config)
- Configure SPF/DKIM/DMARC
- Limit exposed interfaces

---

# 16. Key Files and Locations

|File|Purpose|
|---|---|
|`/etc/postfix/main.cf`|Main configuration|
|`/var/mail/`|Mail storage|
|`/home/<user>/Maildir/`|Maildir storage|
|`/var/log/mail.log`|Logs|

---

# 17. High-Value Commands Summary

## Package Installation

sudo apt-get install postfix  
sudo apt-get install mailutils  
sudo apt-get install courier-imap  
sudo apt-get install heirloom-mailx

---

## User Setup

sudo useradd -m -s /bin/bash incoming  
sudo passwd incoming

---

## SMTP Testing

telnet localhost 25

---

## Service Management

sudo postfix status  
sudo postfix start  
sudo service postfix restart  
sudo /etc/init.d/postfix restart

---

## Maildir Configuration

sudo postconf -e "home_mailbox = Maildir/"

---

## Network Configuration

sudo postconf -e "mydestination = mail.example.com, localhost.localdomain, localhost"  
  
sudo postconf -e "mynetworks = 127.0.0.0/8, 192.168.1.0/24"  
  
sudo postconf -e "inet_interfaces = all"  
  
sudo postconf -e "recipient_delimiter = +"

---

# 18. Blue-Team Skill Relevance

This setup supports training in:

- Linux service management
- SMTP/IMAP protocol understanding
- Email infrastructure troubleshooting
- Log analysis
- Network port validation
- Mail-based alert workflows

---

# 19. Red-Team and Threat Simulation Relevance

Postfix infrastructure enables:

- Email phishing simulations
- Mail relay exploitation labs
- SMTP fuzzing testing
- Mail-based payload delivery testing

---

# 20. Practical Learning Value

Mastering this workflow builds foundational competency in:

- Linux-based messaging infrastructure
- Network service exposure
- Authentication workflows
- Protocol-level troubleshooting
- Incident-driven email handling

These skills directly intersect with:

- SOC operations
- Incident response
- Email threat analysis
- Infrastructure security engineering