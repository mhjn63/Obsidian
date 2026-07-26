> HTML Page: [[HTML Pages/THM/CTI-for-Alert-Triage.html|Open HTML Page]]

# CTI for Alert Triage: Turning Indicators Into Investigative Context

Most analysts first encounter threat intelligence through lookup tools. An IP address goes into VirusTotal. A domain is checked in AbuseIPDB. A file hash is submitted to a reputation service. The result comes back red, green, or somewhere in between.

That is useful, but it is not yet cyber threat intelligence.

A lookup answers a narrow question: **has someone reported this indicator as malicious?** CTI answers a broader one: **what does this indicator mean, which campaign is it connected to, who may be behind it, what else should we hunt for, and has our organization seen the same activity before?**

The difference is context. A suspicious domain stops being an isolated value and becomes part of a campaign, linked to infrastructure, malware, techniques, historical sightings, and response guidance.

## Threat Intelligence Is More Than Reputation

Lookup services and CTI platforms often begin with the same raw material: IP addresses, domains, URLs, hashes, and other indicators. They differ in what they return.

| Question | Indicator Lookup | CTI |
|---|---|---|
| Is this an attack indicator? | Yes: `Trojan:JS/Agent` | Yes, with evidence explaining why |
| Which actor or campaign is it tied to? | Usually unavailable | Mini Shai Hulud campaign or TeamPCP activity |
| Which related indicators exist? | Usually unavailable | Domains, IPs, hashes, tools, and MITRE techniques |
| Has the organization seen this before? | Usually unavailable | Internal sightings and previous related activity |

A lookup produces a verdict. CTI produces a relationship.

That relationship is where the investigative value begins.

## L1 Uses Intelligence to Decide; L2 Uses It to Explain

Threat intelligence supports every SOC tier, but the depth changes with responsibility.

Level 1 analysts usually work at the indicator level. They check reputation, compare the alert with known malicious behavior, and decide whether the case should be closed or escalated.

Level 2 analysts use intelligence to contextualize the threat. They need to determine how the activity fits into a campaign, which adversary techniques are present, which systems may also be exposed, what should be hunted, and which response actions match the threat.

L1 asks whether the activity is malicious.

L2 asks what the malicious activity means.

## Why Advanced CTI Matters

For a small SOC, the most practical value of CTI is investigative acceleration. Once the analyst understands an adversary’s methods, infrastructure, motivation, and likely objectives, the investigation becomes easier to scope. The analyst can spend less time rediscovering known behavior and more time proving how much of it occurred internally.

In larger organizations, government teams, and law enforcement environments, CTI serves a broader strategic role.

### Attribution

CTI can connect activity to a threat actor, campaign, criminal service, or infrastructure provider. Attribution should never rest on one indicator, but it becomes important when legal action, sanctions, public reporting, or coordinated response are involved.

### Proactive Defence

Tracking adversary tooling and TTPs allows defenders to prepare before the organization is attacked. Intelligence can drive new detections, hunt queries, blocklists, tabletop exercises, and hardening priorities.

### Information Sharing

Indicators and campaign context can be shared with trusted communities so other organizations can defend against the same threat.

### Incident Response

Known adversary behavior can accelerate containment and remediation. If an incident resembles a documented campaign, responders can immediately search for the persistence, credential theft, lateral movement, and exfiltration methods associated with it.

CTI becomes high priority whenever the organization cares who is attacking, why they are attacking, how the threat landscape is changing, or how indicators should be exchanged.

## MISP and OpenCTI

Two open-source platforms appear frequently in operational CTI programs.

**MISP** focuses on sharing, organizing, and tracking indicators of compromise. It is effective for hashes, IP addresses, domains, URLs, malware samples, confidence tags, distribution controls, and indicator decay.

**OpenCTI** goes further into relationships. It connects indicators with threat actors, malware families, tools, campaigns, intrusion sets, vulnerabilities, ATT&CK techniques, reports, victims, and sectors.

MISP is strongest when the unit of work is the indicator or event. OpenCTI is strongest when the analyst needs the wider knowledge graph around the threat.

# Scenario: A Suspicious Node.js Credential-Access Alert

Consider a morning SOC queue. A Level 1 analyst receives a medium-severity alert and escalates it for deeper review.

The triggering process is signed `node.exe`, but it has accessed SSH keys, AWS credentials, environment files, and npm configuration data. The alert resembles credential theft, yet it does not reveal which script caused the behavior or how the code reached the workstation.

```text
[SIEM Alert]
Rule Name      : Credential access attempt by untrusted process
Timestamp      : 2026-06-01 08:14:32 UTC

[Triggering Event]
event_type     : file_access
process_name   : node.exe
process_id     : 4821
process_sig    : Signed (Node.js)
user           : CORP\j.okafor
host           : WKSTN-047
accessed_files : ~/.ssh/id_rsa, ~/.aws/credentials, .env, .npmrc
```

The alert justifies an investigation, but it does not yet explain the incident.

## Step 1: Build the Local Story First

Threat intelligence should not replace log analysis. Begin with the organization’s own evidence.

The analyst should determine what launched `node.exe`, which script it executed, whether the process was expected, which files it accessed, and which network destinations it contacted.

The process tree provides the first coherent sequence:

```text
explorer.exe          [08:06:55]
└── code.exe          [08:13:34]
    ├── npm install   [08:14:10]
    └── node index.js [08:14:32]
```

The activity began after the user opened Visual Studio Code and ran `npm install`. That operation launched `node index.js`, which then accessed multiple developer secrets.

The sequence is consistent with an npm supply-chain compromise. It remains a hypothesis, but it is now grounded in internal telemetry.

## Basic Reputation and Malware Analysis

The next step is to examine `index.js`. Static or dynamic analysis should establish what the script does and extract the indicators required for further investigation.

```text
[Static Analysis]
Analyzed File    : index.js
VT Detections    : 7/61 (InfoStealer:JS)
Notable String   : python-requests/2.31.0 (User-Agent)
Notable String   : https://api.github.com (URL Address)
Notable String   : AWS_SECRET_ACCESS_KEY (Sensitive)
SHA256           : d8d170af3de17bb9b217c52aaaffdf9395f35ef015a57ef676e406c121e5e223
```

```text
[Dynamic Analysis]
High-risk        : Cloud and Kubernetes secrets enumeration
High-risk        : Enumeration of common secrets' location
Informational    : Suspicious GitHub API activity
```

The evidence now supports a malicious verdict. The script searches common credential locations, targets cloud and Kubernetes secrets, and communicates with the GitHub API.

But local telemetry still leaves important questions unanswered: which package introduced the script, whether this belongs to a known campaign, which additional packages are compromised, and what remediation is required beyond deleting one file.

This is where CTI extends the investigation.

## Step 2: Move From Indicators to Campaign Context

A practical CTI workflow compares the local evidence with internal and external intelligence.

### Search Internal MISP Data

Check whether the available indicators already exist in the organization’s MISP instance. Relevant values include the SHA-256 hash, domains, IP addresses, package names, User-Agent strings, and ATT&CK techniques.

Internal matches are especially valuable because they may reveal previous incidents, affected customers, approved response actions, or intelligence written by analysts who understand the environment.

### Search OpenCTI by Behavior

Search for reports that match the observed behavior rather than relying only on exact indicators. Useful concepts include Node.js information stealer, npm supply-chain compromise, GitHub API abuse, developer credential theft, and cloud-secret enumeration.

Behavioral matches remain useful when hashes, domains, or package versions have changed.

### Search Recent Public Reporting

When private feeds are unavailable, carefully constructed searches can expose relevant public intelligence. Quoted strings are often effective because malware authors reuse libraries, User-Agent values, API paths, and code fragments.

```text
"node.js stealer python-requests/2.31.0"
```

The search may surface recent vendor reports describing npm malware persistence or a Node-gyp supply-chain compromise. From there, the analyst may discover that the activity matches the Miasma campaign and identify the compromised npm packages involved.

The report does not replace internal evidence. It gives the evidence a name, a wider scope, and a more complete response path.

## What a Strong Report Adds

A useful CTI report may provide:

- Compromised npm package names and versions.
- Related file hashes and repositories.
- Installation scripts and persistence behavior.
- Token-exfiltration methods.
- Additional credential locations.
- Campaign timelines.
- Detection queries.
- Recommended remediation.

The analyst can then hunt for every affected package, identify other developers who installed it, revoke exposed credentials, rotate GitHub, npm, cloud, SSH, and Kubernetes secrets, and search for related persistence or infrastructure.

# CTI Must Be Correlated, Not Followed Blindly

Threat reports are not playbooks that should be copied directly into an incident ticket.

A public report may resemble the activity in the SIEM and still be incomplete, stale, misleading, or unrelated.

A report may list only half the indicators. The initial access may match while later stages differ. Similar behavior may belong to a different campaign. Reposted intelligence may lose caveats, confidence levels, and technical context.

The operating rule is simple:

> Correlate every CTI claim against evidence in the organization’s own logs.

The SIEM establishes what happened locally. CTI helps explain, enrich, and expand it.

# Not Every Reported Indicator Is Malicious

One of the most common CTI mistakes is treating every value mentioned in a report as something that should be blocked.

Adversaries frequently use legitimate infrastructure. A report may mention a service because malware contacted it, not because the service itself is malicious.

## Example 1: A Legitimate IP-Lookup Service

Suppose a report states that the adversary used:

```text
https://api.myip.com/
```

The malware queried the service to discover the victim’s public IP address.

An inexperienced analyst searches historical logs, finds that more than twenty laptops contacted the same service, and declares them all infected.

That conclusion is unsupported. Both benign and malicious software can contact `api.myip.com`. Its presence may contribute to a behavioral pattern, but it does not prove compromise by itself.

A stronger conclusion would combine the service call with additional evidence such as an untrusted parent process, credential-file access, suspicious package installation, token enumeration, or follow-on exfiltration.

## Example 2: Blocking Shared Cloudflare Infrastructure

A report may state that an adversary used Cloudflare CDN infrastructure for command-and-control communication.

The analyst checks the IP in VirusTotal, sees several malicious detections, and adds it to the enterprise blocklist.

This can create widespread disruption. Cloudflare addresses are shared across large numbers of unrelated websites and services. A reputation score may reflect one malicious hostname behind that IP, not every customer using it.

The analyst must consider whether the IP is shared, which hostname was contacted, whether SNI or HTTP Host evidence identifies the malicious service, and whether a domain-level control is safer than an IP-level block.

Experienced analysis separates malicious use of infrastructure from malicious ownership of infrastructure.

# Indicators Have Expiration Dates

Even accurate indicators lose value over time.

A cloud-hosting IP used as command-and-control infrastructure today may be reassigned to an unrelated customer next week. A compromised domain may be cleaned, transferred, or repurposed. A temporary URL may disappear within hours.

Threat intelligence platforms address this through decay models, validity periods, or time-to-live values.

| Indicator Type | Active Blocking | Longer-Term Use |
|---|---:|---|
| File hashes | Permanent | Permanent alerting and retrospective hunting |
| Domain names | 1 year | Continue alerting after blocking expires |
| IP addresses | 14 days | Alert for another 30–90 days |

These values are policy examples, not universal rules. The correct lifetime depends on infrastructure type, ownership, confidence, campaign behavior, whether the resource is shared, and the cost of a false positive.

A file hash is stable because the file’s content does not change. An IP address is volatile because ownership can change while the address stays the same.

# How to Weigh Conflicting Intelligence

Threat intelligence is often uncertain. Different sources may disagree about attribution, confidence, infrastructure, timing, or whether an indicator remains malicious.

Three principles provide a reliable hierarchy.

## 1. Confirmed Internal Activity Outweighs External Claims

What the organization directly observes should take priority.

When SIEM, endpoint, identity, network, or cloud logs confirm malicious behavior, CTI should supplement that evidence. It may add indicators, explain the campaign, or suggest attribution. It should not override verified local activity.

## 2. Internal Intelligence Usually Outweighs External Intelligence

A MISP event authored by the organization’s own CTI team generally has more operational value than an anonymous public pulse. Internal intelligence can account for local assets, previous incidents, business activity, existing controls, and organization-specific exposure.

## 3. Linked Indicators Outweigh Isolated Indicators

An indicator connected to a campaign, malware family, kill chain, and supporting evidence carries more value than an isolated “bad” domain.

A contextualized indicator can show when it was active, which actor used it, which malware contacted it, what stage of the intrusion it supported, and which related indicators exist.

An isolated reputation hit may provide none of that.

A linked indicator is intelligence. An uncontextualized indicator is closer to a lookup result.

# Applying Confidence in CTI Production

CTI teams must decide how much uncertain information to publish.

A broad report maximizes coverage but increases false-positive risk. A narrow report contains only what can be strongly verified but may miss emerging infrastructure. A confidence-tagged report preserves a wider set of findings while expressing uncertainty at the attribute level.

The confidence-tagged model is usually the most useful:

- High-confidence indicators may justify blocking.
- Medium-confidence indicators may justify alerting and enrichment.
- Low-confidence indicators may be restricted to hunting.
- Expired indicators may remain useful for retrospective investigations.

The goal is not to remove uncertainty. It is to express it clearly enough that downstream teams can act safely.

# A Practical CTI-Assisted Triage Workflow

1. Build the incident from internal telemetry.
2. Extract indicators and behaviors.
3. Search internal MISP, OpenCTI, previous cases, and historical alerts.
4. Search trusted external intelligence and original reporting.
5. Correlate every report statement against local evidence.
6. Hunt for the wider campaign across systems, users, packages, and infrastructure.
7. Apply controls according to indicator type, confidence, ownership, and validity.
8. Record the source, confidence, last-seen date, intended use, and expiry.
9. Close the case with campaign context, response actions, and remaining intelligence gaps.

# Final Perspective

Threat intelligence does not improve alert triage simply by giving analysts more indicators. It improves triage by giving indicators meaning.

The hash from `index.js` matters. The User-Agent string matters. The GitHub API activity matters. The compromised package matters. Their real value appears when they are connected into a coherent model of the incident.

The strongest analysts do not choose between telemetry and intelligence. They use telemetry to establish what happened, then use intelligence to understand how far the evidence reaches.

They do not treat public reports as truth. They test them.

They do not block every value coloured red by a reputation service. They examine ownership, context, confidence, and validity.

They do not preserve indicators forever because they were once malicious. They manage them according to decay.

CTI should never replace investigation. It should make investigation faster, broader, and harder to misinterpret.
