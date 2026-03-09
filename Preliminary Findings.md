# Preliminary Findings Structure
---

## Layer 1: Chrome Extension

### Detection Accuracy
*[Results to be filled in after structured testing with volunteers]*

### False Positive Rate
*[Results to be filled in after structured testing]*

**Observed so far:**
- Some legitimate emails (e.g. marketing/notification emails) triggered false positives due to tracking URLs
- Planned fix: domain whitelist

### Visual Warning Indicators
*[Screenshots and observations to be added after styles.css is fully integrated]*

### Performance
*[Scan time and resource usage to be measured]*

---

## Layer 2: DNS/RPZ Blocking

### Blocking Verification
- Known phishing domains return NXDOMAIN from both server and client so it was confirmed working
- Legitimate domains resolve correctly 

### Performance Impact
*[DNS resolution time before vs. after RPZ to be measured and recorded]*

---

## Layer 3: SIEM Integration

*[This layer has not yet been implemented, results are to be filled in during upcoming weeks]*

### Log Sources
*[To be documented]*

### Alert Rules
*[To be documented]*

### Dashboard Output
*[Screenshots and observations to be added]*

---

## Integrated System

### Defense-in-Depth Evaluation
*[To be completed once all three layers are operational]*

### Overall Effectiveness
*[Block rate, false positive rate, and performance overhead across all layers — to be calculated]*
