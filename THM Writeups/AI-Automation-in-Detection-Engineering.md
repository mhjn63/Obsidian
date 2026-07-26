> HTML Page: [[HTML Pages/THM/AI-Automation-in-Detection-Engineering.html|Open HTML Page]]

Detection engineering has a speed problem.

At Aurora Logistics, every new detection took three weeks or more to reach production. The process was familiar: research the threat, write the rule, test it manually, ask a colleague to review it, then deploy it carefully across the SOC stack.

Under pressure to accelerate delivery, one engineer began removing what looked like unnecessary friction.

Peer review was skipped for “simple” rules. Testing periods disappeared. Threat research was shortened. Rules were pushed directly into the SIEM and treated as finished.

At first, the decision looked successful. Delivery accelerated. The backlog moved. Stakeholders saw faster output.

Then the operational cost appeared.

Alerts began firing without enough context for analysts to understand what had happened. A production rule stopped working, but no one could determine when it broke or who changed it. False positives increased by roughly 60 percent, and the SOC became buried in noise.

The failure was not simply that some detections were poorly written. The deeper problem was that the organization had removed the safety net around detection delivery.

This is the central tension in modern detection engineering: teams need to move faster, but every shortcut around review, testing, versioning, and deployment control increases the risk that speed will degrade security instead of improving it.

Automation solves that problem only when it strengthens the process rather than bypassing it.

---

## The Real Risk Is Not Slow Delivery

Manual detection engineering is expensive because too much of the workflow depends on repeated human effort.

Research must be gathered. Data sources must be reviewed. Rules must be written. Syntax must be checked. Queries must be tested. Peers must review the logic. Deployments must be performed consistently. Maintenance must continue after the rule reaches production.

The obvious temptation is to accelerate delivery by removing steps.

That is the wrong target.

The problem is not that review, testing, or change control exist. The problem is that they are slow when performed manually and inconsistently.

A mature detection program does not choose between speed and quality. It automates the controls that protect quality so the team can move faster without weakening the process.

That is the role of Detection-as-Code.

---

## Detection-as-Code Turns Rule Delivery Into an Engineering System

Detection-as-Code, commonly shortened to DaC, applies software engineering practices to security detections.

The idea is straightforward: detections should be treated like code because they behave like code. They have logic, dependencies, versions, failure modes, deployment targets, and production impact.

A detection may be written as a Sigma YAML file, a native SIEM query, an EDR rule, or a custom format. The syntax matters less than the operating model around it.

With DaC, detections are stored in a repository, reviewed before release, tested automatically, and deployed through a repeatable pipeline rather than edited directly in production.

The repository becomes the source of truth.

The SIEM becomes the execution environment.

That separation is what makes safe scale possible.

---

## Version Control Makes Every Change Traceable

In a manual environment, the current production rule may exist only inside the SIEM.

That creates immediate problems:

- The team may not know who changed the rule.
- The reason for the change may be lost.
- Previous versions may be difficult to recover.
- Rollback may depend on memory or screenshots.
- Different environments may silently drift apart.

With Git-based version control, every change is recorded. The team can see who changed the rule, what changed, when it changed, why the change was made, which version is deployed, and which previous version should be restored if the update fails.

This matters because detections are operational systems. A small field-name change, parsing assumption, threshold adjustment, or data-source migration can silently break coverage.

Version control turns that failure from an untraceable event into a reversible change.

---

## Peer Review Becomes Part of Delivery

Manual review often depends on social effort.

An engineer finishes a rule, messages a colleague, waits for availability, and hopes the review is detailed enough to catch hidden problems.

Detection-as-Code formalizes the process through pull requests.

A pull request is not just a request to merge a file. It is a structured checkpoint where another engineer can evaluate:

- Whether the detection logic matches the intended threat behavior.
- Whether the required data sources exist.
- Whether fields are mapped correctly.
- Whether exclusions are justified.
- Whether the alert includes enough context.
- Whether the rule duplicates existing coverage.
- Whether the expected false-positive profile is acceptable.
- Whether the documentation is complete.

Review stops being optional and becomes part of the system.

Aurora Logistics did not fail because engineers lacked knowledge. It failed because the workflow allowed knowledge checks to be skipped.

---

## Automated Checks Catch Problems Before Production

The CI/CD pipeline is the operational core of Detection-as-Code.

Whenever an engineer proposes a change, the pipeline runs a series of automated checks before the rule is accepted.

A mature detection pipeline may validate:

- YAML or JSON syntax.
- Required metadata.
- Naming conventions.
- Rule identifiers.
- ATT&CK mappings.
- Severity and confidence values.
- Log-source declarations.
- Query conversion.
- Backend compatibility.
- Unit tests.
- Historical-data performance.
- Expected alert volume.
- False-positive rate.
- Documentation completeness.

This is where Aurora Logistics could have caught the 60 percent increase in noise.

Instead of discovering the problem after deployment, the pipeline could have evaluated the rule against historical data and flagged the change before it reached analysts.

Automation shifts failure left. The earlier a mistake is found, the cheaper it is to fix.

---

## Controlled Deployment Removes Configuration Drift

Manual deployment creates inconsistency.

An engineer may copy a query into one SIEM, forget a field mapping in another, apply a different schedule in a third environment, or miss a required suppression setting.

A controlled pipeline deploys the same approved rule in the same way every time.

Once the pull request is approved and merged, the pipeline can:

1. Convert the generic rule into the target platform’s query format.
2. Validate the generated query.
3. Push it to the SIEM or EDR.
4. Apply the correct schedule, severity, and metadata.
5. Record the deployment result.
6. Alert the team if any target fails.
7. Roll back when required.

This removes an entire category of human error and makes multi-platform environments manageable.

---

## Where Detection-as-Code Fits in the Lifecycle

Detection-as-Code does not replace the full detection engineering lifecycle.

It is strongest in the stages where rules are developed, reviewed, deployed, and maintained.

Earlier stages usually remain outside the core DaC pipeline, but they can still be automated.

### Backlog Management

Detection requirements are typically managed in systems such as Jira.

Each request can capture the threat or technique to detect, business priority, required data sources, expected severity, ownership, acceptance criteria, and related incidents or intelligence.

Automation can enrich tickets, assign priorities, identify duplicates, and link related coverage gaps.

### Threat Research

Threat research is often one of the slowest parts of detection development.

Automation can ingest threat intelligence reports, advisories, malware analyses, and incident-response write-ups, then extract indicators, ATT&CK techniques, process names, command-line patterns, registry paths, network behaviors, and candidate detection opportunities.

This does not remove the need for analyst judgment. It reduces the amount of manual reading required before that judgment begins.

### Data Review

A rule cannot detect behavior the organization does not log.

An automated data inventory can answer:

- Which log sources exist?
- Which systems are covered?
- Which fields are populated?
- How fresh is the data?
- Are schemas consistent?
- Which sources are missing?

This prevents engineers from writing rules that look correct on paper but cannot function in the environment.

### Rule Development, Deployment, and Maintenance

This is the core Detection-as-Code zone.

Rules are written, reviewed, tested, converted, deployed, monitored, tuned, and retired through the pipeline.

The lifecycle becomes repeatable because controls are embedded in the system rather than dependent on memory.

---

## Detection-as-Code Is a Pattern, Not a Product

A common implementation may use:

- Sigma for generic detection logic.
- Git for version control.
- GitHub Actions or GitLab CI for automation.
- The Sigma toolchain for query conversion.
- Platform APIs for deployment.
- Historical SIEM data for testing.

But Detection-as-Code does not depend on any single tool.

Some environments use data sources or logic that Sigma cannot represent cleanly. Others deploy to platforms with no mature Sigma backend. Some organizations need custom metadata, validation, or proprietary deployment logic.

In those cases, teams build their own system: a custom rule format, custom conversion, custom tests, and custom deployment scripts.

The tooling changes. The engineering principles do not.

The essential requirements remain:

- Store detections outside the production platform.
- Track every change.
- Require review.
- Test before deployment.
- Deploy consistently.
- Preserve rollback.
- Maintain a complete audit trail.

---

## Why DaC Lives Outside the SIEM

A SIEM is designed to evaluate detections against large volumes of events.

It is not always designed to manage detections as a collaborative engineering product.

Most SIEM and EDR platforms are strong at execution but weaker at pull-request review, cross-platform version control, automated pre-production testing, unified rollback, and change auditing across multiple tools.

This becomes especially important in an MSSP.

An MSSP may operate several SIEMs, EDR platforms, cloud environments, and customer-specific stacks. No single platform can govern the entire detection estate.

Without an external repository and pipeline, the engineer must repeat the same lifecycle separately in every technology.

Detection-as-Code creates one control plane outside those tools.

The repository governs what should exist.

The pipeline governs how it reaches production.

The SIEM executes the final detection.

---

## AI Accelerates the Human-Heavy Parts of the Process

Detection-as-Code creates the delivery machine.

AI accelerates the slowest stages inside and around it.

Large language models can reduce repetitive work across the lifecycle, especially when used as engineering assistants rather than autonomous authorities.

Useful applications include:

- Summarizing threat intelligence reports.
- Extracting ATT&CK techniques.
- Drafting Sigma rules.
- Converting natural-language requirements into detection logic.
- Suggesting field mappings.
- Generating test cases.
- Writing rule documentation.
- Producing analyst triage guidance.
- Comparing similar detections.
- Identifying missing metadata.
- Proposing false-positive exclusions.
- Translating rules between query languages.
- Reviewing historical alerts for tuning candidates.

This can remove hours of mechanical work.

But it does not remove the need for validation.

AI can produce a rule quickly. It cannot guarantee that the rule is correct for the organization’s data, architecture, risk tolerance, or threat model.

---

## Speed Creates a New Review Problem

AI introduces a paradox.

The faster detections can be generated, the more carefully teams must decide what to trust.

A model may produce logic that looks technically polished while containing serious flaws:

- Hallucinated fields.
- Invalid syntax.
- Incorrect ATT&CK mappings.
- Overly broad conditions.
- Missing exclusions.
- Assumptions about data that does not exist.
- Rules copied from incompatible platforms.
- Unsafe thresholds.
- Weak investigation context.
- Duplicated coverage.
- Logic that detects tools instead of malicious behavior.

This is especially dangerous because AI output often appears complete.

A weak manually written rule may look unfinished. A weak AI-generated rule may look authoritative.

That makes human review more important, not less.

---

## AI Should Draft, Not Approve

The safest operating model is simple:

AI creates a candidate.

The engineering system decides whether it is production-ready.

An AI-generated rule should still pass through:

- Human pull-request review.
- Syntax validation.
- Schema checks.
- Query conversion.
- Historical-data testing.
- False-positive estimation.
- Environment-specific validation.
- Controlled deployment.
- Post-deployment monitoring.

AI can increase throughput without changing the trust boundary.

The human reviewer remains accountable for the decision to merge.

The pipeline remains accountable for enforcing technical controls.

The production environment remains protected from unreviewed output.

---

## Agentic Detection Development

The next step beyond standalone AI assistance is agentic detection development.

Instead of using one model for one task, several specialized agents can be chained into a single workflow.

A typical pipeline may include:

### Requirement Agent

Converts a plain-language request into structured requirements, including the target behavior, ATT&CK techniques, required data, expected severity, environmental assumptions, and acceptance criteria.

### Research Agent

Reviews threat reports, incident write-ups, and intelligence sources to extract detection-relevant behaviors.

### Rule Authoring Agent

Drafts the detection in Sigma or another supported format.

### Conversion Agent

Translates the generic detection into the target SIEM or EDR query language.

### Test Agent

Generates positive and negative test cases, validates syntax, and compares the rule against historical data.

### Documentation Agent

Produces rule descriptions, analyst investigation steps, false-positive notes, response recommendations, and change summaries.

### Reviewer Agent

Performs a fast pre-check for missing logic, incomplete metadata, weak assumptions, or obvious inconsistencies.

This can turn a plain-language request into a drafted, tested, and documented detection in minutes.

But the reviewer agent is not the final approver.

It is a pre-review layer.

The output must still enter the normal Detection-as-Code process:

1. Human pull-request review.
2. CI validation.
3. Automated tests.
4. Controlled deployment.
5. Production monitoring.

The pipeline becomes faster, but the approval boundary remains human.

---

## AI Can Improve the Front and Back of the Lifecycle

Agentic rule creation is only one use case.

AI also has value before development begins and after deployment is complete.

### At the Front: Threat Research

An AI system can process an incident-response report or threat-intelligence article and extract observable behaviors, relevant ATT&CK techniques, candidate log sources, process trees, command lines, persistence mechanisms, network patterns, and potential detection requirements.

This gives engineers a structured starting point instead of forcing them to read every report line by line.

### At the Back: Maintenance

AI can analyze alert history and identify detections that may need attention.

It can surface rules that are excessively noisy, rarely useful, duplicative, missing context, producing repeated false positives, no longer supported by active data sources, or candidates for retirement.

Maintenance is often neglected because new rule requests appear more urgent. AI can make maintenance visible before degraded detections overwhelm the SOC.

---

## A Safer Operating Model for AI-Assisted Detection Engineering

A reliable program should follow several non-negotiable principles.

### Never Deploy AI Output Directly

Generated rules must never bypass review or testing.

### Keep the Repository as the Source of Truth

AI may propose changes, but all accepted changes must exist in version control.

### Test Against Real Organizational Data

Generic examples are not enough. Rules must be evaluated against the actual schemas, field values, noise patterns, and business activity of the environment.

### Require Explainable Logic

Every rule should clearly state what behavior it detects, why the logic works, which assumptions it makes, which data it requires, what may cause false positives, and how analysts should investigate it.

### Preserve Human Accountability

The model does not own the detection. A named engineer or team does.

### Measure Production Impact

After deployment, monitor alert volume, false-positive rate, true-positive value, analyst handling time, detection latency, data-source health, rule failures, and coverage overlap.

AI changes how quickly detections are produced. It does not change the need to prove that they work.

---

## Rebuilding Aurora Logistics

Aurora Logistics does not need fewer controls.

It needs automated controls.

The detection team should replace direct SIEM editing with a Detection-as-Code pipeline that includes:

1. A Git repository for all detection logic.
2. Mandatory pull requests.
3. Required metadata and documentation.
4. Automated syntax and schema validation.
5. Historical-data testing.
6. False-positive and alert-volume thresholds.
7. Controlled deployment through platform APIs.
8. Post-deployment health checks.
9. Rollback support.
10. Continuous maintenance metrics.

AI can then be added safely to accelerate threat research, initial rule drafting, test generation, documentation, query conversion, and tuning recommendations.

The process should become faster because repetitive work is automated—not because review, testing, and accountability are removed.

---

## The Future of Detection Engineering

The future of detection engineering will be defined by scale.

Threat intelligence will arrive faster. Adversaries will automate more of their operations. Security teams will be expected to create and update detections at a pace that manual workflows cannot sustain.

Teams that remain dependent on ad hoc development, direct SIEM editing, and informal review will fall behind.

But teams that pursue speed without governance will produce noise, outages, blind spots, and analyst fatigue.

The winning model combines three elements:

- Automation for consistency.
- AI for acceleration.
- Human judgment for trust.

Detection-as-Code provides the structure.

AI provides the velocity.

Engineering discipline determines whether that velocity becomes defensive advantage or operational debt.

The lesson from Aurora Logistics is simple: the safest path to faster detections is not to skip steps. It is to automate the steps that should never have been optional.
