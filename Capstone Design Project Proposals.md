# Capstone Design Project Proposals

## Project Idea: Advanced Anti-Phishing Chrome Extension with DNS-Level Protection

**Problem Statement:** Phishing attacks continue to be one of the most prevalent cybersecurity threats, with over 3.4 billion phishing emails sent daily and a 61% increase in phishing attempts in 2023. Current browser-based phishing protection relies primarily on blacklist approaches that are reactive rather than proactive, often missing newly created phishing domains and zero-day campaigns. Traditional email security solutions operate at the server level, leaving end-users vulnerable when accessing personal email accounts through web browsers. Organizations lack comprehensive visibility into phishing attempts targeting their employees through personal email accounts, creating blind spots in their security monitoring. The gap between enterprise-grade threat detection and consumer-level browser protection creates a critical vulnerability that sophisticated phishing campaigns exploit.

**Proposed Solution:** This project will develop a multi-layered anti-phishing system combining a Chrome extension for real-time email protection with DNS-level blocking via Response Policy Zones (RPZ) and comprehensive SIEM integration. The Chrome extension will analyze URLs in real-time as users interact with email interfaces (Gmail, Outlook, Yahoo Mail), cross-referencing detected domains against threat intelligence feeds from multiple sources including PhishFort, URLhaus, and custom-maintained blacklists. A dedicated DNS server with RPZ configuration will provide network-level blocking of malicious domains, while automated scripts will update threat intelligence hourly from GitHub repositories and security feeds. All detection events will be logged to both primary SIEM systems and backup storage for comprehensive security monitoring. User research with 50+ participants will validate the system's effectiveness and usability through controlled phishing simulations and real-world deployment scenarios.

**Scope:**
* Chrome extension development using Manifest V3 with real-time URL analysis for major email providers
* DNS server deployment with BIND9 and RPZ configuration for network-level domain blocking
* Integration with at least 3 threat intelligence feeds with automated hourly updates
* SIEM logging implementation with both primary and backup event storage
* User research study with minimum 50 participants across 3 demographic groups
* Performance optimization to maintain sub-200ms response times for URL analysis
* Development of machine learning model for enhanced phishing detection accuracy
* Chrome Web Store publication preparation with comprehensive documentation

**What will NOT be included:**
* Mobile browser extension development (iOS Safari, Android Chrome)
* Integration with email clients outside of web browsers (Outlook desktop, Thunderbird)
* Advanced natural language processing for email content analysis
* Enterprise deployment automation or centralized management console
* Support for browsers other than Chrome/Chromium-based browsers

**References:**
Proofpoint. (2024). State of the Phish 2024 Report. Retrieved from https://www.proofpoint.com/us/resources/threat-reports/state-of-phish  
APWG. (2024). Phishing Activity Trends Report Q1 2024. Anti-Phishing Working Group.  
Verizon. (2024). Data Breach Investigations Report. Retrieved from https://www.verizon.com/business/resources/reports/dbir/  
PhishFort Lists. (2024). GitHub Repository. Retrieved from https://github.com/phishfort/phishfort-lists  
Internet Systems Consortium. (2024). Response Policy Zones (RPZ). Retrieved from https://www.isc.org/rpz/
