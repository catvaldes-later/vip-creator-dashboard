# VIP Creator Intelligence Dashboard — Operational Runbook

**Owner:** Cat Valdes
**Dashboard URL:** https://catvaldes-later.github.io/vip-creator-dashboard/
**Live File:** `index.html` (single-file React 18 app, GitHub Pages)
**Last Updated:** 2026-02-27

---

## 1. Architecture Overview

The dashboard is a **single `index.html` file** containing:
- All data embedded as JavaScript constants (no external API calls at runtime)
- Pre-compiled `React.createElement` calls — **NO JSX, NO Babel**
- Inline styles using a theme object `M`
- Served via GitHub Pages from `catvaldes-later.github.io/vip-creator-dashboard/`

### Data Constants (in order of appearance in file)
| Constant | Source | Description |
|---|---|---|
| `HUBSPOT_CREATORS_RAW` | HubSpot API | VIP creator deal records |
| `HUBSPOT_PIPELINE_RAW` | HubSpot API | Pipeline/deal stage data |
| `WALMART_CREATOR_CATEGORIES` | Domo CSV export | Per-creator top 3 Walmart categories by revenue (all-time) |
| `WALMART_CATEGORIES` | Domo CSV export | Network-wide Walmart category GMV, 6 months |

### Tabs
| Tab | Content |
|---|---|
| Dashboard | KPI cards, pipeline funnel, creator table |
| Walmart HQ | Creator leaderboard (with per-creator top 3 categories) + Top Categories by GMV card |
| Creator Rolodex | Searchable creator detail cards |

---

## 2. Data Sources & Pull Procedures

### 2a. HubSpot Creator Data
- **Source:** HubSpot API (deals pipeline)
- **How:** Export from HubSpot or pull via API
- **Format:** JSON array of deal objects
- **Key fields:** Creator name, GMV metrics, conversion rate, risk level, onboarding status, deal stage

### 2b. Walmart Creator Category Data (Domo)
- **Source:** Domo — "Walmart Category and Creator Data" report
- **How:** Scheduled Domo email report → CSV attachment arrives in Gmail
- **CSV columns:** `Walmart.category`, `User.id`, `User.name`, `Walmart.GMV`, `Walmart.Revenue`
- **Granularity:** Per-creator, per-category (all-time totals, no date dimension)

#### Processing Rules:
1. Parse CSV, group by `User.name`
2. For each creator, sort categories by `Walmart.Revenue` descending
3. Keep top 3 categories per creator
4. **ALWAYS EXCLUDE "Food" category** — Mavely earns no commission on Food
5. Match creator names to VIP dashboard names (case-insensitive, first+last fuzzy matching)
6. Store as `WALMART_CREATOR_CATEGORIES` object: `{"Creator Name": ["Cat1", "Cat2", "Cat3"], ...}`
7. Only include creators that match VIP names in the dashboard (keeps constant small)

#### Name Matching Notes:
- HubSpot uses `firstname + " " + lastname` from deal properties
- Domo uses `User.name` which may differ in formatting
- Match by: exact lowercase match first, then first+last name match
- ~30% of VIP creators may not match due to different display names across systems — this is expected. Show blank for unmatched.

#### Known Name Aliases (Domo → HubSpot):
| Domo Name | HubSpot Name |
|---|---|
| Taylor Moore | Taylor Mitchell |

*Add new aliases here as they're discovered.*

### 2c. Walmart Network Category GMV (Domo)
- **Source:** Domo dashboard "Walmart Categories / Products - Weekly Reporting" at `mavely-life.domo.com`
- **How:** Scheduled Domo email report → CSV attachment arrives in Gmail
- **CSV columns:** `Category`, `Date`, `GMV`
- **Granularity:** Network-wide aggregated (NOT per-creator)
- **Date format in CSV:** `YYYY-Mon` (e.g., `2026-Feb`)

#### Processing Rules:
1. Parse CSV, group by Category
2. Keep only the **last 6 months** of data
3. Sort categories by most recent month's GMV descending
4. **ALWAYS EXCLUDE the "Food" category** — Mavely does not earn commission on Food
5. Store as `WALMART_CATEGORIES` constant in index.html

### 2d. Chorus Call Logs
- **Source:** Chorus via Gmail
- **How:** Pull from Gmail search

### 2e. Onboarding Status
- **Source:** HubSpot meetings data, cross-referenced with Managed VIP deals
- **How:** Check meeting activity against deal records

---

## 3. Code Modification Procedures

### 3a. Updating Data Constants

When replacing a data constant (e.g., `WALMART_CATEGORIES`):

1. **Find the exact constant** using grep: `grep -n "const WALMART_CATEGORIES" index.html`
2. **Identify the start and end lines** of the constant
3. **Replace the entire constant** with new data
4. **Run syntax verification** (see Section 5) BEFORE uploading

### 3b. Adding New UI Components

When inserting new React components into the tab structure:

1. **Count parens before editing.** Use acorn tokenizer to get the paren balance at the insertion point
2. **After inserting, verify paren balance is exactly 0** using acorn parser
3. **Never trust `node --check` alone** — see Known Pitfall #1

### 3c. Modifying Existing Components

1. Read the surrounding 50 lines of context before and after the edit target
2. Make surgical edits — don't rewrite large blocks unnecessarily
3. Run full syntax verification after every edit

---

## 4. Deployment Procedure

1. Save updated `index.html` to outputs folder
2. Run syntax verification (Section 5)
3. Upload to GitHub repository
4. **Hard refresh required** — GitHub Pages caches aggressively
   - Cmd+Shift+R on Mac, Ctrl+Shift+R on Windows
   - Or test in Incognito/Private window
5. Verify the live site loads and all tabs render correctly

---

## 5. Syntax Verification (MANDATORY)

**Every single edit to index.html MUST pass this verification before being shared or uploaded.**

### Step 1: Extract the correct script

```javascript
// CORRECT — matches the main inline script (no attributes on <script> tag)
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)];
const main = scripts.reduce((a, b) => a[1].length > b[1].length ? a : b);
```

### Step 2: Parse with acorn

```javascript
require('acorn').parse(mainScript, { ecmaVersion: 2020, sourceType: 'script' });
```

Both steps must succeed. If acorn reports an error, **do not deliver the file.**

---

## 6. KNOWN PITFALLS — Do Not Repeat

### Pitfall #1: node --check regex matches wrong script tag
- **What happened:** Used `/<script[^>]*>([\s\S]*?)<\/script>/` which matched CDN `<script src="...">` tags (React, ReactDOM) instead of the main inline `<script>` block. `node --check` reported "SYNTAX OK" on a 200-byte CDN stub while the 500KB+ main script had a syntax error.
- **Rule:** ALWAYS use `/<script>([\s\S]*?)<\/script>/` (no `[^>]*`) and select the **largest** match. Or better yet, use acorn to parse the extracted script.
- **Severity:** Critical — caused a blank dashboard to be shipped to production.

### Pitfall #2: Missing closing parentheses when inserting new Cards
- **What happened:** Inserted a new `Card` component between two existing sections. The closing paren sequence `})))))),` was short by 1 `)`, causing `SyntaxError: missing ) after argument list`.
- **Rule:** After any insertion between React.createElement blocks, use the acorn tokenizer to verify paren balance = 0. Count the opens and closes in the inserted block explicitly.
- **Tip:** In this codebase, deeply nested `React.createElement` chains commonly have 6-9 closing parens in a row. One missing paren = blank screen, no error visible to the user.
- **Severity:** Critical — caused production outage (blank screen).

### Pitfall #3: Food category must always be excluded
- **What happened:** Initial Walmart Categories build included Food. Mavely does not earn commission on Food sales.
- **Rule:** When processing the Domo Walmart Category CSV, ALWAYS filter out rows where `Category === "Food"` before building the constant.
- **Severity:** Medium — incorrect data displayed to stakeholders.

### Pitfall #4: Wrong Domo navigation for data export
- **What happened:** Directed user to "Data Science" section in Domo to find raw datasets. That section is for ML/AI tools, not data management or export.
- **Rule:** To export data from Domo, use the export options directly on dashboard cards (kebab menu → Export), or schedule email reports from the dashboard itself. Do NOT send users to Data Science, Workflows, or other admin sections.
- **Severity:** Low — wasted user time, no data impact.

### Pitfall #5: Network-wide data is not per-creator data
- **What happened:** User asked for per-creator top 3 Walmart categories. The first Domo CSV (`GMV_GrowthByMonth.csv`) only had `Category, Date, GMV` — network-wide aggregated data with no creator/influencer column. Instead of flagging this as the wrong dataset, built a network-wide categories card and moved on. User had to point out the mistake.
- **Rule:** When the user requests per-creator data, ALWAYS verify the CSV has a creator/influencer/user column BEFORE building anything. If the data is aggregated (no creator dimension), immediately tell the user this isn't the right export and help them find the correct one.
- **The correct CSV for per-creator data:** "Walmart Category and Creator Data" — columns: `Walmart.category`, `User.id`, `User.name`, `Walmart.GMV`, `Walmart.Revenue`
- **Severity:** Medium — built the wrong feature, wasted time, confused the user.

### Pitfall #6: Browser caching after GitHub Pages deploy
- **What happened:** User uploaded new index.html but saw the old version. GitHub Pages serves cached content aggressively.
- **Rule:** Always remind the user to hard refresh (Cmd+Shift+R) or check in Incognito after uploading to GitHub.
- **Severity:** Low — causes confusion but no data risk.

---

## 7. Pre-Flight Checklist (Run Before Every Refresh)

Use this checklist every time you modify index.html:

- [ ] **Data constants updated** with fresh data from all sources
- [ ] **Food category excluded** from WALMART_CATEGORIES
- [ ] **Syntax verified** using acorn parser (NOT just node --check)
- [ ] **Correct script extraction regex** used: `/<script>([\s\S]*?)<\/script>/g` → largest match
- [ ] **Paren balance = 0** confirmed by acorn (no unclosed parentheses)
- [ ] **All tabs tested mentally**: Dashboard, Walmart HQ, Creator Rolodex — does the component tree close properly?
- [ ] **File saved** to outputs folder
- [ ] **User reminded** to hard refresh after GitHub upload

---

## 8. Scheduled Task Configuration

| Field | Value |
|---|---|
| Task ID | `nightly-domo-dashboard-refresh` |
| Schedule | `0 7 * * *` (7:00 AM daily, local time) |
| Enabled | Yes |

### Task Execution Order:
1. **Read this runbook first** — check for any new pitfalls or changed procedures
2. Pull Domo report CSV from Gmail (Walmart Category GMV)
3. Pull HubSpot creator data
4. Pull Chorus call logs from Gmail
5. Pull onboarding status from HubSpot meetings
6. Process all data, build updated constants
7. Update index.html with new constants
8. Run syntax verification (Section 5)
9. Run pre-flight checklist (Section 7)
10. Push to GitHub Pages
11. DM Cat on Slack with sync status

---

## 9. Open Items / Future Work

| Item | Status | Notes |
|---|---|---|
| Per-creator Walmart category data | DONE | Uses "Walmart Category and Creator Data" Domo CSV. Top 3 categories by revenue per creator, displayed as pills in Walmart HQ leaderboard. ~30% name mismatch between HubSpot and Domo is expected. |
| Managed VIP cross-reference for onboarding | In progress | Need to cross-reference HubSpot meeting activity with Managed VIP deal records |
| Automated GitHub push | Not yet set up | Currently manual upload by Cat |

---

*This runbook is a living document. Every time a mistake is made or a new procedure is learned, add it here so it never happens again.*
