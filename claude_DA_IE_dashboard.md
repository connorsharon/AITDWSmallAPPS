# DA / IE Pipeline Dashboard — Project Instructions

## Purpose
Single-file HTML dashboard giving the **Data Analysis (DA)** and **Integrity Engineering (IE)** teams 
advance visibility into upcoming ILI jobs that require their work. Built from Salesforce Opportunity 
data (pre-order pipeline), not confirmed orders.

---

## Source Files

### 1. `Opp_data_SF.xls` — Salesforce Opportunities (main input)
HTML-disguised XLS. Read with `pd.read_html()`.

| Column | Notes |
|---|---|
| `Primary Quote` | Quote number (Q-XXXXXX) — the unique job identifier. Multiple rows per quote (one per line item/product). |
| `Opportunity Name` | Full job description — contains pipeline name, diameter, miles (e.g. "ILI, MDS Pro, 26" x 32.48 miles, TCO NPS 26...") |
| `Account Name` | Customer |
| `Product Name` | Line item product — **this is the primary field to match run types against** |
| `Close Date` | Proposed/target date (M/D/YYYY) |
| `Stage` | Pipeline stage: Explore → Investigate → Evaluate → Confirm → Commit |
| `Probability (%)` | Win probability — use for weighted revenue calc |
| `Total Price` | Line item price (USD) — sum per quote for total opp value |
| `Total Price Currency` | Usually USD |
| `Management Entity: Entity` | Entity number |
| `Management Entity: Management Region` | "Services" = ILI/pipeline work. "Products" and "Canada" are excluded. |
| `Opportunity Owner` | SF owner / sales rep |
| `Description` | Free text — less reliable, don't use for matching |

**Always filter to `Management Entity: Management Region == 'Services'` first.**

### 2. `report177XXXXXXX.xls` — Salesforce Confirmed Orders (reference only)
Same HTML-XLS format. Used as reference to understand which opps have already converted to orders.
Opp → Order conversion is Commit stage → Order. Don't double-count.

| Column | Notes |
|---|---|
| `Order Number` | Confirmed order ID |
| `Product Name` | Same product codes as Opp file |
| `Project Manager` | Key for FSA matching (separate tool) |
| `Management Entity: Entity` | Same entity system |
| `Status` | Filter to "Order Submitted Successfully" |

### 3. `run_types.csv` — Run Type Reference Table
Two columns: `Tool Type` (run type code) and `Type` (team assignment).
- Column 1: run type/product code (e.g. `DENT.STRAIN.DA`, `IES.FIA.GROWTHONLY`)
- Column 2: team — either `Data Analysis` or `Integrity Engineering`

---

## Region / Entity Filtering

**Include:** `Management Entity: Management Region == 'Services'` only.

**ILI-relevant quotes** = any quote where at least one line item product name contains:
`MFL`, `DEF`, `EMAT`, `SMFL`, `XYZ INSPECTION`, `MDS`, `LFM`, `FLEX`, `GMFL`, `UMFL`, `INSPECTION`

OR the quote has a matched DA/IE run type product.

**Exclude Products region** (14,180 rows) — unrelated to ILI analysis.

---

## Run Type Matching Logic

Match against **`Product Name` only** (not Opportunity Name) to avoid false positives.
One product name → one run type. Use the mapping table below (check in order, most specific first).

### Data Analysis (DA) products → run types
| Product Name Pattern (case-insensitive) | Run Type |
|---|---|
| DENT STRAIN | DENT.STRAIN.DA |
| BEND STRAIN or BENDING STRAIN | BEND.STRAIN.DA |
| WRINKLE BEND or WRINKLEBEND | WRINKLEBEND.ANALYSIS |
| HARD SPOT or HARDSPOT | HARDSPOT.ANALYSIS |
| SEAM WELD CORROSION CLASS or SSWC | SSWC.CLASSIFICATION |
| PIPE JOINT CLASSIF or PJC | PJC.CLASSIFICATION |
| EXPEDITED REPORT | EXPEDITED.REPORTS |
| XYZ INSPECTION or XYZ SURVEY | XYZ Survey |
| GPS CENTERLINE or GPS TRACK | GPS TRACK |
| CSCC PRIORITIZATION | ES.AIA.CSCC |
| DATA ANALYSIS (standalone) | ANALYSIS |
| PIPELINE CLEANING-PCS | PCS |
| SUBSBU-MULTIPLE | RUN COMPARISON |
| IMU (standalone word) | IMU |
| SPC (standalone word) | SPC |

### Integrity Engineering (IE) products → run types
| Product Name Pattern | Run Type |
|---|---|
| CORROSION GROWTH ONLY | IES.FIA.GROWTHONLY |
| CORROSION GROWTH & FUTURE | IES.FIA.FUTURE |
| CORROSION GROWTH or SIGNAL-TO-SIGNAL | IES.FIA.GROWTHONLY |
| LINE MOVEMENT | IES.FIA.LINEMOVEMENT |
| PPDA or PIPE POPULATION DISCREPANCY | IES.IIA.PPDA |
| CRACK FATIGUE | IES.FIA.CRACKFATIGUE |
| CRACK PROFILE | IES.IIA.CRACKPROFILE |
| CRACK SCREEN | IES.AIA.CRACK.SCREEN |
| CRACK FEA | IES.FIA.CRACK.FEA |
| DENT FEA or DENT CODE | IES.IIA.DENT.CODE |
| BENDING STRAIN ASSESSMENT | IES.IIA.BEND.STRAIN |
| CUSTOMER TRAINING or TRAINING ON EQUIPMENT | IES.TRAINING |
| FEA (standalone) | IES.ADV.FEA |

**Note:** `PCS` should only match as standalone word (`\bPCS\b`), NOT inside "HTPCS" 
(Hot Tap and Pressure Control Services — unrelated to Pipeline Condition Survey).

---

## Miles Extraction

Extract pipeline miles from `Opportunity Name` using regex:
1. Pattern `x {NUMBER} miles` or `x {NUMBER} mi` (most common format)
2. Fallback: `{NUMBER} miles` anywhere in name
3. Validate: must be between 0.1 and 5000

Example: `"ILI, MDS Pro, 26" x 32.48 miles, TCO NPS 26..."` → `32.48 miles`

Note: Some opps use feet (e.g. "12,538'") — these are short segments, can be ignored or noted.

---

## Data Architecture

### Quote-level rollup (primary unit)
One row per `Primary Quote`. Aggregate line items:
- `total_value` = sum of all `Total Price` in quote
- `weighted_value` = `total_value × (probability / 100)`
- `da_types` = comma-joined set of matched DA run types in quote
- `da_products` = pipe-joined set of matched DA product names
- `da_value` = sum of DA line item prices
- `ie_types` = comma-joined set of matched IE run types
- `ie_products` = pipe-joined set of matched IE product names
- `ie_value` = sum of IE line item prices
- `miles` = extracted from `Opportunity Name`

### Stage pipeline (funnel order)
`Explore → Investigate → Evaluate → Confirm → Commit`
- **Evaluate + Confirm** = primary working pipeline (most actionable for teams)
- **Commit** = likely converting to order imminently
- **Explore/Investigate** = early stage, lower confidence

### Probability weighting
- Evaluate = 40% typical
- Confirm = 60% typical
- Commit = 90%+ 
(Use actual `Probability (%)` field, not hardcoded values)

---

## Dashboard Requirements

### Views needed

**1. Pipeline Summary (Dashboard)**
- Stat cards: Total ILI quotes | DA work items | IE work items | Weighted pipeline value
- Funnel chart: quotes by Stage
- Monthly close date chart (bar) — showing DA vs IE vs ILI-only
- Top customers by weighted value
- Top opportunity owners

**2. DA Team View**
- Table of all quotes with DA work identified
- Columns: Quote | Customer | Opp Name | Miles | Close Date | Stage | Prob% | DA Run Types | DA Products | DA Value | Total Quote Value | Owner
- Filter by: Stage, Customer, Close Month, Run Type
- Sorted by Close Date default

**3. IE Team View**  
- Same structure as DA view but for IE work
- Columns: Quote | Customer | Opp Name | Miles | Close Date | Stage | Prob% | IE Run Types | IE Products | IE Value | Total Quote Value | Owner

**4. All ILI Opportunities**
- Full ILI pipeline including jobs with no DA/IE work yet identified
- Flag rows with DA and/or IE work with color coding
- Shows 381 quotes total

**5. Import**
- Drag-and-drop for refreshed Opp export
- Same matching engine applied on upload

### Key UX requirements
- Color coding: DA = blue (#4b8be8), IE = teal (#4be8b8), No analysis = gray
- Stage color: Confirm = green, Evaluate = amber, others = gray
- Show both raw value AND weighted value
- Miles displayed where available, "—" where not
- Clicking a row opens detail panel with all line items
- Filter by close month (important for team capacity planning)
- Toggle: Show All Stages vs Confirm+Evaluate only

---

## Known Data Quirks

1. **Multiple rows per quote** — always roll up by `Primary Quote` before displaying
2. **Training products** (40 rows match IES.TRAINING) — equipment training, not IE analysis. 
   Confirm with user whether to include or exclude.
3. **Miles only in ~30% of opp names** — many are short-segment or equipment jobs without miles stated
4. **International opps** — entities 906, 960, 703, 711 are international. 
   Consider separate tab or filter.
5. **Close dates are proposed dates** — field dates may differ. Don't treat as hard deadlines.
6. **Currency** — almost all USD in Services region. Check `Total Price Currency` to be safe.

---

## Files to Keep in Project

- `Opp_data_SF.xls` — latest opportunity export
- `run_types.csv` — run type → team mapping table (update as new types added)
- `report177XXXXXXX.xls` — latest confirmed orders (for reference/dedup)
- `claude.md` — this file

## Refresh Cadence
SF Opportunity data should be re-exported and re-imported weekly or bi-weekly.
The run_types.csv mapping is maintained manually — update when new analysis products are added to SF catalog.
