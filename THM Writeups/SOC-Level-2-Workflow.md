[🏠 Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Practitioner-grade cybersecurity notes, cert prep guides, and courses. All premium notes available at **[buymeacoffee.com/notescatalog/extras](https://buymeacoffee.com/notescatalog/extras)** or [shop.motasem-notes.net](https://shop.motasem-notes.net)

# SOC Level 2 Workflow: From Escalated Alert to Complete Incident Resolution

A Level 2 SOC analyst does not begin with a blank screen. The work starts with a trigger.

For a Level 1 analyst, that trigger is usually a new security alert. For Level 2, it is more often a reported incident: an alert escalated by L1, an urgent request from management, or a high-priority case raised by an MSSP customer.

That difference matters. Level 1 is designed to determine whether an alert deserves deeper attention. Level 2 is responsible for turning that escalation into a defensible conclusion. The analyst must investigate what happened, decide how serious it is, coordinate the response, and capture what the organization should learn from the incident.

The Level 2 workflow can be reduced to three responsibilities:

1. Investigate the incident.
2. Respond to the threat.
3. Preserve the lessons, evidence, and operational improvements that follow.

The work still happens under an SLA. Speed still matters. But at Level 2, speed is no longer the only measure of performance. The verdict must be correct, the response must match the risk, and the case must be closed without leaving hidden persistence, missed indicators, or unresolved exposure behind.

---

## Level 1 and Level 2 Triage Serve Different Purposes

The distinction between L1 and L2 is not simply that one role is junior and the other is senior. Each tier exists to answer a different operational question.

Level 1 asks:

> Does this alert require escalation?

Level 2 asks:

> What happened, how far did it go, and what must be done now?

An L1 analyst may review an alert, validate its basic context, and escalate it within ten minutes. The L2 analyst may then spend another two hours reconstructing the incident, correlating logs, contacting users or IT teams, containing affected assets, and confirming that the attacker no longer has access.

| Triage Aspect | Level 1 | Level 2 |
|---|---|---|
| **Trigger** | New security alert | Escalated alert or reported incident |
| **SLA** | Applicable through metrics such as MTTA and MTTR | Equally applicable |
| **Primary focus** | Rapid alert review and initial classification | Deeper log analysis, incident validation, and response |
| **Main platforms** | Primarily the ticketing system and SIEM | A wider collection of SOC, identity, endpoint, network, and IT tools |
| **Typical response** | Quarantine a malicious download or approve a SOAR playbook | Manually remove malware, disable accounts, isolate hosts, coordinate remediation, and validate recovery |

The most important shift is analytical depth. L1 focuses on identifying enough evidence to justify escalation. L2 must produce a reliable account of the incident from beginning to end.

---

## The Level 2 Role Changes With the Organization

There is no universal Level 2 job description.

Large enterprises and mature MSSPs may have dedicated Level 3 analysts, digital forensics and incident response teams, threat intelligence units, malware analysts, detection engineers, and security platform teams. In that environment, Level 2 may focus heavily on investigation, scoping, and initial containment before handing specialized work to another function.

Smaller organizations operate differently. A Level 2 analyst may become the investigator, incident responder, threat hunter, malware analyst, identity specialist, and technical liaison to IT—all within the same case.

That broader exposure can accelerate learning, but it also carries more responsibility. Analysts evaluating a Level 2 position should examine the actual operating model rather than relying on the title. The same title can describe a focused escalation role in one organization and a full-spectrum incident response position in another.

---

## Log Analysis Begins With Understanding the Detection

A deeper investigation should not begin with random searches across every available log source.

Start with the rule.

Before opening additional queries, understand what technique the detection is designed to identify, which event fields caused it to fire, and what assumptions are built into the logic. Without that context, the analyst risks collecting large volumes of unrelated telemetry while missing the exact behavior the rule was intended to surface.

Three operating principles should guide Level 2 triage:

- Start only after understanding the detection rule and the technique it targets.
- Avoid working multiple active cases at the same time unless one is blocked or awaiting action from another party.
- Stay within the SLA whenever possible. When the investigation is taking too long and the risk is material, contain the threat first and continue the analysis afterward.

The purpose of log analysis is not to accumulate events. It is to reconstruct behavior.

---

## First, Establish What Happened

Before building a detailed timeline, create a high-level version of the incident.

Consider an alert titled:

`Rundll32 Spawned by PowerShell`

The first SIEM searches should answer four immediate questions:

1. Why did the rule trigger?
2. What did `rundll32.exe` do after it launched?
3. What caused PowerShell to execute?
4. Does the sequence match a known attack pattern?

These questions turn disconnected telemetry into a working hypothesis.

The analyst may discover that a user launched an unfamiliar application, the application spawned PowerShell, PowerShell executed `rundll32.exe`, and `rundll32.exe` loaded a newly created DLL before connecting to an external server.

That sequence is not yet a complete incident timeline, but it provides the story required to guide the next searches. Without this step, the analyst is likely to search too broadly. With it, every query can test or expand a specific theory.

---

## Then Build the Full Timeline

A high-level story explains the incident. A detailed timeline exposes its indicators.

The distinction is critical. A case can appear understandable while still hiding persistence, secondary payloads, credential access, additional command execution, or lateral movement.

Using the same scenario, the analyst should timeline the process, file, and network activity associated with the suspicious application—such as `NetDbg`—then repeat the process for `rundll32.exe` and every relevant child process.

A useful Splunk timeline might:

- Filter for key Sysmon events generated by the `NetDbg` application.
- Identify reusable indicators such as `lxdhk.dll`, `netupdate-pro.shop`, and `159.89.143.156`.
- Show that a suspicious outbound connection was followed by a DLL drop and execution.
- Establish whether `NetDbg` is compromised software or malware masquerading as a legitimate application.

The timeline should answer more than “what triggered the alert?” It should reveal:

- The earliest known point of compromise.
- The process ancestry.
- Every file created, modified, or executed.
- Every domain and IP address contacted.
- Any persistence mechanism.
- Any affected account.
- Any evidence of credential access.
- Any child process or follow-on payload.
- The last confirmed malicious action.

The goal is not to produce the largest possible timeline. It is to produce the smallest timeline that still explains the entire incident without leaving meaningful gaps.

---

## Investigation Becomes a Threat-Hunting Loop

Many Level 2 cases can be resolved with a single well-built timeline. More complex incidents cannot.

The `NetDbg` scenario may still leave important questions unanswered:

- Is the application approved by the organization?
- Who installed it?
- How did the user or host first enter the environment?
- Is the software itself malicious, or was a legitimate installation compromised?
- Did the same indicators appear on other systems?
- Did the attacker gain persistence elsewhere?

At this point, log analysis becomes a threat-hunting loop:

1. Form a working story from the alert and available context.
2. Build a timeline that tests the story and fills its gaps.
3. Decide whether the sequence is clear from initial access to final activity.
4. When the story remains incomplete, revise the hypothesis and repeat the process.

This loop prevents premature closure. It forces the analyst to distinguish between a plausible explanation and a verified one.

A weak investigation stops when the first malicious event is found. A strong investigation continues until the analyst understands the scope, the sequence, and the remaining uncertainty.

---

## Response Turns Analysis Into Security

Investigation has no operational value unless it leads to the right action.

This is where Level 1 and Level 2 work converge. L1 identified the event. L2 established what it means. Response converts that understanding into containment, remediation, and risk reduction.

Ideally, response begins after the investigation is complete and the analyst has reached a confident verdict. In practice, that sequence is not always possible. Some incidents require action before every detail is known.

The analyst must balance two risks:

- Acting too early and disrupting legitimate business activity.
- Acting too late and allowing an attacker to continue operating.

That judgment is one of the defining responsibilities of Level 2 work.

---

## Verify Ambiguous Activity With the People Closest to It

Not every suspicious event can be resolved through telemetry alone.

Imagine an anomalous-login alert showing that a user accessed a Microsoft account through Proton VPN. The actions after authentication appear legitimate, but the login location and network path remain unusual.

Only the user can confirm whether the activity was expected.

The analyst should contact the user directly. When the risk is high and the user does not respond quickly, temporarily disabling the account may be justified until ownership is confirmed.

Other verification paths include:

- Asking IT to explain the purpose of an unfamiliar application such as `NetDbg`.
- Confirming with IT support whether a new administrator account was intentionally created.
- Asking DevOps whether a suspicious API call came from an approved automation script.
- Contacting the red team to determine whether the activity belongs to an authorized penetration test.

Verification is not a substitute for technical analysis. It is another source of evidence.

Human confirmation is especially valuable when the action could be legitimate but the logs cannot establish intent.

---

## Responding to True Positives

Once the analyst has verified the activity and assigned a final true-positive or false-positive verdict, the response should match the scale of the incident.

A localized infection, such as adware on a single employee laptop, may require only an EDR quarantine and confirmation that no persistence remains.

A broader intrusion may require the analyst and supporting teams to:

- Access affected systems through RDP or SSH.
- Isolate hosts or entire network segments.
- Disable users in Microsoft Entra ID.
- Revoke sessions and application tokens.
- Remove malicious files and persistence mechanisms.
- Reset compromised credentials.
- Rotate exposed secrets.
- Patch exploited vulnerabilities.
- Block malicious domains, IP addresses, and file hashes.
- Search the environment for the same indicators.
- Monitor recovered systems for reinfection.

Response work is often the most demanding part of the case because it connects the SOC to identity, endpoint, network, cloud, DevOps, and IT operations. It also forces the analyst to move from observation to action.

---

## Major Incidents Require Containment Before Certainty

During a major incident, waiting for a complete investigation can create more damage than acting on partial but credible evidence.

When logs show active data exfiltration, the immediate priority is to stop the transfer. The analyst may not yet know the initial access vector, the full attacker path, or every affected system. Those questions still matter, but they come after containment.

Major-incident response is usually divided into three phases:

### Containment

Stop the threat from spreading or continuing.

Examples include isolating hosts, disabling users, revoking active sessions, blocking network destinations, or segmenting affected systems.

### Eradication

Remove the attacker’s foothold and eliminate the cause.

This may include deleting malware, removing persistence, rotating stolen passwords, revoking privileges, replacing exposed keys, and closing the exploited access path.

### Recovery

Return systems to normal operation without reintroducing the threat.

Recovery may involve lifting isolation, restoring services, applying patches, validating system integrity, and monitoring for evidence of reinfection.

The phases may overlap. A host may be isolated while other systems are still under investigation. Credentials may be rotated before all affected accounts are known. Recovery may begin for one business unit while eradication continues elsewhere.

Playbooks provide structure, but they cannot replace judgment. Level 2 analysts must make decisions under pressure when the incident does not follow the expected sequence.

---

## False Positives Still Create Work

A false positive is not always a case with no security value.

The alert may be incorrect while still exposing a weak detection rule, a vulnerable system, or a gap in analyst performance.

| Example | Level 2 Response | Common Name |
|---|---|---|
| The rule triggered on unrelated system activity because of flawed query logic | Plan a detection fix or report the issue to the engineering team | False-positive tuning |
| The alert was false, but it exposed an Internet-facing RDP service | Work with IT to place the service behind a VPN or another controlled access layer | Security hardening |
| The alert was simple, but the Level 1 analysis was incomplete or incorrect | Explain the analytical gaps or arrange focused mentoring | Team improvement |

This is one of the clearest signs of a mature SOC: false positives are not discarded and forgotten. They are converted into better detections, safer architecture, and stronger analyst performance.

---

## Resolving the Case

A case is not resolved simply because the analyst understands what happened.

Resolution requires two conditions:

1. The final verdict is documented.
2. The threat has been fully remediated or the remaining risk has been formally accepted.

Ticketing workflows differ between organizations, but three practices should remain constant.

### Preserve Evidence of Every Action

Record the investigation, verdict, containment steps, remediation work, communications, and validation results in a centralized system such as the incident ticket.

The case should show what was known, what was done, when it was done, and who approved or performed each action.

### Inform the SOC

Share incidents that are new, unusual, or likely to recur.

A short internal briefing can prevent another analyst from rebuilding the same knowledge during the next alert. New indicators, detection gaps, and response lessons should become shared operational knowledge.

### Keep Stakeholders Updated

Management, IT teams, business owners, and MSSP customers need concise status updates.

The communication should explain:

- What happened.
- What systems or users were affected.
- What actions were taken.
- Whether the threat is contained.
- What remains unresolved.
- What follow-up work is required.

A technically correct response can still fail operationally when stakeholders do not know the current risk or the state of remediation.

---

## The Real Standard for Level 2 Work

Level 2 analysis is not defined by the number of queries executed, the volume of logs reviewed, or the amount of time spent in the SIEM.

The standard is whether the analyst can produce a defensible answer to five questions:

1. What happened?
2. How did it happen?
3. What was affected?
4. What stopped it?
5. What must change to prevent it from happening again?

A complete Level 2 workflow begins with an escalation and ends with more than a closed ticket. It ends with the threat contained, the evidence preserved, the stakeholders informed, and the SOC better prepared for the next incident.
