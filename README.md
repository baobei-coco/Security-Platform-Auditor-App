# Security Platform Auditor

A self-contained, browser-based audit tool for security platform configurations. Runs entirely client-side — no server, no dependencies, no data transmission. Open `index.html` and go.

**Live:** https://baobei-coco.github.io/Security-Platform-Auditor-App

---

## What it does

The auditor ingests configuration exports from your security platforms and runs automated checks against them, surfacing misconfigurations, coverage gaps, overlapping rules, and policy hygiene issues. Each finding includes a severity rating, contextual detail, and a remediation recommendation.

Each platform is scored 0–100 based on the weighted severity of detected issues. Scores are graded A–F and displayed on a per-module dashboard.

### Supported platforms

| Platform | Modules |
|---|---|
| Proofpoint | CASB, Email Security, DLP, TAP, Users, Data Security WB, Isolation, Human Risk, Risk Config, Data Classification, Admin |
| Microsoft Purview | DLP Policies, Sensitivity Labels, Retention, Insider Risk, Compliance, Admin |
| CrowdStrike | Prevention, Detection, Response, IOA, Admin |
| Zscaler | URL Filtering, App Segmentation, DLP, Firewall, Admin |
| Symantec DLP | Policies, Response, Classifiers, Admin |
| Forcepoint DLP | Policies, Classifiers, Destinations, Admin |
| Microsoft Defender | Endpoint, Email, Cloud App, ASR, Admin |
| Splunk / SIEM | Searches, Alerts, Indexes, Notables, Admin |
| SentinelOne | Policies, Exclusions, Threats, Admin |
| Okta | Policies, MFA, Apps, Users, Admin |
| Azure / Entra ID | RBAC, Conditional Access, NSG, Storage, Admin |
| OORT | Identities, Dormant Accounts, Anomalies, Admin |
| Google Workspace | Sharing, OAuth, Users, Audit, Admin |
| ExtraHop | Detections, Triggers, Traffic, Admin |
| AWS | IAM, Security Hub, GuardDuty, S3, Admin |

---

## How to use it

### Demo mode

1. Open `index.html` in any modern browser (or navigate to the live URL above)
2. Click any platform card on the landing page
3. Click **⚡ Load All Demo** on the platform dashboard — this runs the full audit against built-in synthetic data
4. Navigate modules via the left sidebar
5. Click any audit section to expand findings

### With real data

1. Open the platform, click **📥 Import Data** on the dashboard
2. Select the module tab you want to load
3. Paste a JSON array or CSV, or upload a file directly
4. Click **Import & Save**

Imported data is saved to `localStorage` and automatically reloaded on your next visit. A green dot appears next to each module with stored data.

---

## Importing real data

### Supported formats

**JSON array** — a flat array of objects, one per record (rule, policy, user, etc.):
```json
[
  { "id": "r01", "name": "Block SSN Email", "enabled": true, "action": "Block" },
  { "id": "r02", "name": "Monitor All", "enabled": true, "action": "" }
]
```

**JSON object with module keys** — auto-splits across multiple modules in one import:
```json
{
  "policies": [ ... ],
  "users":    [ ... ],
  "admin":    [ ... ]
}
```
The keys must match the module IDs for that platform (visible in the import modal tabs). Matched modules are loaded simultaneously; unmatched keys are ignored.

**CSV** — header row required, imports into whichever module tab is currently active:
```
id,name,enabled,action
r01,Block SSN Email,true,Block
r02,Monitor All,true,
```

### Field names

The `analyze()` function for each module reads specific field names from the imported data. If your export uses different field names, you have two options:

- Rename the columns before importing (easiest for CSV)
- Edit the `analyze()` function for that module in `index.html` to match your field names

The demo data in `DEMO_DATA` at the top of `index.html` shows the exact field shapes expected by each module's analyzer — use those as a reference.

### Resetting to demo data

In the import modal, click **🗑 Reset all modules** to clear all stored data for that platform and revert to demo data. To reset a single module, click **Reset to demo** in the green stored-data bar above the module view.

---

## How to add a new platform

All platform data is defined in three places in `index.html`. Search for each section by its comment header.

### 1. Add to `DEMO_DATA`

At the top of the `<script>` block, add a new key under `DEMO_DATA`:

```js
DEMO_DATA['myplatform'] = {
  policies: [
    { id: 'p01', name: 'Block External Share', enabled: true, action: 'Block' },
    // ...
  ],
  admin: [
    { id: 'a01', name: 'Admin User', mfa: false },
  ]
};
```

### 2. Add to `PLATFORMS`

In the `PLATFORMS` object, add your platform definition:

```js
myplatform: {
  name: 'My Platform',
  icon: '🔒',           // emoji or inline SVG string
  color: '#e91e8c',
  cls: 'platform-myplatform',
  desc: 'Short description for the landing card',
  modules: [
    { id: 'dash',     icon: '⬡', label: 'Dashboard' },
    { id: 'policies', icon: '📋', label: 'Policies' },
    { id: 'admin',    icon: '🔧', label: 'Administration' },
  ]
},
```

Also add a CSS accent rule near the top of the `<style>` block:
```css
.platform-card.myplatform::before { background: linear-gradient(135deg, rgba(233,30,140,.08), transparent); }
```

### 3. Add to `MODULE_DEFS`

For each module (except `dash`, which is auto-generated), add `analyze()` and `render()` functions:

```js
myplatform: {
  policies: {
    analyze(data) {
      const issues = { noAction: [], disabled: [] };
      data.filter(r => !r.enabled).forEach(r =>
        issues.disabled.push({
          sev: 'warning',
          name: r.name,
          detail: 'Policy is disabled.',
          tags: [{ v: 'DISABLED', c: 'y' }],
          rec: 'Re-enable or remove.'
        })
      );
      return makeResult(issues, calcScore(issues, { disabled: 5 }), data, 'policies');
    },
    render(pid, mid, result) {
      wrapRender(pid, mid, result, 'Policy audit',
        [{ n: result.policies.length, l: 'Policies', c: 'blue' }],
        [{ id: `${pid}-${mid}-dis`, icon: '⭕', title: 'Disabled', desc: 'Off policies', issues: result.issues.disabled }]
      );
    }
  }
}
```

### 4. Add the landing page card

In the `platform-grid` div in the HTML body:

```html
<div class="platform-card myplatform" onclick="launchPlatform('myplatform')">
  <div class="pc-icon">🔒</div>
  <div class="pc-name">My Platform</div>
  <div class="pc-desc">Short description</div>
  <div class="pc-modules">2 modules · Full audit</div>
  <div class="pc-accent myplatform"></div>
</div>
```

### 5. Add to the platform switcher dropdown

In the `ps-dropdown` div:

```html
<div class="ps-option" data-p="myplatform" onclick="switchPlatform('myplatform')">
  <span class="pso-icon">🔒</span>
  <div><div class="pso-name">My Platform</div><div class="pso-sub">2 modules</div></div>
  <span class="ps-badge" id="psb-myplatform"></span>
</div>
```

---

## Code structure

```
index.html
├── <style>
│   ├── :root {}              CSS variables — all theme colors defined here
│   └── ...                   Component styles
└── <script>
    ├── DEMO_DATA {}          Sample records for every platform/module
    ├── PLATFORMS {}          Platform metadata (name, icon, modules list)
    ├── Storage helpers       localStorage read/write (getData, saveToStorage, etc.)
    ├── Utility functions     esc(), scoreGrade(), calcScore(), makeResult()
    ├── Render utilities      wrapRender(), buildSection(), renderIssue()
    ├── Orchestration         launchPlatform(), switchModule(), processModule()
    ├── Import modal          openImportModal(), applyImport(), parseImportInput()
    └── MODULE_DEFS {}        analyze() + render() for every platform/module
```

### Severity levels

| Value | Meaning | Score impact |
|---|---|---|
| `critical` | Active misconfiguration, exploitable gap | High (weight × 3) |
| `warning` | Hygiene issue, incomplete config | Medium (weight × 1) |
| `info` | Coverage gap, best practice deviation | Low (weight × 0.3) |

### Score formula

`calcScore(issues, weights)` — for each issue category, deducts `weight × count × severity_multiplier` from 100, floored at 0.

---

## Contributing

Since this is a single-file app, all changes go into `index.html`.

**Workflow:**
1. Edit `index.html` locally
2. Open in browser and click **⚡ Load All Demo** to verify existing platforms still work
3. Test your new platform/module with demo data, then with a real export if available
4. Commit and push to `main` — GitHub Pages deploys automatically

**To update the live site:**
- Go to the repo → click `index.html` → pencil icon → edit → commit, or
- Use **Add file → Upload files** to drag and replace `index.html` directly

---

## Notes

- All data stays in the browser. Nothing is sent anywhere.
- `localStorage` is per-browser and per-origin — imported data on one machine won't appear on another.
- The file is ~300KB. GitHub Pages serves it with no issues; avoid opening via `file://` on some browsers as localStorage may be restricted.
- Dark Reader and similar browser extensions can interfere with SVG rendering — the app includes `<meta name="darkreader-lock"/>` to prevent this.
