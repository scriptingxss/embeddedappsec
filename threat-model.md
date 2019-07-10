# Threat Modeling

## Threat Modeling

Threat modeling is an exercise that helps with quantifying threats to understand how attackers \(threat actors\) may be able to compromise a system and then make the appropriate mitigations to thwart the potential risks posed. Typically, threat modeling is an exercise that takes place before deployment to production systems as part of the design phase but can also be used in the beginning stages of any security testing. Threat modeling will usually include the following activities:

1. Identifying all assets in a system, creating an architecture overview
2. Decomposing the system \(or device\)
3. Identification of threats
4. Document all the threats with their respective scenarios, and 
5. Rate each threat by its likelihood as well as impact using a rating system

A framework used to help enumerate and identify threats is [STRIDE](https://en.wikipedia.org/wiki/STRIDE_%28security%29), which is a mnemonic described in the following table:

|  | Threat | Property Violated | Threat Definition | Examples |
| :--- | :--- | :--- | :--- | :--- |
| **S** | Spoofing | Authenticity | Pretending to be someone or something you are not | Impersonating another user |
| **T** | Tampering | Integrity | Modifying data or code | Deleting log files to conceal activity |
| **R** | Repudiation | Non-reputability | Claiming to have not performed an action. | I never executed that DROP TABLE in production |
| **I** | Information disclosure | Confidentiality | Exposing information to someone not authorized to see it | Hardcoded usernames and passwords in source code |
| **D** | Denial of Service | Availability | Deny, limit, or degrade service to users | Exhaust server side infrastructure with requests |
| **E** | Elevation of Privilege | Authorization | Gain capabilities without proper authorization | Gaining admin privileges |

Common risk rating systems used in threat modeling are [DREAD](https://en.wikipedia.org/wiki/DREAD_%28risk_assessment_model%29), and [CVSS](https://www.first.org/cvss/) but several others are also available. DREAD, another mnemonic, is scored on a scale of 1 to 3 according to each category described below and a final score is the average. 1 is low risk, 2 is medium risk, and 3 is high risk. Often, the scoring scale is modified to fit the need of an organization, such as 0 to 10. 

|  | Name | Description | High | Medium | Low |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **D** | Damage | How bad would an attack be? | Can subvert all security controls and get full trust to take over the whole  ecosystem. | Could leak sensitive information. | Could leak trivial information. |
| **R** | Reproducibility | How easy is it to reproduce the attack? | The attack is always reproducible. | The attack can be reproduced only within a timed window or specific condition. | It's very difficult to reproduce the attack, even with specific information about the vulnerability. |
| **E** | Exploitability | How much work is it to launch the attack? | A novice attacker could execute the exploit. | A skilled attacker could make the attack repeatedly. | Allows a skilled attacker with in- depth knowledge to perform the attack. |
| **A** | Affected users | How many people or users will be impacted by a successful attack? | All users, default configurations, all devices. | Affects some users, some devices, and custom configurations. | Affects a small percentage of users and/or devices through an obscure feature. |
| **D** | Discoverability | How easy is it to discover the threat? | Attack explanation can be easily found in a publication. | Affects a seldom-used feature where an attacker would need to be very creative to discover a malicious use for it. | Is obscure and unlikely an attacker would discover a way to exploit the bug. |

Threat models should answer the following four questions:

* What are we building?
  * Use data flow diagrams \(DFD\) to assist with modeling components and how they interact locally as well as with external services
  * DFDs should show each process, user, entity, data store, and protocols that connect assets
  * Make use of tools such as Microsoft Threat Modeling Tool 2016, OWASP Threat Dragon, or online diagram software such as [https://draw.io/](https://draw.io/) or [https://www.lucidchart.com](https://www.lucidchart.com).
* What can go wrong?
  * Leverage STRIDE to help identify and enumerate threats
* What are we going to do about the issues that can go wrong?
  * DREAD aids with risk scoring and prioritization. The higher the risk, the higher priority the threat should be addressed or mitigated. 
* How well was our analysis?
  * Conduct a retrospective activity to check the overall quality, progress, and future planning. 

Threat modeling should be done **early, and as often as possible**. Threat model owners are best in the hands of the software teams and should considered a **living document** that is updated as new features are planned. 

A great book and also an authoritative reference is "[Threat Modeling: Designing for Security](https://www.amazon.com/Threat-Modeling-Designing-Adam-Shostack/dp/1118809998)". 

### Additional References <a id="additional-references"></a>

* [OWASP Application Threat Modeling](https://www.owasp.org/index.php/Application_Threat_Modeling)
* [OWASP Threat Dragon](https://github.com/mike-goodwin/owasp-threat-dragon-desktop)
* [Microsoft Threat Modeling Tool 2016](https://www.microsoft.com/en-us/download/details.aspx?id=49168)

