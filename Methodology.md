# Methodology - Multi-Layered Anti-Phishing System

## Overview

This project implements a three-layer anti-phishing protection system combining browser-level detection, network-level blocking, and centralized security monitoring. The system uses a Chrome extension for real-time email protection, DNS-based blocking using Response Policy Zones (RPZ), and SIEM logging for security event management. A controlled before-and-after comparison study will evaluate the effectiveness of each protection layer independently.
## Experimental Design

The experiment follows a classic pretest-posttest design with three distinct protection layers that can be evaluated both independently and as an integrated system. This methodology allows for measurement of each layer's effectiveness while assessing the overall protection coverage of the multi-layered approach.

### Three-Layer Architecture

**Layer 1: Browser-Level Protection (Chrome Extension)**
- Real-time URL analysis in web email interfaces (Gmail)
- Local PhishFort blocklist cache for fast lookups
- Google Safe Browsing API integration for ML-enhanced detection
- Visual warning system to alert users before clicking malicious links
- Event logging to SIEM for monitoring and analysis

**Layer 2: Network-Level Protection (DNS/RPZ)**
- BIND9 DNS server with Response Policy Zones configuration
- PhishFort blocklists converted to RPZ zone format
- Network-wide domain blocking (catches bypass attempts)
- Secondary defense layer if users ignore browser warnings
- Event logging to SIEM for comprehensive coverage tracking

**Layer 3: Monitoring and Analysis (SIEM)**
- Syslog-based logging infrastructure (with potential ELK Stack upgrade for visualization)
- Aggregates detection events from both Chrome extension and DNS server
- Correlates phishing attempts across protection layers
- Provides results on effectiveness of each layer
- Dashboard visualization of threat patterns and system performance

### Experimental Flow

```
[Pretest Phase - Baseline]
    |
    |--> Setup infrastructure (3 VMs, Chrome extension, SIEM)
    |--> Configure DNS server WITHOUT RPZ blocking
    |--> Deploy Chrome extension WITHOUT detection enabled
    |--> Test access to known phishing domains (baseline - should work)
    |--> Test access to legitimate sites (baseline)
    |--> Measure DNS resolution times (baseline)
    |--> Record all baseline results
    |
    v
[Treatment Phase - Layer 1: Chrome Extension]
    |
    |--> Enable Chrome extension phishing detection
    |--> Load PhishFort blocklist into extension cache
    |--> Integrate Google Safe Browsing API
    |--> Configure extension → SIEM logging
    |--> Test phishing domain detection in browser
    |--> Verify warning displays work correctly
    |
    v
[Treatment Phase - Layer 2: DNS/RPZ]
    |
    |--> Download PhishFort blocklists
    |--> Configure BIND9 RPZ zones
    |--> Import phishing domains to RPZ
    |--> Enable DNS-level blocking
    |--> Configure DNS → SIEM logging
    |--> Verify network-level blocking functions
    |
    v
[Post-test Phase - Integrated System]
    |
    |--> Test known phishing domains (should be blocked by Layer 1 or 2)
    |--> Test legitimate sites (should work normally)
    |--> Measure DNS resolution times with both layers active
    |--> Simulate user bypass attempts (ignore extension warnings)
    |--> Collect SIEM logs from both layers
    |--> Analyze which layer caught which threats
    |
    v
[Real-World Testing Phase]
    |
    |--> Deploy to 2 on-campus friends
    |--> Monitor normal browsing activity for defined period
    |--> Collect user feedback on false positives
    |--> Gather usability feedback
    |--> Document any issues or edge cases
    |
    v
[Analysis & Reporting]
    |
    |--> Compare pretest vs post-test results
    |--> Calculate block rate for each layer
    |--> Measure false positive rates
    |--> Analyze performance overhead
    |--> Evaluate defense-in-depth effectiveness
    |--> Document findings and create implementation guide
```

## Experimental Environment

### Virtual Machine Infrastructure

**VM #1: DNS Server (Ubuntu Server)**
- **Purpose:** Host BIND9 DNS server with RPZ configuration and SIEM logging
- **OS:** Ubuntu Server 22.04 LTS
- **Resources:** 2-4 vCPUs, 4GB RAM, 20GB storage
- **Software:** BIND9, rsyslog for SIEM integration, monitoring tools
- **Network:** Static IP on isolated lab network
- **Justification:** Sufficient resources to handle DNS queries, large RPZ blocklists in memory, and logging operations

**VM #2: Windows Client 1 (Primary Test System)**
- **Purpose:** Primary testing platform for Chrome extension and integrated system evaluation
- **OS:** Windows 10/11
- **Resources:** 2 vCPUs, 4GB RAM, 40GB storage
- **Software:** Chrome browser, testing tools
- **DNS Configuration:** Points to VM #1 for DNS resolution
- **Justification:** Simulates typical end-user workstation for realistic testing scenarios

**VM #3: Windows Client 2 (Validation System)**
- **Purpose:** Secondary testing to verify consistency across multiple machines
- **OS:** Windows 10/11
- **Resources:** 2 vCPUs, 4GB RAM, 40GB storage
- **Software:** Chrome
- **DNS Configuration:** Points to VM #1 for DNS resolution
- **Justification:** Validates that protection works consistently across different client configurations

All VMs deployed on school's Proxmox infrastructure with isolated network segment for controlled testing.

### Chrome Extension Architecture

**Technology Stack:**
- Manifest V3 (latest Chrome extension standard)
- JavaScript for core functionality
- Chrome Web Request API for URL interception
- Chrome Storage API for local blocklist cache
- Fetch API for Google Safe Browsing integration

**Key Components:**
1. **Background Service Worker:** Monitors email interfaces and intercepts URL clicks
2. **Content Scripts:** Inject warning overlays into email web pages
3. **Local Blocklist Cache:** PhishFort domains stored locally for fast lookup
4. **API Integration:** Queries Google Safe Browsing for unknown URLs
5. **Logging Module:** Sends detection events to SIEM server

**Detection Workflow:**
```
User clicks link in email
    ↓
Content script intercepts click
    ↓
Background worker checks local cache
    ↓
If not in cache → Query Google Safe Browsing API
    ↓
Calculate threat score
    ↓
If malicious → Display warning overlay + Log to SIEM
If safe → Allow navigation + Log to SIEM
```

### DNS/RPZ Configuration

**BIND9 Setup:**
- Latest stable BIND9 version on Ubuntu Server
- RPZ zones configured for phishing domain blocking
- Forwarding to upstream DNS (Google DNS 8.8.8.8, Cloudflare 1.1.1.1) for legitimate queries
- Query logging enabled for SIEM integration
- Performance monitoring for resolution time analysis

**RPZ Zone Structure:**
```
zone "rpz.phishing.local" {
    type master;
    file "/etc/bind/rpz/phishing-domains.db";
};
```

**Blocklist Processing Pipeline:**
1. Download PhishFort lists from GitHub
2. Parse and validate domain formats
3. Remove duplicates and whitelist false positives
4. Convert to RPZ zone file format:
   ```
   evil-site.com CNAME .
   phishing-domain.com CNAME .
   ```
5. Reload BIND9 to activate new rules
6. Verify blocking with test queries

### SIEM Integration Architecture

**Logging Infrastructure:**
- **Phase 1:** Syslog-based logging for rapid deployment
- **Phase 2 (if time permits):** ELK Stack (Elasticsearch, Logstash, Kibana) for visualization

**Log Sources:**
1. **Chrome Extension Logs:**
   - Timestamp, detected URL, threat score, API response, user action
   - Format: JSON over HTTPS to syslog endpoint
   
2. **DNS Server Logs:**
   - Timestamp, queried domain, client IP, RPZ action (block/allow), response time
   - Format: BIND9 query logs parsed and forwarded to syslog

**SIEM Functionality:**
- Centralized event aggregation from both protection layers
- Correlation of multi-layer detections (same threat caught by both layers)
- Real-time alerting for high-severity phishing attempts
- Historical analysis of threat patterns
- Performance results dashboard
- False positive tracking and analysis

## Data Collection Sources

### Phishing Domain Blocklists

**Primary Source: PhishFort GitHub Repository**
- Actively maintained community-driven phishing database
- Updated multiple times daily with new threats
- Format: Plain text list of domains
- URL: https://github.com/phishfort/phishfort-lists

**Blocklist Integration Process:**
```
[PhishFort GitHub] 
    ↓ Download
[Raw Domain Lists]
    ↓ Process
[Validation & Deduplication]
    ↓ Split
[Extension Cache]  [RPZ Zone File]
    ↓                  ↓
[Chrome Extension] [BIND9 Server]
```

### Machine Learning Detection

**Google Safe Browsing API:**
- Google's ML-trained phishing detection system
- Checks URLs against constantly updated threat database
- Returns threat classification and confidence scores
- Free tier: 10,000 queries per day (sufficient for testing)
- Provides zero-day protection for newly created phishing sites not yet in blocklists

**API Integration:**
```javascript
// Simplified example
async function checkURL(url) {
    const response = await fetch('https://safebrowsing.googleapis.com/v4/threatMatches:find', {
        method: 'POST',
        body: JSON.stringify({
            threatInfo: {
                threatTypes: ["SOCIAL_ENGINEERING"],
                threatEntries: [{"url": url}]
            }
        })
    });
    return response.json();
}
```

### Testing Domain Categories

**1. Known Phishing Domains (from PhishFort)**
- Verified malicious domains for controlled testing
- Already taken down or sandboxed for safe testing
- Used to measure block rate effectiveness

**2. Legitimate Domain Baseline**
- 50-100 popular legitimate websites:
  - Email providers (gmail.com)
  - Banking sites (major US banks)
  - E-commerce (amazon.com, ebay.com)
  - Social media (facebook.com, twitter.com, linkedin.com)
  - News outlets (cnn.com, nytimes.com, wsj.com)
- Used to measure false positive rate

**3. Edge Case Domains**
- Newly registered domains (test ML detection)
- Typosquatting examples (g00gle.com, paypa1.com)
- URL shorteners (bit.ly, tinyurl.com)
- International domains with IDN homographs

## Evaluation Results

### Primary Results

**Block Rate (Target: >95%)**
- Percentage of known phishing domains successfully blocked
- Measured separately for:
  - Layer 1 (Chrome extension) block rate
  - Layer 2 (DNS/RPZ) block rate  
  - Combined system block rate
- Formula: (Blocked Domains / Total Phishing Domains Tested) × 100

**False Positive Rate (Target: <1%)**
- Percentage of legitimate sites incorrectly blocked or flagged
- Critical for usability and user trust
- Formula: (False Positives / Total Legitimate Sites Tested) × 100

**Detection Layer Analysis**
- Which layer catches threats first (extension vs DNS)
- Percentage caught by Layer 1 only
- Percentage caught by Layer 2 only
- Percentage caught by both layers (redundancy measure)

### Performance results

**DNS Resolution Time**
- Average query response time before RPZ implementation
- Average query response time after RPZ implementation
- Performance overhead calculation
- Target: <50ms additional latency

**Extension Response Time**
- Time from URL click to warning display (target: <200ms)
- API query latency to Google Safe Browsing
- Local cache hit rate (should be >80% after initial population)

**System Resource Usage**
- DNS server CPU and memory utilization
- Chrome extension memory footprint
- Network bandwidth for API queries and SIEM logging

### User Experience results

**Real-World Testing (2 On-Campus Friends)**
- Deployment duration: 1-2 weeks per friend
- Data collected:
  - Number of warnings displayed
  - False positive reports
  - Legitimate blocks (actual phishing attempts caught)
  - User feedback on warning clarity and usefulness
  - Performance perception (does browsing feel slower?)
  - Usability issues or edge cases discovered

**Feedback Collection Methods:**
- Post-deployment survey
- Informal interviews about user experience
- Log analysis of user interactions with warnings
- Documentation of any reported issues

## Pretest Phase

### Environment Setup

1. **DNS Server Configuration**
   - Install Ubuntu Server 22.04 on VM #1
   - Install BIND9 without RPZ zones configured
   - Configure basic DNS forwarding to upstream servers
   - Set static IP address
   - Verify basic DNS resolution works
   - Take VM snapshot: "Clean BIND9 - Pre-RPZ"

2. **Client Configuration**
   - Install Windows 10/11 on VM #2 and VM #3
   - Install Chrome browser
   - Configure DNS settings to point to VM #1
   - Verify network connectivity and DNS resolution
   - Install Chrome extension in "disabled" mode
   - Take VM snapshots: "Windows Client - Baseline"

3. **SIEM Setup**
   - Configure rsyslog on DNS server
   - Set up log collection directory
   - Test basic logging functionality
   - Document baseline log format

### Baseline Measurements

**DNS Performance Baseline**
- Use `dig` command to measure resolution times
- Test 100 queries to popular domains
- Record: min, max, average, median response times
- Establish baseline before any blocking is enabled

**Phishing Domain Accessibility Test**
- Attempt to access 50 known phishing domains (from safe/inactive list)
- All should resolve and be accessible (this is baseline)
- Document which domains resolve successfully
- Record DNS query times for these domains

**Legitimate Site Access Test**
- Access all 50-100 legitimate baseline domains
- Verify all load correctly
- Document any that fail (pre-existing issues)
- Record load times and DNS resolution times

**Extension Disabled Baseline**
- With extension disabled, click email links
- No warnings should appear (baseline behavior)
- Document normal user flow
- Record click-to-page-load times

## Treatment Phase - Layer 1 (Chrome Extension)

### Extension Development and Deployment

1. **Core Functionality Implementation**
   - Build Manifest V3 extension structure
   - Implement URL interception for Gmail)
   - Create warning overlay UI components
   - Develop local blocklist cache system
   - Build logging module for SIEM integration

2. **PhishFort Integration**
   - Download latest PhishFort blocklists
   - Process and validate domain list
   - Load into extension's local storage
   - Implement cache refresh mechanism

3. **Google Safe Browsing API Integration**
   - Obtain API key from Google Cloud Console
   - Implement API query function
   - Add rate limiting and error handling
   - Cache API responses to minimize queries

4. **SIEM Logging Configuration**
   - Configure extension to send logs to syslog endpoint
   - Define log format and event types
   - Test logging for all detection scenarios
   - Verify logs appear in SIEM

### Extension Testing

- Deploy extension to both test VMs
- Test phishing URL detection with known bad domains
- Verify warning displays correctly
- Test API fallback when domain not in cache
- Confirm logs appear in SIEM
- Measure response times (target: <200ms)

## Treatment Phase - Layer 2 (DNS/RPZ)

### RPZ Configuration

1. **Zone File Creation**
   - Download PhishFort blocklists
   - Convert to RPZ zone file format
   - Add zone configuration to BIND9
   - Validate syntax

2. **RPZ Activation**
   ```bash
   # Example configuration in named.conf.local
   response-policy { 
       zone "rpz.phishing.local"; 
   };
   
   zone "rpz.phishing.local" {
       type master;
       file "/etc/bind/rpz/phishing-domains.db";
   };
   ```

3. **BIND9 Integration**
   - Reload BIND9 configuration
   - Verify RPZ is active with test queries
   - Configure query logging
   - Set up log forwarding to SIEM

### DNS Testing

- Test phishing domain queries (should return NXDOMAIN)
- Verify legitimate domains still resolve
- Measure resolution time impact
- Confirm logs appear in SIEM with correct format
- Test from both Windows clients
- Take snapshot: "RPZ Active - Full Blocking"

## Post-test Phase - Integrated System

### System Testing

**Multi-Layer Detection Verification**
1. Test known phishing domains:
   - Extension should display warning (Layer 1)
   - If warning ignored, DNS should block (Layer 2)
   - Both events should log to SIEM
   - Calculate detection rate for each layer

2. Test legitimate domains:
   - No warnings from extension
   - Normal DNS resolution
   - Measure any false positives
   - Log all access attempts

3. Test edge cases:
   - New domains not in blocklist (API catches via ML)
   - URL shorteners
   - Typosquatting domains
   - International domains with special characters

**Performance Analysis**
- Measure DNS resolution times with full system active
- Compare to baseline measurements
- Calculate performance overhead
- Test under load (rapid successive queries)
- Verify extension doesn't slow browsing

**SIEM Analysis**
- Review aggregated logs from both layers
- Identify correlation patterns (threats caught by both)
- Calculate coverage statistics
- Generate effectiveness reports
- Create visualization dashboards (if ELK Stack implemented)

## Real-World Testing Phase

### Friends and Family

**Deployment Process:**
1. Install Chrome extension on friends devices
2. Configure DNS to use test server (if on campus network)
3. Provide usage instructions and feedback form
4. Monitor SIEM for events from friends laptops
5. Weekly check-ins for feedback
6. Duration: 1-2 weeks 

**Data Collection:**
- Number of phishing warnings displayed
- Any false positives reported
- Actual phishing attempts caught
- User feedback on:
  - Warning clarity and usefulness
  - Perceived browsing speed
  - Any usability issues
  - Overall satisfaction
