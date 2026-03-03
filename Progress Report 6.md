# Progress Report 6
## Overview

This progress report documents the development of content.js, the core of the PhishBait Chrome extension. As Layer 1 of the three-layer anti-phishing architecture, the Chrome extension provides real-time link scanning within gmail and gives visual warnings before they click potentially malicious links. This report provides a complete logic breakdown of the content.js script, explaining the design decisions, detection methodology, and implementation rationale from a cybersecurity perspective.

---

## content.js — Complete Implementation

The content.js file is structured into four logical sections: configuration, analysis, application, and initialization. Below is the complete script followed by a detailed breakdown of each component.

### Full Script

```javascript
/**
 * PhishBait Content Script
 * Author: Dylan George
 * Purpose: Scans links in webmail interfaces (Gmail, Outlook) for phishing indicators
 * 
 */

// Configuration object containing all detection rules
const PhishBaitConfig = {
    // Keywords commonly found in phishing URLs
    suspiciousKeywords: [
        'login', 'signin', 'sign-in', 'log-in',
        'verify', 'verification', 'confirm', 'confirmation',
        'account', 'password', 'credential',
        'secure', 'security', 'authenticate',
        'update', 'upgrade', 'expire', 'expired',
        'suspend', 'suspended', 'restrict', 'restricted',
        'unusual', 'unauthorized', 'alert',
        'bank', 'paypal', 'amazon', 'microsoft', 'apple',
        'urgent', 'immediate', 'action', 'required',
        'click-here', 'click.here'
    ],

    // Top-level domains frequently abused by phishers
    suspiciousTLDs: [
        '.tk', '.ml', '.ga', '.cf', '.gq',      
        '.xyz', '.top', '.club', '.online',     
        '.buzz', '.rest', '.fit', '.work',      
        '.click', '.link', '.surf'              
    ],

    // Used to detect typosquatting attempts
    brandDomains: [
        'google', 'gmail', 'microsoft', 'outlook', 'office365',
        'apple', 'icloud', 'amazon', 'aws',
        'facebook', 'instagram', 'twitter',
        'paypal', 'stripe', 'chase', 'bankofamerica',
        'netflix', 'spotify', 'dropbox'
    ],

    // Scoring thresholds for risk classification
    thresholds: {
        low: 2,      // Score >= 2: Low risk (yellow warning)
        high: 5      // Score >= 5: High risk (red warning)
    }
};

// Statistics tracking for the popup display
let scanStats = {
    totalLinksScanned: 0,
    highRiskCount: 0,
    mediumRiskCount: 0,
    lowRiskCount: 0,
    lastScanTime: null
};

/**
 * Analyzes a URL for phishing indicators and returns a risk assessment
 * 
 * @param {string} url - The URL to analyze
 * @returns {Object} - Contains score and all of the reasons
 */
function analyzeUrl(url) {
    let result = {
        score: 0,
        reasons: [],
        riskLevel: 'safe'
    };

    const lowerUrl = url.toLowerCase();
    
    // Attempt to parse URL for detailed analysis
    let urlObject;
    try {
        urlObject = new URL(url);
    } catch (error) {
        result.score += 2;
        result.reasons.push('Malformed URL structure');
        return result;
    }

    const hostname = urlObject.hostname.toLowerCase();
    const pathname = urlObject.pathname.toLowerCase();
    const fullPath = hostname + pathname;

    // Check IP address instead of domain name
    const ipAddressPattern = /^(\d{1,3}\.){3}\d{1,3}$/;
    if (ipAddressPattern.test(hostname)) {
        result.score += 5;
        result.reasons.push('URL uses IP address instead of domain name');
    }

    // Check Suspicious keywords in URL
    for (const keyword of PhishBaitConfig.suspiciousKeywords) {
        if (fullPath.includes(keyword)) {
            result.score += 1;
            result.reasons.push(`Contains suspicious keyword: "${keyword}"`);
            
            // Only report first 3 keyword matches to avoid cluttering
            if (result.reasons.filter(r => r.includes('keyword')).length >= 3) {
                break;
            }
        }
    }

    // Check Suspicious TLD
    for (const tld of PhishBaitConfig.suspiciousTLDs) {
        if (hostname.endsWith(tld)) {
            result.score += 3;
            result.reasons.push(`Uses suspicious TLD: "${tld}"`);
            break;
        }
    }

    // Check Excessive subdomains
    const subdomainCount = (hostname.match(/\./g) || []).length;
    if (subdomainCount > 3) {
        result.score += 2;
        result.reasons.push(`Excessive subdomains (${subdomainCount} levels deep)`);
    }

    // Check Brand name in subdomain
    for (const brand of PhishBaitConfig.brandDomains) {
        // Check if brand appears in hostname but isn't the actual domain
        if (hostname.includes(brand) && !hostname.endsWith(brand + '.com') && 
            !hostname.endsWith(brand + '.org') && !hostname.endsWith(brand + '.net')) {
            result.score += 4;
            result.reasons.push(`Possible brand impersonation: "${brand}" in URL`);
            break;
        }
    }

    // Check Unusual port numbers
    const port = urlObject.port;
    if (port && port !== '80' && port !== '443') {
        result.score += 2;
        result.reasons.push(`Non-standard port number: ${port}`);
    }

    // Check URL contains @ abuse
    if (url.includes('@')) {
        result.score += 4;
        result.reasons.push('URL contains @ symbol (possible URL obfuscation)');
    }

    // Check Extremely long URL
    if (url.length > 100) {
        result.score += 1;
        result.reasons.push('Unusually long URL');
    }

    // Check Multiple redirects indicated in URL
    const redirectIndicators = ['redirect', 'url=http', 'goto=', 'link=http', 'out=http'];
    for (const indicator of redirectIndicators) {
        if (lowerUrl.includes(indicator)) {
            result.score += 2;
            result.reasons.push('URL contains redirect indicator');
            break;
        }
    }

    // Check Homograph attack detection 
    const mixedCharPattern = /[а-яА-Я]|[α-ωΑ-Ω]/; // Cyrillic or Greek characters
    if (mixedCharPattern.test(url)) {
        result.score += 5;
        result.reasons.push('Possible homograph attack (mixed character sets)');
    }

    // Determine risk level based on total score
    if (result.score >= PhishBaitConfig.thresholds.high) {
        result.riskLevel = 'high';
    } else if (result.score >= PhishBaitConfig.thresholds.low) {
        result.riskLevel = 'medium';
    } else {
        result.riskLevel = 'safe';
    }

    return result;
}

/**
 * Applies visual warning indicators to a link element based on risk level
 * 
 * @param {HTMLElement} linkElement - The anchor tag to modify
 * @param {Object} analysis - The result from analyzeUrl()
 */
function applyWarningToLink(linkElement, analysis) {
    linkElement.classList.remove('phishbait-safe', 'phishbait-medium', 'phishbait-high');
    
    // Apply appropriate warning class based on risk level
    if (analysis.riskLevel === 'high') {
        linkElement.classList.add('phishbait-high');
        scanStats.highRiskCount++;
    } else if (analysis.riskLevel === 'medium') {
        linkElement.classList.add('phishbait-medium');
        scanStats.mediumRiskCount++;
    } else {
        linkElement.classList.add('phishbait-safe');
        scanStats.lowRiskCount++;
    }

    // Create tooltip with detailed information
    const reasonsList = analysis.reasons.length > 0 
        ? analysis.reasons.join('\n• ') 
        : 'No suspicious indicators detected';
    
    const tooltipText = `PhishBait Analysis:\nRisk Level: ${analysis.riskLevel.toUpperCase()}\nScore: ${analysis.score}\n\nIndicators:\n• ${reasonsList}`;
    
    linkElement.setAttribute('data-phishbait-tooltip', tooltipText);
    linkElement.setAttribute('data-phishbait-risk', analysis.riskLevel);
    linkElement.setAttribute('data-phishbait-score', analysis.score);
}

/**
 * Scans all links on the current page
 * This is the main function that starts the scanning process
 */
function scanAllLinks() {
    console.log('PhishBait: Starting link scan...');
    
    // Reset statistics for this scan
    scanStats = {
        totalLinksScanned: 0,
        highRiskCount: 0,
        mediumRiskCount: 0,
        lowRiskCount: 0,
        lastScanTime: new Date().toISOString()
    };

    // Find all anchor tags with href attributes
    const allLinks = document.querySelectorAll('a[href]');
    
    for (const link of allLinks) {
        const href = link.getAttribute('href');
        
        // Skip empty hrefs, javascript links, and anchor links
        if (!href || href.startsWith('#') || href.startsWith('javascript:')) {
            continue;
        }

        if (href.startsWith('mailto:') || href.startsWith('tel:')) {
            continue;
        }

        if (href.startsWith('/') && !href.startsWith('//')) {
            continue;
        }

        // Analyze the URL
        const analysis = analyzeUrl(href);
        
        // Apply visual warning if any risk detected
        applyWarningToLink(link, analysis);
        
        scanStats.totalLinksScanned++;
    }

    // Save stats to storage for popup display
    chrome.storage.local.set({ scanStats: scanStats });
    
    console.log(`PhishBait: Scan complete. Analyzed ${scanStats.totalLinksScanned} links.`);
    console.log(`  High Risk: ${scanStats.highRiskCount}`);
    console.log(`  Medium Risk: ${scanStats.mediumRiskCount}`);
    console.log(`  Low Risk: ${scanStats.lowRiskCount}`);
}

/**
 * Sets up a MutationObserver to detect dynamically loaded content
 * Email interfaces like Gmail load content dynamically, so we need to re-scan when new content appears
 */
function setupDynamicContentObserver() {
    const observerConfig = {
        childList: true,      // Watch for added/removed child elements
        subtree: true,        // Watch all descendants, not just direct children
        attributes: false,    // Don't watch attribute changes
        characterData: false  // Don't watch text content changes
    };

    // avoid excessive scanning
    let scanTimeout = null;
    const DEBOUNCE_DELAY = 1000; // Wait 1 second after last change before scanning

    // Create the observer
    const observer = new MutationObserver(function(mutations) {
        // Check if any mutations added new nodes
        const hasNewNodes = mutations.some(mutation => mutation.addedNodes.length > 0);
        
        if (hasNewNodes) {
            // Clear existing timeout
            if (scanTimeout) {
                clearTimeout(scanTimeout);
            }
            
            // Set new timeout for debounced scan
            scanTimeout = setTimeout(function() {
                scanAllLinks();
            }, DEBOUNCE_DELAY);
        }
    });

    observer.observe(document.body, observerConfig);
    
    console.log('PhishBait: Dynamic content observer initialized');
}

/**
 * Adds event listener for tooltip display on hover
 */
function setupTooltipHandlers() {
    const tooltip = document.createElement('div');
    tooltip.id = 'phishbait-tooltip';
    tooltip.className = 'phishbait-tooltip-hidden';
    document.body.appendChild(tooltip);

    document.addEventListener('mouseenter', function(event) {
        const target = event.target;
        
        // Check if this is a PhishBait-scanned link
        if (target.hasAttribute && target.hasAttribute('data-phishbait-tooltip')) {
            const tooltipText = target.getAttribute('data-phishbait-tooltip');
            const riskLevel = target.getAttribute('data-phishbait-risk');
            
            tooltip.textContent = tooltipText;
            tooltip.className = `phishbait-tooltip phishbait-tooltip-${riskLevel}`;
            
            const rect = target.getBoundingClientRect();
            tooltip.style.left = rect.left + 'px';
            tooltip.style.top = (rect.bottom + 5) + 'px';
        }
    }, true);

    document.addEventListener('mouseleave', function(event) {
        const target = event.target;
        
        if (target.hasAttribute && target.hasAttribute('data-phishbait-tooltip')) {
            tooltip.className = 'phishbait-tooltip-hidden';
        }
    }, true);
}

/**
 * Initialize PhishBait when the page loads
 */
function initialize() {
    console.log('PhishBait: Extension loaded on ' + window.location.hostname);
    
    // Perform initial scan
    scanAllLinks();
    
    // Set up observer for dynamic content
    setupDynamicContentObserver();
    
    // Set up tooltip handlers
    setupTooltipHandlers();
}

// Run initialization when DOM is ready
if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initialize);
} else {
    // DOM already loaded, initialize immediately
    initialize();
}
```

---

## Logic Breakdown and Design Rationale

### Section 1: PhishBaitConfig — Detection Rules Configuration

The configuration object centralizes all detection parameters, making the system maintainable and tunable without needing to touch the core logic.

#### Suspicious Keywords Array

```javascript
suspiciousKeywords: [
    'login', 'signin', 'sign-in', 'log-in',
    'verify', 'verification', 'confirm', 'confirmation',
]
```

**Why these keywords?**

Phishing attacks rely on social engineering and the URLs themselves often contain keywords designed to create urgency or legitimacy. I selected these keywords based on common phishing patterns documented in industry threat reports:

- **Authentication terms** (`login`, `signin`, `password`, `credential`) — Phishers need victims to enter credentials, so these words appear frequently in fake login pages
- **Urgency terms** (`urgent`, `immediate`, `expire`, `suspended`) — Creates panic
- **Action terms** (`verify`, `confirm`, `update`, `action required`) — Mimic legitimate account verification 
- **Brand terms** (`bank`, `paypal`, `amazon`, `microsoft`) — Often appear in URLs alongside impersonation attempts

The array is intentionally broad because false positives at the keyword level only add +1 to the score — it takes multiple indicators to trigger a warning.

#### Suspicious TLDs Array

```javascript
suspiciousTLDs: [
    '.tk', '.ml', '.ga', '.cf', '.gq',      
    '.xyz', '.top', '.club', '.online',     
    '.buzz', '.rest', '.fit', '.work',      
]
```

**Why these TLDs?**

Research from APWG and Proofpoint consistently shows that certain top-level domains are disproportionately used for phishing:

- **Free TLDs** (`.tk`, `.ml`, `.ga`, `.cf`, `.gq`) — Freenom offered these for free until 2023, and they became notorious for abuse
- **Cheap TLDs** (`.xyz`, `.top`, `.club`) — Low registration costs often under $1 attract attackers who register domains in bulk
- **Tricky names** (`.click`, `.link`, `.surf`) — Sound action-oriented

I deliberately excluded common legitimate TLDs like `.io`, `.co`, and `.app` despite some abuse, because the false positive rate would be too high given their widespread legitimate use in the tech industry.

#### Brand Domains Array

```javascript
brandDomains: [
    'google', 'gmail', 'microsoft', 'outlook', 'office365',
    'apple', 'icloud', 'amazon', 'aws',
    'paypal', 'stripe', 'chase', 'bankofamerica',
    // ... etc
]
```

**Why these brands?**

These are the most commonly impersonated brands according to Verizon DBIR and Proofpoint State of Phish reports. The detection logic checks if these strings appear in a URL's hostname *without* the URL actually belonging to that brand. This catches typosquatting and subdomain impersonation attacks.

#### Scoring Thresholds

```javascript
thresholds: {
    low: 2,      // Score >= 2: Medium risk (yellow warning)
    high: 5      // Score >= 5: High risk (red warning)
}
```

**Why these values?**

The thresholds were calibrated to balance detection sensitivity against false positive rates:

- **Score 0-1 is Safe** — A single minor indicator
- **Score 2-4 is Medium Risk** — Multiple minor indicators or one moderate indicator
- **Score 5+ is High Risk** — user should not click

---

### Section 2: analyzeUrl() — The Detection Engine

#### Why Cumulative Scoring?

I chose cumulative scoring over binary detection for several reasons:

- **phishing URLs often combine multiple subtle indicators** — No single indicator is enough but the combination is suspicious
- **Reduces false positives** — A legitimate URL is unlikely to match several rules
- **Provides graduated risk assessment** — Users can make informed decisions based on severity

#### Detection Checks Breakdown

| Check | Points | Rationale |
|-------|--------|-----------|
| IP address hostname | +5 | Legitimate sites almost never display IP addresses to users |
| Suspicious keywords | +1 each | Common in phishing but also in legitimate URLs |
| Suspicious TLD | +3 | These TLDs have high abuse rates but aren't exclusively bad |
| Excessive subdomains (>3) | +2 | Used to hide the real domain |
| Brand impersonation | +4 | Strong indicator when brand appears in URL but domain doesn't match |
| Non-standard port | +2 | Unusual for legitimate sites |
| @ symbol in URL | +4 | Classic URL obfuscation technique |
| URL length >100 chars | +1 | long URLs can hide suspicious elements |
| Redirect indicators | +2 | Open redirects are commonly abused; `url=http` patterns suggest redirect chains |
| Homograph characters | +5 | Cyrillic/Greek lookalike characters are almost always malicious |

#### URL Parsing and Error Handling

```javascript
let urlObject;
try {
    urlObject = new URL(url);
} catch (error) {
    result.score += 2;
    result.reasons.push('Malformed URL structure');
    return result;
}
```

**Why catch parsing errors?**

Malformed URLs that fail JavaScript's URL parser may indicate:

- Encoding tricks
- Malicious payloads embedded in URL structure

Rather than silently failing, the script treats unparseable URLs as suspicious +2 points and continues with that 

---

### Section 3: applyWarningToLink() — Visual Feedback

```javascript
function applyWarningToLink(linkElement, analysis) {
    linkElement.classList.remove('phishbait-safe', 'phishbait-medium', 'phishbait-high');
    
    if (analysis.riskLevel === 'high') {
        linkElement.classList.add('phishbait-high');
        scanStats.highRiskCount++;
    } else if (analysis.riskLevel === 'medium') {
        linkElement.classList.add('phishbait-medium');
        scanStats.mediumRiskCount++;
    } else {
        linkElement.classList.add('phishbait-safe');
        scanStats.lowRiskCount++;
    }
    
    // Store analysis data for tooltip
    linkElement.setAttribute('data-phishbait-tooltip', tooltipText);
    linkElement.setAttribute('data-phishbait-risk', analysis.riskLevel);
    linkElement.setAttribute('data-phishbait-score', analysis.score);
}
```

**Design Decisions:**

1. **CSS classes over inline styles** — Allows styling to be managed in styles.css
2. **Data attributes for metadata** — Stores analysis results directly on the element for tooltip access without additional lookups
3. **Statistics tracking** — Enables the popup to display scan results without re-analyzing
4. **Idempotent application** — Removes existing classes before adding new ones so rescans don't cause duplicate styling

---

### Section 4: scanAllLinks() — Orchestration

```javascript
function scanAllLinks() {
    const allLinks = document.querySelectorAll('a[href]');
    
    for (const link of allLinks) {
        const href = link.getAttribute('href');
        
        // Skip non-scannable links
        if (!href || href.startsWith('#') || href.startsWith('javascript:')) {
            continue;
        }
        if (href.startsWith('mailto:') || href.startsWith('tel:')) {
            continue;
        }
        if (href.startsWith('/') && !href.startsWith('//')) {
            continue;
        }

        const analysis = analyzeUrl(href);
        applyWarningToLink(link, analysis);
        scanStats.totalLinksScanned++;
    }

    chrome.storage.local.set({ scanStats: scanStats });
}
```

**Why these exclusions?**

- **`#` anchors** — Page-internal navigation
- **`javascript:` URLs** — Execute code
- **`mailto:` and `tel:`** — Open email clients
- **Relative paths (`/path`)** — Internal site navigation within Gmail itself

---

### Section 5: setupDynamicContentObserver() — Handling Modern Web Apps

```javascript
function setupDynamicContentObserver() {
    const observerConfig = {
        childList: true,
        subtree: true,
        attributes: false,
        characterData: false
    };

    let scanTimeout = null;
    const DEBOUNCE_DELAY = 1000;

    const observer = new MutationObserver(function(mutations) {
        const hasNewNodes = mutations.some(mutation => mutation.addedNodes.length > 0);
        
        if (hasNewNodes) {
            if (scanTimeout) {
                clearTimeout(scanTimeout);
            }
            scanTimeout = setTimeout(function() {
                scanAllLinks();
            }, DEBOUNCE_DELAY);
        }
    });

    observer.observe(document.body, observerConfig);
}
```

**Why is this necessary?**

Gmail and Outlook are Single Page Applications that load content dynamically via JavaScript. When a user opens an email, the content is fetched asynchronously and injected into the DOM. Without a MutationObserver, the initial scan would miss all links in subsequently opened emails.

---

### Section 6: Initialization Logic

```javascript
function initialize() {
    console.log('PhishBait: Extension loaded on ' + window.location.hostname);
    scanAllLinks();
    setupDynamicContentObserver();
    setupTooltipHandlers();
}

if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', initialize);
} else {
    initialize();
}
```

**Why check document.readyState?**

Content scripts can be injected at different times depending on the `run_at` setting in manifest.json and browser behavior. This check ensures:

- If the DOM is still loading, we wait for `DOMContentLoaded`
- If the DOM is already ready, initialize immediately

---

## Integration with Layer 2 (DNS/RPZ)

The Chrome extension and DNS blocking layer complement each other:

| Scenario | Layer 1 (Extension) | Layer 2 (DNS) |
|----------|---------------------|---------------|
| Known phishing domain in blocklist | May or may not detect because it's pattern-dependent | **Blocks** (NXDOMAIN) |
| New phishing domain with suspicious patterns | **Warns user** | May not block because its not yet in blocklist |
| Obfuscated URL with redirect | **Warns user** | Blocks final destination if known |
| User ignores browser warning | Warning displayed | **Blocks** |

This layered approach ensures that weaknesses in one detection method are covered by strengths in another.

