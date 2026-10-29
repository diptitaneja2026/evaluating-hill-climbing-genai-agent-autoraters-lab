# Low-Level Design (LLD) — Evaluating and Hill Climbing a Gen AI Agent Using Autoraters (L400-APEX006)

> **Lab URL:** [https://explore.qwiklabs.com/classrooms/21504/labs/115876](https://explore.qwiklabs.com/classrooms/21504/labs/115876)  
> **Lab Code:** `L400-APEX006`

---

## 1. Component Inventory & Codebase Layout

```text
autoraters/
├── .env                                            # Vertex AI & Gemini runtime config (GOOGLE_GENAI_USE_VERTEXAI=TRUE)
├── requirements.txt                                # Python 3.13 dependencies (google-adk==2.4.0, google-cloud-aiplatform[evaluation]==1.160.0)
├── best_criteria_submission.txt                    # Task 5 submission: monthly-tax-holding,environmental-esa,flood-zone-flagging
├── tools_data/
│   ├── lots_database.json                          # 65 vacant lot records (acreage, zoning, flood_zone, adjacent_parcels, etc.)
│   └── tax_database.json                           # 65 official county tax records indexed by parcel_apn
├── re_analyst/
│   ├── agent.py                                    # ADK App ('re_analyst') & root Agent ('re_opportunity_analyst')
│   ├── tools/
│   │   ├── lot_tool.py                             # get_lot_data(plot_id: str) -> str
│   │   └── tax_tool.py                             # get_tax_assessment_data(parcel_apn: str) -> str
│   └── skills/
│       ├── analyze-opportunity/SKILL.md            # Core property investigation, cost-per-acre derivation & neighbor discovery
│       ├── analyze-flood-risk/SKILL.md             # Subject & neighboring property FEMA flood zone & hydrological analysis
│       └── analyze-tax-implications/SKILL.md       # Official tax assessment lookup, monthly holding cost & land value/acre
└── eval_harness/
    ├── create_re_agent_evalset.py                  # Async batch runner building ADK 2.0 EvalSet JSON traces
    ├── run_geap_eval.py                            # Vertex AI GenAI Evaluation Service (GEAP) k-pass runner
    ├── metrics_calc.py                             # Krippendorff's Alpha, Pairwise Agreement, and N x k matrix statistics
    ├── single_criterion_analysis.py                # Deep-dive N x k grades matrix and summary stats for 1 criterion
    ├── all_criteria_analysis.py                    # Comparative statistical summary across all 19 criteria
    ├── run_hillclimb.py                            # Automated hill-climbing loop (k=1) for cost-per-acre & neighboring-flood-risk
    ├── explain_criteria_results.py                 # Filters and prints LLM judge rationales for scores <= threshold
    ├── utils.py                                    # Dynamic ADK app loader and rating DataFrame pivot helpers
    ├── rubrics.json                                # 19 pointwise evaluation criteria & 1-5 rating rubrics
    └── hillclimb_lots.txt                          # 10 target Lot IDs for baseline & hill-climbing runs
```

---

## 2. ADK Agent & Toolset Specification (`re_analyst/`)

### 2.1 Environment Configuration (`.env`)
```ini
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=qwiklabs-gcp-01-b40696192cf2
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
```

### 2.2 Agent & SkillToolset Wiring (`re_analyst/agent.py`)
- **Agent Name:** `re_opportunity_analyst` (`App` name: `re_analyst`)
- **Model Wrapper:** `google.adk.models.Gemini(model=MODEL, retry_options=types.HttpRetryOptions(initial_delay=1, attempts=3))`
- **Tool Registration:**
  - `lot_tool = FunctionTool(func=get_lot_data)`
  - `tax_tool = FunctionTool(func=get_tax_assessment_data)`
  - `skill_toolset = SkillToolset(skills=[opportunity_skill, flood_skill, tax_skill], additional_tools=[lot_tool, tax_tool])`
- **Mandatory Memorandum Output Schema (`SYSTEM_INSTRUCTION`):**
  1. `# Initial Site Evaluation Memorandum: [Lot ID]`
  2. `## Executive Summary`
  3. `## Acquisition Basis and Holding Costs`
  4. `## Zoning and Infrastructure Readiness`
  5. `## Risk Assessment`
  6. `## Recommendation`

### 2.3 Data Schemas (`tools_data/`)

#### `lots_database.json` Record Schema
```json
{
  "LOT-8472": {
    "plot_id": "LOT-8472",
    "parcel_apn": "049-382-011-000",
    "acreage": 1.2,
    "zoning": "R-3 (Multifamily)",
    "flood_zone": "Zone AE (High Risk)",
    "transit_proximity_miles": 0.4,
    "last_sale_price": 450000,
    "estimated_annual_taxes": 8500,
    "environmental_status": "Phase I ESA - Clean",
    "utility_access": "City Water & Sewer at boundary",
    "topography": "0-2% grade (Flat)",
    "adjacent_parcels": ["LOT-8471", "LOT-8473"],
    "soil_classification": "Urban Land-Inwood complex",
    "easement_type": "Public Utility (Rear 5ft)"
  }
}
```

#### `tax_database.json` Record Schema
```json
{
  "049-382-011-000": {
    "parcel_apn": "049-382-011-000",
    "plot_id": "LOT-1011",
    "tax_year": 2025,
    "annual_tax_amount": 23600.0,
    "assessed_land_value": 856000,
    "assessed_improvement_value": 0,
    "total_assessed_value": 856000,
    "tax_jurisdiction": "County Assessor Office",
    "payment_status": "Current - Paid in Full"
  }
}
```

---

## 3. Evaluation & Statistical Reliability Engine (`eval_harness/`)

### 3.1 ADK Dataset Builder (`create_re_agent_evalset.py`)
- Reads target lot IDs (`LOT-8472`, `LOT-8471`, `LOT-8473`, `LOT-1000` through `LOT-1006`) from `eval_harness/hillclimb_lots.txt`.
- Constructs `EvalCase` instances with:
  - `starting_prompt`: `"Please evaluate investment suitability for lot {lot_id} and generate a report."`
  - `conversation_plan`: `"User requests initial site evaluation memorandum for lot {lot_id}."`
- Uses `asyncio.Semaphore(max_concurrency=10)` and `google.adk.runners.Runner` with `InMemorySessionService` to run all 10 cases concurrently and convert session events into ADK 2.0 `EvalInvocation` traces via `EvaluationGenerator.convert_events_to_eval_invocations(session.events)`.

### 3.2 Vertex AI Gen AI Evaluation Runner (`run_geap_eval.py`)
- **Workaround for Vertex AI SDK PromptTemplate Bug:**
  ```python
  template = PointwiseMetricPromptTemplate(
      criteria=r["criteria"],
      rating_rubric=r["rating_rubric"],
      input_variables=["response"],
  )
  # Populate uninitialized 'variables' attribute due to missing super().__init__() in SDK
  template.variables = template._get_variables()
  ```
- **Workaround for `pyOpenSSL` Concurrent Connection Error:**
  ```python
  os.environ["GOOGLE_API_USE_MTLS"] = "never"
  os.environ["GOOGLE_API_USE_CLIENT_CERTIFICATE"] = "false"
  ```
- Dispatches `k_passes` sequential `EvalTask.evaluate()` runs to `us-central1` and saves assembled prompts + judge scores/explanations to JSON.

### 3.3 Statistical Reliability Formulas (`metrics_calc.py`)
Given an $N \times k$ ratings matrix $G$ where rows $i \in \{1, \dots, N\}$ are property reports and columns $j \in \{1, \dots, k\}$ are independent evaluation passes:

1. **Row Pairwise Percent Agreement ($\text{PA}_i$):**
   For report $i$ with $n_i$ non-null ratings and count $c_{i,v}$ for each distinct rating value $v$:
   $$\text{PA}_i = \frac{\sum_{v} c_{i,v}(c_{i,v} - 1)}{n_i(n_i - 1)}, \quad \text{Mean Pairwise Agree} = \frac{1}{N}\sum_{i=1}^{N} \text{PA}_i$$
2. **Krippendorff's Alpha ($\alpha$, Interval Scale):**
   $$\alpha = 1 - \frac{D_o}{D_e}$$
   where $D_o$ is the observed disagreement across passes within the same report and $D_e$ is the expected disagreement by chance across the entire dataset.
   - **Why the Homogeneity Trap Occurs:** When an autorater assigns the exact same score (e.g., `5`) to every report across every pass (`general-tone`, `logical-flow`, `professionalism`, `readability`), expected variation across reports approaches zero (`sd_report_means = 0`). Even though `pairwise %_agree = 100.00%` and library output prints `alpha = 1`, the criterion cannot distinguish between different reports (`min_rating = 5`, `max_rating = 5`), making it useless as an evaluation signal.
3. **Useful Metrics vs. Distractors for Autorater Self-Consistency:**

| Metric | Classification | Role in Reliability Analysis |
| :--- | :--- | :--- |
| **`alpha` (Krippendorff's Alpha)** | **Primary Reliability Metric** | Global chance-corrected agreement across passes ($\alpha > 0.85$ required; must be paired with high `sd_report_means`). |
| **`pairwise %_agree`** | **Primary Reliability Metric** | Average exact-match percentage across pass pairs within each report. |
| **`max range report`** | **Primary Reliability Metric** | Worst-case instability ($\max - \min$) on a single report across $k$ passes (ideal is $\le 1$). |
| **`sd report means`** | **Variance / Anti-Homogeneity Check** | Ensures the criterion spreads scores across reports (`> 0.30`); `0.0000` flags the **Homogeneity Trap**. |
| **`sd pass means`** | **Pass Drift Check** | Checks if an entire pass was systematically harsher/more lenient than another pass (low is expected). |
| **`mean` / `min rating` / `max rating`** | **Distractor (for consistency) / Scale Context** | Measures agent performance level and scale usage, not rater consistency by itself. |
| **`max range pass`** | **Distractor** | Measures score spread across *different* reports within a single pass; a high value (`3` or `4`) simply means the dataset has both good and bad reports. |

### 3.4 Complete 19-Criteria Statistical Analysis (`N=32, k=15`)

| Criterion ID | Mean | Min | Max | Max Range Report | SD Report Means | Pairwise % Agree | Alpha ($\alpha$) | Verdict |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **`monthly-tax-holding`** | `2.2764` | `2` | `5` | **`1`** | **`0.8872`** | **`99.17%`** | **`0.997`** | **SELECTED (#1 Reliable)** |
| **`flood-zone-flagging`** | `4.8447` | `3` | `5` | **`1`** | **`0.3357`** | **`94.41%`** | **`0.897`** | **SELECTED (#2 Reliable)** |
| **`environmental-esa`** | `4.2772` | `2` | `5` | `3` | **`0.7042`** | `79.10%` | **`0.897`** | **SELECTED (#3 Reliable)** |
| `risk-assessment` | `4.9532` | `3` | `5` | `2` | `0.1843` | `96.25%` | `0.803` | Borderline (Low variance) |
| `utility-topography` | `4.7756` | `4` | `5` | `1` | `0.3081` | `82.57%` | `0.751` | Moderate Alpha (`0.751`) |
| `neighboring-flood-risk` | `1.0316` | `1` | `2` | `0` | `0.1768` | `100.00%` | `1` | Floor Effect (31/32 scored 1) |
| `general-tone` | `5.0000` | `5` | `5` | `0` | `0.0000` | `100.00%` | `1` | **Homogeneity Trap** (All 5s) |
| `logical-flow` | `5.0000` | `5` | `5` | `0` | `0.0000` | `100.00%` | `1` | **Homogeneity Trap** (All 5s) |
| `professionalism` | `5.0000` | `5` | `5` | `0` | `0.0000` | `100.00%` | `1` | **Homogeneity Trap** (All 5s) |
| `readability` | `5.0000` | `5` | `5` | `0` | `0.0000` | `100.00%` | `1` | **Homogeneity Trap** (All 5s) |
| `data-density` | `4.9979` | `4` | `5` | `1` | `0.0118` | `99.58%` | `0.499` | Near-Homogeneity Trap |
| `professional-tone` | `4.9958` | `4` | `5` | `1` | `0.0164` | `99.17%` | `0.498` | Near-Homogeneity Trap |
| `persuasiveness` | `4.8840` | `4` | `5` | `1` | `0.2022` | `86.51%` | `0.672` | Unreliable ($\alpha < 0.70$) |
| `analyst-insight` | `4.7294` | `4` | `5` | `1` | `0.2570` | `71.59%` | `0.639` | Unreliable ($\alpha < 0.70$) |
| `vocabulary-choice` | `4.9349` | `4` | `5` | `1` | `0.1365` | `90.83%` | `0.623` | Unreliable ($\alpha < 0.70$) |
| `core-specifications` | `4.3623` | `4` | `5` | `1` | `0.2648` | `65.05%` | `0.621` | Unreliable ($\alpha < 0.70$) |
| `cost-per-acre` | `1.3771` | `1` | `3` | `2` | `0.2163` | `58.97%` | `0.565` | Noisy in baseline ($\alpha = 0.565$) |
| `report-length` | `4.7622` | `4` | `5` | `1` | `0.1715` | `67.20%` | `0.549` | Unreliable ($\alpha < 0.60$) |
| `document-formatting` | `4.8361` | `4` | `5` | `1` | `0.1488` | `75.22%` | `0.548` | Unreliable ($\alpha < 0.60$) |

---

## 4. Hill-Climbing Skill Refinements (Task 6 LLD)

### 4.1 Root Cause Diagnosis from `hillclimb_1`

1. **`cost-per-acre` (Baseline Mean: `1.1000`)**:
   - **Rubric Requirement for Score 5:** *"Accurately calculates and highlights exact cost-per-acre as a core valuation metric."* (Score 4: *"Correctly computes cost-per-acre with clear derivation, but does not benchmark or emphasize it as a core valuation metric."*)
   - **Root Cause:** None of the three baseline skills (`analyze-opportunity`, `analyze-tax-implications`, `analyze-flood-risk`) instructed the agent to divide `last_sale_price` by `acreage` or emphasize cost-per-acre as a core valuation metric.
   - **Target Lot Mathematical Reference Table:**

| Lot ID | Last Sale Price | Acreage | Exact Acquisition Cost-Per-Acre ($\text{Price} / \text{Acreage}$) | Adjacent Parcels |
| :--- | :---: | :---: | :---: | :--- |
| `LOT-8472` | `$450,000` | `1.2` | **`$375,000.00 / acre`** | `LOT-8471`, `LOT-8473` |
| `LOT-8471` | `$410,000` | `1.0` | **`$410,000.00 / acre`** | `LOT-8472` |
| `LOT-8473` | `$520,000` | `1.5` | **`$346,666.67 / acre`** | `LOT-8472` |
| `LOT-1000` | `$310,000` | `3.1` | **`$100,000.00 / acre`** | `LOT-1001` |
| `LOT-1001` | `$900,000` | `1.5` | **`$600,000.00 / acre`** | `LOT-1000`, `LOT-1002` |
| `LOT-1002` | `$680,000` | `4.4` | **`$154,545.45 / acre`** | `LOT-1001`, `LOT-1003` |
| `LOT-1003` | `$500,000` | `2.0` | **`$250,000.00 / acre`** | `LOT-1002`, `LOT-1004` |
| `LOT-1004` | `$500,000` | `1.6` | **`$312,500.00 / acre`** | `LOT-1003`, `LOT-1005` |
| `LOT-1005` | `$380,000` | `2.6` | **`$146,153.85 / acre`** | `LOT-1004`, `LOT-1006` |
| `LOT-1006` | `$450,000` | `1.7` | **`$264,705.88 / acre`** | `LOT-1005`, `LOT-1007` |

2. **`neighboring-flood-risk` (Baseline Mean: `1.0000`)**:
   - **Rubric Requirement for Score 5:** *"Exhaustively investigates neighboring property flood zones and critically analyzes potential drainage, runoff, and shared hydrological impacts on the subject parcel's development feasibility."*
   - **Root Cause:** Although `get_lot_data` returns `"adjacent_parcels": [...]`, `analyze-flood-risk/SKILL.md` did not declare `get_lot_data` in `metadata.adk_additional_tools` and only instructed the agent to check the subject parcel's `flood_zone`.

### 4.2 Skill Modifications Applied (`re_analyst/skills/`)
- **`analyze-opportunity/SKILL.md`:**
  - Added `get_tax_assessment_data` alongside `get_lot_data` in `metadata.adk_additional_tools`.
  - Added mandatory step to iterate over `adjacent_parcels` and invoke `get_lot_data` for every neighboring lot ID.
  - Added mandatory step to derive exact **Acquisition Cost-Per-Acre** ($\text{Last Sale Price} / \text{Acreage}$), show the explicit formula, highlight it as a core valuation metric in both `Executive Summary` and `Acquisition Basis and Holding Costs`, and benchmark it against Assessed Land Value Per Acre and neighboring lots' Cost-Per-Acre.
- **`analyze-flood-risk/SKILL.md`:**
  - Added `metadata.adk_additional_tools: [get_lot_data]`.
  - Added mandatory step to query all `adjacent_parcels` via `get_lot_data`, build a comparative table of FEMA flood zone classifications, topography, soil classifications, and easements, and critically analyze cross-boundary stormwater runoff, subsurface hydrology, floodplain storage displacement / FEMA No Adverse Impact (NAI), and development feasibility mitigations.
- **`analyze-tax-implications/SKILL.md`:**
  - Added `get_lot_data` to `metadata.adk_additional_tools`.
  - Reinforced exact derivation of **Acquisition Cost-Per-Acre** ($\text{Last Sale Price} / \text{Acreage}$), **Assessed Land Value Per Acre** ($\text{Assessed Land Value} / \text{Acreage}$), and **Monthly Tax Holding Cost** ($\text{Official Annual Tax} / 12$).
