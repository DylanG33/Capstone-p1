# Methodology Draft

## Overview

A controlled before-and-after comparison study will be conducted to evaluate the effectiveness of DNS-based phishing protection using Response Policy Zones (RPZ). This experiment will measure whether implementing RPZ with phishing blocklists can successfully prevent access to known phishing domains while maintaining normal access to legitimate websites. The following sections provide an outline of the methodology that will be expanded and refined during subsequent progress reports.

## Experimental Design

The experiment follows a classic pretest–posttest design where measurements are taken before and after implementing the DNS-based blocking system. This allows for direct comparison of the system's effectiveness by observing changes in phishing site accessibility and overall network performance.

### Experimental Flow Diagram

```
[Pretest Phase]
    |
    |--> Setup DNS Server (no RPZ)
    |--> Configure Client VMs
    |--> Test phishing site access (baseline)
    |--> Test legitimate site access
    |--> Measure DNS performance
    |--> Record all data
    |
    v
[Treatment Phase]
    |
    |--> Download PhishFort blocklists
    |--> Configure RPZ on BIND9
    |--> Import phishing domains to RPZ zone
    |--> Enable blocking mechanism
    |--> Verify blocking works
    |
    v
[Post-test Phase]
    |
    |--> Retest phishing site access (should be blocked)
    |--> Retest legitimate site access (should work)
    |--> Remeasure DNS performance
    |--> Collect user feedback
    |--> Compare pretest vs post-test data
    |
    v
[Analysis & Reporting]
```

## Experimental Environment

A controlled network environment will be established using virtual machines to safely test phishing protection mechanisms:

### Network Topology

```
                    [School Network]
                          |
         _________________|_________________
        |                 |                 |
        |                 |                 |
   [DNS Server VM]   [Client VM 1]    [Client VM 2]
   Ubuntu Server     Windows 10/11    Windows 10/11
   BIND9 + RPZ       Chrome/Firefox   Chrome/Firefox/Edge
   Static IP         |                |
        |            |                |
        |____________|________________|
                     |
              DNS Queries Flow
                Through Here
                     |
                     v
           [Internet/Upstream DNS]
           Google DNS (8.8.8.8)
           Cloudflare (1.1.1.1)


[On-Campus Friend 1]  [On-Campus Friend 2]
Personal Laptop       Personal Desktop
Configured to use --> [DNS Server VM]
test DNS server
```

### DNS Server Setup

A virtual machine will be used to host the DNS server implementation:

- **Virtual Machine**: A VM will be borrowed from the school's resources to host the DNS server. The specific hypervisor platform (VMware, VirtualBox, Proxmox, etc.) will be documented once access is obtained.
- **Operating System**: Ubuntu Server LTS (latest stable version) will be installed on the VM. The exact version will be specified during implementation.
- **Resource Allocation**: Initial estimates suggest 2-4 CPU cores, 4-8GB RAM, and 20-50GB storage will be sufficient. These specifications will be adjusted based on performance during testing.
- **Network Configuration**: The VM will be configured with a static IP address on the school network to allow client connections. Details on network bridging or NAT configuration will be provided in progress reports.

### Understanding BIND9 and DNS-Based Blocking

BIND9 (Berkeley Internet Name Domain version 9) is DNS server software that translates website names into IP addresses. It acts like a phone book for the internet, when you type "google.com" into your browser, BIND9 looks up and returns Google's actual IP address so your computer knows where to connect.

#### How Normal DNS Works

1. **You type a URL**: You enter "amazon.com" in your browser
2. **DNS query**: Your computer asks a DNS server "What's the IP address for amazon.com?"
3. **DNS response**: The DNS server responds with the IP address
4. **Connection**: Your browser uses that IP address to connect to Amazon's servers

#### Why BIND9 is Needed for This Project

BIND9 supports Response Policy Zones (RPZ), which is critical for implementing DNS-based phishing protection. RPZ allows you to create custom rules for how DNS queries are answered, you can tell BIND9 "If someone asks for badphishingsite.com, don't give them the real IP, block it instead."

#### How BIND9 + RPZ Blocks Phishing Sites

The blocking workflow works as follows:

1. **Normal DNS query**: User's browser asks the BIND9 server "What's the IP for legitbank-login.phishing.com?"
2. **RPZ check**: Before responding, BIND9 checks its RPZ zone file (the blocklist)
3. **Match found**: The domain is on the phishing blocklist
4. **Block response**: Instead of returning the real IP address, BIND9 returns NXDOMAIN (pretends the site doesn't exist), a redirect to a warning page, or a localhost address that goes nowhere
5. **Result**: The user never reaches the phishing site

#### System Architecture

```
[Client VM] ---DNS Query---> [BIND9 Server with RPZ] ---Check Blocklist--->
                                      |
                                      |
                              Is domain blocked?
                                      |
                        Yes /                  \ No
                           /                    \
                    Return NXDOMAIN         Forward query to
                    or warning page         upstream DNS, return
                                           real IP address
```

## Virtual Machine Requirements

### VM Specifications Request

For this project, three virtual machines will be requested from the school's Proxmox infrastructure to create a testing environment for DNS-based phishing protection.

#### VM #1: DNS Server (Ubuntu Server)

**Purpose**: Host the BIND9 DNS server with RPZ configuration for phishing domain blocking

**Specifications**:
- **Operating System**: Ubuntu Server 22.04 LTS or newer
- **CPU**: 2-4 vCPUs
- **RAM**: 4GB
- **Storage**: 20GB
- **Network**: 1 network interface with static IP capability

**Justification**: The DNS server is the core of this project and needs sufficient resources to handle DNS queries from multiple clients, maintain large blocklists in memory, and log all DNS activity. BIND9 is relatively lightweight, but enough RAM ensures smooth operation when loading thousands of phishing domains into the RPZ zone. The storage requirement accounts for the OS, BIND9 installation, blocklist files, and DNS query logs generated during testing.

#### VM #2: Client Machine 1 (Windows 10/11)

**Purpose**: Primary test client for simulating end-user browsing behavior and testing phishing site blocking

**Specifications**:
- **Operating System**: Windows 10 or Windows 11
- **CPU**: 2 vCPUs
- **RAM**: 4GB
- **Storage**: 40GB
- **Network**: 1 network interface configured to use DNS Server VM

**Justification**: This VM simulates a typical user workstation and will be used to test whether phishing sites are successfully blocked while legitimate sites remain accessible. The 40GB storage accommodates the Windows OS and multiple browser installations.

#### VM #3: Client Machine 2 (Windows 10/11)

**Purpose**: Secondary test client for validating consistency of DNS blocking across multiple machines

**Specifications**:
- **Operating System**: Windows 10 or Windows 11
- **CPU**: 2 vCPUs
- **RAM**: 4GB
- **Storage**: 40GB
- **Network**: 1 network interface configured to use DNS Server VM

**Justification**: Having a second client VM is essential for verifying that DNS blocking works consistently across different machines and isn't dependent on a single client configuration. This VM will run the same tests as Client Machine 1 to ensure reproducible results. Testing from multiple clients also simulates a more realistic scenario where a DNS server handles requests from multiple users simultaneously, allowing for performance and scalability assessment.

#### Network Configuration Requirements

All three VMs should be placed on the same network segment within the Proxmox environment to ensure:
- Low latency communication between clients and DNS server
- Easy configuration of static IP addressing
- Isolated testing environment 
- Ability for the DNS server to forward queries to external upstream DNS servers 

### Blocking Example Comparison

**Without RPZ (Pretest):**
```
User: "What's the IP for evil-phishing-site.com?"
BIND9: "It's 192.168.1.100"
User: *visits phishing site*
```

**With RPZ (Post-test):**
```
User: "What's the IP for evil-phishing-site.com?"
BIND9: *checks blocklist* "That domain doesn't exist (NXDOMAIN)"
User: *gets error page, can't visit phishing site*
```

### DNS Server Software Installation

The process of setting up the DNS server will follow these general steps:

- Install BIND9 on the Ubuntu Server VM
- Configure basic DNS functionality and verify it can resolve standard domain queries
- Configure BIND9 to forward queries for legitimate sites to upstream DNS servers (like Google DNS or Cloudflare)
- Secure the DNS server with appropriate firewall rules and access controls
- Test basic DNS resolution before implementing RPZ to ensure the foundation is working
- Document all configuration files and settings for reproducibility

#### BIND9 Configuration Components

The BIND9 setup will consist of several key components:

1. **Main Config File** (`named.conf`): Tells BIND9 how to operate, what zones to load, and where to find RPZ rules
2. **RPZ Zone File**: Contains the actual blocklist and the phishing domains to block
3. **Logging Configuration**: Records all DNS queries to track what's being blocked
4. **Upstream DNS Settings**: Configures which external DNS servers BIND9 forwards legitimate queries to

Specific commands, configuration file examples, and troubleshooting steps will be detailed as the implementation progresses.

### Client Machine Setup

Multiple client VMs will be used to simulate end-user environments:

- **Client VMs**: Additional virtual machines will be borrowed from the school to serve as test clients
- **Operating Systems**: Windows 10/11 will be installed on client VMs to represent typical user environments
- **Browser Configuration**: Multiple browsers (Chrome, Firefox, Edge) will be installed for cross-browser testing
- **DNS Configuration**: Client machines will be configured to point to the test DNS server for all domain resolution
- **Network Isolation**: All VMs (DNS server and clients) will be on the same school network segment to minimize latency and ensure connectivity

The number of client VMs and their specific configurations will be determined based on availability from the school and documented in progress reports.

## Data Collection Sources

### Blocklist Integration Process

```
[PhishFort GitHub Repo]
    |
    | Download phishing domain lists
    v
[Raw Data Files]
    | .txt or .csv format
    | Contains thousands of phishing domains
    v
[Processing Script]
    | Parse and format domains
    | Remove duplicates
    | Validate domain format
    v
[RPZ Zone File Format]
    | evil-site.com CNAME .
    | phishing.com CNAME .
    | fake-bank.com CNAME .
    v
[Import to BIND9]
    | Load into RPZ configuration
    | Activate blocking rules
    v
[Active Protection]
```

Three categories of domain lists will be compiled for testing:

* **Phishing Blocklists**: PhishFort Lists from GitHub will serve as the primary source of known phishing domains. Additional blocklists may be incorporated if needed for comprehensive coverage.
* **Test Phishing Domains**: A curated collection of verified phishing URLs (already taken down or sandboxed) will be used for safe testing without exposing systems to active threats.
* **Legitimate Domain Baseline**: A list of 50-100 popular legitimate websites (e.g., Google, Amazon, banking sites, news outlets) will be compiled to test for false positives.

The exact number of domains in each category and the selection criteria will be detailed in progress reports.

## Pretest Phase

Before implementing any blocking mechanisms, baseline measurements will be established:

### Environment Setup
- Install and configure the DNS server on the borrowed VM without RPZ enabled
- Set up client VMs and configure them to use the test DNS server
- Verify connectivity and proper DNS resolution across all client machines
- Document initial system configurations and network settings

### Baseline Measurements
- Test access to the curated list of known phishing domains from client VMs to confirm they are currently reachable (using safe, inactive phishing URLs)
- Measure and record DNS resolution times for both legitimate and phishing domains
- Document the success rate of accessing each category of websites from multiple client machines
- Record baseline network latency and performance metrics
- Verify that all legitimate websites in the test set load correctly without issues across different browsers

### Data Recording
- Create spreadsheets or databases to log all test results from each client VM
- Establish a standardized testing procedure that can be replicated in the post-test phase
- Take screenshots or recordings of access attempts for documentation
- Test from multiple client VMs to ensure consistency

The specific tools used for measurement (e.g., dig, nslookup, custom scripts) will be determined and documented during implementation.

## Treatment Phase

The treatment consists of implementing and activating RPZ-based phishing protection:

### RPZ Configuration
- Download and import phishing domain lists from PhishFort into a usable format
- Configure the DNS server to implement Response Policy Zones according to ISC guidelines
- Create RPZ zone files that include all phishing domains from the blocklists
- Set the appropriate action for blocked domains (NXDOMAIN, redirect to warning page, etc.)

### Policy Implementation
- Apply the RPZ policy to the DNS server configuration
- Restart or reload DNS services to activate the blocking mechanism
- Verify that RPZ is functioning by testing a small sample of blocked domains from client VMs

### Logging and Monitoring
- Enable DNS query logging to track blocked requests from all client machines
- Set up monitoring to capture both successful blocks and any potential false positives
- Configure alerts for unusual DNS activity or system performance issues

The exact RPZ configuration syntax, file formats, and implementation details will be provided in subsequent progress reports as they are developed.

## Post-hoc Test Phase

Following the implementation of RPZ filtering, the same tests conducted in the pretest will be repeated:

### Replication of Pretest
- Attempt to access the same set of phishing domains tested during the pretest from all client VMs
- Test all legitimate websites to identify any that are incorrectly blocked
- Measure DNS resolution times with RPZ active across different client machines
- Document any error messages or blocking behaviors encountered
- Test across multiple browsers on each client VM to ensure consistent blocking

### Effectiveness Measurements
- Calculate the percentage of phishing domains successfully blocked
- Identify any phishing sites that bypassed the filter
- Document false positive rate (legitimate sites blocked incorrectly)
- Compare pre and post-treatment DNS resolution speeds
- Assess overall impact on browsing experience from the client VMs

### On-Campus User Testing
- Two friends on campus will be asked to configure their personal devices to use the test DNS server
- They will use the DNS-based blocking during their normal browsing activities for a defined testing period
- Feedback will be collected through informal interviews or surveys regarding their experience
- Any issues, false positives, or performance concerns will be documented
- This real-world usage will provide qualitative insights beyond the controlled VM testing

### Data Analysis
- Compare pretest and post-test results to quantify the effectiveness of RPZ
- Identify patterns in blocked vs. unblocked domains
- Analyze any performance degradation caused by RPZ implementation
- Incorporate qualitative feedback from the two on-campus testers into the evaluation
- Compare results across different client VMs and browsers to identify any inconsistencies

## Evaluation Metrics

The success of the RPZ implementation will be measured using the following criteria:

- **Block Rate**: Percentage of known phishing domains successfully blocked (target: >95%)
- **False Positive Rate**: Percentage of legitimate sites incorrectly blocked (target: <1%)
- **DNS Resolution Time**: Average time to resolve domain names, comparing before and after RPZ implementation
- **System Performance**: Overall impact on network speed and browsing experience across client VMs
- **Cross-Browser Consistency**: Verification that blocking works uniformly across Chrome, Firefox, and Edge
- **User Feedback**: Qualitative assessment from on-campus testers regarding usability and effectiveness
- **Scalability**: Assessment of whether the solution can handle multiple simultaneous client connections and the full blocklist without performance issues

## Limitations and Considerations

Several limitations are:

- The experiment uses known phishing domains rather than zero-day phishing sites
- Testing is conducted in a controlled VM environment on the school network
- The blocklist used may not include all possible phishing domains
- Real-world user testing is limited to two on-campus volunteers
- VM performance and network conditions may differ from production environments
- Testing period for on-campus users will be limited by the semester timeline

These limitations will be addressed in more detail in the final methodology and discussion sections.
