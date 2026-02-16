# Progress Report 5
## Overview

This progress report documents the implementation of RPZ for DNS-based phishing protection. Building on the previous session's BIND9 installation, this session focused on downloading phishing blocklists, converting them to RPZ format, and successfully enabling network-level domain blocking.

---

## Work Completed

### 1. Phishing Blocklist Acquisition

Successfully downloaded an active phishing domain blocklist from the Phishing.Database repository on GitHub.

**Source URL:**
```
https://raw.githubusercontent.com/Phishing-Database/Phishing.Database/refs/heads/master/phishing-domains-ACTIVE/phishing-domains-ACTIVE1.txt
```

**Commands Executed:**
```bash
sudo mkdir -p /etc/bind/rpz
sudo wget https://raw.githubusercontent.com/Phishing-Database/Phishing.Database/refs/heads/master/phishing-domains-ACTIVE/phishing-domains-ACTIVE1.txt -O /etc/bind/rpz/phishing-raw.txt
wc -l /etc/bind/rpz/phishing-raw.txt
```

**Results:**
| Metric | Value |
|--------|-------|
| Domains Downloaded | 377,754 |
| File Size | 10.0 MB |

---

### 2. RPZ Zone File Conversion

Converted the raw phishing domain list to BIND9 RPZ zone format. Initial conversion attempts encountered issues with malformed domains in the blocklist.

**Challenges Encountered:**
- Unbalanced parentheses in some domain entries
- Domain labels exceeding 63-character DNS limit
- Special characters in domain names

**Initial Script I wrote:**
```
#!/bin/bash

INPUT="/etc/bind/rpz/phishing-raw.txt"
OUTPUT="/etc/bind/rpz/db.rpz.phishing"
SERIAL=$(date +%Y%m%d%H)

echo "Input: $INPUT"
echo "Output: $OUTPUT"

cat > $OUTPUT << EOF
\$TTL 300
@       IN      SOA     localhost. root.localhost. (
                        $SERIAL  ; serial
                        3600     ; refresh
                        600      ; retry
                        86400    ; expire
                        300      ; minimum
                        )
        IN      NS      localhost.

EOF

TOTAL=$(wc -l < "$INPUT")
echo "Loading $TOTAL domains"

COUNT=0
while IFS= read -r domain || [ -n "$domain" ]; do
    # Skip empty lines and comments
    [[ -z "$domain" ]] && continue
    [[ "$domain" =~ ^# ]] && continue
    [[ "$domain" =~ ^// ]] && continue
    
    domain=$(echo "$domain" | tr -d '[:space:]' | tr '[:upper:]' '[:lower:]')
    
    [[ -z "$domain" ]] && continue
    
    echo "$domain CNAME ." >> $OUTPUT
    echo "*.$domain CNAME ." >> $OUTPUT
    
    ((COUNT++))
    
    # Progress indicator every 50000 domains
    if (( COUNT % 50000 == 0 )); then
        echo "Processed $COUNT domains"
    fi

done < "$INPUT"

echo "Conversion finished"
echo "Total domains converted: $COUNT"
echo "Output file: $OUTPUT"
```

**Final Conversion Command:**
```bash
sudo bash -c 'echo "\$TTL 300
@ IN SOA localhost. root.localhost. 2026021618 3600 600 86400 300
  IN NS localhost." > /etc/bind/rpz/db.rpz.phishing && grep -E "^[a-zA-Z0-9]" /etc/bind/rpz/phishing-raw.txt | grep -v "[()\"\\]" | awk "length(\$0)<64 {print \$1\" CNAME .\"; print \"*.\"\$1\" CNAME .\"}" >> /etc/bind/rpz/db.rpz.phishing'
```

**RPZ Zone File Format:**
```
$TTL 300
@ IN SOA localhost. root.localhost. 2026021618 3600 600 86400 300
  IN NS localhost.
malicious-domain.xyz CNAME .
*.malicious-domain.xyz CNAME .
another-phishing-site.com CNAME .
*.another-phishing-site.com CNAME .
```
---

### 3. BIND9 RPZ Configuration

Configured BIND9 to load and use the RPZ zone for query filtering.

**File:** `/etc/bind/named.conf.local`

**Configuration Added:**
```
zone "rpz.phishing" {
    type master;
    file "/etc/bind/rpz/db.rpz.phishing";
    allow-query { none; };
};
```

**File:** `/etc/bind/named.conf.options`

**Configuration Added:**
```
    response-policy { zone "rpz.phishing"; };
```

---
### 4. RPZ Blocking Verification

Tested the RPZ implementation to confirm phishing domains are blocked while legitimate domains remain accessible.

**Test 1: Legitimate Domain (Google.com)**

```bash
dig @localhost google.com
```

**Results:**

<img src="https://github.com/user-attachments/assets/3e32ac6b-8d1a-416d-8210-1dd3391ef067" />

**Test 2: Phishing Domain (from blocklist)**

```bash
dig @localhost 0000000000000000000000000000000000000.xyz
```

**Results:**

<img src="https://github.com/user-attachments/assets/623e81e8-3a91-4d6d-ad29-64ddb9dd15cb" />

### 6. Proxmox Snapshot

Created a snapshot to preserve the working RPZ configuration.

---

## Technical Challenges and Solutions

### Challenge 1: Malformed Domain Entries
**Problem:** Zone file failed to load due to unbalanced parentheses  
**Solution:** Filtered entries containing parentheses and special characters

### Challenge 2: DNS Label Length Exceeded
**Problem:** Some domains had labels exceeding 63-character DNS limit  
**Solution:** Added length filter to exclude oversized entries

### Challenge 3: Zone Header Corruption
**Problem:** Cleanup commands accidentally removed SOA/NS records  
**Solution:** Regenerated zone file with proper header and filtered domains in single command

