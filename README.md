# KnowBe4 Training Report — Google Apps Script

Automated daily training/assessment report for KnowBe4 campaigns, generated entirely inside a Google Sheet. No Python, no local install, no servers. It pulls data straight from the KnowBe4 API, splits users into per-site tabs, and builds a summary — refreshable on demand or on a daily schedule.

A reusable template: clone it, swap in your own site names and API key, and it works. See **[Adapting this template to your organization](#adapting-this-template-to-your-organization)**.

---

## What it does

On each refresh, the script:

1. Pulls fresh **enrollment status** from the KnowBe4 API for a chosen campaign.
2. Enriches each record with **user profile data** (location, manager, groups), cached to stay fast.
3. Filters out vendor/external accounts.
4. Splits users into **per-site tabs** based on their location.
5. Writes a **SUMMARY** tab with completion ratios per site and an all-sites total.
6. Stamps each sheet with **last-updated timestamps** so viewers know if the data is current.

Everything overwrites in place, so the same sheet stays up to date.

---

## Output structure

**SUMMARY tab** — site-by-site overview:

| Column | Meaning |
|--------|---------|
| Tab | Short site code (HQ, SITE_A, SITE_B, …) |
| Company | Full company name |
| Total Users | Unique users in that site |
| (module columns) | Passed / Total per training module |
| Completion Ratio % | Overall passed ÷ total for the site |

The top two rows show **Last status update** and **Last user data update**. The bottom row is an **ALL** grand total across every site (vendor/external users excluded).

**Per-site tabs** (HQ, BR, SITE_A, SITE_B, … — whatever you define in `SITE_RULES`) — 8 columns, one row per user per module:

```
Email | First Name | Last Name | Manager Name | Manager Email | Location | Content | Status
```

Status cells are color-coded: green for Passed, red for Past Due.

---

## Setup (one-time, ~5 minutes)

1. Create a new Google Sheet.
2. **Extensions → Apps Script**.
3. Delete the default code, paste in `kb4_report_apps_script.js`, and **Save** (Ctrl+S).
4. Click the gear icon (**Project Settings**) → **Script Properties** → add:
   - `KB4_API_KEY` = your KnowBe4 API key
   - `KB4_CAMPAIGN_ID` = the campaign ID (or leave blank to auto-detect the first active campaign)
5. **Approve permissions (the step people miss):** in the Apps Script editor, set the function dropdown to `refreshReport` and click **Run**. Approve the Google permission popup. This only happens once.
6. Reload the Google Sheet — a new **KnowBe4 Report** menu appears.
7. Click **KnowBe4 Report → Refresh Now**.

> **🔑 API key — never hardcode it.** The key goes in **Script Properties** (`KB4_API_KEY`), never inline in the source. Keeping it out of the code means you can safely commit / publish this script. The script reads it via `PropertiesService.getScriptProperties()`.
>
> **Where to find your API key:** KnowBe4 console → Account Settings → API → copy the key.
> **EU tenant?** In the script's `apiGet` function, change `us.api.knowbe4.com` to `eu.api.knowbe4.com`.
> **Timezone:** Project Settings → Time zone → set to your local zone (e.g. `America/New_York`) so timestamps display correctly.

---

## The menu

| Item | What it does | When to use |
|------|--------------|-------------|
| 🔄 **Refresh Now** | Rebuilds the report — fresh status, cached user profiles | Daily / anytime you want current numbers |
| 👥 **Refresh User Data** | Clears the user-profile cache so the next refresh re-pulls it | After adding employees or changing locations/managers |
| ⚙️ **Set API Key** | Stores the API key via a prompt | If you'd rather not use Script Properties |

After **Refresh User Data**, run **Refresh Now** to actually re-pull and rebuild.

---

## Adapting this template to your organization

Everything you need to change lives near the **top of the script**. The matching logic below it stays untouched.

### 1. `SITE_RULES` — your site → tab mapping

This is the main thing to edit. Each entry maps users to a tab:

```javascript
["TAB", "Full Company Name", ["KEYWORD1", "KEYWORD2"], locationOnly]
```

- **TAB** — short tab code (e.g. `HQ`, `SITE_A`). Keep it 2–8 characters, uppercase, letters/digits/`_`/`-`.
- **Full Company Name** — shown in the SUMMARY tab's *Company* column.
- **Keywords** — case-insensitive substrings. A user matches the rule if any keyword appears in their Location (or Group, see below).
- **locationOnly** — `true` = match on the Location field **only**; `false` = match on Location first, then fall back to Group.

**Order matters — first match wins.** Two conventions to follow:

- **Put `locationOnly: true` office rules FIRST.** Use these for a headquarters or shared office that should win over company membership — anyone physically there lands in that tab regardless of which company they belong to. (The Group field often holds catch-all values like *"All HQ Office Users"*, which is why these rules ignore Group.)
- **Put broad rules — like a parent/holding company — LAST,** so specific sites get matched before the catch-all.

The template ships with placeholders you replace:

```javascript
const SITE_RULES = [
  ["HQ",  "Headquarters Office (All Companies)", ["HEADQUARTERS", "HQ CITY"], true ],
  ["BR",  "Branch Office",                       ["BRANCH CITY"],            true ],

  ["SITE_A", "Company A", ["SITE_A", "COMPANY A KEYWORD"], false],
  ["SITE_B", "Company B", ["SITE_B", "COMPANY B KEYWORD"], false],
  // … add one row per site …

  ["GROUP_HO", "Parent Group (Holding)", ["GROUP", "HOLDING", "CORPORATE"], false],
];
```

Add or remove rows freely — the logic doesn't care how many there are.

### 2. `EXCLUDED_TABS` — tabs to drop from output

```javascript
const EXCLUDED_TABS = ["GROUP_HO", OTHER_TAB];
```

Tabs listed here are computed but **not rendered**. By default that's the parent/holding bucket and the `OTHER` catch-all (which collects anyone who matched no rule). Remove a name from this list if you *do* want that tab to appear.

> Note: if your tab codes contain underscores (like `SITE_A`), keep the underscore in the cleanup regex inside `cleanupUnusedSheets` (it's already included in this template). That regex is what removes stale auto-generated tabs on each run.

### 3. `EXCLUDED_EMAILS` — vendor / external accounts

```javascript
const EXCLUDED_EMAILS = ["vendor@example.com", "another@vendor.com"];
```

These addresses never appear in the report. Two exclusion mechanisms exist:

- **Prefix rule:** any email whose local part starts with **`V_`** (e.g. `V_contractor@…`) is dropped automatically — a common convention for external/vendor accounts.
- **Explicit list:** the addresses in `EXCLUDED_EMAILS`.

> Only the `V_` prefix (uppercase letter + underscore) is filtered. A name like `victor_lee@example.com` is kept.

### 4. Script Properties (not in the code)

- `KB4_API_KEY` — **required.** Store it here, never in the source.
- `KB4_CAMPAIGN_ID` — optional; blank = auto-detect the first active campaign.

### 5. Region & timezone

- **EU tenant:** change `us.api.knowbe4.com` → `eu.api.knowbe4.com` in `apiGet`.
- **Timezone:** Project Settings → Time zone → your local zone.

---

## Scheduled daily auto-refresh

In the Apps Script editor, click the clock icon (**Triggers**) → **Add Trigger**:

- Function: `refreshReport`
- Event source: Time-driven
- Type: Day timer
- Time: e.g. 7am–8am

The sheet then refreshes automatically each morning. Share the sheet link with site admins and they always see the latest data.

---

## How the caching works

User profile data (location, groups, manager) changes rarely, but status changes constantly. So:

- **Status** is fetched fresh on every refresh.
- **User profiles** are fetched from the `/users` endpoint and cached in a hidden `_USERCACHE` sheet, re-fetched only when older than `USER_CACHE_HOURS` (default 24).

This keeps refreshes fast: the first run of the day pays the user-fetch cost, and later refreshes are quick.

To change the cache window, edit near the top of the script:

```javascript
const USER_CACHE_HOURS = 24;   // 168 = weekly, 1 = hourly
```

---

## Site grouping logic

Each user is assigned to a tab by a **two-pass match**:

1. **Location pass** — the user's Location is checked against every site rule, in order. A real location (a named site office) wins here regardless of group membership.
2. **Group pass** — only if Location matched nothing, the Group field is checked (skipping `locationOnly` rules like `HQ`/`BR`, which are location-only).

Special rule: **the HQ office wins over company.** Anyone whose location is the headquarters city office goes to the **HQ** tab regardless of company. A second office (e.g. `BR`) works the same way. Everyone else goes to their company's site tab.

To adjust the rules, edit the `SITE_RULES` array (see [Adapting this template](#adapting-this-template-to-your-organization)). Each entry is:

```javascript
["TAB", "Full Company Name", ["KEYWORD1", "KEYWORD2"], locationOnly]
```

- Keywords are matched as case-insensitive substrings.
- `locationOnly: true` means the rule only matches on Location, never Group.
- Order matters — first match wins, so put broad/priority rules first.

---

## Excluded accounts

These never appear in the report:

- Any email whose local part starts with **`V_`** (vendor/external accounts).
- Specific vendor accounts listed in `EXCLUDED_EMAILS` (placeholder: `vendor@example.com`).

To exclude more accounts, add them to `EXCLUDED_EMAILS`:

```javascript
const EXCLUDED_EMAILS = ["vendor@example.com", "another@vendor.com"];
```

> Note: only the `V_` prefix (with underscore) is filtered. A name like `victor_lee@example.com` is kept.

---

## Troubleshooting

| Symptom | Cause / Fix |
|---------|-------------|
| `Cannot call SpreadsheetApp.getUi() from this context` | You ran `onOpen` directly. Switch the dropdown to `refreshReport` and run that. `onOpen` runs automatically — ignore this error. |
| Clicking the menu does nothing | Permissions not approved. Run `refreshReport` once from the editor and approve the popup. |
| `You can't create a filter in a sheet that already has a filter` | Fixed in current version (`getOrCreateSheet` removes old filters). Make sure you're on the latest script. |
| All users land in OTHER / blank Location & Manager | The `/users` enrichment isn't returning data. Check the API key is valid and the account is the right region (US vs EU). |
| Users in the wrong tab | A user's Location is blank and their Group put them somewhere unexpected. Check that user's profile in KnowBe4, then **Refresh User Data**. |
| Timestamps show wrong time | Set the script timezone in Project Settings. |

For a deeper look at what the API returns, add a small helper that logs a sample of `/users` and `/training/enrollments` records to the execution log.

---

## Configuration quick reference

All near the top of the script:

| Setting | Default | Purpose |
|---------|---------|---------|
| `SITE_RULES` | (placeholder rules) | Location/group → tab mapping — **edit this** |
| `EXCLUDED_TABS` | `["GROUP_HO", OTHER_TAB]` | Tabs computed but not rendered |
| `EXCLUDED_EMAILS` | `["vendor@example.com"]` | Specific addresses to drop |
| `USER_CACHE_HOURS` | `24` | How long user profiles are cached |
| `STATUS_LABELS` | KB4 → display | Status text shown in the report |

Script Properties (Project Settings):

| Property | Purpose |
|----------|---------|
| `KB4_API_KEY` | KnowBe4 API key (**required — store here, never hardcode**) |
| `KB4_CAMPAIGN_ID` | Target campaign (blank = auto-detect) |

---

## Requirements

- A Google account with access to Google Sheets / Apps Script.
- A KnowBe4 account with API access enabled and an API key.
- The KnowBe4 API permissions to read training campaigns, enrollments, and users.

---

## License

MIT (or your preferred license). This is a generic template with no affiliation to any specific organization or to KnowBe4, Inc.
