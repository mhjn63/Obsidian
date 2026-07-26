> HTML Page: [Open HTML Page](HTML%20Pages/Free%20Notes/Cybersecurity/Bug%20Bounty/Bug%20Bounty%20Methodology.html)

🏠 [Main Site](https://motasem-notes.net/) · [🛒 Store](https://shop.motasem-notes.net/) · [▶ YouTube](https://www.youtube.com/@MotasemHamdan) · [☕ Membership](https://buymeacoffee.com/notescatalog/membership)

> Get the full BUH BOUNTY PACK in PDF [here](https://buymeacoffee.com/notescatalog/e/528934)

**Table of Contents**
- [Your Path to Bug Bounty Hunting](#Your%20Path%20to%20Bug%20Bounty%20Hunting)
	- [](#Your%20Path%20to%20Bug%20Bounty%20Hunting#Your%20Path%20to%20Bug%20Bounty%20Hunting#How%20to%20Choose%20and%20Master%20Your%20First%20Vulnerability|How%20to%20Choose%20and%20Master%20Your%20First%20Vulnerability)
		- [](#How%20to%20Choose%20and%20Master%20Your%20First%20Vulnerability#How%20to%20Choose%20and%20Master%20Your%20First%20Vulnerability#Step%201|Step%201)
		- [](#How%20to%20Choose%20and%20Master%20Your%20First%20Vulnerability#How%20to%20Choose%20and%20Master%20Your%20First%20Vulnerability#Step%202|Step%202)
	- [](#Your%20Path%20to%20Bug%20Bounty%20Hunting#Your%20Path%20to%20Bug%20Bounty%20Hunting#Sample%20Workflow|Sample%20Workflow)
	- [](#Your%20Path%20to%20Bug%20Bounty%20Hunting#Your%20Path%20to%20Bug%20Bounty%20Hunting#How%20Do%20You%20Know%20You're%20on%20the%20Right%20Path?|How%20Do%20You%20Know%20You're%20on%20the%20Right%20Path?)
- [Report Writing](#Report%20Writing)
	- [](#Report%20Writing#Report%20Writing#Structural%20Foundations%20of%20Effective%20Bug%20Reports|Structural%20Foundations%20of%20Effective%20Bug%20Reports)
	- [](#Report%20Writing#Report%20Writing#Respecting%20Scope%20:%20A%20Critical%20Operational%20Discipline|Respecting%20Scope%20:%20A%20Critical%20Operational%20Discipline)
	- [](#Report%20Writing#Report%20Writing#Impact%20:%20The%20Core%20of%20Reward%20Justification|Impact%20:%20The%20Core%20of%20Reward%20Justification)
	- [](#Report%20Writing#Report%20Writing#Supporting%20Evidence%20:%20Enhancing%20Reproducibility|Supporting%20Evidence%20:%20Enhancing%20Reproducibility)
		- [](#Supporting%20Evidence%20:%20Enhancing%20Reproducibility#Supporting%20Evidence%20:%20Enhancing%20Reproducibility#HTTP%20Requests%20and%20Responses|HTTP%20Requests%20and%20Responses)
		- [](#Supporting%20Evidence%20:%20Enhancing%20Reproducibility#Supporting%20Evidence%20:%20Enhancing%20Reproducibility#Screenshots|Screenshots)
		- [](#Supporting%20Evidence%20:%20Enhancing%20Reproducibility#Supporting%20Evidence%20:%20Enhancing%20Reproducibility#Video%20Demonstrations|Video%20Demonstrations)
	- [](#Report%20Writing#Report%20Writing#Maintaining%20Professional%20Conduct|Maintaining%20Professional%20Conduct)
	- [](#Report%20Writing#Report%20Writing#The%20Standard%20Bug%20Bounty%20Report%20Template|The%20Standard%20Bug%20Bounty%20Report%20Template)
	- [](#Report%20Writing#Report%20Writing#Title|Title)
	- [](#Report%20Writing#Report%20Writing#Summary|Summary)
	- [](#Report%20Writing#Report%20Writing#Description|Description)
	- [](#Report%20Writing#Report%20Writing#Steps%20to%20Reproduce|Steps%20to%20Reproduce)
	- [](#Report%20Writing#Report%20Writing#Steps%20to%20Reproduce:|Steps%20to%20Reproduce:)
	- [](#Report%20Writing#Report%20Writing#Example%20script%20used%20during%20exploitation|Example%20script%20used%20during%20exploitation)
	- [](#Report%20Writing#Report%20Writing#Supporting%20Material|Supporting%20Material)
	- [](#Report%20Writing#Report%20Writing#Supporting%20Material:|Supporting%20Material:)
	- [](#Report%20Writing#Report%20Writing#Impact|Impact)
	- [](#Report%20Writing#Report%20Writing#Mitigation%20(Optional)|Mitigation%20(Optional))
	- [](#Report%20Writing#Report%20Writing#Example%20Markdown%20Report%20Template|Example%20Markdown%20Report%20Template)
		- [](#Example%20Markdown%20Report%20Template#Example%20Markdown%20Report%20Template#Summary:|Summary:)
		- [](#Example%20Markdown%20Report%20Template#Example%20Markdown%20Report%20Template#Description:|Description:)
		- [](#Example%20Markdown%20Report%20Template#Example%20Markdown%20Report%20Template#Steps%20to%20Reproduce:|Steps%20to%20Reproduce:)
		- [](#Example%20Markdown%20Report%20Template#Example%20Markdown%20Report%20Template#Supporting%20Material:|Supporting%20Material:)
		- [](#Example%20Markdown%20Report%20Template#Example%20Markdown%20Report%20Template#Impact:|Impact:)
		- [](#Example%20Markdown%20Report%20Template#Example%20Markdown%20Report%20Template#Mitigation:|Mitigation:)
	- [](#Report%20Writing#Report%20Writing#Core%20Operational%20Insight|Core%20Operational%20Insight)
- [Bug Bounty Report Sample Template](#Bug%20Bounty%20Report%20Sample%20Template)
- [Issue Description](#Issue%20Description)
- [Affected URL/Area](#Affected%20URL/Area)
- [Risk Rating](#Risk%20Rating)
	- [](#Risk%20Rating#Risk%20Rating#Impact|Impact)
	- [](#Risk%20Rating#Risk%20Rating#Attack%20Scenario|Attack%20Scenario)
- [Steps to Reproduce/PoC](#Steps%20to%20Reproduce/PoC)
	- [](#Steps%20to%20Reproduce/PoC#Steps%20to%20Reproduce/PoC#Request|Request)
	- [](#Steps%20to%20Reproduce/PoC#Steps%20to%20Reproduce/PoC#Response|Response)
	- [](#Steps%20to%20Reproduce/PoC#Steps%20to%20Reproduce/PoC#Screenshots|Screenshots)
- [Affected Demographic/User Base](#Affected%20Demographic/User%20Base)
- [Recommended Fix](#Recommended%20Fix)
- [References](#References)
- [Bug Bounty Platforms](#Bug%20Bounty%20Platforms)
	- [](#Bug%20Bounty%20Platforms#Bug%20Bounty%20Platforms#Bug%20Bounty%20Programs|Bug%20Bounty%20Programs)

## Your Path to Bug Bounty Hunting
### How to Choose and Master Your First Vulnerability

A great first choice is **Cross-Site Scripting (XSS)**. Why?
- It's one of the most common web vulnerabilities.
- It appears in many different forms (reflected, stored, DOM-based).
- Learning to find it teaches you a ton about how websites process user input and how browsers render content.

Another solid option is **Insecure Direct Object References (IDORs)**, which will teach you about access control and how applications handle user-specific data.

**Let's stick with XSS for this example.** Here’s a structured way to master it:
#### Step 1
Forget aimlessly hunting for a bit. Your primary goal is to understand XSS inside and out. The single best free resource for this is **PortSwigger's Web Security Academy**.

1. **Go to the Cross-site scripting section.**
2. **Read the theory.** Understand what XSS is, the difference between reflected, stored, and DOM-based XSS, and the concept of "contexts" (HTML, attribute, JavaScript, etc.).
3. **Solve the labs.** This is non-negotiable. Do every single lab. The `Apprentice` and `Practitioner` level labs will build your foundation. The `Expert` labs will show you how to bypass filters and use advanced techniques.

This focused practice is the `how` you're missing. It will give you the hands-on experience in a controlled environment where you _know_ a vulnerability exists. This builds confidence and pattern recognition.

#### Step 2
Now, let's connect this back to your recon.

1. **Choose a Program:** Pick a bug bounty program with a broad scope (e.g., a `*.example.com` target).
2. **Run Your Recon:** Do your usual subdomain enumeration with `subfinder` and `httpx`.
3. **Analyze the Output:** Instead of just taking screenshots, look at your list of live domains from `httpx`. Which ones look interesting?
    - Are there any with old-looking titles? (e.g., "Login Portal," "Admin," "UAT," "Test")
    - Which ones have a lot of user-interactive features like search bars, forms, profiles, or comment sections? These are prime targets for XSS.
4. **Manual Inspection is Key:** Pick one or two interesting subdomains and explore them manually. Click every button. Fill out every form. Look at the URL parameters.
5. **Start Hunting for XSS:**
    - **Find Input Points:** Where can you enter data? Search bars, contact forms, username fields, URL parameters (`?q=`, `?id=`, `?redirect_url=`), etc.
    - **Test for Reflections:** Enter a simple, non-malicious string like `cybertest`. Now, look at the page source (`Ctrl+U` or `Cmd+U`). Is your string reflected anywhere?
    - **Understand the Context:**
        - If it's reflected inside an HTML tag like `<h1>cybertest</h1>`, try injecting a simple payload: `<h1><script>alert(1)</script></h1>`.
        - If it's inside an attribute like `<input value="cybertest">`, you might need to break out of the attribute: `"><script>alert(1)</script>`.
        - If it's inside a JavaScript block like `<script>var query = "cybertest";</script>`, you'll need a different payload to break out of the string: `";alert(1)//`.
    - **Check Your DevTools:** Open the browser's developer tools (F12). The **Console** will show you any errors, and the **Network** tab will let you inspect the requests and responses. The **Elements** tab shows you the live DOM, which is crucial for finding DOM XSS.

### Sample Workflow
A more effective workflow that goes beyond just recon:

1. **Recon Phase (Your current workflow):** Enumerate subdomains (`subfinder`), find live hosts (`httpx`).
2. **Triage Phase (The missing link):**
    - Use tools like `gospider` or `katana` to crawl the live hosts and find endpoints/parameters.
    - Manually review the output. Look for parameters with names like `?next=`, `?redirect=`, `?url=`, `?q=`, `?search=`, `?id=`. These are hints!
    - Manually browse the top 5-10 most interesting-looking applications you found. What do they do? How do they work?
3. **Focused Manual Testing Phase (The "Real" Hacking):**
    - Pick **one** application and **one** vulnerability class (e.g., XSS).
    - Test every input point on that application for that specific vulnerability. Use the knowledge you gained from PortSwigger.
    - Take notes on what you tried and what the outcome was. This is crucial for learning.

### How Do You Know You're on the Right Path?
You're on the right path when you can answer "why" you're doing something.
- **Instead of:** "I'm running `subjs` because the workflow told me to."
- **It becomes:** "I'm running `subjs` to extract JavaScript files. I'll then manually read these files to find interesting API endpoints, developer comments, or client-side logic that might lead to a vulnerability like DOM XSS."

**Success isn't just finding a bug.** Success is understanding _why_ a payload worked or _why_ it didn't. When you can explain the context of a reflection and why a specific payload is needed, you've "got it."

## Report Writing
Most learning material in offensive security concentrates heavily on discovering vulnerabilities, yet comparatively little attention is given to the discipline of documenting findings effectively. This imbalance creates a major skill gap, because the ability to discover vulnerabilities alone does not guarantee recognition, reward, or professional credibility. The value of a vulnerability is not realized until it is clearly communicated in a format that enables reproduction, validation, and remediation.

A well-written bug bounty report directly accelerates the triage workflow. Security analysts responsible for vulnerability validation process large volumes of submissions daily, often under strict operational timelines. When a report lacks clarity, contains incomplete steps, or presents technical findings without structure, the analyst must spend additional time interpreting intent, reconstructing missing details, and attempting multiple reproduction paths. This slows resolution cycles and frequently results in unnecessary communication loops between researcher and triage team. Conversely, when a report is structured logically, includes reproducible instructions, and clearly explains impact, triage teams can validate the issue quickly and progress toward remediation without delay. This efficiency significantly reduces friction and shortens response timelines.

High-quality reporting also influences financial outcomes. Reward increases typically occur through three separate mechanisms. First, when reports are complete and self-contained, the researcher spends less time responding to clarification requests and more time identifying new vulnerabilities. This increased productivity naturally improves overall earnings potential. Second, many bug bounty programs provide discretionary bonuses when reports help engineering teams rapidly identify root causes or reduce debugging time. Third, structured reporting improves the likelihood of securing reward eligibility in duplicate scenarios. Many programs award bounties only to the first submission that demonstrates reliable reproduction. If a duplicate report arrives earlier but lacks reproducible steps, a later but fully validated report may still be rewarded.

Professional reputation is also heavily influenced by reporting quality. When reports are publicly disclosed after remediation, they serve as technical proof of capability. These disclosures function as a professional portfolio, demonstrating not only technical discovery skills but also communication discipline and engineering awareness. In hiring contexts, the ability to show real-world vulnerability documentation carries significant weight, particularly when compared to candidates who can only demonstrate theoretical knowledge.

---

### Structural Foundations of Effective Bug Reports

Effective vulnerability documentation relies fundamentally on structure. The organization of information determines whether the recipient can easily interpret the vulnerability lifecycle from identification to exploitation. Poor structure introduces ambiguity, while structured reporting creates clarity and traceability.

Before submitting any report, the central evaluation question should be whether the document can be followed without external explanation. A reliable method for validating clarity involves re-reading the report from the perspective of someone unfamiliar with the application. Any location that introduces confusion must be rewritten until the workflow becomes obvious.

Formatting also plays a decisive role. Most modern bug bounty platforms support Markdown rendering, enabling structured presentation of technical material. Without proper formatting, reports containing raw HTTP requests, scripts, or logs appear visually disorganized and difficult to interpret. Code blocks, numbered lists, and headings transform dense technical content into readable technical narratives.

---

### Respecting Scope : A Critical Operational Discipline
One of the most frequent operational failures among inexperienced researchers involves ignoring program scope definitions. Every bug bounty program publishes a formal policy describing permitted targets, prohibited assets, and excluded vulnerability classes. Failing to consult this document leads to wasted effort and damaged reputation metrics.

When a report is submitted, the first validation step performed by triage teams involves confirming whether the asset exists within the defined scope. If the affected system or vulnerability type falls outside permitted boundaries, the submission is immediately closed. Such closures negatively affect reputation scoring and trust metrics.

Reputation metrics directly influence workflow efficiency. On platforms such as HackerOne, high-reputation researchers bypass certain automated filtering stages designed to reduce noise. This privilege results in faster processing, reduced delays, and increased invitation opportunities to higher-value private programs. Maintaining consistent adherence to scope policies therefore functions as both an operational necessity and a strategic advantage.

---

### Impact : The Core of Reward Justification
A vulnerability without a clearly defined impact has limited practical value. The impact section represents the strongest opportunity to demonstrate severity and justify reward magnitude. This section must describe not only what vulnerability exists, but how it affects system security in measurable terms.

For example, if unauthorized access to cloud storage is possible, the presence of sensitive data must be verified. Demonstrating that confidential records exist within the storage environment significantly increases severity classification compared to listing static or publicly accessible files. Similarly, in subdomain takeover scenarios, simply identifying an unclaimed domain is insufficient. Demonstrating successful domain control establishes tangible exploitability and confirms risk validity.

Impact statements should emphasize real-world consequences such as data exposure, account compromise, service disruption, or infrastructure compromise. The objective is to translate technical findings into operational risk language understandable to decision-makers.

---

### Supporting Evidence : Enhancing Reproducibility
Supporting documentation strengthens credibility and simplifies vulnerability validation. Effective support material removes ambiguity and provides verification checkpoints for analysts.

#### HTTP Requests and Responses
Whenever vulnerabilities involve HTTP communication, both the triggering request and resulting response should be included. The most important elements of these interactions typically include:

- Request method
- Target endpoint
- Vulnerable parameter
- Response indicators

Excessively large payload dumps should be avoided unless required for analysis. Only essential components should be included to preserve clarity.

Proper formatting remains mandatory. HTTP messages must be placed inside Markdown code blocks to visually separate them from descriptive text.

Example structure:
```http
POST /vulnerable-endpoint HTTP/1.1  
Host: target.com  
Content-Type: application/json  
  
{"param":"malicious_payload"}
```

---

#### Screenshots
Screenshots function as visual confirmation tools. They are particularly useful when:

- User interfaces contain complex navigation paths
- Vulnerable input locations are not obvious
- Exploitation results produce visual proof

For instance, screenshots demonstrating successful data extraction or unauthorized access provide immediate validation of exploit success.

---

#### Video Demonstrations
Video documentation becomes necessary when exploitation workflows involve numerous dependent steps. Multi-stage vulnerabilities, chained attacks, or conditional behaviors benefit significantly from short demonstration recordings.

Videos should remain concise, structured, and focused exclusively on reproduction sequences. Spoken narration improves clarity when explaining decision logic or intermediate observations.

---

### Maintaining Professional Conduct
Technical expertise alone does not determine long-term success in vulnerability research. Professional communication behavior directly affects reputation stability and triage cooperation.

Disagreements regarding reproducibility occasionally occur due to environmental differences between researcher and analyst systems. When such issues arise, emotional reactions degrade credibility and may result in premature report closure. Maintaining professionalism ensures that disagreements remain technical rather than interpersonal.

Providing additional evidence, such as video demonstrations or environment configuration details, often resolves reproduction conflicts. Maintaining composure during these interactions preserves professional standing and supports long-term collaboration.

---

### The Standard Bug Bounty Report Template
Structured templates dramatically simplify report creation and improve consistency. The following template represents a widely effective format used across multiple professional engagements.

---

### Title

The report title functions as the initial point of context. It must communicate vulnerability type, affected asset, endpoint location, and vulnerable parameter in a single descriptive line.

Example:
**Reflected Cross-Site Scripting on xyz.com via `/search` endpoint using `query` parameter**

This level of specificity allows triage teams to immediately understand the vulnerability classification and scope.

---

### Summary
The summary provides a concise explanation of the vulnerable functionality. It introduces the application feature involved and describes how the vulnerability manifests at a conceptual level.

This section should remain brief while still providing enough context to prepare readers for detailed reproduction steps.

---

### Description
The description section expands technical depth beyond the summary. Complex vulnerabilities often require additional explanation covering underlying logic flaws, parameter manipulation, or security misconfigurations.

Separating summary and description improves readability by allowing fast scanning while preserving technical completeness.

---

### Steps to Reproduce

This section represents the procedural core of the report. Each step must be reproducible without assumptions. Instructions should follow chronological order and include all required parameters.

Example structure:

### Steps to Reproduce:  
  
1. Navigate to https://target.com/login  
2. Enter payload into username field  
3. Submit the request  
4. Observe response behavior

When scripts are used, they must be included:

### Example script used during exploitation  
```python
import requests  
  
url = "https://target.com/api"  
payload = {"input":"malicious_data"}  
  
requests.post(url, json=payload)
```
Every included instruction must be validated by executing the process from beginning to end.

---

### Supporting Material
Supporting evidence consolidates validation artifacts such as:

- Screenshots
- Video demonstrations
- Script files
- Traffic captures

Embedding screenshots directly within Markdown improves readability and reduces context switching.

Example placeholder:

### Supporting Material:  
Screenshot demonstrating unauthorized access.  
  
Script used during exploitation:  
`script content here`

---

### Impact
This section explains the operational risk created by the vulnerability. It must describe measurable damage scenarios such as data exposure, privilege escalation, remote code execution, or denial of service.

Strong impact explanations significantly influence reward classification.

---

### Mitigation (Optional)
Mitigation suggestions demonstrate advanced analytical depth. If the root cause has been identified, recommending defensive solutions can shorten debugging cycles and improve remediation speed.

Typical mitigation recommendations include:

- Input validation enforcement
- Authentication hardening
- Permission isolation
- Patch deployment

Providing mitigation guidance occasionally results in discretionary bonuses due to engineering time savings.

---

### Example Markdown Report Template

#### Summary:  
An introduction to the application's feature and your vulnerability.  
  
#### Description:  
In-depth technical details in case the bug is complex.  
  
#### Steps to Reproduce:  
1. Step 1  
2. Step 2  

Raw HTTP request and response goes here

  
#### Supporting Material:  
Screenshots, video  

script you have used goes here

  
#### Impact:  
Clearly explain how the vulnerability affects the system.  
  
#### Mitigation:  
Optional remediation guidance.

---

### Core Operational Insight
Technical discovery without documentation discipline results in reduced recognition, delayed validation, and diminished rewards. Structured reporting transforms vulnerability discovery into operational impact, financial gain, and professional credibility.

## Bug Bounty Report Sample Template
# Title

## Issue Description

A generic overview of the issue, I usually use the default text from OWASP as it explains the issue well. Include a more specific description of the issue identified within the application.

## Affected URL/Area
- The affected urls or area of the application where the issue exists.

## Risk Rating
- Risk: **Critical / High / Medium / Low / Informational**
- Difficulty to Exploit: **Low / Medium / High**
- Authentication Required: **Yes / No**
- User Interaction Required: **Yes / No**
- CVSS 3.1 Score: [X.X](https://www.first.org/cvss/calculator/3.1#CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:N)

### Impact
- What kind of attacker could exploit this? (external, authenticated user, admin)
- What access/privileges do they need?
- What can they achieve? (data theft, privilege escalation, service disruption)
- Who else does it affect? (other users, administrators, the organization)

### Attack Scenario
Describe a realistic attack scenario showing how this vulnerability could be exploited in a real-world situation.

## Steps to Reproduce/PoC
A clear outline of the steps required to execute the payload as an attacker, this can include how to setup the payload and launch it.

1. Step one...
2. Step two...
3. Step three...

### Request
```python
POST /endpoint HTTP/1.1
Host: target.com
Content-Type: application/json

{"example": "payload"}
```

### Response
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"result": "response showing vulnerability"}
```

### Screenshots
- screenshot1.png - Description of what it shows
- screenshot2.png - Description of what it shows

## Affected Demographic/User Base
- Explain who this issue affects?
- Is it everyone or just a select amount of users?
- How can this occur in normal usage?
- What is the potential scale of impact?

## Recommended Fix
- How do you fix the issue?
- What is the recommended remediation actions required to successfully fix issue x?
- Are there any quick mitigations available while a full fix is developed?

## References
Include additional reading for the client to further backup the issues explained or elaborate more on other potential issues chained to the one identified.
- [1] [Reference 1](https://example.com/)
- [2] [Reference 2](https://example.com/)

## Bug Bounty Platforms
- [YesWeHack](https://yeswehack.com/)
- [intigriti](https://intigriti.com/)
- [HackerOne](https://hackerone.com/)
- [Bugcrowd](https://bugcrowd.com/)
- [Cobalt](https://cobalt.io/)
- [Bountysource](https://www.bountysource.com/)
- [Bounty Factory](https://bountyfactory.io/)
- [Coder Bounty](http://www.coderbounty.com/)
- [FreedomSponsors](https://freedomsponsors.org/)
- [FOSS Factory](http://www.fossfactory.org/)
- [Synack](https://www.synack.com/)
- [HackenProof](https://hackenproof.com/)
- [Detectify](https://cs.detectify.com/)
- [Bugbountyjp](https://bugbounty.jp/)
- [Safehats](https://safehats.com/)
- [BugbountyHQ](https://www.bugbountyhq.com/)
- [Hackerhive](https://hackerhive.io/)
- [Hacktrophy](https://hacktrophy.com/)
- [AntiHACK](https://www.antihack.me/)
- [CESPPA](https://www.cesppa.com/)
- [BountyHub](https://bountyhub.dev/)

### Bug Bounty Programs
- [123Contact Form](http://www.123contactform.com/security-acknowledgements.htm)
- [99designs](https://hackerone.com/99designs)
- [Abacus](https://bugcrowd.com/abacus)
- [Acquia](mailto:security@acquia.com)
- [ActiveCampaign](mailto:security@activecampaign.com)
- [ActiveProspect](mailto:security@activeprospect.com)
- [Adobe](https://hackerone.com/adobe)
- [AeroFS](mailto:security@aerofs.com)
- [Airbitz](https://cobalt.io/airbitz)
- [Airbnb](https://hackerone.com/airbnb)
- [Algolia](https://hackerone.com/algolia)
- [Altervista](http://en.altervista.org/feedback.php?who=feedback)
- [Altroconsumo](https://go.intigriti.com/altroconsumo)
- [Amara](mailto:security@amara.org)
- [Amazon Web Services](mailto:aws-security@amazon.com)
- [Amazon.com](mailto:security@amazon.com)
- [ANCILE Solutions Inc.](https://bugcrowd.com/ancile)
- [Anghami](https://hackerone.com/anghami)
- [ANXBTC](https://cobalt.io/anxbtc)
- [Apache httpd](https://hackerone.com/ibb-apache)
- [Appcelerator](mailto:Infosec@appcelerator.com)
- [Apple](mailto:product-security@apple.com)
- [Apptentive](https://www.apptentive.com/contact)
- [Aptible](mailto:security@aptible.com)
- [Ardour](http://tracker.ardour.org/my_view_page.php)
- [Arkane](https://go.intigriti.com/arkanenetwork)
- [ARM mbed](mailto:whitehat@polarssl.org)
- [Asana](mailto:security@asana.com)
- [ASP4all](mailto:support@asp4all.nl)
- [AT&T](https://bugbounty.att.com/bugform.php)
- [Atlassian](https://securitysd.atlassian.net/servicedesk/customer/portal/2)
- [Attack-Secure](mailto:admin@attack-secure.com)
- [Authy](mailto:security@authy.com)
- [Automattic](https://hackerone.com/automattic)
- [Avast!](mailto:bugs@avast.com)
- [Avira](mailto:vulnerabilities@avira.com)
- [AwardWallet](https://cobalt.io/awardwallet)
- [Badoo](https://corp.badoo.com/en/security/#send_bid)
- [Barracuda](https://bugcrowd.com/barracuda)
- [Base](https://go.intigriti.com/base)
- [Basecamp](mailto:security@basecamp.com)
- [Beanstalk](https://wildbit.wufoo.com/forms/wildbit-security-response)
- [BillGuard](https://cobalt.io/billguard)
- [Billys Billing](https://cobalt.io/billys-billing)
- [Binary.com](https://hackerone.com/binary)
- [Binary.com Cashier](https://hackerone.com/binary_cashier)
- [BitBandit.eu](https://cobalt.io/bitbandit-eu)
- [Bitcasa](mailto:security@bitcasa.com)
- [BitCasino](https://cobalt.io/bitcasino)
- [BitGo](https://cobalt.io/bitgo)
- [BitHealth](https://cobalt.io/bithealth)
- [BitHunt](https://hackerone.com/bithunt)
- [BitMEX](https://cobalt.io/bitmex)
- [Bitoasis](https://cobalt.io/bitoasis)
- [Bitpagos](https://cobalt.io/bitpagos)
- [Bitrated](https://cobalt.io/bitrated)
- [Bitreserve](https://cobalt.io/bitreserve)
- [Bitspark](https://cobalt.io/bitspark)
- [Bitwage](https://cobalt.io/bitwage)
- [BitWall](mailto:request@bitwall.io)
- [BitYes](https://cobalt.io/bityes)
- [BlackBerry](https://global.blackberry.com/secure/report-an-issue/en.html)
- [Blackboard](mailto:learnsecurity@blackboard.com)
- [Blackphone](https://bugcrowd.com/blackphone)
- [Blesta](mailto:security@blesta.com)
- [Block.io](https://hackerone.com/blockio)
- [Block.io, Inc.](https://cobalt.io/block-io-inc)
- [Blockchain.info](https://cobalt.io/blockchain-info)
- [BlockScore](https://cobalt.io/blockscore)
- [Bookfresh](https://hackerone.com/bookfresh)
- [Box](mailto:security-reports@box.com)
- [Braintree](mailto:security@braintreepayments.com)
- [Brussels Airlines](https://go.intigriti.com/brusselsairlines)
- [BTC_sx](https://cobalt.io/btc-sx)
- [Buffer](mailto:security@bufferapp.com)
- [BX.in.th](https://cobalt.io/bx-in-th)
- [C2FO](https://hackerone.com/c2fo)
- [Campaign Monitor](https://help.campaignmonitor.com/contact)
- [CARD.com](https://bugcrowd.com/card)
- [Catchafire](https://cobalt.io/catchafire)
- [Caviar](https://hackerone.com/caviar)
- [CCBill](mailto:bugrewards@ccbill.com)
- [CERT/CC](https://hackerone.com/cert)
- [Certly](https://hackerone.com/certly)
- [ChainPay](https://cobalt.io/chainpay)
- [ChangeTip](https://cobalt.io/changetip)
- [Chargify](https://bugcrowd.com/chargify)
- [Chromium Project](https://code.google.com/p/chromium/issues/entry?template=Security%20Bug)
- [Circle](https://cobalt.io/circle)
- [CircleCI](mailto:security@circleci.com)
- [Cisco](http://www.cisco.com/web/about/security/psirt/security_vulnerability_policy.html#roosfassv)
- [ClickUp](https://clickup.com/bug-bounty)
- [Clojars](mailto:contact@clojars.org)
- [CloudFlare](https://hackerone.com/cloudflare)
- [Cobalt](https://cobalt.io/cobalt)
- [Code Climate](mailto:security@codeclimate.com)
- [CodeIgniter](https://hackerone.com/codeigniter)
- [CodePen](https://bugcrowd.com/codepen)
- [Coin Republic](https://cobalt.io/coin-republic)
- [Coin.Space](https://hackerone.com/coinspace)
- [Coinage](https://cobalt.io/coinage)
- [Coinbase](https://hackerone.com/coinbase)
- [CoinDaddy](https://cobalt.io/coindaddy)
- [Coinkite](mailto:feedback@coinkite.com?subject=%5BVulnerability%5D%20-%20)
- [Coinport](https://cobalt.io/coinport)
- [coins.ph](https://cobalt.io/coins-ph)
- [Cointrader.net](https://cobalt.io/cointrader-net)
- [Coinvoy](https://cobalt.io/coinvoy)
- [Collishop](https://go.intigriti.com/collishop)
- [Colruyt](https://go.intigriti.com/colruyt)
- [Compose](mailto:security@compose.io)
- [concrete5](https://hackerone.com/concrete5)
- [Constant Contact](mailto:vulnerability@constantcontact.com)
- [Counterparty](https://cobalt.io/counterparty)
- [Coupa](mailto:security@coupa.com)
- [Coursera](https://hackerone.com/coursera)
- [cPanel](mailto:security@cpanel.net)
- [cPaperless](mailto:support@cPaperless.com)
- [Crix.io](https://cobalt.io/crixio)
- [Cross Border Fines](https://go.intigriti.com/crossborderfines)
- [CrowdShield](https://crowdshield.com/bug-bounty-list.php?bug_bounty_program=crowdshield)
- [Cryptocat](https://github.com/cryptocat/cryptocat/issues)
- [Cupcake](mailto:security@cupcake.io)
- [CustomerInsight](mailto:admin@customerinsight.ca)
- [Cylance](https://hackerone.com/cylance)
- [Dato Capital](mailto:security%40datocapital.com)
- [Detectify](mailto:disclosure@detectify.com)
- [De Volkskrant](https://go.intigriti.com/devolkskrant)
- [Delen Private Bank](https://go.intigriti.com/delen)
- [DigitalOcean](mailto:security@digitalocean.com)
- [DigitalSellz](https://hackerone.com/digitalsellz)
- [Django](https://hackerone.com/django)
- [Doorkeeper](mailto:info@doorkeeper.jp)
- [DoSomething](https://cobalt.io/dosomething)
- [DPD](mailto:security@dpd.zendesk.com)
- [Dragon King](https://hackenproof.com/neverdie/dragon-king)
- [Dreambaby](https://go.intigriti.com/dreamland)
- [Dreamland](https://go.intigriti.com/dream)
- [Dropbox](https://hackerone.com/dropbox)
- [Dropbox Acquisitions](https://hackerone.com/dropbox-acquisitions)
- [Drupal](https://www.drupal.org/node/101494)
- [eBay](http://pages.ebay.com/securitycenter/Researchers.html)
- [Eclipse](mailto:security@eclipse.org)
- [eHealth Hub VZN KUL](https://go.intigriti.com/ehealthhubvznkul)
- [EMC](mailto:security_alert@emc.com)
- [Enano](mailto:security@enanocms.org)
- [Engine Yard](mailto:security@engineyard.com)
- [Envoy](https://hackerone.com/envoy)
- [Eobot](https://cobalt.io/eobot)
- [EthnoHub](mailto:security@ethnohub.com)
- [Etsy](https://www.etsy.com/bounty)
- [EVE](mailto:security@ccpgames.com)
- [Event Espresso](http://eventespresso.com/report-a-security-vulnerability)
- [Everitoken](https://hackenproof.com/everitoken/everitoken-blockchain)
- [Evernote](mailto:security@evernote.com)
- [EURid](https://go.intigriti.com/eurid)
- [Expatistan](mailto:gerardo@expatistan.com)
- [ExpressionEngine](https://hackerone.com/expressionengine)
- [Ezbob](https://cobalt.io/ezbob)
- [Facebook](https://www.facebook.com/whitehat)
- [Faceless](https://hackerone.com/faceless)
- [Factlink](https://hackerone.com/factlink)
- [FanFootage](https://hackerone.com/fanfootage)
- [FastSlots](https://cobalt.io/fastslots)
- [Flash](https://hackerone.com/flash)
- [Flood](mailto:support@flood.io)
- [Flow Dock](mailto:security@flowdock.com)
- [Flox](https://hackerone.com/flox)
- [Fluxiom](mailto:security@fluxiom.com)
- [Fog Creek](http://www.fogcreek.com/contact)
- [FormAssembly](mailto:security@formassembly.com)
- [Founder Bliss](https://cobalt.io/founder-bliss)
- [Foursquare](mailto:security@foursquare.com)
- [Freelancer](mailto:security-reporting@freelancer.com)
- [Gallery](mailto:security@galleryproject.org)
- [Gamma](mailto:security-alert@intergamma.nl)
- [Gemfury](mailto:security@gemfury.com)
- [General Motors](https://hackerone.com/gm)
- [GhostMail](https://hackerone.com/gmguys)
- [GitHub](https://bounty.github.com/submit-a-vulnerability.html)
- [GitLab](https://hackerone.com/gitlab)
- [GlassWire](https://hackerone.com/glasswire)
- [Gliph](mailto:security@gli.ph)
- [GlobaLeaks](https://hackerone.com/globaleaks)
- [Google PRP](mailto:security-patches@google.com)
- [Google VRP](https://www.google.com/about/appsecurity/reward-program/index.html)
- [Grammarly](https://hackerone.com/grammarly)
- [Gratipay](https://hackerone.com/gratipay)
- [GreenAddress](https://cobalt.io/greenaddress)
- [Greenhouse.io](https://hackerone.com/greenhouse)
- [Grok Learning](mailto:security@groklearning.com)
- [HackenProof](https://hackenproof.com/hacken/hackenproof)
- [HackerOne](https://hackerone.com/security)
- [Harmony](mailto:security@collectiveidea.com)
- [Heroku](https://bugcrowd.com/heroku)
- [Hex-Rays](mailto:bugbounty@hex-rays.com)
- [Hive Wallet](https://cobalt.io/hive-wallet)
- [Hootsuite](mailto:security@hootsuite.com)
- [HTC](mailto:security@htc.com)
- [Huawei](mailto:psirt@huawei.com)
- [Hubdia](https://hackerone.com/hubdia)
- [Humble Bundle](https://bugcrowd.com/humblebundle)
- [IAM KU Leuven](https://go.intigriti.com/kuleuvenlogin)
- [Ian Dunn](https://hackerone.com/iandunn-projects)
- [IBM](https://www.ibm.com/scripts/contact/contact/us/en/security_vulnerabilities)
- [ICEcoder](https://bugcrowd.com/icecoder)
- [Iconfinder](mailto:support@iconfinder.com)
- [Ifixit](mailto:security@ifixit.com)
- [Imgur](https://hackerone.com/imgur)
- [ImpressPages](https://cobalt.io/impresspages)
- [Indeed](https://bugcrowd.com/indeed)
- [Independent Reserve](https://cobalt.io/independent-reserve)
- [Informatica](https://hackerone.com/informatica)
- [IntegraXor](http://www.integraxor.com/support.html)
- [Internetwache](mailto:security@internetwache.org)
- [InVision](https://hackerone.com/invision)
- [IRCCloud](https://hackerone.com/irccloud)
- [itBit Exchange](https://hackerone.com/itbit)
- [ITRP](mailto:security@itrp.com)
- [itsme](https://go.intigriti.com/itsme)
- [joola.io](https://hackerone.com/joola-io)
- [Joomla](http://vel.joomla.org/submit-vel)
- [JRuby](mailto:security@jruby.org)
- [jsDelivr](https://hackerone.com/jsdelivr)
- [Juniper](mailto:sirt@juniper.net)
- [Kadira](https://hackerone.com/kadira)
- [Kaneva](mailto:security@kaneva.com)
- [Kayako](http://my.kayako.com/Tickets/Submit)
- [Kenna](https://bugcrowd.com/riskio)
- [Keybase](https://hackerone.com/keybase)
- [Khan Academy](https://hackerone.com/khanacademy)
- [SKB Kontur](https://kontur.ru/.well-known/security.txt)
- [Kraken](mailto:bugbounty@kraken.com)
- [Kinepolis](https://go.intigriti.com/kinepolis)
- [Kuna](https://hackenproof.com/kuna/kuna-crypto-exchange)
- [Lancor Income](https://cobalt.io/lancor-income)
- [LastPass](mailto:security@lastpass.com)
- [LaunchKey](mailto:security@launchkey.com)
- [Lean Testing](https://hackerone.com/leantesting)
- [Librato](mailto:security@librato.com)
- [LibSass](https://hackerone.com/libsass)
- [Liferay](mailto:security@liferay.com)
- [Line](https://bugbounty.linecorp.com/en/)
- [LinkedIn](mailto:security@linkedin.com)
- [LiveEnsure](http://www.liveensure.com/contact.php)
- [LocalBitcoins](https://cobalt.io/localbitcoins)
- [Localize](https://hackerone.com/localize)
- [Logentries](mailto:security@logentries.com)
- [Lookout](mailto:security@lookout.com)
- [Magento](mailto:security@magento.com)
- [MAGIX](mailto:security@magix.net)
- [Mahara](mailto:security@mahara.org)
- [MaiCoin](https://cobalt.io/maicoin)
- [Mail.Ru](https://hackerone.com/mailru)
- [Mailbird](https://cobalt.io/mailbird)
- [MailChimp](http://mailchimp.com/about/security-response/)
- [ManageBGL](https://cobalt.io/managebgl)
- [ManageWP](mailto:security@managewp.com)
- [MapLogin](https://hackerone.com/maplogin)
- [Marietje Schaake](https://go.intigriti.com/marietjeschaake)
- [Marktplatts](https://hackerone.com/marktplaats)
- [Mavenlink](https://hackerone.com/mavenlink)
- [Maximum](https://hackerone.com/maximum)
- [MCProHosting](https://bugcrowd.com/mcprohostings)
- [MEGA](mailto:bugs@mega.co.nz)
- [Mercury](https://cobalt.io/mercury)
- [Meteor](https://hackerone.com/meteor)
- [meXBT](https://cobalt.io/mexbt)
- [Microsoft](mailto:secure@microsoft.com)
- [Mimecast](mailto:disclosure@mimecast.com)
- [Mobile Vikings](https://go.intigriti.com/mobilevikings)
- [Mobile Vikings](https://hackerone.com/mobilevikings)
- [Modus CSR](mailto:security@moduscsr.com)
- [MoneyBird](mailto:security@moneybird.com)
- [MoneyStream](https://hackerone.com/moneystream)
- [Moodle](mailto:security@moodle.org)
- [Motorola Solutions](mailto:security@motorolasolutions.com)
- [Mozilla](https://www.mozilla.org/en-US/security/bug-bounty/)
- [mynxt.info](https://cobalt.io/mynxt-info)
- [NCSC](mailto:cert@ncsc.nl)
- [Nearby Live](https://hackerone.com/nearby)
- [Nest](mailto:security@nest.com)
- [Netflix](mailto:security-report@netflix.com)
- [Neverdie Smart Contract](https://hackenproof.com/neverdie/neverdie-smart-contract)
- [Neverdie Web](https://hackenproof.com/neverdie/neverdie-web)
- [Nexmo](https://cobalt.io/nexmo)
- [Nexuzhealth](https://go.intigriti.com/nexushealth)
- [Nexuzhealth Web PACS](https://go.intigriti.com/nexuzhealthwebpacs)
- [Nginx](https://hackerone.com/ibb-nginx)
- [Nitrous](mailto:security@nitrous.io)
- [Nokia Networks](mailto:security-alert@nokia.com)
- [NoPass](https://cobalt.io/nopass)
- [NZRS](mailto:security@nzrs.net.nz)
- [Offensive Security](mailto:security@offensive-security.com)
- [ok.ru](https://hackerone.com/ok)
- [OKCoin](https://cobalt.io/okcoin)
- [OkCupid](https://hackerone.com/okcupid)
- [Olark](mailto:security@olark.com)
- [OneSpan Mobile](https://go.intigriti.com/vascomobileproducts)
- [OneSpan Server Products](https://go.intigriti.com/vascoserver-sideproducts)
- [Opal Cryptocurrency](https://cobalt.io/opal-cryptocurrency)
- [Openfolio](https://hackerone.com/openfolio)
- [OpenSSL](https://hackerone.com/ibb-openssl)
- [OpenStack](https://security.openstack.org/#how-to-report-security-issues-to-openstack)
- [OpenText](mailto:otst@opentext.com)
- [Opera](https://bugs.opera.com/wizarddesktop)
- [Optimizely](https://cobalt.io/optimizely)
- [Oracle](mailto:secalert_us@oracle.com)
- [ownCloud](https://hackerone.com/owncloud)
- [PagerDuty](mailto:security@pagerduty.com)
- [Panasonic Avionics](https://hackerone.com/panasonic-aero)
- [Pantheon](https://bugcrowd.com/pantheon)
- [Panzura](mailto:security@panzura.com)
- [Paragon Initiative Enterprises](https://hackerone.com/paragonie)
- [Paychoice](mailto:security@paychoice.com.au)
- [PayMill](mailto:security@paymill.com)
- [PayPal](mailto:https://www.paypal.com/bugbounty/register)
- [Paytm](https://bugbounty.paytm.com/)
- [Perl](https://hackerone.com/ibb-perl)
- [Phabricator](https://hackerone.com/phabricator)
- [PHP](https://bugs.php.net/report.php)
- [Pidgin](mailto:security@pidgin.im)
- [PikaPay](mailto:security@pikapay.com)
- [PinoyHackNews](mailto:admin@pinoyhacknews.com)
- [Pinterest](https://bugcrowd.com/pinterest)
- [Piwik Open Source Analytics](https://cobalt.io/piwik-open-source-analytics)
- [Plone](mailto:security@plone.org)
- [Pocket](mailto:security@getpocket.com)
- [Poloniex](https://cobalt.io/poloniex)
- [Postmark](https://wildbit.wufoo.com/forms/wildbit-security-response)
- [Prezi](mailto:security-bug-bounty@prezi.com)
- [Projectplace](https://hackerone.com/projectplace)
- [PullReview](mailto:security@pullreview.com)
- [Puppet labs](mailto:security@puppetlabs.com)
- [PureVPN](https://bugcrowd.com/purevpn)
- [Python](mailto:security@python.org)
- [QIWI](https://hackerone.com/qiwi)
- [Quadriga CX](https://cobalt.io/quadriga-cx)
- [QuickBT](https://cobalt.io/quickbt)
- [Quora](https://hackerone.com/quora)
- [Rackspace](mailto:security@rackspace.com)
- [Rdbhost_service](https://cobalt.io/rdbhost-service)
- [Red Hat](mailto:site-security@redhat.com)
- [Reddit](mailto:security@reddit.com)
- [Relaso](mailto:security@relaso.com)
- [RelateIQ](mailto:security@relateiq.com)
- [Release Wire](http://www.releasewire.com/about/contact)
- [Respondly](https://hackerone.com/respondly)
- [Revive Adserver](https://hackerone.com/revive_adserver)
- [Ribose](https://www.ribose.com/feedbacks/security)
- [Ripio](https://cobalt.io/ripio)
- [Ripple](mailto:bugs@ripple.com)
- [Riskalyze](mailto:security@riskalyze.com)
- [Romit](https://hackerone.com/romit)
- [Ruby](mailto:security@ruby-lang.org)
- [Ruby on Rails](https://hackerone.com/rails)
- [Salesforce](mailto:security@salesforce.com)
- [Samsung TV](https://samsungtvbounty.com/ReportBug.aspx)
- [Sandbox Escape](https://hackerone.com/sandbox)
- [SAP](mailto:secure@sap.com)
- [Schuberg Philis](mailto:abuse@schubergphilis.com)
- [Scorpion Software](mailto:security@scorpionsoft.com)
- [Secret](https://hackerone.com/secret)
- [Secure Works](mailto:security@secureworks.com)
- [Sellfy](http://docs.sellfy.com/contact)
- [Sentiance](https://go.intigriti.com/sentiance)
- [ServiceRocket](https://bugcrowd.com/servicerocket)
- [ShareLaTeX](mailto:team@sharelatex.com)
- [Sherpany](https://cobalt.io/sherpany)
- [Shopify](https://hackerone.com/shopify)
- [Sifter](mailto:security@sifterapp.com?subject=%27Security%20Vulnerability%20Report%27)
- [Silent Circle](https://bugcrowd.com/silentcircle)
- [Simple](https://bugcrowd.com/simple)
- [SiteGround](mailto:responsible-disclosure@siteground.com)
- [Skoodat](mailto:security@skoodat.com)
- [Skrill](https://cobalt.io/skrill)
- [Skyscanner](https://bugcrowd.com/skyscanner)
- [Slack](https://hackerone.com/slack)
- [Snapchat](https://hackerone.com/snapchat)
- [Snappy](mailto:security@userscape.com)
- [Sonatype](mailto:security@sonatype.com)
- [Sony](https://secure.sony.net/form)
- [SoundCloud](https://scsecurity.freshdesk.com/support/tickets/new)
- [Spaargids](https://go.intigriti.com/spaargids)
- [SpectroCoin](https://cobalt.io/spectrocoin)
- [Spendbitcoins](https://cobalt.io/spendbitcoins)
- [SplashID](https://bugcrowd.com/splashid)
- [Splitwise](mailto:security@splitwise.com)
- [Spotify](mailto:security@spotify.com)
- [Sprout Social](mailto:security@sproutsocial.com)
- [Square](https://hackerone.com/square)
- [Square Open Source](https://hackerone.com/square-open-source)
- [StatusPage](https://bugcrowd.com/sunrise)
- [StopTheHacker](https://hackerone.com/stopthehacker)
- [Student Assessment System](https://go.intigriti.com/printscan)
- [Studio 100](https://go.intigriti.com/studio100)
- [Subledger](https://cobalt.io/subledger)
- [Subrosa](https://cobalt.io/subrosa)
- [Sucuri](https://hackerone.com/sucuri)
- [Suivo](https://go.intigriti.com/suivoweb)
- [Symantec](mailto:secure@symantec.com)
- [Taptalk](https://hackerone.com/taptalk)
- [Tarsnap](mailto:cperciva@tarsnap.com)
- [Tata Play](https://www.tataplay.com/bug-bounty-hunter)
- [TeamUnify](mailto:security@teamunify.com)
- [Tele2](mailto:beveiligingsmeldpunt@tele2.com)
- [Telekom](mailto:cert@telekom.de?subject=bug_bounty)
- [Telenet](https://go.intigriti.com/telenet)
- [Test-Aankoop](https://go.intigriti.com/testaankoop)
- [The Internet](https://hackerone.com/internet)
- [The Mastercoin Foundation](https://cobalt.io/the-mastercoin-foundation)
- [ThisData](https://hackerone.com/thisdata)
- [TimeTrex](https://cobalt.io/timetrex)
- [ToyTalk](https://hackerone.com/toytalk)
- [Trello](https://hackerone.com/trello)
- [Tuenti](http://corporate.tuenti.com/en/contact/security)
- [Tweakers](https://go.intigriti.com/tweakers)
- [Twilio](https://bugcrowd.com/twilio)
- [Twitch](mailto:security@twitch.tv)
- [Twitter](https://hackerone.com/twitter)
- [Uber](mailto:security-abuse@uber.com)
- [Ubiquiti Networks](https://hackerone.com/ubnt)
- [Unitag](mailto:security@unitag.io)
- [Urban Dictionary](https://hackerone.com/urbandictionary)
- [Uzbey](https://hackerone.com/uzbey)
- [Valve Software](mailto:security@valvesoftware.com)
- [VeChainThor](https://hackenproof.com/vechain/vechainthor)
- [VeChainThor Wallet](https://hackenproof.com/vechain/vechainthor-wallet)
- [VCE](mailto:security-alerts@vce.com)
- [Venmo](mailto:security@venmo.com)
- [Version Cake](https://hackerone.com/versioncake)
- [Viadeo](mailto:security@viadeo.com)
- [Vimeo](https://hackerone.com/vimeo)
- [VK.com](https://hackerone.com/vkcom)
- [Volusion](https://bugcrowd.com/volusion)
- [VPNSox](https://cobalt.io/vpnsox)
- [vulners.com](https://hackerone.com/vulnerscom)
- [Vultr](https://www.vultr.com/bug-bounty/)
- [Webconverger](mailto:security@webconverger.com)
- [Websecurify](http://campaigns.websecurify.com/money-for-bugs/#contact)
- [Weebly](https://cobalt.io/weebly)
- [WePay](https://hackerone.com/wepay)
- [Whisper](https://hackerone.com/whisper)
- [WHMCS](https://bugcrowd.com/whmcs)
- [Windthorst ISD](http://www.windthorstisd.net/BugReport.cfm)
- [withinsecurity](https://hackerone.com/withinsecurity)
- [WizeHive](mailto:security@wizehive.com)
- [Woorank](https://go.intigriti.com/woorank)
- [WordPoints](https://hackerone.com/wordpoints)
- [Wordware](https://cobalt.io/wordware)
- [WP API](https://hackerone.com/wp-api)
- [Xen Project](mailto:security@xenproject.org)
- [Xmarks](mailto:security@lastpass.com)
- [Yahoo](https://hackerone.com/yahoo)
- [Yandex](https://yandex.com/bugbounty/report)
- [Yanomo](mailto:support@yanomo.com)
- [Yatra](https://www.yatra.com/online/bug-bounty)
- [Yesware](mailto:security@yesware.com)
- [Zapier](mailto:security@zapier.com)
- [Zaption](https://hackerone.com/zaption)
- [ZenCash](mailto:security@zencash.com)
- [Zendesk](https://hackerone.com/zendesk)
- [Zetetic](mailto:support@zetetic.net)
- [Ziggo](mailto:security@ziggo.nl)
- [Zimbra](mailto:security@zimbra.com)
- [Zoho](https://bugbounty.zoho.com/bb/info)
- [Zomato](https://hackerone.com/zomato)
- [Zopim](https://hackerone.com/zopim)
- [Zynga](mailto:whitehat@zynga.com)
- [Coding Ninjas](https://www.codingninjas.com/bug-bounty)