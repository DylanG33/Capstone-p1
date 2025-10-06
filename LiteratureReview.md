# Literature Review, Why These Sources Matter
To understand and combat phishing threats effectively, I need a combination of industry reports, real-time data, and technical resources. These five sources provide both the current state of phishing attacks and the tools needed to build defensive systems. The Proofpoint and APWG reports give me statistical evidence of the problem's scope, Verizon's DBIR contextualizes phishing within broader cybersecurity incidents, while PhishFort and ISC provide the technical foundation for implementing protective measures.

## Proofpoint - State of the Phish 2024 Report (https://www.proofpoint.com/us/resources/threat-reports/state-of-phishAPWG)

- Surveyed thousands of working adults and infosec professionals to understand how people interact with phishing attempts
- Shows that despite increased awareness training, click rates on malicious links remain concerning
- Breaks down attack vectors, with email as the primary delivery method but SMS and voice phishing are growing
- Provides crucial data on who gets targeted and how attacks succeed
- Highlights that many employees still fall for social engineering tactics

## APWG - Phishing Activity Trends Report Q1 2024

- Tracks phishing attacks across the internet with quarterly statistics on attack volumes and targeted industries
- Q1 2024 report shows phishing attacks hitting record numbers
- Financial services and SaaS platforms are the most impersonated brands
- Documents the lifecycle of phishing sites and how quickly they're stood up and taken down
- Gives the macro view of phishing activity and identifies which sectors face the most risk

## Verizon - Data Breach Investigations Report 2024 (https://www.verizon.com/business/resources/reports/dbir/PhishFort)

- One of the most cited security reports in the industry, analyzing thousands of real-world breaches
- Shows phishing is involved in a large percentage of breaches, often serving as the initial access point
- Connects phishing to actual business impact—not just clicks, but compromised credentials and data exfiltration
- Demonstrates the financial losses and real-world consequences of successful phishing attacks
- Provides evidence that phishing isn't just an annoyance but a serious security threat

## PhishFort Lists ([GitHub Repository](https://github.com/phishfort/phishfort-listsInternet)) 

- Maintains community-curated lists of known phishing domains, URLs, and indicators of compromise
- Open-source and regularly updated by security researchers
- Lists are formatted for easy integration into security tools and DNS filtering systems
- Provides actual blocklist data needed to implement phishing protection
- Actively maintained, meaning it contains current threat intelligence rather than outdated information

## Internet Systems Consortium - Response Policy Zones ([RPZ](https://www.isc.org/rpz/))

- Explains how DNS-based blocking works at a technical level
- Shows how RPZ allows DNS servers to return modified responses for known malicious domains
- Blocks access before users even reach a phishing site
- Provides implementation guidelines and best practices for deploying RPZ in enterprise environments
- Essential technical documentation for understanding the mechanism behind DNS-based phishing protection
