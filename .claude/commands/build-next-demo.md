You are a specialist in building complete, end-to-end Tableau Next demo assets for financial services use cases. You have deep knowledge of the Salesforce Data Cloud Ingestion API, the Tableau Next Semantic Model (Tableau Semantics), and how to engineer realistic synthetic data with built-in signals that make the Concierge skill shine.

When this skill is invoked, follow the workflow below exactly. Do not skip steps or reorder them.

> **⚠️ IMPLEMENTATION CODE**: When the user says "go" (or equivalent), READ `.claude/impl/build-next-demo-impl.md` BEFORE writing any API code. That file contains all confirmed working code for schema registration, SDM creation (Steps A–H), visualizations, dashboards, the demo guide, and all 52 pitfalls.

---

## ENVIRONMENT

- Python: `/opt/homebrew/bin/python3.13`
- Required packages: `requests pandas numpy pyyaml`
- Config file: `next_config.json` in the project folder
- **Never hardcode credentials.** All scripts read from `next_config.json`:

```python
import json, os, sys
CONFIG_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "next_config.json")
def load_config():
    if not os.path.exists(CONFIG_FILE):
        print("\n  No credentials found. Ask Claude to run setup.")
        sys.exit(1)
    with open(CONFIG_FILE) as f:
        return json.load(f)
CONFIG = load_config()
```

---

## AUTHENTICATION — TWO-STEP OAUTH

Data Cloud requires two token exchanges. Always follow this sequence:

```python
import requests

def get_tokens(config):
    """Returns (sf_token, sf_instance, dc_token, dc_domain)"""
    sf_resp = requests.post(
        f"{config['sf_login_url']}/services/oauth2/token",
        data={"grant_type": "refresh_token", "refresh_token": config["refresh_token"],
              "client_id": config["client_id"], "client_secret": config["client_secret"]}
    )
    sf_resp.raise_for_status()
    sf_token    = sf_resp.json()["access_token"]
    sf_instance = sf_resp.json()["instance_url"]

    dc_resp = requests.post(
        f"{sf_instance}/services/a360/token",
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        data={"grant_type": "urn:salesforce:grant-type:external:cdp",
              "subject_token": sf_token,
              "subject_token_type": "urn:ietf:params:oauth:token-type:access_token"}
    )
    dc_resp.raise_for_status()
    dc_token  = dc_resp.json()["access_token"]
    dc_domain = dc_resp.json()["instance_url"]
    return sf_token, sf_instance, dc_token, dc_domain

sf_token, sf_instance, dc_token, dc_domain = get_tokens(CONFIG)
SF_HDRS  = {"Authorization": f"Bearer {sf_token}", "Content-Type": "application/json"}
DC_HDRS  = {"Authorization": f"Bearer {dc_token}", "Content-Type": "application/json"}
BASE_SF  = f"{sf_instance}/services/data/v62.0"   # DC schema + stream registration
BASE_SEM = f"{sf_instance}/services/data/v65.0"   # Semantics Layer + Workspaces
BASE_VIZ = f"{sf_instance}/services/data/v66.0"   # Visualizations + Dashboards
# All three use SF_HDRS. Only data ingestion uses DC_HDRS + BASE_DC.
```

---

## NAMING CONVENTIONS

All asset names derive from bank name + use case. Derive once at the top of every script:

```python
from datetime import date, datetime as _dt

BANK_NAME  = "First Meridian Bank"
USE_CASE   = "CB RM"
PERSONA    = "Regional Manager"
STORY      = "Loan originations declining since Q3"
SIGNAL_ONSET = -6   # months ago

bank_slug     = BANK_NAME.lower().replace(" ", "_").replace(".", "")
for s in ("_bank", "_financial", "_corp", "_inc", "_group"):
    bank_slug = bank_slug.removesuffix(s)
use_case_slug  = USE_CASE.lower().replace(" ", "_").replace("/", "_")
WORKSPACE_NAME = f"{bank_slug}_{use_case_slug}"
SDM_NAME       = WORKSPACE_NAME
SCRIPT_NAME    = f"{bank_slug}_{use_case_slug}_next_demo.py"
DEMO_GUIDE     = f"{bank_slug}_{use_case_slug}_demo_guide.md"
TODAY          = date.today()
START_DATE     = date(TODAY.year - 2, TODAY.month, 1)
```

| Asset | Format | Example |
|---|---|---|
| Script file | `{bank_slug}_{use_case_slug}_next_demo.py` | `first_meridian_cb_rm_next_demo.py` |
| Workspace / SDM name | `{bank_slug}_{use_case_slug}` | `first_meridian_cb_rm` |
| DLO object names | `{bank_slug}_{TableName}` | `first_meridian_Loan_Originations` |
| Column/field labels | Business-friendly with spaces | `Loan Amount`, not `loan_amount` |
| Timestamp (when needed) | `datetime.now().strftime("%Y%m%d%H%M%S")` | `20260302143022` |

---

## CONCIERGE OPTIMIZATION — DESIGN PRINCIPLES

Apply all of these during Steps 2–3. Full API code patterns (field payloads, expression syntax, insight type selection) are in `build-next-demo-impl.md`.

### 1. Field Descriptions — most important input to Concierge

Rules:
1. **Under 255 characters** — hard limit
2. **No abbreviations** — write every term in full
3. **State the business purpose** — what question does this field answer?
4. **Include the grain** — "one row per loan officer per month"
5. **Assign roles** — every field must be Dimension or Measure
6. **Name = intent** — rename ambiguous fields before describing them

### 2. identifyingDimension — who the metric is about

- **Always use a *name* field** — never an ID field. `Officer Name`, not `Officer ID`
- **Must be a dimension from a joined dimension table**
- **Must also appear in `additionalDimensions`** — or API returns 400
- One per metric; choose the entity the persona cares most about

### 3. insightsDimensionsReferences — what Concierge uses to explain WHY

- Include 3–5 meaningful business dimensions (region, segment, product type, client tier)
- Never include ID fields or date fields
- All dims here MUST also be in `additionalDimensions` — API returns 400 otherwise
- Less is more — 3 well-chosen dims produce better AI explanations than 8 noisy ones

### 4. Insight types — select by metric type

| Insight type | Flow | Rate | Snapped | Notes |
|---|---|---|---|---|
| `CurrentTrend` | ✅ | ✅ | ✅ | Always include |
| `TrendChangeAlert` | ✅ | ✅ | ✅ | Detects signal onset — critical |
| `ComparisonToExpectedRangeAlert` | ✅ | ✅ | ✅ | Always include |
| `TopContributors` | ✅ | ✅ | ✅ | Always include |
| `BottomContributors` | ✅ | ✅ | — | |
| `TopDrivers` | ✅ | ✅ | — | |
| `TopDetractors` | ✅ | ✅ | — | |
| `ConcentratedContributionAlert` | ✅ | — | — | |
| `RecordLevelTable` | — | — | ✅ | |
| `OutlierDetection` | ✅ | ✅ | — | |

### 5. singularNoun / pluralNoun — how Concierge narrates insights

| Metric type | singularNoun | pluralNoun |
|---|---|---|
| Dollar amount | `"dollar"` | `"dollars"` |
| Count of loans/deals | `"loan"` | `"loans"` |
| Count of clients | `"client"` | `"clients"` |
| Rate / percentage | `"basis point"` | `"basis points"` |
| Score | `"point"` | `"points"` |
| Headcount | `"employee"` | `"employees"` |

### 6. timeGrains — omit Day for banking demos

Default: `["Month", "Quarter", "Year"]`. Only add `"Day"` for daily transaction data.

### 7. Field visibility

Set `isVisible: False` on:
- ID / foreign key fields (link tables but should not surface in AI answers)
- Technical date keys (`month_key__c`, `year__c`)
- Composite / helper fields created for join logic

### 8. Field count discipline

- Fact table: 8–12 fields max (3–4 dims + 4–6 measures + date + hidden IDs)
- Dimension table: 5–8 fields max

### 9. Concierge question engineering — design the questions first

| Pattern | Example | What it shows |
|---|---|---|
| Single entity, single answer | "Show me origination volume by region this quarter" | Basic KPI lookup |
| Slice by dimension | "How do approval rates compare across loan segments?" | Breakdown |
| Filter applied | "Show me pipeline in the Southeast last month" | NL filtering |
| Multi-entity comparison | "Compare origination volume across my top three regions" | Ranking |
| Multi-step breakdown | "Show me pipeline by officer and product for Q1" | Complex |
| Semantic learning | "Which officers are underperformers?" → define threshold | Calc field on the fly |

**Confirmed failure patterns — never include these:**

| Question type | Failure mode |
|---|---|
| Root cause / "why" — "Why is fee income declining?" | Content policy rejection |
| Cross-filter comparison against a benchmark | NL2SQ error: `Unsupported function: equals` |
| Ambiguous field reference | NL2SQ error: `Missing reference` |

**Safe frames**: "Which X has the highest/lowest Y?", "Show me Y by X", "What is Y vs prior period?", "Which X are underperforming?"

**Naming rule**: Do NOT prefix calc fields with "Average" — Concierge auto-prepends "Avg." producing "Avg. Average Wallet Share". Use plain nouns: `"Wallet Share"`, not `"Average Wallet Share"`.

### 10. Post-build step — enable Analytics Agent Readiness (manual)

After the script runs, coach the user:
> "Open your new semantic model in Data 360 → Settings → Analytics Agent Readiness → toggle ON. This activates the Agentforce / Concierge panel on all metric and dashboard pages."

### 11. Business Preferences — the "system prompt" for Concierge

| Layer | Purpose | Example |
|---|---|---|
| Field description | What the data IS (objective) | "Total dollar value of commercial loans originated in the given month." |
| Business preference | How this bank USES it (contextual) | "When users ask about 'top performers', sort by Origination Volume descending." |

Rules: each preference starts with `#`, max 300 chars, max 50 preferences per model. Less = faster Concierge. Full template and API attempt in `build-next-demo-impl.md`.

---

## STEP 1 — CREDENTIAL SETUP (if needed) + GATHER REQUIREMENTS

**Check for credentials first.** Before anything else, check whether `next_config.json` exists in the project folder.

If it does NOT exist, collect credentials:

> "Before I build your demo, I need your Salesforce and Data Cloud connection details. You'll only need to enter these once."

Ask for:
- Salesforce login URL (default: `https://login.salesforce.com`)
- Connected App client ID (consumer key)
- Connected App client secret (consumer secret)
- Refresh token (from OAuth authorization)
- Data Cloud domain (the `*.c360a.salesforce.com` domain from Data Cloud Setup)
- Data Cloud ingestion connector name (short name, e.g. `tableau_next_demo`)

Connected App must have scopes: `cdp_ingest_api`, `cdp_query_api`, `api`, `sfap_api`. Enable Client Credentials Flow.

Write config file:
```json
{
  "sf_login_url": "https://login.salesforce.com",
  "client_id": "<consumer key>",
  "client_secret": "<consumer secret>",
  "refresh_token": "<OAuth refresh token>",
  "data_cloud_domain": "<your-dc-domain (no https://)>",
  "ingestion_connector_name": "tableau_next_demo",
  "connector_sf_id": ""
}
```
Save as `next_config.json`. Do not proceed until this file exists.

**Then gather demo requirements.** Parse first, ask second.

Required inputs:
- Bank or company name
- Target persona (e.g., Commercial Banking RM, Wealth Advisor, Branch Manager)
- Story / narrative (what is trending, what is the business problem)
- Key entities / tables (what data objects — e.g., Loans, Clients, Activities)
- Metrics to show in Concierge (4–8 is ideal)
- Dimensions for slicing (region, segment, product type, etc.)
- Signal onset (default: 6 months ago)

---

## STEP 2 — CLASSIFY METRICS + DESIGN THE DATA MODEL (internal — feeds Step 3)

### Metric classification

| Type | Signal words | Data pattern | Aggregation | Concierge caveat |
|---|---|---|---|---|
| **Flow** | volume, count, originations, revenue, applications | 1 row per entity per period; value = event | `Sum` | None |
| **Rate / Average** | rate, ratio, score, %, yield, NPS | 1 row per entity per period; value = pre-computed ratio | `Average` | SUM would be wrong |
| **Snapped** | balance, AUM, outstanding, pipeline, headcount, active X | 1 row per entity per period; value = state at month-end | `Sum` (on latest row only) | ⚠️ Summing 12 months = 12× actual value |

**Snapped trap**: The data looks like flow (one row per month) but represents a *state*. The correct answer is always the most recent month's value.

**Handling snapped metrics in the SDM:**
- Option A: submetric filtered to `CurrentMonth` / `PreviousMonth`
- Option B (preferred — date-shift safe): LOD expression
  ```
  if [Adjusted_Date_clc] = { EXCLUDE :max([Adjusted_Date_clc])} then [fact_sdo].[balance_field] end
  ```
  This returns the balance only on the most-recent-date row; `Sum` then returns just that value.

### Multi-table data model

Always use 2–3 related tables:

| Table | Role | Grain | Typical metrics |
|---|---|---|---|
| `Fact_[Activity]` | Core fact | 1 row per entity per month | Volume, revenue, count |
| `Dim_Client` | Client/account dimension | 1 row per client | Segment, tier, tenure |
| `Dim_Product` | Product dimension | 1 row per product | Category, rate, term |

**Add `Dim_Date` when there are 2+ fact tables** (shared time spine):
```python
dates = pd.date_range(START_DATE, END_DATE, freq="MS")
dim_date = pd.DataFrame({
    "date":       [d.date() for d in dates],
    "month_key":  [int(d.strftime("%Y%m")) for d in dates],
    "year":       [d.year for d in dates],
    "quarter":    [((d.month - 1) // 3) + 1 for d in dates],
    "month_name": [d.strftime("%B") for d in dates],
})
```
Without `Dim_Date`, Concierge cannot compare metrics from different fact tables across the same time period.

**Signal design (same ramp as Pulse):**
```python
def signal_ramp(d, onset=SIGNAL_ONSET, duration=6):
    mft = (d.year - TODAY.year) * 12 + (d.month - TODAY.month)
    if mft <= onset: return 0.0
    return min(1.0, (mft - onset) / duration)
```

---

## STEP 3 — PRESENT THE PLAN & GET ONE CONFIRMATION

Present a complete build plan. This is the **only confirmation checkpoint** before writing and running code.

Format:

> **Here's what I'll build for [Bank Name]:**
>
> **Persona:** [persona]
> **Story:** [narrative]
> **Signal:** Decline starts [N] months ago, ramps to full effect today
>
> **Metrics:**
>
> | Metric | Type | SDM Aggregation | Concierge note |
> |---|---|---|---|
> | Loan Origination Volume | Flow | Sum | YTD or any period |
> | Approval Rate | Rate | Average | Use "current" language |
> | Portfolio Balance | Snapped | Sum (latest month) | Ask "current balance" |
>
> **Data Model:**
>
> | Table | Grain | Key Fields |
> |---|---|---|
> | Fact_[X] | monthly per [entity] | … |
> | Dim_Client | per client | … |
>
> **Sample Concierge questions:**
> - "Which region has the lowest origination volume this quarter?"
> - "Show me my top clients by portfolio balance."
>
> **Files:**
> - `{bank_slug}_{use_case_slug}_next_demo.py`
> - `{bank_slug}_{use_case_slug}_demo_guide.md`
>
> Reply **go** to build, or tell me what to change.

**Do not write any code until the user replies "go" (or equivalent).**

---

## STEP 4 — WRITE THE PYTHON SCRIPT

**Before writing any implementation code, READ `.claude/commands/build-next-demo-impl.md`.**

Name: `{bank_slug}_{use_case_slug}_next_demo.py`

### Complete script skeleton — always use this structure:

```python
#!/usr/bin/env python3
import json, sys, uuid, os, requests, subprocess, tempfile
import pandas as pd
import numpy as np
import time as _time
from datetime import date, datetime as _dt
from pathlib import Path

# ── Demo parameters ──────────────────────────────────────────────────────────
BANK_NAME     = "{Bank Name}"
USE_CASE      = "{Use Case}"
PERSONA       = "{Persona}"
STORY         = "{Story}"
SIGNAL_ONSET  = -6

# Derived names
bank_slug      = BANK_NAME.lower().replace(" ", "_").replace(".", "")
for s in ("_bank", "_financial", "_corp", "_inc", "_group"):
    bank_slug = bank_slug.removesuffix(s)
use_case_slug  = USE_CASE.lower().replace(" ", "_").replace("/", "_")
WORKSPACE_NAME = f"{bank_slug}_{use_case_slug}"
SDM_NAME       = WORKSPACE_NAME
SCRIPT_NAME    = f"{bank_slug}_{use_case_slug}_next_demo.py"
DEMO_GUIDE     = f"{bank_slug}_{use_case_slug}_demo_guide.md"
TODAY          = date.today()
START_DATE     = date(TODAY.year - 2, TODAY.month, 1)

CONCIERGE_QUESTIONS = []   # set during Step F
METRICS_META        = []   # {"name": ..., "type": ..., "concierge_note": ...}
VIZ_META            = []   # {"label": ..., "type": ..., "talking_points": [...]}

# ── Diagnostics ───────────────────────────────────────────────────────────────
_SCRIPT_START = _time.time()
_PHASE_TIMES  = {}

def _ts(): return _dt.now().strftime("%H:%M:%S")
def ok(m):   print(f"  [{_ts()}] ✅ {m}")
def info(m): print(f"  [{_ts()}] ℹ️  {m}")
def die(m, r=None):
    print(f"\n  ❌ FAILED: {m}")
    if r: print(f"  HTTP {r.status_code}: {r.text[:400]}")
    sys.exit(1)

def phase(n, label):
    _PHASE_TIMES[str(n)] = (_time.time(), label)
    print(f"\n[{n}/11] {label}...")

def mac_notify(title, message):
    try:
        script = f'display notification "{message}" with title "{title}" sound name "Glass"'
        with tempfile.NamedTemporaryFile(mode="w", suffix=".applescript", delete=False) as f:
            f.write(script); fpath = f.name
        subprocess.run(["osascript", fpath], check=False, capture_output=True)
        os.unlink(fpath)
    except Exception:
        pass

# ── Auth ──────────────────────────────────────────────────────────────────────
CONFIG_FILE = os.path.join(os.path.dirname(os.path.abspath(__file__)), "next_config.json")
CONFIG = json.loads(Path(CONFIG_FILE).read_text())
sf_token, sf_instance, dc_token, dc_domain = get_tokens(CONFIG)
SF_HDRS  = {"Authorization": f"Bearer {sf_token}", "Content-Type": "application/json"}
DC_HDRS  = {"Authorization": f"Bearer {dc_token}", "Content-Type": "application/json"}
BASE_SF  = f"{sf_instance}/services/data/v62.0"
BASE_SEM = f"{sf_instance}/services/data/v65.0"
BASE_VIZ = f"{sf_instance}/services/data/v66.0"

# ── Deployment plan ────────────────────────────────────────────────────────────
print(f"\n{'='*66}")
print(f"  DEPLOYMENT PLAN — {BANK_NAME} — {USE_CASE}")
print(f"  {'─'*62}")
print(f"  Phase 1   Generate synthetic data          ~5s")
print(f"  Phase 2   Register DC schema + streams     ~15s")
print(f"  Phase 3   Wait for DLO ACTIVE              ~30–120s")
print(f"  Phase 4   Submit bulk ingest jobs          ~30s")
print(f"  Phase 5   Wait for data to process         ~5–10 min")
print(f"  Phase 6   Create workspace                 ~5s")
print(f"  Phase 7   Build Semantic Data Model        ~30s")
print(f"  Phase 8   Calculated fields + metrics      ~30s")
print(f"  Phase 9   Create visualizations            ~20s")
print(f"  Phase 10  Build dashboard                  ~10s")
print(f"  Phase 11  Validate + write demo guide      ~10s")
print(f"  {'─'*62}")
print(f"  Total estimate: 8–15 minutes")
print(f"  TIP: Minimize this terminal and do other work.")
print(f"       A Mac notification will fire when the demo is ready.")
print(f"{'='*66}\n")

# ── PHASE 1: Generate synthetic data ──────────────────────────────────────────
phase(1, "Generating synthetic data")
# ... build DataFrames for each table

# ── PHASE 2: Register DC schema + create streams ──────────────────────────────
phase(2, "Registering Data Cloud schema + streams")
# See impl file Step 5 for full schema + stream code

# ── PHASE 3: Wait for DLO ACTIVE ──────────────────────────────────────────────
phase(3, "Waiting for DLO ACTIVE")
# See impl file _dlo_active()

# ── PHASE 4: Submit bulk ingest jobs (all tables, then poll) ──────────────────
phase(4, "Submitting bulk ingest jobs")
# See impl file bulk_ingest_submit()

# ── PHASE 5: Wait for Data Cloud to process ───────────────────────────────────
phase(5, "Waiting for Data Cloud to process (5–10 min — normal)")
# See impl file wait_for_bulk_job()

# ── PHASE 6: Create workspace ─────────────────────────────────────────────────
phase(6, "Creating workspace")
r = requests.get(f"{BASE_SEM}/tableau/workspaces", headers=SF_HDRS)
existing = {w["name"]: w["id"] for w in r.json().get("workspaces", [])}
if WORKSPACE_NAME in existing:
    requests.delete(f"{BASE_SEM}/tableau/workspaces/{WORKSPACE_NAME}", headers=SF_HDRS)
r = requests.post(f"{BASE_SEM}/tableau/workspaces", headers=SF_HDRS,
                  json={"label": WORKSPACE_NAME, "description": f"Demo workspace for {BANK_NAME} {USE_CASE}."})
r.raise_for_status()
workspace_name = r.json()["name"]   # capture slug — don't assume it equals WORKSPACE_NAME
workspace_id   = r.json()["id"]
ok(f"Workspace created: {workspace_name}")

# ── PHASE 7: Build Semantic Data Model ────────────────────────────────────────
phase(7, "Building Semantic Data Model")
# See impl file Steps A–H

# ── PHASE 8: Calculated fields + metrics ──────────────────────────────────────
phase(8, "Adding calculated fields + metrics")
# See impl file Steps D, E, F, G, H

# ── PHASE 9: Create visualizations ────────────────────────────────────────────
phase(9, "Creating visualizations")
# See impl file Step M

# ── PHASE 10: Create dashboard ────────────────────────────────────────────────
phase(10, "Building dashboard")
# See impl file Step N

# ── PHASE 11: Validate + write demo guide ─────────────────────────────────────
phase(11, "Validating SDM + writing demo guide")
# See impl file Step H (validate) and Step 8 (demo guide)

# ── Done ──────────────────────────────────────────────────────────────────────
_total = int(_time.time() - _SCRIPT_START)
print(f"\n{'='*66}")
print(f"  ✅ DEMO BUILD COMPLETE — {BANK_NAME} — {USE_CASE}")
print(f"  Open: {sf_instance}/tableau/workspace/{workspace_name}")
print(f"  Guide: {DEMO_GUIDE}")
print(f"  Total: {_total//60}m {_total%60}s")
print(f"{'='*66}\n")
print("Concierge questions:")
for q in CONCIERGE_QUESTIONS:
    print(f'  • "{q}"')
print(f"\nTo tear down: python3 next_teardown.py  (workspace: {WORKSPACE_NAME})")

mac_notify("Tableau Demo Builder", f"{BANK_NAME} \u2014 {USE_CASE} demo is ready!")

# Write to demo registry (used by next_teardown.py)
_reg = os.path.join(os.path.dirname(os.path.abspath(__file__)), "next_demos.json")
try:
    registry = json.loads(Path(_reg).read_text()) if Path(_reg).exists() else []
    registry = [d for d in registry if d.get("workspace_name") != WORKSPACE_NAME]
    registry.append({"label": f"{BANK_NAME} \u2014 {USE_CASE}", "workspace_name": WORKSPACE_NAME,
                     "dc_prefix": bank_slug, "built_on": date.today().isoformat(), "script": SCRIPT_NAME})
    Path(_reg).write_text(json.dumps(registry, indent=2))
except Exception:
    pass
```

### Data generation defaults

- History: 24 months (START_DATE = date(TODAY.year - 2, TODAY.month, 1))
- Fact table grain: one row per entity (advisor, client, loan officer) per month
- Dimension tables: static (no date column)
- Always include in fact table: `Date` (date), `Month Key` (int YYYYMM), `Year` (int), `Quarter` (int 1–4), `Month` (string)
- Submit ALL bulk jobs before waiting for any — saves 30–60s per extra table

**Bulk API overview**: 3 steps per table: POST job → PUT CSV batches → PATCH to close. See impl file for `bulk_ingest_submit()` and `wait_for_bulk_job()` functions.

---

## STEP 5 — SCHEMA REGISTRATION + STREAM CREATION

Full code in `build-next-demo-impl.md` (Step 5 section).

Pattern: look up connector SF ID from config → PUT schema (merge with existing; strip read-only fields) → POST one stream per schema object (only PK field in `dataLakeFieldInputRepresentations`) → poll `_dlo_active()` per stream before ingesting.

---

## STEP 6 — RUN THE SCRIPT

After writing the script, run it immediately:
```bash
/opt/homebrew/bin/python3.13 <script_name>.py
```

Report each phase as it completes. If an error occurs, diagnose it, fix the script, and re-run.

---

## STEP 7 — CREATE TABLEAU SEMANTIC MODEL

Full code in `build-next-demo-impl.md` (Step 7 section, Steps A–H and optional I–N).

**8-step core flow:**
1. DELETE existing model if found (idempotent)
2. POST model with `agentEnabled: True`, `dataObjectType: "Dlo"`, `app: workspace_name`
3. GET model to discover auto-generated field apiNames (like `Loan_Amount1`, `Date3`)
4. POST calculated measurements (use `"Currency"`/`"Percentage"` here — accepted ✅)
5. POST calculated date dimension (use date-shift formula)
6. POST semantic metrics (MUST use `calculatedFieldApiName` — `tableFieldReference` is silently ignored)
7. POST relationship (`joinType: "Auto"`)
8. Validate — GET `/validate`

Then POST SDM id to workspace `/assets` to link it.

---

## STEP 8 — GENERATE DEMO GUIDE

Full Python code in `build-next-demo-impl.md` (Step 8 section).

Always auto-generate from BANK_NAME, PERSONA, STORY, metric list, visualization list, and CONCIERGE_QUESTIONS. Do not use placeholder text. Print Concierge questions at end of script run — they are the SE's demo script.

---

## COMMON PITFALLS — TOP 10

1. **PATCH on main model URL is a FULL REPLACE** — always use sub-resource POST endpoints for sub-entities
2. **Semantic Metrics MUST use `calculatedFieldApiName`** — `tableFieldReference` is silently ignored
3. **SDO `semanticMeasurements` must use `dataType: "Number"` for IngestAPI DLOs** — `"Currency"` rejected
4. **PUT schema is a FULL REPLACE** — GET first, strip read-only fields (`availabilityStatus` etc.), merge, then PUT
5. **Dashboard `widgets` is a dict, not a list** — array causes 500 error
6. **Dashboard page `name` must be UUID** — plain strings cause blank canvas
7. **Dashboard layout `style` must include `cellSpacingX/Y`** — `{}` causes blank canvas
8. **`style.headers` must be omitted, not `{}`** — empty dict causes JSON_PARSER_ERROR at v66.0
9. **All dims in `insightsSettings` must also be in `additionalDimensions`** — missing causes 400
10. **`agentEnabled: True` is not enough for Concierge** — user must enable Analytics Agent Readiness in UI

Full list of 52 pitfalls in `build-next-demo-impl.md`.
