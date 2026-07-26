
## Elastic Defend Deep Dive: Turning the Elastic Stack into a Full EDR and Hunting Real Activity

The moment most teams start comparing endpoint security tools, the conversation usually shifts from brand names to capabilities; visibility, prevention, detection logic, and how fast analysts can reconstruct what actually happened on a compromised system.

That comparison is what sparked this exploration: Elastic Security, particularly the **Elastic Defend** component, positioned not just as a log pipeline, but as a complete endpoint detection and response (EDR) platform inside a SOC environment.

What follows is a deep technical walk-through of how Elastic Defend operates, how to deploy it properly, and how analysts use it to reconstruct attack chains from raw telemetry into actionable intelligence.

# From SIEM to Full EDR: What Elastic Defend Actually Adds

Many teams already use the Elastic Stack; **Elasticsearch**, **Kibana**, and associated pipelines for log ingestion and search-driven investigations.

That baseline gives you visibility.

Elastic Defend turns that visibility into control.

Instead of relying only on centralized logs, Elastic Defend extends monitoring directly onto endpoints using the **Elastic Agent**, allowing the platform to:

- Observe processes
- Track file system changes
- Monitor network connections
- Detect suspicious behavior
- Prevent active threats

In practice, this transforms Elastic from a passive analysis tool into an active defensive layer capable of stopping threats not just logging them.

---

# Where Elastic Defend Runs: Endpoint Coverage Scope

Elastic Defend is designed to operate across heterogeneous environments.

Supported endpoints include:

- Windows desktops and servers
- macOS systems
- Linux systems
- Kubernetes nodes
- Cloud workloads
- Traditional enterprise endpoints

The operational dependency is straightforward:

**Elastic Agent must be installed on every endpoint you want to monitor.**

Once deployed, the agent becomes the telemetry bridge between the endpoint and the Elastic infrastructure.

Without that agent, there is no endpoint visibility.

---

# The Two Core Operating Modes: Detection vs Prevention

Elastic Defend operates in two fundamental modes.

Understanding the difference is critical because many deployments fail simply by enabling only monitoring without prevention.

## Mode 1 ; Telemetry Collection (Detection Mode)

This mode behaves similarly to an IDS.

It collects activity data such as:

- Process creation
- File modification
- Network activity
- User privilege changes
- System-level actions

Once collected, this data is analyzed against detection logic to produce alerts when suspicious patterns are observed.

This mode answers:

**What happened?**

But not:

**Did we stop it?**

---

## Mode 2 ; Endpoint Protection (Prevention Mode)

Prevention mode builds on detection.

Instead of just alerting, Elastic Defend actively blocks malicious activity.

Typical prevention actions include:

- Blocking malware execution
- Preventing ransomware spread
- Stopping credential theft attempts
- Terminating malicious processes

Running both modes together is what elevates Elastic Defend into a full EDR platform.

Without prevention enabled, the system behaves more like visibility tooling than security enforcement.

---

# Policy Configuration: Where Defensive Behavior Is Defined

After installing Elastic Defend, the next major step is policy configuration.

Policies determine:

- What to monitor
- What to block
- Which endpoints receive which protections

Typical policy options include:

- Malware protection
- File modification monitoring
- Blocklists
- Memory threat detection
- Behavioral analysis
- Ransomware protection

Each operating system can have its own policy configuration.

That separation is not cosmetic—it prevents over-monitoring low-risk systems while enforcing stricter controls on critical assets.

For example:

- Linux servers may prioritize process and privilege escalation monitoring
- Windows workstations may enable registry change tracking
- macOS systems may focus on network and file activity

This granular control is what allows Elastic Defend to scale across enterprise environments.

---

# Endpoint Visibility: What Data Elastic Actually Collects

Once policies are active and agents are deployed, Elastic begins collecting telemetry across multiple behavioral domains.

Typical Windows telemetry includes:

- File modifications
- API calls
- DLL loading events
- DNS activity
- Network connections
- Process creation
- Registry changes
- Security context changes

macOS telemetry includes:

- File activity
- Network connections
- Process execution

Linux telemetry focuses heavily on:

- Process chains
- Privilege transitions
- Command execution
- System file access

The key principle:

**Every action becomes searchable telemetry.**

And searchable telemetry becomes reconstructable attack timelines.

---

# Creating a Data View: Turning Raw Logs Into Investigatable Data

Raw telemetry is useless until analysts can query it.

That’s where **data views** come into play.

Inside Kibana, analysts create a data view tied to Elastic Defend indices.

Typical index pattern:

logs-endpoint.events.*

The wildcard is critical.

It allows ingestion of:

- File activity logs
- Network telemetry
- Process execution events

Once created, the data view becomes selectable inside **Discover**, where analysts can begin active investigation workflows.

---

# The Investigation Scenario: Simulating Suspicious Activity

To demonstrate real detection workflow, a scripted enumeration scenario was executed on a monitored system.

The script triggered:

- System reconnaissance commands
- Privilege enumeration
- File listing operations
- User context evaluation

Examples of commands executed:

- `id`
- `uname`
- `sudo`
- `ls`
- Access to `/etc/passwd`

The goal was not to memorize commands.

The goal was to simulate attacker reconnaissance behavior.

From the SOC perspective:

**All that was known was that a suspicious script executed.**

Everything else had to be reconstructed.

---

# Reconstructing the Attack Chain Using Process Telemetry

Investigation begins with a single clue:

**Script name**

Example:

`enumeration.sh`

From there, analysts filter:

process.args contains `sh`

This isolates shell execution events.

Once identified, additional fields are added:

- Event action
- Process arguments
- Process executable
- Parent process
- Username
- User ID

The output reveals a structured process lifecycle:

- fork
- exec
- UID change
- termination

This lifecycle reconstruction is essential because:

It reveals whether privilege escalation occurred.

And in this case, it did.

---

# Privilege Escalation Detection: UID Transition

One of the most telling indicators in the investigation was:

UID changed from 1000 to 0

That shift represents:

User → Root escalation

In Linux systems:

UID 0 = root

This is a high-value indicator because privilege escalation dramatically increases attacker capabilities.

Elastic captures this transition automatically.

No manual instrumentation required.

---

# Using Entity IDs to Track Process Relationships

Every process inside Elastic telemetry receives a unique identifier:

process.entity_id

This identifier enables:

- Tracking parent-child relationships
- Rebuilding execution chains
- Mapping lateral process spawning

By filtering on a parent entity ID, analysts can reconstruct:

- Child processes spawned
- Commands executed
- Timeline order

This produces the attack chain view—one of the most critical elements in threat hunting.

---

# Detection Rules: The Core of Automated Security

Elastic Defend includes a library of pre-built detection rules.

These rules evaluate telemetry against known malicious behavior patterns.

Examples of rule triggers include:

- Suspicious file access
- Credential dumping behavior
- Abnormal privilege escalation
- Malware execution patterns

If telemetry matches a rule condition:

**An alert is generated automatically.**

No manual query required.

This reduces analyst workload while increasing detection consistency.

---

# Triggering a Real Alert: Accessing the Shadow File

To demonstrate rule-based detection, a high-risk action was executed:

cat /etc/shadow

This file contains:

- Password hashes
- User credential metadata

Accessing it is a classic post-exploitation behavior.

Attackers typically read this file to:

- Extract hashes
- Perform offline cracking
- Reuse credentials

Elastic Defend immediately detected the activity and generated an alert.

---

# Alert Analysis: Understanding What Triggered Detection

Once the alert appeared, analysts reviewed:

- Rule name
- Risk score
- Trigger reason
- Associated process tree

Example alert type:

Potential shadow file read via command line utility

Risk score:

47

Severity:

Medium

This structured output allows analysts to:

- Assess threat level
- Identify root cause
- Determine response actions

---

# Visualizing the Attack: The Process Analyzer Graph

Reading logs is slow.

Visualizing relationships is fast.

Elastic provides a process analyzer graph that reconstructs the full execution chain.

Observed chain:

sudo → su → bash → cat

Execution order:

1. sudo invoked
2. su escalated privileges
3. bash executed
4. cat accessed shadow file

Each step included timestamp precision measured in milliseconds.

This timeline-based reconstruction allows analysts to:

- Identify attack origin
- Understand sequence logic
- Validate privilege escalation paths

It converts raw telemetry into human-readable attack flow.

---

# Why Behavioral Visibility Matters More Than Signatures

Traditional antivirus relies heavily on signatures.

Modern attacks rarely rely on static signatures.

Instead, attackers execute legitimate system tools in malicious ways.

Examples:

- `sudo`
- `cat`
- `bash`
- `ls`

These tools are not malware.

They are normal utilities.

Detection relies on behavior patterns—not filenames.

Elastic Defend’s detection logic focuses on:

**What the process does—not what it is called.**

That shift is the foundation of modern EDR effectiveness.

---

# Building Analyst Workflow Discipline

Elastic Defend introduces a structured investigative workflow.

That workflow typically follows this pattern:

1. Detect suspicious event
2. Identify originating process
3. Track parent-child relationships
4. Analyze privilege transitions
5. Correlate timestamps
6. Generate incident narrative

This structured workflow reduces guesswork.

And guesswork is where investigations fail.

---

# The Strategic Advantage: Unified Detection and Investigation

Elastic Defend’s real strength is integration.

Instead of maintaining separate systems for:

- Logging
- Detection
- Investigation
- Visualization

Everything exists inside the same environment.

That reduces:

- Tool switching overhead
- Correlation complexity
- Investigation latency

It also improves analyst response speed—often the difference between containment and breach expansion.
## THM Elastic: Using Elastic Defend Room Answers

What version of the Elastic Defend integration is installed on your Linux host machine?

`9.2.0`


Which default protection level does Elastic Defend use for Malware?

`Prevent`

Highlight the process telemetry data being collected on your VM with event.category: process.
What is the full process.name field value for the Python process?

`python3.12`

Next, highlight the network events with event.category: network.
What is the top value for the network.type field?

`ipv4`



Head back to Discover and enter the query: process.args: *discovery.sh*.
What is the process.parent.executable field value for the event you located?

`/usr/bin/bash`

Locate the process.entity_id field value from the event above.
Use it against the process.parent.entity_id field to track the process chain.
What is the first command executed by the script?

`find /home -name *creds*`

Continue looking through the commands that were executed.
What is the name of the directory created in /tmp?

`creds`


Investigate the malware prevention alert created from the EICAR test file.
What is the event.risk_score field value?

`73`


What event.code field value is assigned to the above alert?

`malicious_file`


What is the risk score assigned to the /etc/shadow file read alert?

`47`


Using the flyout menu for the /etc/shadow file read alert, investigate the Overview section.
What is the first MITRE ATT&CK tactic name associated with the alert?

`Privilege Escalation`

How many MITRE Tactics are associated with the Cron Job Created or Modified alert?

`3`

Investigate the printf child process using the Analyzer graph and flyout panel.
What is the process.executable field value?

`/usr/bin/printf`


Which IP address and port number were used for the reverse shell attempt?

`10.10.10.100 4444`


Check out the remaining chmod child process.
What is the name of the cron job whose permissions were set?
`system-update`









