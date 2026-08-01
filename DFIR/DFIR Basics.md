🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](shop.motasem-notes.net)

- [[#Full DFIR Study Notes|Full DFIR Study Notes]]
- [[#1. Cheatsheet Purpose|1. Cheatsheet Purpose]]
- [[#2. The Golden Rule of Computer Investigations|2. The Golden Rule of Computer Investigations]]
- [[#3. Investigation Preparation Mindset|3. Investigation Preparation Mindset]]
	- [[#3. Investigation Preparation Mindset#Investigation Goals|Investigation Goals]]
- [[#4. Computer Crime vs. Corporate Policy Violation|4. Computer Crime vs. Corporate Policy Violation]]
	- [[#4. Computer Crime vs. Corporate Policy Violation#Computer Crime Scenario|Computer Crime Scenario]]
	- [[#4. Computer Crime vs. Corporate Policy Violation#Company Policy Violation Scenario|Company Policy Violation Scenario]]
- [[#5. Systematic Investigation Workflow|5. Systematic Investigation Workflow]]
	- [[#5. Systematic Investigation Workflow#Practical Rule|Practical Rule]]
- [[#6. Case Assessment Template|6. Case Assessment Template]]
	- [[#6. Case Assessment Template#Example Assessment Logic|Example Assessment Logic]]
- [[#7. Investigation Planning Checklist|7. Investigation Planning Checklist]]
- [[#8. Evidence Security and Chain of Custody|8. Evidence Security and Chain of Custody]]
	- [[#8. Evidence Security and Chain of Custody#Chain of Custody|Chain of Custody]]
	- [[#8. Evidence Security and Chain of Custody#Evidence Custody Form Fields|Evidence Custody Form Fields]]
	- [[#8. Evidence Security and Chain of Custody#Single-Evidence vs. Multi-Evidence Forms|Single-Evidence vs. Multi-Evidence Forms]]
	- [[#8. Evidence Security and Chain of Custody#Secure Storage Requirements|Secure Storage Requirements]]
- [[#9. Evidence Handling Rules|9. Evidence Handling Rules]]
- [[#10. Corporate High-Tech Investigation Playbooks|10. Corporate High-Tech Investigation Playbooks]]
	- [[#10. Corporate High-Tech Investigation Playbooks#A. Employee Termination Cases|A. Employee Termination Cases]]
	- [[#10. Corporate High-Tech Investigation Playbooks#B. Internet Abuse Investigations|B. Internet Abuse Investigations]]
	- [[#10. Corporate High-Tech Investigation Playbooks#C. E-mail Abuse Investigations|C. E-mail Abuse Investigations]]
	- [[#10. Corporate High-Tech Investigation Playbooks#D. Attorney-Client Privilege Investigations|D. Attorney-Client Privilege Investigations]]
	- [[#10. Corporate High-Tech Investigation Playbooks#E. Media Leak Investigations|E. Media Leak Investigations]]
	- [[#10. Corporate High-Tech Investigation Playbooks#F. Industrial Espionage Investigations|F. Industrial Espionage Investigations]]
- [[#11. Interviews vs. Interrogations|11. Interviews vs. Interrogations]]
	- [[#11. Interviews vs. Interrogations#Preparation Questions|Preparation Questions]]
	- [[#11. Interviews vs. Interrogations#Good Interview Traits|Good Interview Traits]]
- [[#12. Forensic Workstation Essentials|12. Forensic Workstation Essentials]]
	- [[#12. Forensic Workstation Essentials#Minimum Core Setup|Minimum Core Setup]]
	- [[#12. Forensic Workstation Essentials#Useful Extras|Useful Extras]]
	- [[#12. Forensic Workstation Essentials#Tool Reality|Tool Reality]]
- [[#13. Why Normal Windows Booting Is Dangerous|13. Why Normal Windows Booting Is Dangerous]]
- [[#14. Bit-Stream Copy and Image Concepts|14. Bit-Stream Copy and Image Concepts]]
	- [[#14. Bit-Stream Copy and Image Concepts#Bit-Stream Copy|Bit-Stream Copy]]
	- [[#14. Bit-Stream Copy and Image Concepts#Bit-Stream Image|Bit-Stream Image]]
	- [[#14. Bit-Stream Copy and Image Concepts#Backup vs. Forensic Image|Backup vs. Forensic Image]]
	- [[#14. Bit-Stream Copy and Image Concepts#Visual Model|Visual Model]]
- [[#15. Acquisition Workflow with ProDiscover Basic|15. Acquisition Workflow with ProDiscover Basic]]
	- [[#15. Acquisition Workflow with ProDiscover Basic#Data Acquistion|Data Acquistion]]
	- [[#15. Acquisition Workflow with ProDiscover Basic#USB Acquisition Flow|USB Acquisition Flow]]
	- [[#15. Acquisition Workflow with ProDiscover Basic#Key Acquisition Outputs|Key Acquisition Outputs]]
	- [[#15. Acquisition Workflow with ProDiscover Basic#Key Acquisition Terms|Key Acquisition Terms]]
	- [[#15. Acquisition Workflow with ProDiscover Basic#Remember:|Remember:]]
- [[#16. Analysis Workflow with ProDiscover Basic|16. Analysis Workflow with ProDiscover Basic]]
	- [[#16. Analysis Workflow with ProDiscover Basic#Load Evidence Image|Load Evidence Image]]
	- [[#16. Analysis Workflow with ProDiscover Basic#Search for Evidence|Search for Evidence]]
	- [[#16. Analysis Workflow with ProDiscover Basic#Search Strategy|Search Strategy]]
	- [[#16. Analysis Workflow with ProDiscover Basic#Boolean Search Logic|Boolean Search Logic]]
- [[#17. Deleted File Logic|17. Deleted File Logic]]
- [[#18. Exporting Evidence Files|18. Exporting Evidence Files]]
- [[#19. Reporting Workflow|19. Reporting Workflow]]
	- [[#19. Reporting Workflow#Report Requirements|Report Requirements]]
	- [[#19. Reporting Workflow#Repeatable Findings|Repeatable Findings]]
- [[#20. Investigator Journal|20. Investigator Journal]]
- [[#21. Completing the Case|21. Completing the Case]]
- [[#22. Case Critique Checklist|22. Case Critique Checklist]]
- [[#Key Terms|Key Terms]]
- [[#24. Fast Investigation Memory Aid|24. Fast Investigation Memory Aid]]
- [[#25. Practical Mistakes to Avoid|25. Practical Mistakes to Avoid]]
- [[#26. One-Page Field Checklist|26. One-Page Field Checklist]]


## Full DFIR Study Notes

**Fourthwall**
```
https://shop.motasem-notes.net/products/dfir-study-notes
```

**BuyMeaCoffee**
```
https://buymeacoffee.com/notescatalog/e/142831
```
## 1. Cheatsheet Purpose

Cheatsheet teaches how to manage a computer investigation from intake to final reporting. The core lesson is simple: **a forensic investigation is not just “finding files.” It is a controlled process for preserving evidence, proving repeatable findings, and producing defensible conclusions.**

The Cheatsheet centers on six practical capabilities:

1. Preparing a computer investigation.
2. Applying a systematic investigation approach.
3. Handling corporate high-tech investigations.
4. Setting up a data recovery / forensic workstation.
5. Conducting acquisition and analysis.
6. Completing, reporting, and critiquing the case.

---

## 2. The Golden Rule of Computer Investigations

> **Preserve the original evidence. Analyze only a forensic copy.**

In practice, this means:

- Do not browse around the original drive.
- Do not boot the evidence system casually.
- Do not let the operating system write to the evidence disk.
- Use write-blocking when acquiring evidence.
- Create a bit-stream image.
- Validate the image when possible.
- Document every transfer, action, and finding.

A weak investigation usually fails because of poor evidence handling, not because the investigator failed to find a file.

---

## 3. Investigation Preparation Mindset

A computer forensics professional gathers evidence from a suspect system and determines whether the evidence supports a crime, policy violation, or other allegation.

### Investigation Goals

| Goal | Meaning |
|---|---|
| Preserve evidence | Keep original data intact and uncontaminated. |
| Establish chain of custody | Prove who handled evidence, when, why, and where. |
| Identify relevant artifacts | Locate files, deleted data, e-mails, logs, metadata, or hidden content. |
| Maintain repeatability | Another qualified investigator should be able to reproduce the same result. |
| Report clearly | Explain what happened in language useful to management, lawyers, or court. |

---

## 4. Computer Crime vs. Corporate Policy Violation

Chapter 2 separates investigation examples into two broad categories.

### Computer Crime Scenario

A law enforcement case may involve seized computers, removable media, phones, PDAs, or storage devices. The investigator receives evidence already collected by officers and must examine it without altering it.

Key points:

- Officers should document seized items.
- Running systems should be photographed before shutdown.
- Live acquisition may be needed before powering down, especially for RAM.
- Evidence can include intact files, deleted files, e-mail, hidden files, password-protected files, and removable media.
- Legacy operating systems may require older forensic tools.

### Company Policy Violation Scenario

A corporate case may involve employee misuse of company systems, such as side-business activity, Internet abuse, e-mail abuse, or data leakage.

Key points:

- Corporate investigations depend heavily on company policies.
- Employees may have reduced privacy expectations only if policies and banners are clear.
- Policy violations can still become civil or criminal matters.
- Chain of custody still matters, even for internal HR investigations.


### Key Terms in Crime Scene Processing
`probable cause`: The standard specifying whether a police officer has the right to make an
arrest, conduct a personal or property search, or obtain a warrant for arrest.
`professional curiosity` : The motivation for law enforcement and other professional personnel
to examine an incident or crime scene to see what happened.
`Scientific Working Group on Digital Evidence (SWGDE)` : A group that sets standards for
recovering, preserving, and examining digital evidence.
`Secure Hash Algorithm version 1 (SHA-1)` : A forensic hashing algorithm created by NIST to
determine whether data in a file or on storage media has been altered.
`sniffing` : Detecting data transmissions to and from a suspect’s computer and a network
server to determine the type of data being transmitted over a network.

`4-mm DAT` : Magnetic tapes that store about 4 GB of data, but like CD-Rs, are slow to read
and write data.
`Automated Fingerprint Identification Systems (AFIS)` : A computerized system for identifying
fingerprints that’s connected to a central database; used to identify criminal suspects and
review thousands of fingerprint samples at high speed.
`computer-generated records` : Data generated by a computer, such as system log files or
proxy server logs.
`computer-stored records` : Digital files generated by a person, such as electronic spreadsheets.
`covert surveillance` : Observing people or places without being detected, often using
electronic equipment, such as video cameras or key stroke/screen capture programs.
`Cyclic Redundancy Check (CRC)` : A mathematical algorithm that translates a file into a
unique hexadecimal value.
`digital evidence` : Evidence consisting of information stored or transmitted in electronic form.
`extensive-response field kit` : A portable kit designed to process several computers and a variety of operating systems at a crime or incident scene involving computers. This kit should contain two or more types of software or hardware computer forensics tools, such as extra storage drives.
`hash value` : A unique hexadecimal value that identifies a file or drive.
`hazardous materials (HAZMAT)`: Chemical, biological, or radiological substances that can
cause harm to people.
`initial-response` : field kit A portable kit containing only the minimum tools needed to
perform disk acquisitions and preliminary forensics analysis in the field.
`innocent information` : Data that doesn’t contribute to evidence of a crime or violation.
`International Organization on Computer Evidence (IOCE)` : A group that sets standards for
recovering, preserving, and examining digital evidence.
`keyed hash set` : A value created by an encryption utility’s secret key.
`limiting phrase` : Wording in a search warrant that limits the scope of a search for evidence.
`low-level investigations Corporate`: cases that require less effort than a major criminal case.
`Message Digest 5 (MD5)` : An algorithm that produces a hexadecimal value of a file or
storage media. Used to determine whether data has been changed.
`National Institute of Standards and Technology (NIST)`  One of the governing bodies
responsible for setting standards for various U.S. industries.
nonkeyed hash set A unique hash numbered generated by a software tool and used to
identify files.
person of interest Someone who might be a suspect or someone with additional knowledge
that can provide enough evidence of probable cause for a search warrant or arrest.
plain view doctrine When conducting a search and seizure, objects in plain view of a law
enforcement officer, who has the right to be in position to have that view, are subject to
seizure without a warrant and can be introduced as evidence. As applied to executing
searches of computers, the plain view doctrine’s limitations are less clear.
5
probable cause The standard specifying whether a police officer has the right to make an
arrest, conduct a personal or property search, or obtain a warrant for arrest.
professional curiosity The motivation for law enforcement and other professional personnel
to examine an incident or crime scene to see what happened.
Scientific Working Group on Digital Evidence (SWGDE) A group that sets standards for
recovering, preserving, and examining digital evidence.
Secure Hash Algorithm version 1 (SHA-1) A forensic hashing algorithm created by NIST to
determine whether data in a file or on storage media has been altered.
sniffing Detecting data transmissions to and from a suspect’s computer and a network
server to determine the type of data being transmitted over a network.

---

## 5. Systematic Investigation Workflow

Use this sequence as the backbone for almost every case.

| Phase | Practical Action |
|---|---|
| 1. Initial assessment | Determine the case type, accusation, evidence location, involved people, and requester. |
| 2. Preliminary approach | Outline the likely steps before touching evidence. |
| 3. Detailed checklist | Convert the approach into step-by-step actions with estimated time. |
| 4. Resource planning | Identify tools, OS support, storage, experts, and legal contacts needed. |
| 5. Evidence acquisition | Obtain and copy evidence media using a forensic process. |
| 6. Risk identification | List what could go wrong, such as encryption, damaged media, passwords, legacy systems, or booby traps. |
| 7. Risk mitigation | Create backup images, use alternate tools, involve specialists, or adjust the plan. |
| 8. Test the design | Validate copies, review hashes, and confirm the plan is working. |
| 9. Analyze evidence | Search, recover, inspect, and correlate artifacts. |
| 10. Investigate recovered data | Determine whether artifacts support or refute the allegation. |
| 11. Complete report | Document actions, findings, limitations, and conclusions. |
| 12. Critique the case | Review what worked, what failed, and what should change next time. |

### Practical Rule

For small cases, a simple plan may be enough. For cases involving many systems, multiple media types, or legal exposure, build a detailed plan and update it as new facts emerge.

---

## 6. Case Assessment Template

Use this template before acquiring or analyzing evidence.

```text
Case name:
Case number:
Requester:
Case type:
Nature of allegation:
Business/legal context:
Known systems involved:
Known storage media:
Operating system:
Known file system:
Evidence location:
Who collected the evidence:
Current evidence custodian:
Potential risks:
Required tools:
Required specialists:
Expected artifacts:
Reporting audience:
Deadline:
```

### Example Assessment Logic

For a suspected employee side-business case:

- **Situation:** Employee abuse / corporate resource misuse.
- **Nature:** Non-work activity performed on company systems.
- **Evidence:** USB drive, workstation, browser history, office documents, e-mail, spreadsheet records.
- **Likely artifacts:** Domain names, client lists, invoices, spreadsheets, letters, timestamps, deleted files.
- **Key question:** Did the employee conduct personal business using company time or systems?

---

## 7. Investigation Planning Checklist

After assessment, convert the case into an operational plan.

```text
[ ] Acquire the evidence from the current custodian.
[ ] Complete evidence custody documentation.
[ ] Transport evidence safely.
[ ] Place original evidence in an approved secure container.
[ ] Prepare forensic workstation.
[ ] Retrieve evidence only when ready to image it.
[ ] Create a forensic image / bit-stream copy.
[ ] Validate the image if possible.
[ ] Return original evidence to secure storage.
[ ] Analyze only the image or working copy.
[ ] Export relevant files carefully.
[ ] Record all tool actions, logs, and notes.
[ ] Produce a report.
[ ] Preserve reports, images, notes, and exported files.
[ ] Critique the investigation after closure.
```

---

## 8. Evidence Security and Chain of Custody

### Chain of Custody

**Chain of custody** is the route evidence takes from collection to case closure or court. It proves evidence integrity by documenting each transfer and handling event.

A broken chain of custody can make otherwise strong evidence questionable.

### Evidence Custody Form Fields

A good custody form should capture:

| Field | Purpose |
|---|---|
| Case number | Links item to investigation. |
| Investigating organization | Identifies responsible entity. |
| Investigator | Names the person handling the evidence. |
| Evidence description | Describes the item clearly. |
| Vendor / manufacturer | Helps identify hardware. |
| Model / serial number | Uniquely identifies device where possible. |
| Condition | Notes damage, labels, seals, or status. |
| Date/time collected | Establishes timeline. |
| Collected by | Identifies original collector. |
| Location collected | Shows where evidence originated. |
| Transfer log | Records each handoff. |
| Purpose of transfer | Explains why evidence was accessed. |
| Signatures | Confirms custody handoff. |

### Single-Evidence vs. Multi-Evidence Forms

| Form Type | Best Used When |
|---|---|
| Single-evidence form | You need detailed tracking for each item. |
| Multi-evidence form | A case contains many related items and a summary form is enough. |

### Secure Storage Requirements

Use an **approved secure container**, such as a locked fireproof cabinet, locker, or safe with limited access. Original media should go back into secure storage immediately after imaging.

---

## 9. Evidence Handling Rules

```text
DO:
[ ] Use antistatic bags for drives and electronic media.
[ ] Use antistatic wrist straps or pads when handling components.
[ ] Keep media away from heat, moisture, magnets, and radio equipment.
[ ] Transport evidence in a safe environment.
[ ] Record who handled the evidence and why.
[ ] Store originals securely.
[ ] Work from forensic copies.

DON'T:
[ ] Boot an evidence drive in a normal Windows environment.
[ ] Browse the original media directly.
[ ] Let unauthorized personnel access evidence.
[ ] Leave evidence forms unsecured.
[ ] assume removable media is safe from static or environmental damage.
```

---

## 10. Corporate High-Tech Investigation Playbooks

Chapter 2 gives practical procedures for several corporate investigation types.

---

### A. Employee Termination Cases

Use when an employee is leaving, has been fired, or is suspected of taking company data.

**Focus areas:**

- Workstation and laptop storage.
- USB drives and removable media.
- E-mail folders.
- File shares.
- Cloud or remote access logs.
- Recent file access and transfer activity.

**Practical steps:**

```text
[ ] Coordinate with HR, legal, and management.
[ ] Identify systems and accounts used by the employee.
[ ] Preserve relevant devices and storage.
[ ] Disable access at the proper time.
[ ] Acquire images before reuse or reassignment.
[ ] Search for company data, personal archives, exfiltration traces, and deletion activity.
[ ] Report only relevant, supportable findings.
```

---

### B. Internet Abuse Investigations

Use when an employee is suspected of inappropriate browsing, unauthorized downloads, or misuse of Internet access.

**Evidence sources:**

- Proxy server logs.
- Browser cache.
- Browser history.
- Download folders.
- DNS logs.
- Endpoint artifacts.
- Time/date metadata.

**Workflow:**

```text
[ ] Confirm the company has acceptable-use policies.
[ ] Review applicable privacy laws.
[ ] Collect proxy/firewall logs.
[ ] Acquire the suspect workstation image.
[ ] Compare recovered artifacts with server-side logs.
[ ] Verify whether activity came from company network or external ISP.
[ ] Report whether evidence supports the allegation.
```

**Important:** Privacy law differs by jurisdiction. In international environments, what is allowed in one country may be restricted in another.

---

### C. E-mail Abuse Investigations

Use for spam, harassment, threats, offensive content, or unauthorized communication.

**Evidence needed:**

- Copy of offending e-mail with full headers.
- E-mail server logs, if available.
- Server mailbox data.
- Local Outlook `.pst` or `.ost` files.
- Webmail traces from browser cache/history.
- Forensic image of local machine when needed.

**Workflow:**

```text
[ ] Preserve offending messages with headers.
[ ] Contact e-mail administrator for logs and mailbox exports.
[ ] Acquire local machine if messages are stored locally.
[ ] Search for relevant e-mail addresses, names, subjects, and phrases.
[ ] Examine message headers.
[ ] Correlate sender, recipient, timestamp, IP, and server records.
```

---

### D. Attorney-Client Privilege Investigations

Use when working under direction of an attorney.

**Core rule:** The attorney controls the investigation. Findings must remain confidential.

**Workflow:**

```text
[ ] Obtain written memorandum from attorney before starting.
[ ] Confirm the memo states privileged communication / work product.
[ ] Get keyword list from attorney.
[ ] Make two bit-stream images, preferably with different tools.
[ ] Compare hash values where possible.
[ ] Search allocated and unallocated space.
[ ] Extract relevant data into organized folders.
[ ] Label written communication as privileged and confidential.
[ ] Use secure communication for sensitive updates.
```

**Critical risk:** Work performed before privilege is established may be discoverable by opposing counsel.

---

### E. Media Leak Investigations

Use when confidential company information appears in news, blogs, forums, or public channels.

**Evidence sources:**

- Corporate e-mail.
- Webmail traces on company computers.
- Proxy logs.
- Internet message boards.
- Search engine results.
- Phone records.
- Workstations of persons with access to leaked data.

**Workflow:**

```text
[ ] Interview management privately.
[ ] Identify who had access to the leaked information.
[ ] Identify the media outlet/source.
[ ] Build keyword list.
[ ] Search e-mail and proxy logs.
[ ] Review phone records.
[ ] Acquire and analyze systems of persons of interest.
[ ] Expand only when evidence supports expansion.
[ ] Report progress routinely to management.
```

**Watch for:** Scope creep. Leak investigations can expand quickly and consume resources.

---

### F. Industrial Espionage Investigations

Industrial espionage cases should be treated as potential criminal matters, especially when sensitive technology or foreign nationals are involved.

**Potential team members:**

| Role | Purpose |
|---|---|
| Computing investigator | Performs disk forensic examinations. |
| Technology specialist | Understands compromised technical data. |
| Network specialist | Reviews logs and deploys monitors. |
| Threat/legal specialist | Understands ITAR, EAR, espionage, and legal exposure. |

**Workflow:**

```text
[ ] Determine whether the case may involve industrial espionage.
[ ] Consult legal and upper management early.
[ ] Define what evidence is needed to substantiate the allegation.
[ ] Generate keywords for disk and network monitoring.
[ ] Collect resources and assign roles.
[ ] Set surveillance or network monitoring if approved.
[ ] Discreetly acquire suspect systems where appropriate.
[ ] Review e-mail, proxy, network, physical access, and phone logs.
[ ] Report regularly to management and counsel.
[ ] Reassess scope before expanding.
```

**Important:** Do not accidentally become an agent of law enforcement before the allegation is substantiated and legal strategy is defined.

---

## 11. Interviews vs. Interrogations

| Term | Meaning |
|---|---|
| Interview | A conversation to collect facts from a witness or suspect. |
| Interrogation | A process intended to obtain a confession or admission. |

The forensic examiner’s role is often to support the trained investigator by providing technical questions, expected answers, and context.

### Preparation Questions

```text
[ ] What facts do I need from this person?
[ ] What technical topics must I explain to the interviewer?
[ ] What answers would be consistent with the forensic findings?
[ ] What indirect questions can expose contradictions?
[ ] Do I need to research a technology before the interview?
```

### Good Interview Traits

- Patience.
- Prepared questions.
- Ability to rephrase.
- Tenacity.
- Calm tone.
- Enough technical confidence to avoid being misled.

---

## 12. Forensic Workstation Essentials

A **forensic workstation** is a system configured for evidence acquisition and analysis.

### Minimum Core Setup

| Component | Purpose |
|---|---|
| Windows forensic workstation | Runs GUI tools such as ProDiscover, FTK, EnCase, or X-Ways. |
| Write-blocker | Prevents writes to evidence media. |
| Acquisition tool | Creates forensic images. |
| Analysis tool | Searches, recovers, and reviews artifacts. |
| Target drive | Stores images and extracted data. |
| Spare PATA/SATA ports | Connects internal drives. |
| USB ports | Handles removable media. |

### Useful Extras

- NIC.
- Additional USB ports.
- FireWire ports.
- SCSI card.
- Disk editor.
- Text editor.
- Graphics viewer.
- Specialized file viewers.

### Tool Reality

No single tool recovers everything. A strong examiner develops skill across multiple tools and operating environments.

---

## 13. Why Normal Windows Booting Is Dangerous

Starting Windows while examining an evidence disk can alter the disk. Windows may write system data, update metadata, interact with the Recycle Bin, or record device information.

Use:

- Hardware write-blockers.
- Software write-protection where appropriate.
- MS-DOS or Linux acquisition methods where suitable.
- Forensic acquisition tools designed for evidence handling.

---

## 14. Bit-Stream Copy and Image Concepts

### Bit-Stream Copy

A **bit-stream copy** is a bit-by-bit duplicate of the original storage medium. It copies sectors, not just visible files.

### Bit-Stream Image

A **bit-stream image** is the file containing that forensic copy.

### Backup vs. Forensic Image

| Backup Copy | Bit-Stream Image |
|---|---|
| Copies known files and folders. | Copies sectors across the medium. |
| Usually misses deleted files. | Can preserve deleted files and fragments. |
| Not ideal for evidence. | Standard forensic method. |
| Focuses on restoration. | Focuses on evidentiary integrity. |

### Visual Model

Chapter 2’s imaging diagram shows the investigative chain as:

```text
Original Evidence Media  →  Forensic Image  →  Target / Working Copy for Analysis
```

This model matters because the original should remain preserved while analysis happens on the copied evidence.

---

## 15. Acquisition Workflow with ProDiscover Basic

### Data Acquistion
Data acquisition is the process of copying data. For computer forensics, it’s the task of
collecting digital evidence from electronic media. 

There are two types of data acquisition: `Static acquisitions and live acquisitions`. 

The future of data acquisitions is shifting toward live acquisitions because of the use of disk
encryption with newer operating systems (OSs). 

In addition to encryption concerns, collecting any data that’s active in a suspect’s computer RAM is becoming more important to digital investigations. 

The processes and data integrity requirements for static and live acquisitions are the same.
The only shortcoming with live acquisitions is not being able to perform repeatable processes,
which are critical for collecting digital evidence. 

With static acquisitions, if you have preserved the original media, making a second static acquisition should produce the same results. The data on the original disk is not altered, no matter how many times an acquisition is done. Making a second live acquisition while a computer is running collects new data because of dynamic changes in the OS.

Your goal when acquiring data for a static acquisition is to preserve the digital evidence.
Many times, you have only one chance to create a reliable copy of disk evidence with a data
acquisition tool. Although these tools are generally dependable, you should still take steps to
make sure you acquire an image that can be verified.

In addition, failures can and do occur, so you should learn how to use several acquisition tools and methods; you work with a few different tools in this chapter. Other data acquisition tools that work in Windows, MS-DOS 6.22, and Linux are described briefly in the last section, but the list of vendors and methods is by no means conclusive. You should always search for newer and better tools to ensure the integrity of your forensics acquisitions.

### USB Acquisition Flow

```text
[ ] Put USB drive in write-protect mode if available.
[ ] Connect USB to forensic workstation.
[ ] Start ProDiscover Basic.
[ ] Choose Action > Capture Image.
[ ] Select the USB drive as source.
[ ] Choose destination path for image.
[ ] Enter technician name.
[ ] Enter image number.
[ ] Start acquisition.
[ ] Review log output for errors.
[ ] Preserve MD5 output/hash file.
[ ] Exit tool and document the action.
```

### Key Acquisition Outputs

| Output | Why It Matters |
|---|---|
| Image file | Working forensic copy. |
| Log file | Shows acquisition activity and errors. |
| MD5 output | Supports later validation and integrity checks. |
### Key Acquisition Terms

`Logical acquisition`: This data acquisition method captures only specific files of interest to the case or specific types of files, such as Outlook PST files. 
`Raw format`: A data acquisition format that creates simple sequential flat files of a suspect drive or data set. 
`Redundant array of independent disks (RAID)`: Two or more disks combined into one large drive in several configurations for special needs. Some RAID systems are designed for redundancy to ensure continuous operations if one disk fails. Another configuration spreads data across several disks to improve access speeds for reads and writes. 
`Sparse acquisition`: Like logical acquisitions, this data acquisition method captures only specific files of interest to the case, but it also collects fragments of unallocated (deleted) data. 
`Static acquisitions`:  A data acquisition method used when a suspect drive is write-protected and can’t be altered. If disk evidence is preserved correctly, static acquisitions are repeatable. 
`Whole disk encryption`: An encryption technique that performs a sector-by-sector encryption of an entire drive. Each sector is encrypted in its entirety, making it unreadable when copied with a static acquisition method.
### Remember:
- Forensics data acquisitions are stored in three different formats: raw, proprietary, and
AFF. Most proprietary formats and AFF store metadata about the acquired data in
the image file.
- The four methods of acquiring data for forensics analysis are disk-to-image file, diskto-
disk copy, logical disk-to-disk or disk-to-data file, or sparse data copy of a folder or file.
- Large disks might require using tape backup devices. With enough tapes, any size
drive or RAID drive can be backed up. Tape backups run more slowly but are a reliable method for forensics acquisitions.
- Lossless compression for forensics acquisitions doesn’t alter the data when it’s restored, unlike lossy compression. Lossless compression can compress up to 50% for most data. If data is already compressed on a drive, lossless compression might not save much more space.
- If there are time restrictions or too much data to acquire from large drives or RAID drives, a logical or sparse acquisition might be necessary. Consult with your lead attorney or supervisor first to let them know that collecting all the data might not be possible.
- You should have a contingency plan to ensure that you have a forensically sound acquisition and make two acquisitions if you have enough data storage. The first acquisition should be compressed, and the second should be uncompressed. If one acquisition becomes corrupt, the other one is available for analysis.
- Write-blocking devices or utilities must be used with GUI acquisition tools in both Windows and Linux. Practice with a test drive rather than suspect drive, and use a hashing tool on the test drive to verify that no data was altered.
- Always validate your acquisition with built-in tools from a forensics acquisition program, a hexadecimal editor with MD5 or SHA-1 hashing functions, or the Linux md5sum or sha1sum commands.
- A Linux Live CD provides many useful tools for computer forensics acquisitions.
- The preferred Linux acquisition tool is dcfldd instead of dd because it was designed for forensics acquisition. Always validate the acquisition with the hashing features of dcfldd and md5sum or sha1sum.
- When using the Linux dd or dcfldd commands, remember that reversing the output
most data. If data is already compressed on a drive, lossless compression might not save much more space. 
- If there are time restrictions or too much data to acquire from large drives or RAID drives, a logical or sparse acquisition might be necessary. Consult with your lead attorney or supervisor first to let them know that collecting all the data might not be possible. 
- You should have a contingency plan to ensure that you have a forensically sound acquisition and make two acquisitions if you have enough data storage. The first acquisition should be compressed, and the second should be uncompressed. If one acquisition becomes corrupt, the other one is available for analysis. 
- Write-blocking devices or utilities must be used with GUI acquisition tools in both Windows and Linux. Practice with a test drive rather than suspect drive, and use a hashing tool on the test drive to verify that no data was altered. 
- Always validate your acquisition with built-in tools from a forensics acquisition program, a hexadecimal editor with MD5 or SHA-1 hashing functions, or the Linux md5sum or sha1sum commands.
- A Linux Live CD provides many useful tools for computer forensics acquisitions. 
- The preferred Linux acquisition tool is dcfldd instead of dd because it was designed for forensics acquisition. Always validate the acquisition with the hashing features of dcfldd and md5sum or sha1sum. 
- When using the Linux dd or dcfldd commands, remember that reversing the output field (of=) and input field (if=) of suspect and target drives could write data to the wrong drive, thus destroying your evidence. If available, you should always use a physical write-blocker device for acquisitions. 
- To acquire RAID disks, you need to determine the type of RAID and then which acquisition tool to use. With a firmware-hardware RAID, acquiring data directly from the RAID server might be necessary. 
- Remote network acquisition tools require installing a remote agent on the suspect’s computer. The remote agent can be detected if suspects install their own security programs, such as a firewall.

## 16. Analysis Workflow with ProDiscover Basic

### Load Evidence Image

```text
[ ] Start ProDiscover Basic.
[ ] Create a new project.
[ ] Add the forensic image file.
[ ] Load image content view.
[ ] Review all files.
[ ] Inspect file contents in the data pane.
```

### Search for Evidence

```text
[ ] Open Search.
[ ] Choose content search.
[ ] Select ASCII or appropriate search mode.
[ ] Enter keyword(s).
[ ] Select the image to search.
[ ] Review search results.
[ ] Open hits and determine relevance.
[ ] Export files of interest.
[ ] Add comments for report inclusion.
```

### Search Strategy

Use targeted keywords. Common terms create too many false positives.

Examples:

```text
George
domain
client
invoice
ISP
registration
business name
known e-mail address
known phone number
known domain name
```

### Boolean Search Logic

| Operator | Use |
|---|---|
| AND | Require both terms. |
| OR | Match either term. |
| NOT | Exclude unwanted hits. |

Boolean search reduces noise and helps avoid excessive false-positive hits.

---

## 17. Deleted File Logic

Deleted files often remain recoverable until overwritten. When a file is deleted, its storage space is normally marked as available, but the original content may still exist physically on disk.

Practical implication:

- Deleted file recovery is possible.
- File fragments can remain in unallocated space.
- Fast action improves recovery chance.
- Analysis must include allocated and unallocated areas when appropriate.

---

## 18. Exporting Evidence Files

When a file of interest is found:

```text
[ ] Open or preview it in the forensic tool.
[ ] Confirm relevance.
[ ] Export a copy to a controlled working folder.
[ ] Document the export.
[ ] Preserve original image unchanged.
[ ] Use an appropriate viewer to examine exported copy.
[ ] Add comments or notes for reporting.
```

For binary files or application-specific files, use the correct software viewer. Examples include spreadsheets, CAD files, graphics viewers, and text editors.

---

## 19. Reporting Workflow

A forensic report should answer the six Ws:

| Question | Report Meaning |
|---|---|
| Who | People, accounts, custodians, suspects, witnesses. |
| What | Evidence acquired, actions taken, artifacts found. |
| When | Collection time, file timestamps, activity windows. |
| Where | Evidence location, system path, storage source. |
| Why | Purpose of investigation and relevance of findings. |
| How | Tools, process, acquisition method, analysis method. |

### Report Requirements

```text
[ ] State what you did.
[ ] State what you found.
[ ] Explain tools used.
[ ] Explain technical processes for nontechnical readers.
[ ] Include timestamps and metadata where relevant.
[ ] Attach or reference tool-generated reports/logs.
[ ] Avoid unsupported conclusions.
[ ] Keep findings repeatable.
```

### Repeatable Findings

A finding is valuable only if the same process can produce the same result again. Repeatability is what makes forensic work defensible.

---

## 20. Investigator Journal

Maintain a written journal throughout the investigation.

Record:

- Dates and times.
- Evidence received.
- People contacted.
- Tools used.
- Image names.
- Hashes and validation results.
- Searches performed.
- Exported files.
- Decisions made.
- Problems encountered.
- Deviations from plan.
- Reporting actions.

Assume your notes may be reviewed in legal proceedings. Write professionally.

---

## 21. Completing the Case

Before closing the case, confirm:

```text
[ ] Have you answered the original allegation?
[ ] Did the evidence support or refute the allegation?
[ ] Is the chain of custody intact?
[ ] Are image files, logs, reports, and exports preserved?
[ ] Are all findings repeatable?
[ ] Are limitations clearly stated?
[ ] Has the requester received the correct level of detail?
[ ] Has original evidence been returned to secure storage?
```

For the chapter’s employee side-business scenario, the final report should show whether the employee used company equipment/time for personal business, which files prove it, and whether timestamps support the allegation.

---

## 22. Case Critique Checklist

After closing the case, review the investigation to improve future work.

```text
[ ] What went well?
[ ] What slowed the investigation?
[ ] Were the results expected?
[ ] Did new facts change the plan?
[ ] Was documentation complete?
[ ] Was the chain of custody strong?
[ ] Were the tools effective?
[ ] Were extra tools or specialists needed?
[ ] Did the requester provide feedback?
[ ] What should be changed in future cases?
```

A good forensic team treats every case as a feedback loop.

---

## Key Terms

| Term | Practical Meaning |
|---|---|
| Approved secure container | Locked, fireproof, access-controlled storage for evidence. |
| Attorney-client privilege | Protected legal communication between attorney and client. |
| Bit-stream copy | Bit-by-bit copy of original media. |
| Bit-stream image | File containing the bit-stream copy. |
| Chain of custody | Documented evidence handling path. |
| Evidence bag | Protective bag for transporting evidence media/components. |
| Evidence custody form | Form documenting who handled evidence and when. |
| Forensic copy | Another term for a bit-stream image. |
| Forensic workstation | System configured for forensic acquisition and analysis. |
| Interrogation | Questioning intended to obtain confession/admission. |
| Interview | Fact-gathering conversation with witness or suspect. |
| Multi-evidence form | Custody form listing multiple items in one case. |
| Password-cracking software | Tool used to recover or guess passwords. |
| Password protected | Access restricted through password control. |
| Repeatable findings | Same forensic process produces same result each time. |
| Single-evidence form | Custody form dedicated to one evidence item. |

---

## 24. Fast Investigation Memory Aid

```text
ASSESS → PLAN → SECURE → IMAGE → VALIDATE → ANALYZE → REPORT → CRITIQUE
```

Expanded:

```text
Assess the case.
Plan the investigation.
Secure the original evidence.
Image the evidence.
Validate the copy.
Analyze the working image.
Report findings clearly.
Critique the case after closure.
```

---

## 25. Practical Mistakes to Avoid

```text
[ ] Starting analysis without knowing the case scope.
[ ] Failing to document who handled the evidence.
[ ] Working directly on original media.
[ ] Booting evidence in a normal OS environment.
[ ] Forgetting environmental risks during transport.
[ ] Using only one tool and assuming it found everything.
[ ] Running broad keyword searches that create excessive false positives.
[ ] Reporting technical artifacts without explaining their meaning.
[ ] Making conclusions stronger than the evidence supports.
[ ] Skipping the post-case critique.
```

---

## 26. One-Page Field Checklist

```text
CASE INTAKE
[ ] Identify requester.
[ ] Identify allegation.
[ ] Identify systems/media.
[ ] Confirm authority to investigate.
[ ] Confirm legal/privacy constraints.

EVIDENCE HANDLING
[ ] Photograph or document condition.
[ ] Complete custody form.
[ ] Use evidence bags / antistatic protection.
[ ] Transport securely.
[ ] Store original in approved container.

ACQUISITION
[ ] Prepare forensic workstation.
[ ] Use write-blocker.
[ ] Create bit-stream image.
[ ] Capture tool logs.
[ ] Capture hash/validation data.
[ ] Return original to secure storage.

ANALYSIS
[ ] Load image into forensic tool.
[ ] Review file system.
[ ] Search targeted keywords.
[ ] Review deleted files / unallocated data where needed.
[ ] Export relevant evidence.
[ ] Add comments and notes.
[ ] Preserve repeatability.

REPORTING
[ ] Explain who, what, when, where, why, and how.
[ ] Include evidence source and tool process.
[ ] Include relevant timestamps.
[ ] Attach logs/reports where useful.
[ ] State limitations.
[ ] Avoid speculation.

CLOSURE
[ ] Deliver report.
[ ] Preserve evidence and work product.
[ ] Critique case.
[ ] Update procedures if needed.
```

---

