# Progress Report 2

## Summary

This week, I have researched the technical implementation of BIND9 DNS servers with Response Policy Zones and identified available phishing domain blocklists:

* Studied Internet Systems Consortium's RPZ documentation detailing configuration syntax, zone file formats, and policy response types
* Analyzed BIND9 architecture and confirmed RPZ support in current stable releases for DNS query interception
* Evaluated PhishFort Lists GitHub repository as primary blocklist source with regularly updated phishing domains
* Researched RPZ deployment best practices including upstream DNS forwarding, logging configurations, and performance optimization techniques
* Compared alternative DNS filtering solutions like Pi-hole and DNSFilter to validate BIND9+RPZ as appropriate academic research platform

## Documentation

1. Internet Systems Consortium. (2024). Response Policy Zones (RPZ). https://www.isc.org/rpz/
2. PhishFort Lists. (2024). GitHub Repository. https://github.com/phishfort/phishfort-lists
3. BIND9 Administrator Reference Manual sections on RPZ configuration
4. Comparative analysis document: BIND9 vs alternative DNS filtering platforms for research purposes
