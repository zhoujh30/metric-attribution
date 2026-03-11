---
name: metric-attribution
description: "Metric attribution analysis skill (primary domain: e-commerce; extensible to other industries). Triggered when a user needs to diagnose why a metric changed. Two scenarios: ① AB attribution — experiment group vs. control group, locating root causes of GMV/GPM shifts; ② AA attribution — no explicit control group, constructing a reference via period comparison, peer group, DiD, or synthetic control. Trigger phrases: 'help me analyze this AB data', 'experiment result is bad, why', 'traffic went up but GMV didn't', 'why did GMV drop', 'I want to do an attribution analysis', 'help me figure out why this metric changed', 'why is this category underperforming'. Core output: ① adaptive classification (2 metrics → 4-quadrant ABCD; 1 metric → positive/negative contribution segments) ② multi-dimensional drill-down ③ effect decomposition (GPM Mix-Rate or contribution ranking + structure/efficiency split) ④ prioritized action plan. Output format: self-contained HTML (shareable via GitHub Secret Gist link) + Excel list. Output language defaults to English; switch to Chinese if user writes in Chinese or requests it."
---

# Metric Attribution Skill

> **Primary domain**: E-commerce (GMV / GPM / CTR / CO / AOV). Methodology is extensible to other industries.

---

## Overview

Two stages:

- **Phase 0: Requirements** (0a pre-data + 0b post-data)
- **Phase 1–5: Analysis** (dynamically shaped by Phase 0 — not a fixed template)

**Core principle: clarify what to answer, how to compare, and what action follows — before writing any code.**

**Output language**: Default English. Switch to Chinese if the user writes in Chinese or explicitly requests it (`请用中文` / `switch to Chinese`).

---

## Phase 0a: Intent Capture (Before Data Arrives)

**When to trigger**: User expresses intent but has not provided data yet (e.g., "I want to do an attribution analysis", "help me figure out why GMV dropped").

**Skip 0a** if the user provides data and full requirements in the same message — go directly to 0b.

### Question Design

Use `ask_user_input` to present structured options, minimizing user effort. Max 4 questions per session. Always include a free-text fallback ("Other").

**Q1 — Analysis background (single select):**
- AB experiment: explicit experiment and control groups exist
- Metric anomaly: a metric moved unexpectedly, no explicit control group
- Strategy evaluation: assessing the impact of a change, no randomized groups
- Other (free text)

**Q2 — What have you already observed? (open-ended)**

> Be as specific as possible: e.g., "experiment group GMV -10%, traffic up 30%", "GMV dropped 15% starting Wednesday", "suspect high-price items have a conversion issue". "Don't know yet" is fine too.

**Q3 — What decision does this analysis need to support? (multi-select)**
- Assortment adjustment (add / remove / expand product pool)
- Algorithm / strategy tuning (fix model or distribution logic)
- Traffic allocation optimization (adjust weights across products / categories)
- Upward reporting (clear conclusion + visualization)
- Other (free text)

**Q4 — Data situation (open-ended)**

> What data do you have? Format (Excel / CSV / query output)? Time range? Any known data quality issues?

### Adaptive Logic

After receiving answers:
- Q1 = AB experiment → ask user to upload data, proceed to 0b
- Q1 = Metric anomaly / Strategy evaluation → one follow-up: **Do you have historical data or a comparable peer group?** (determines AA method, see 0b.3)
- Q3 already specifies action direction → don't repeat in 0b

---

## Phase 0b: Requirements Alignment (After Data Arrives)

### 0b.1 Silent Exploration

Silently explore the data first (don't show this to the user). Use findings to anchor the requirements conversation:
- Read table structure: column names, data types, row count
- Check for experiment / version / group columns → **determine analysis type (AB / AA)**
- Identify core metric columns (PV / Click / Order / GMV / Subsidy, etc.)
- Identify drillable dimension columns (category hierarchy, price tier, user type, seller attributes, etc.)
- Compute top-level totals (PV / GMV / GPM for both groups)
- Count core metrics involved → **determine 4-quadrant vs. 2-segment**

### 0b.2 Analysis Type Detection

| Data characteristics | Analysis type | Phase 2 classification |
|---------------------|--------------|----------------------|
| Explicit experiment / control column + PV and GMV both present | AB attribution | 4-quadrant (A/B/C/D) |
| Explicit experiment / control column, but user only cares about 1 metric | AB attribution (single metric) | 2-segment (positive / negative contribution) |
| No explicit control group; reference must be constructed | AA attribution | 2-segment (positive / negative contribution) |

**Single-metric detection**: if the user's Q3 focuses on one outcome metric only, or PV/traffic data is absent, treat as single-metric.

### 0b.3 AA Method Selection (AA attribution only)

Based on Q4 and data exploration, recommend a primary method and state its core assumption:

| Condition | Recommended method | Core assumption |
|-----------|-------------------|----------------|
| Stable history (≥3 comparable periods), no obvious external shock | **Period comparison** (WoW / YoY) | Historical trend is a valid counterfactual baseline |
| External shock present (major promo, policy change, platform overhaul) making history non-comparable | **DiD (Difference-in-Differences)** | Shock has parallel trend effects on treatment and control |
| No reliable history, but comparable lateral groups exist (other categories, regions, user cohorts unaffected by the change) | **Peer group comparison** | Control group is homogeneous with treatment on key attributes |
| Sufficient data points but none of the above apply | **Synthetic control** | Counterfactual can be constructed via weighted combination |
| Insufficient data for any method | Clearly state this; suggest what data to collect | — |

**Robustness check triggers** (assessed automatically, no extra questions):
- Period comparison: strong seasonality detected → add YoY validation
- DiD: parallel trend assumption appears violated → add placebo test
- Peer group comparison: control group also shows movement → add time-series consistency check

Present the method recommendation and its assumption to the user. Proceed after confirmation.

### 0b.4 Confirm Analysis Plan

Summarize into a concise plan. **Only ask for information that is still missing.** Example:

> **Analysis plan — please confirm:**
> - **Type**: AB attribution, 2 metrics (PV + GMV), 4-quadrant classification
> - **Question**: Exp PV +29.5% but GMV -1.4% — locate root cause of GPM decline
> - **Dimensions**: Category L1→L2→L3, price tier, user type, seller tier
> - **Action direction**: Assortment adjustment + algorithm tuning
> - **Output**: HTML dashboard (Gist link) + D-class blacklist Excel

Proceed to Phase 1 after confirmation.

---

## Phase 1: Data Processing & Metric Computation

### 1.1 Granularity Check

One `product_id` may split across multiple rows by `user_type`, `price_range`, etc. Choose aggregation based on analysis needs:

```python
# Product-level aggregation (for quadrant / segment classification)
agg_exp  = df[df['version_id'] == exp_id].groupby('product_id')[metric_cols].sum()
agg_ctrl = df[df['version_id'] == ctrl_id].groupby('product_id')[metric_cols].sum()
merged   = agg_exp.join(agg_ctrl, lsuffix='_exp', rsuffix='_ctrl', how='inner')

# Attach product attributes (category, price tier, etc. — first row after dedup)
attrs  = df[df['version_id'] == exp_id].drop_duplicates('product_id').set_index('product_id')[attr_cols]
merged = merged.join(attrs)
```

For AA attribution, `_exp` = current period; `_ctrl` = constructed reference period / group.

### 1.2 Metric Framework

| Layer | Metrics | Formula |
|-------|---------|---------|
| Scale | PV, GMV, Orders, Subsidy | Direct sum |
| Conversion funnel | CTR, CO | Click/Show, Order/Click |
| Monetization efficiency | GPM, OPM, AOV | GMV/PV×1000, Order/PV×1000, GMV/Order |

**Display rule: relative change (%) is primary — large text. Absolute value is secondary — small text.**

---

## Phase 2: Adaptive Classification

### Mode A: 4-Quadrant (2 core metrics, e.g., PV + GMV)

| Quadrant | PV | GMV | Meaning | Focus |
|----------|-----|------|---------|-------|
| A | ↑ | ↑ | Double-up (benchmark) | Extract positive traits → expand pool |
| B | ↓ | ↑ | Efficiency gain (existing inventory) | Find conversion improvement levers |
| C | ↓ | ↓ | Double-down (shrinkage) | Traffic hijacked by D? Subsidy cut? |
| D | ↑ | ↓ | High exposure, low conversion (waste) | **Core audit** — why does traffic not convert? |

```python
def classify_4q(row):
    pv_up  = row['pv_diff']  > 0
    gmv_up = row['gmv_diff'] > 0
    if pv_up  and gmv_up:  return 'A'
    if not pv_up and gmv_up:  return 'B'
    if not pv_up and not gmv_up: return 'C'
    return 'D'
```

Every quadrant must output: PV (Δ%), GMV (Δ%), CTR, CO, GPM, AOV, Subsidy.

Diagnose C/D by comparing GPM and AOV shifts:
- CO large drop + AOV stable → conversion failure (no one buying)
- CO stable + AOV large drop → price-point collapse (buying cheaper)

### Mode B: 2-Segment (1 core metric, e.g., GMV only)

| Segment | Definition | Focus |
|---------|------------|-------|
| Positive contributors | Metric went up vs. reference in this segment | Replicate and expand |
| Negative contributors | Metric went down vs. reference in this segment | Diagnose root cause, prioritize treatment |

**Within each segment, rank by absolute contribution value.**

Key signal: **a dimension that cleanly separates positive from negative contributors is a high-priority analysis entry point and action lever.**

---

## Phase 3: Multi-Dimensional Drill-Down

For D-class items (4-quadrant) or negative contributors (2-segment), drill by the dimensions confirmed in Phase 0. **Rank each dimension by negative contribution (absolute value).**

### Standard Dimension Menu

| Dimension | Purpose | Key question |
|-----------|---------|-------------|
| Category (L1 → L2 → L3) | Locate bleeding categories | Which L3 categories are hurting most? |
| Price tier | Price-fit quality | High-price collapse or low-price traffic inflation? |
| User type | Traffic-fit quality | New users routed to irrelevant products? |
| Seller tier | Supply quality | Traffic flowing to low-tier sellers? |
| Dimension combo (category × price tier) | Cross-feature localization | What are the Top 10 negative combos? |

**Standard drill-down table columns**: Dimension value, SKU count, ΔGMV, PV Δ%, GPM current/reference, GPM Δ%, CO current/reference

Extend flexibly based on dimensions added by the user in Phase 0 (e.g., subsidy efficiency, seller-level dimensions).

---

## Phase 4: Effect Decomposition

### Mode A: GPM Mix-Rate Decomposition (4-quadrant)

**Key to a qualitative conclusion — separates structural problems from efficiency problems.**

```
GPM change = Mix Effect (traffic structure shift) + Rate Effect (within-category conversion change)

Mix Effect  = Σ (share_exp_i  - share_ctrl_i) × gpm_ctrl_i
Rate Effect = Σ  share_exp_i  × (gpm_exp_i   - gpm_ctrl_i)
```

| Result | Meaning | Typical action |
|--------|---------|---------------|
| Rate Effect dominates | Conversion rates dropped across categories — algorithm efficiency issue | Audit pCVR model, adjust distribution logic |
| Mix Effect dominates | Traffic shifted toward low-GPM categories | Rebalance traffic weights across categories |
| Both significant | Compound problem | Treat in layers |

Break down by L1 category; output each category's Mix and Rate contribution values.

### Mode B: Contribution Ranking + Structure / Efficiency Split (2-segment)

**Step 1: Contribution ranking by dimension (primary output)**

```python
contributions = {}
for dim_val in df[dimension].unique():
    mask  = df[dimension] == dim_val
    delta = df.loc[mask, 'metric_current'].sum() - df.loc[mask, 'metric_reference'].sum()
    contributions[dim_val] = delta
# Sort by absolute value; display positive (green) and negative (red) separately
```

Output as waterfall or bar chart. **Highlight dimensions that cleanly separate positive from negative** — these are the analysis entry points and action levers.

**Step 2: Structure / efficiency split on top negative dimensions**

```
Change in metric for a dimension =
  Structure effect + Efficiency effect

Structure effect  = (share_current - share_reference) × metric_per_unit_reference
Efficiency effect =  share_current × (metric_per_unit_current - metric_per_unit_reference)
```

| Result | Meaning | Action direction |
|--------|---------|----------------|
| Structure effect dominates | More resources allocated to low-efficiency segment | Rebalance allocation weights |
| Efficiency effect dominates | Per-unit efficiency dropped within the segment | Diagnose and fix efficiency issue |
| Both significant | Compound problem | Treat in layers |

---

## Phase 5: Action Plan Output

**Content is fully determined by the action direction confirmed in Phase 0. Compose from the modules below as needed.**

### Module A: Assortment Adjustment

**Positive product trait extraction (expansion pool):**
- Common profile: core categories, price tiers, seller tiers, seller types
- Positive contribution ranking: by L1 / L3 category, dimension combos
- Expansion recommendations: which L3 categories to source, which price tiers to anchor, which seller tiers to prioritize

**Negative product blacklist (Excel export):**
- Full list sorted by ΔGMV
- Three disposal tiers:
  - **Remove immediately**: control group had sales, experiment group has zero
  - **Strongly recommend removal**: both groups zero sales (pure waste)
  - **Rate-limit and observe**: has sales but GMV is negative
- **Common trait summary → convert to future sourcing exclusion rules** (more valuable than the blacklist itself)

### Module B: Algorithm / Strategy Tuning

- pCVR calibration recommendations (which categories have miscalibrated estimates)
- Per-user-segment strategy recommendations (differentiated distribution for returning vs. new users)
- Conversion gate recommendations (e.g., high-price non-essential items should require preference signals before being served)

### Module C: Traffic Allocation Optimization

- PV efficiency by quadrant / segment (GMV generated per 1,000 PV)
- Traffic reflow recommendations (where should PV freed from D-class flow to — A / B class products)

### Module D: Subsidy Efficiency Audit

- Subsidy cost per unit PV, comparison across groups
- ROAS comparison by quadrant and seller tier
- High-risk waste flagging

### Module E: C-Class Recovery

- Diagnosis: subsidy cut vs. traffic hijacked by D-class
- Recovery strategies: floor traffic protection, subsidy restoration, priority product tagging

### Priority Framework

All recommendations ranked P0–P3:
- **P0 — Stop the bleeding**: remove zero-conversion products, rate-limit top negatives, free up PV
- **P1 — Fix root cause**: audit model / adjust distribution logic
- **P2 — Grow positives**: expand product pool based on positive traits
- **P3 — Build long-term**: monitoring dashboard + distribution strategy rebuild

---

## Output Specifications

### Interactive HTML Dashboard

Pure HTML + CSS + Chart.js (CDN). No React.

**Structure:**
- Header: analysis name, group comparison description, core KPI one-line summary
- Sticky tab nav: Overview | Classification | Drill-Down | Effect Decomposition | Action Plan
- Drill-down tab: sub-tabs to switch between dimensions
- Action plan tab: sub-tabs to switch between modules

**Design rules:**
- Dark theme (bg: `#0c0e14`, card: `#161923`)
- Fonts: JetBrains Mono (numbers) + system sans-serif (text)
- Color semantics: A=`#10b981`, B=`#3b82f6`, C=`#eab308`, D=`#ef4444`; positive=`#10b981`, negative=`#ef4444`
- Up/down colors: up=`#34d399`, down=`#f87171`
- Metric cards: % change large text, absolute value small text
- Insight boxes: `danger` / `warn` / `success` / `info`
- Top 3–5 rows highlighted in tables (most severe first)

### Shareable Link (GitHub Secret Gist)

After generating the HTML, create a **Secret Gist** via the GitHub API (`public: false` — not searchable, accessible only via link):

```python
import requests, os

def create_secret_gist(html_content: str, title: str) -> str:
    token = os.environ.get('GITHUB_TOKEN')  # one-time setup, requires gist scope
    headers = {
        'Authorization': f'token {token}',
        'Accept': 'application/vnd.github.v3+json',
    }
    payload = {
        'description': title,
        'public': False,
        'files': {f'{title}.html': {'content': html_content}},
    }
    resp     = requests.post('https://api.github.com/gists', json=payload, headers=headers)
    raw_url  = resp.json()['files'][f'{title}.html']['raw_url']
    return f'https://htmlpreview.github.io/?{raw_url}'
```

**One-time setup**: set `GITHUB_TOKEN` in your environment or `~/.claude/settings.json` (requires `gist` scope).

If no token is configured: output a local HTML file and display setup instructions.

### Excel List (Separate File)

Full negative product list: `product_id`, ΔGMV, PV Δ%, GPM current/reference, L1/L2/L3 category, price tier, seller tier/type, disposal recommendation (three tiers).

Positive product whitelist Excel available on request.

---

## Response Templates

**AA method recommendation:**
> "You have X weeks of historical data with no obvious external shocks, so I recommend **WoW period comparison** as the baseline. Core assumption: the historical trend is a valid counterfactual. Does that hold for this period?"

**GPM attribution conclusion:**
- Rate dominates: "This is a systemic algorithm efficiency issue, not a traffic mix problem. Rate Effect accounts for X% of the GPM decline."
- Mix dominates: "Traffic structure has shifted materially toward low-GPM categories. Mix Effect accounts for X% of the GPM decline."

**C/D diagnosis:**
- Conversion issue: "The core problem is 'no one is buying', not 'prices are too low'. D-class CO -X%, AOV change is mild."
- Price collapse: "The issue is price-point collapse. D-class AOV -X%, low-price items absorbed most of the traffic."

**Dimension key finding:**
> "[Dimension] cleanly separates positive from negative contributors — [value A] pulled up +$X, [value B] dragged down -$X. This is the highest-priority analysis entry point."

---

## Checklist

**Phase 0a (intent capture):**
- [ ] Analysis background collected (AB / metric anomaly / strategy evaluation)
- [ ] User's existing observations and hypotheses captured
- [ ] Action intent clarified (assortment / algorithm / traffic / reporting)
- [ ] Data situation and known quality issues understood

**Phase 0b (requirements alignment):**
- [ ] Data silently explored; top-level metrics computed
- [ ] Analysis type determined (AB / AA) and classification mode chosen (4-quadrant / 2-segment)
- [ ] AA mode: primary method recommended with assumption stated; user confirmed
- [ ] Analysis plan confirmed; only missing critical info was asked

**Phase 1–4 (analysis execution):**
- [ ] Top-level metric dashboard complete (relative change % primary)
- [ ] Classification complete (4-quadrant with full metric comparison / 2-segment with contribution ranking)
- [ ] All user-confirmed dimensions drilled (category traversed L1→L2→L3)
- [ ] Effect decomposition complete with clear qualitative conclusion (Rate/Mix or Structure/Efficiency)
- [ ] AA mode: robustness check conditions reviewed; additional validation added if triggered

**Phase 5 (action plan):**
- [ ] Action plan covers all directions confirmed in Phase 0
- [ ] Positive traits extracted (if assortment adjustment requested)
- [ ] Negative product blacklist Excel exported (if assortment adjustment requested)
- [ ] Blacklist common traits summarized and converted to future sourcing exclusion rules
- [ ] All recommendations ranked P0–P3

**Output check:**
- [ ] All HTML tabs / sub-tabs switch correctly
- [ ] Chart.js charts render correctly
- [ ] All tables sorted by contribution magnitude
- [ ] Gist link generated and accessible (or local file exported with setup note)
- [ ] Excel exported as a separate file
