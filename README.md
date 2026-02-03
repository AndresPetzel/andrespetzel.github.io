# Tracking Implementation Fix: CheckoutChamp Architecture
## Candidate Submission for eJam | Tracking Engineer

### 📋 Executive Summary
This report details the remediation of the pixel over-firing issue within the CheckoutChamp (CHAMP) funnel. The implemented solution successfully resolves a **17% CPA discrepancy** ($116.44 reported vs. $136.44 actual) by ensuring event idempotency. This fix provides Meta's optimization algorithms with clean, accurate data, directly improving ROAS calculations and scaling decisions.

### 🧐 Root Cause Analysis
An audit of the `legacy-broken` implementation revealed that the inflated conversion data was caused by:
1. **Uncontrolled Native Firing:** The `index.js` funnel configuration had the native `Purchase_Event_FB` key set to `"2"`. This triggered the Purchase event automatically on every DOM load of the Thank You page.
2. **State Blindness:** The platform architecture, which relies on redirects and potential page refreshes, lacked a mechanism to prevent duplicate signals when users reloaded the page, navigated back, or accessed the receipt from multiple tabs.

### 🛠️ Technical Implementation & CHAMP-Specific Fix
The solution follows a "State-Aware" architecture designed to handle CHAMP’s specific redirect and reload behavior:

1. **Automation Bypass:** Native auto-firing was disabled in `index.js` by setting the `keyValue` to `"0"`.
2. **Idempotent Tracking Script:** A custom JavaScript block was engineered and injected into `thank_you.html`.
3. **Logic Flow:**
   - **Data Extraction:** The script retrieves the `orderId` from URL parameters and the `grandTotal` value directly from the DOM (element ID `grandTotal`).
   - **State Verification:** Before firing, the script checks `localStorage` for a unique key tied to the specific transaction (e.g., `fb_purchase_sent_ORDER123`).
   - **Execution:** The `fbq('track', 'Purchase')` event is only triggered if the key is missing.
   - **Lock Persistence:** Once fired, the script sets the key in `localStorage` to "true".

**Why LocalStorage?** Unlike `sessionStorage`, `localStorage` persists across multiple tabs and browser restarts. This is critical for CHAMP environments where users frequently open confirmation links in new windows or return to the page later.

### ✅ QA & Validation Steps (Quality Assurance)
The `stable-implementation` was stress-tested under the following scenarios:
* **Standard Funnel Completion:** Event fires exactly once with the correct dynamic value.
* **Thank You Page Reload (F5):** The script successfully detects the existing storage lock and suppresses duplicate events.
* **Network Throttling:** Validated under "Slow 3G" conditions to ensure the storage lock is established before secondary page triggers occur.
* **Cross-Tab Collision:** Confirmed that opening the Thank You URL in a separate browser window does not trigger a second conversion.
* **Mobile Safari:** Verified persistence behavior on iOS Safari to ensure compatibility with CHAMP’s high mobile traffic volume.

### 📈 Monitoring Plan
To ensure long-term data reliability and prevent future discrepancies:
1. **Discrepancy Alerting:** Implement a daily automated audit comparing CRM total orders vs. Meta Purchase events. Trigger an alert if the delta exceeds **10%**.
2. **Event ID Deduplication:** Ensure the `orderId` is passed as the `event_id` parameter to allow Meta’s servers to perform secondary deduplication.
3. **Pixel Uptime Monitoring:** Deploy monitoring to detect if the Base Code fails to load or if the custom tracking script encounters runtime errors.

---
**Author:** Andrés Petzel
**Role:** Tracking Implementation Engineer Candidate
