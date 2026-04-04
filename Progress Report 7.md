## Overview

This progress report documents the development of the popup dashboard and visual styling for the PhishBait Chrome extension. This includes popup.js, popup.html, and styles.css.

---

## popup.js Itself

```javascript
/**
 * PhishBait Popup Script
 * Author: Dylan George
 * Purpose: Scan statistics and controls in popup
 */

// Get all the elements from the HTML
const statusBadge = document.getElementById('status-badge');
const totalCount = document.getElementById('total-count');
const highCount = document.getElementById('high-count');
const mediumCount = document.getElementById('medium-count');
const safeCount = document.getElementById('safe-count');
const highBar = document.getElementById('high-bar');
const mediumBar = document.getElementById('medium-bar');
const safeBar = document.getElementById('safe-bar');
const lastScanTime = document.getElementById('last-scan-time');
const rescanBtn = document.getElementById('rescan-btn');

/**
 * Updates the status badge 
 */
function updateStatusBadge(high, medium) {
    if (high > 0) {
        statusBadge.textContent = '🔴 At Risk';
        statusBadge.className = 'status-badge status-high';
    } else if (medium > 0) {
        statusBadge.textContent = '🟡 Caution';
        statusBadge.className = 'status-badge status-medium';
    } else {
        statusBadge.textContent = '✓ Safe';
        statusBadge.className = 'status-badge status-safe';
    }
}

/**
 * last scan time formatting
 */
function formatTime(isoString) {
    if (!isoString) {
        return 'Never';
    }
    const date = new Date(isoString);
    return date.toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' });
}

/**
 * Updates all the UI elements
 */
function updateUI(stats) {
    // Get the values, default to 0 if missing
    const total = stats.totalLinksScanned || 0;
    const high = stats.highRiskCount || 0;
    const medium = stats.mediumRiskCount || 0;
    const safe = stats.lowRiskCount || 0;

    // Update the number displays
    totalCount.textContent = total;
    highCount.textContent = high;
    mediumCount.textContent = medium;
    safeCount.textContent = safe;

    // Update the progress bar widths
    if (total > 0) {
        highBar.style.width = Math.round((high / total) * 100) + '%';
        mediumBar.style.width = Math.round((medium / total) * 100) + '%';
        safeBar.style.width = Math.round((safe / total) * 100) + '%';
    } else {
        highBar.style.width = '0%';
        mediumBar.style.width = '0%';
        safeBar.style.width = '0%';
    }

    // Update the status badge
    updateStatusBadge(high, medium);

    lastScanTime.textContent = total + ' scanned • ' + formatTime(stats.lastScanTime);
}

/**
 * Gets the current stats from the content script
 */
function fetchStats() {
    chrome.tabs.query({ active: true, currentWindow: true }, function(tabs) {
        chrome.tabs.sendMessage(tabs[0].id, { action: 'getStats' }, function(response) {
            if (chrome.runtime.lastError) {
                // content script not loaded
                lastScanTime.textContent = 'Not on supported page';
                return;
            }
            
            if (response && response.stats) {
                updateUI(response.stats);
            }
        });
    });
}

/**
 * Tells the content script to rescan all links
 */
function rescanPage() {
    // Disable button while scanning
    rescanBtn.disabled = true;
    rescanBtn.textContent = '🔄 Scanning';

    chrome.tabs.query({ active: true, currentWindow: true }, function(tabs) {
        chrome.tabs.sendMessage(tabs[0].id, { action: 'rescan' }, function(response) {
            // Waits for the scan to finish
            setTimeout(function() {
                fetchStats();
                rescanBtn.disabled = false;
                rescanBtn.textContent = '🔄 Rescan Page';
            }, 1500);
        });
    });
}

rescanBtn.addEventListener('click', rescanPage);

fetchStats();
```

By storing references to all the DOM elements at the start, I avoid repeatedly calling `document.getElementById()` every time the UI updates. Grab elements once, reuse the references.

---

### Time Formatting

```javascript
function formatTime(isoString) {
    if (!isoString) {
        return 'Never';
    }
    const date = new Date(isoString);
    return date.toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' });
}
```

The content script stores `lastScanTime` as an ISO string which is unreadable to users. This converts it to readable format and if there's no timestamp yet, it returns "Never" as a default.

---

### UI Update Function

```javascript
function updateUI(stats) {
    const total = stats.totalLinksScanned || 0;
    const high = stats.highRiskCount || 0;
    // ...
    
    if (total > 0) {
        highBar.style.width = Math.round((high / total) * 100) + '%';
        // ...
    }
}
```

The `|| 0` pattern provides fallback values in case any stat is undefined. Checking `if (total > 0)` before calculating percentages avoids division by zero which would break the progress bars.

---

### Fetching Stats from Content Script

```javascript
function fetchStats() {
    chrome.tabs.query({ active: true, currentWindow: true }, function(tabs) {
        chrome.tabs.sendMessage(tabs[0].id, { action: 'getStats' }, function(response) {
            if (chrome.runtime.lastError) {
                lastScanTime.textContent = 'Not on supported page';
                return;
            }
            if (response && response.stats) {
                updateUI(response.stats);
            }
        });
    });
}
```

The popup and content script run in separate contexts and can't share variables directly. I used Chrome's messaging API: find the active tab, send a message, content script responds with stats.

---

### Rescan Functionality

```javascript
function rescanPage() {
    rescanBtn.disabled = true;
    rescanBtn.textContent = '🔄 Scanning...';

    chrome.tabs.query({ active: true, currentWindow: true }, function(tabs) {
        chrome.tabs.sendMessage(tabs[0].id, { action: 'rescan' }, function(response) {
            setTimeout(function() {
                fetchStats();
                rescanBtn.disabled = false;
                rescanBtn.textContent = '🔄 Rescan Page';
            }, 1500);
        });
    });
}
```

Disabling the button prevents multiple rapid clicks and the 1.5 second timeout gives it a chance to complete.

---

### Event Binding and Initialization

```javascript
rescanBtn.addEventListener('click', rescanPage);
fetchStats();
```

This attaches the click handler and immediately calls `fetchStats()` to load statistics when the popup opens. No `DOMContentLoaded` listener needed because the script is at the bottom of the HTML.

---

## popup.html Itself

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>PhishBait</title>
    <style>
        body {
            width: 250px;
            padding: 15px;
            font-family: Arial, sans-serif;
            font-size: 14px;
        }
        .header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 15px;
            padding-bottom: 10px;
            border-bottom: 1px solid #ccc;
        }
        .title {
            font-weight: bold;
            font-size: 16px;
        }
        .status-badge {
            font-size: 12px;
        }
        .status-safe { color: green; }
        .status-medium { color: orange; }
        .status-high { color: red; }
        .row {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 8px;
        }
        .bar-container {
            width: 100px;
            height: 8px;
            background: #eee;
            border-radius: 4px;
            margin-left: 10px;
        }
        .bar {
            height: 100%;
            border-radius: 4px;
        }
        .bar-high { background: #dc3545; }
        .bar-medium { background: #ffc107; }
        .bar-safe { background: #28a745; }
        .footer {
            margin-top: 15px;
            padding-top: 10px;
            border-top: 1px solid #ccc;
            font-size: 12px;
            color: #666;
        }
        button {
            width: 100%;
            padding: 8px;
            margin-top: 10px;
            cursor: pointer;
        }
    </style>
</head>
<body>
    <div class="header">
        <span class="title">🎣 PhishBait</span>
        <span id="status-badge" class="status-badge status-safe">✓ Safe</span>
    </div>

    <div class="row">
        <span>🔴 High Risk</span>
        <span id="high-count">0</span>
        <div class="bar-container"><div id="high-bar" class="bar bar-high" style="width: 0%"></div></div>
    </div>

    <div class="row">
        <span>🟡 Medium</span>
        <span id="medium-count">0</span>
        <div class="bar-container"><div id="medium-bar" class="bar bar-medium" style="width: 0%"></div></div>
    </div>

    <div class="row">
        <span>🟢 Safe</span>
        <span id="safe-count">0</span>
        <div class="bar-container"><div id="safe-bar" class="bar bar-safe" style="width: 0%"></div></div>
    </div>

    <div class="footer">
        <span id="total-count">0</span> links • <span id="last-scan-time">Never</span>
    </div>

    <button id="rescan-btn">🔄 Rescan Page</button>

    <script src="popup.js"></script>
</body>
</html>
```

---

## popup.html Structure Breakdown
---

### Inline Styles

```css
body {
    width: 250px;
    padding: 15px;
    font-family: Arial, sans-serif;
}
```

Chrome extension popups don't have a default width. 250px provides enough room for content without being too wide.

---

### Color Classes

```css
.status-safe { color: green; }
.status-medium { color: orange; }
.status-high { color: red; }

.bar-high { background: #dc3545; }
.bar-medium { background: #ffc107; }
.bar-safe { background: #28a745; }
```

These colors match the warning system in styles.css

---

### Progress Bar Structure

```html
<div class="bar-container"><div id="high-bar" class="bar bar-high" style="width: 0%"></div></div>
```

The outer `.bar-container` provides a fixed-width gray background (100%). The inner `.bar` is the colored portion whose width gets set by JavaScript. Setting `style="width: 0%"` inline provides the initial state before JavaScript runs.

---

### Script Placement

```html
    <script src="popup.js"></script>
</body>
```

Placing the script at the bottom guarantees the DOM is loaded before popup.js tries to access elements. This eliminates the need for a `DOMContentLoaded` listener.

---

## styles.css Itself

```css
/**
 * PhishBait Stylesheet
 * Author: Dylan George
 * Purpose: Visual styling for phishing warning indicators
 */

/* High Risk Links - Red warning */
.phishbait-high {
    background-color: rgba(220, 53, 69, 0.2) !important;
    border: 2px solid #dc3545 !important;
    border-radius: 3px !important;
    padding: 1px 4px !important;
    text-decoration: underline wavy #dc3545 !important;
}

/* Medium Risk Links - Yellow/Orange warning */
.phishbait-medium {
    background-color: rgba(255, 193, 7, 0.2) !important;
    border: 2px solid #ffc107 !important;
    border-radius: 3px !important;
    padding: 1px 4px !important;
    text-decoration: underline wavy #ffc107 !important;
}

/* Hidden state */
.phishbait-tooltip-hidden {
    display: none !important;
}

/* Base tooltip styling */
.phishbait-tooltip {
    position: fixed;
    z-index: 999999;
    max-width: 350px;
    padding: 12px 16px;
    border-radius: 6px;
    font-family: 'Segoe UI', Arial, sans-serif;
    font-size: 12px;
    line-height: 1.5;
    white-space: pre-wrap;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    pointer-events: none;
}

/* Tooltip for high risk */
.phishbait-tooltip-high {
    background-color: #dc3545;
    color: white;
    border: 2px solid #a71d2a;
}

/* Tooltip for medium risk */
.phishbait-tooltip-medium {
    background-color: #ffc107;
    color: #333;
    border: 2px solid #d39e00;
}

@keyframes phishbait-pulse {
    0% {
        box-shadow: 0 0 0 0 rgba(220, 53, 69, 0.4);
    }
    70% {
        box-shadow: 0 0 0 6px rgba(220, 53, 69, 0);
    }
    100% {
        box-shadow: 0 0 0 0 rgba(220, 53, 69, 0);
    }
}

/* Apply pulse animation to high-risk links */
.phishbait-high {
    animation: phishbait-pulse 2s ease-in-out infinite;
}
```

---

## styles.css Breakdown

### Tooltip Base Styling

```css
.phishbait-tooltip {
    position: fixed;
    z-index: 999999;
    max-width: 350px;
    padding: 12px 16px;
    border-radius: 6px;
    font-family: 'Segoe UI', Arial, sans-serif;
    font-size: 12px;
    line-height: 1.5;
    white-space: pre-wrap;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    pointer-events: none;
}
```

The tooltip displays analysis details when hovering over flagged links. `position: fixed` ensures it stays in place relative to the viewport. `z-index: 999999` guarantees it appears above Gmail's UI elements. `white-space: pre-wrap` preserves the line breaks in the formatted tooltip text. `pointer-events: none` prevents the tooltip from interfering with mouse interactions.

---

### Tooltip Color Variants

```css
.phishbait-tooltip-high {
    background-color: #dc3545;
    color: white;
    border: 2px solid #a71d2a;
}

.phishbait-tooltip-medium {
    background-color: #ffc107;
    color: #333;
    border: 2px solid #d39e00;
}
```

Tooltips match the link warning colors. High-risk tooltips use white text on red for maximum contrast and urgency. Medium-risk tooltips use dark text on yellow since black-on-yellow is more readable than white-on-yellow.

---

## Content Script Message Handler

Added to the bottom of content.js to enable popup communication:

```javascript
// Listen for messages from the popup
chrome.runtime.onMessage.addListener(function(request, sender, sendResponse) {
    if (request.action === 'getStats') {
        sendResponse({ stats: scanStats });
    } else if (request.action === 'rescan') {
        scanAllLinks();
        sendResponse({ success: true });
    }
    return true;
});
```

The `return true` keeps the message channel open for asynchronous responses.

---

## Manifest.json Update

Added the action block to enable the popup:

```json
"action": {
    "default_popup": "popup.html"
}
```
