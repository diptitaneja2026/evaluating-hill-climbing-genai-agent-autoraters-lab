# Step-by-Step Operational Runbook — Evaluating and Hill Climbing a Gen AI Agent Using Autoraters (L400-APEX006)

> **Lab URL:** [https://explore.qwiklabs.com/classrooms/21504/labs/115876](https://explore.qwiklabs.com/classrooms/21504/labs/115876)  
> **Lab Code:** `L400-APEX006`  
> **Source:** Compiled from verified Cloud Shell session history (`~/.bash_history`) and Antigravity (`agy`) agent execution logs.

---

## Task 1. Set Up Your Environment

### Step 1.1: Configure GCP Project & Authenticate Application Default Credentials (ADC)
```bash
gcloud config set project qwiklabs-gcp-01-b40696192cf2
gcloud auth application-default login
```
*(Replace `qwiklabs-gcp-01-b40696192cf2` with your active Qwiklabs Project ID if running in a new lab instance.)*

### Step 1.2: Download the Lab Codebase from Cloud Storage
```bash
cd ~
gcloud storage cp -r gs://$(gcloud config get-value project)-bucket/autoraters .
```

### Step 1.3: Install Python 3.13 via `pyenv` & Install Lab Dependencies
```bash
sudo apt-get update
sudo apt-get install -y libreadline-dev

curl https://pyenv.run | bash

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc

pyenv install 3.13
pyenv global 3.13

python3 -m pip install -r autoraters/requirements.txt
```

### Step 1.4: Create the `.env` Configuration File for `re_analyst`
```bash
cd ~/autoraters
cat << EOF > .env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project)
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.5-flash
EOF

gcloud config set project $(gcloud config get-value project)
```

---

## Task 2. Run, Test, and Understand the Agent

### Step 2.1: Launch the ADK Web UI
```bash
cd ~/autoraters
source ~/.bashrc
pyenv global 3.13
adk web --allow_origins "regex:https://.*\.cloudshell\.dev"
```

### Step 2.2: Test Interactive Evaluation in ADK Dev UI
1. Open `http://127.0.0.1:8000` via Cloud Shell Web Preview.
2. Select **`re_analyst`** from the top-left application dropdown.
3. Submit the test prompt:
   ```text
   Please evaluate investment suitability for lot LOT-8472 and generate a report.
   ```
4. Observe the agent loading `analyze-opportunity`, `analyze-flood-risk`, and `analyze-tax-implications`, calling `get_lot_data("LOT-8472")` and `get_tax_assessment_data("049-382-011-000")`, and generating the initial site evaluation memorandum.

---

## Task 3. Generate Agent Responses (Batch ADK EvalSet)

### Step 3.1: Activate Python 3.13 in a New Terminal & Run the Dataset Builder
```bash
cd ~/autoraters
source ~/.bashrc
pyenv global 3.13
gcloud config set project $(gcloud config get-value project)

python3 -m eval_harness.create_re_agent_evalset \
  --lots-file eval_harness/hillclimb_lots.txt \
  --out ./re_analyst/hillclimb_lots.evalset.json \
  --agent-dir ./re_analyst
```

**Verified Output:**
```text
🚀 Executing 're_analyst' over 10 properties to populate evaluation traces...
-----------------------------------------------------------------
⚡ Starting concurrent execution across 10 properties (max concurrency: 10)...
✅ [1/10] Completed trace for LOT-8472
...
✅ [10/10] Completed trace for LOT-1006

Cleaning up resources...
✅ Successfully generated real estate evaluation dataset for 10 properties -> ./re_analyst/hillclimb_lots.evalset.json
```

---

## Task 4. Evaluate the Agent Responses with LLM-as-a-Judge

### Step 4.1: Perform a Dry Run (`--dry-run`) to Inspect Compiled Prompts
```bash
cd ~/autoraters
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --dry-run \
  --k 3
```
- Inspect `runs/baseline/geap_calls_and_responses.json` to verify `"prompt_made_to_service"` and `"is_dry_run": true`.

### Step 4.2: Execute Live Multi-Pass Cloud Evaluations (`--k 3`)
```bash
cd ~/autoraters
python3 -m eval_harness.run_geap_eval \
  --traces re_analyst/hillclimb_lots.evalset.json \
  --rubric-file eval_harness/rubrics.json \
  --out runs/baseline/geap_calls_and_responses.json \
  --experiment-name rubric-agreement-check \
  --k 3
```
- Dispatches 3 evaluation passes (`pass-1`, `pass-2`, `pass-3`) across 10 reports $\times$ 19 criteria to the Vertex AI Gen AI Evaluation Service and populates `responses_from_service` with `score` and `explanation`.

---

## Task 5. Analyze Autorater Intra-Rater Reliability (`AT ID: 1`)

### Step 5.1: Inspect Single Criterion Score Matrix (`environmental-esa`)
```bash
cd ~/autoraters
python3 -m eval_harness.single_criterion_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json \
  --criterion environmental-esa
```

**Verified Output Summary (`environmental-esa`):**
```text
=========================================
🔍 Criterion ID: environmental-esa
📋 Name:         Environmental Status (Phase I ESA) Evaluation
=========================================

📈 Criterion Metrics Summary:
| Metric                                            | Short Name       | Value                    |
|---------------------------------------------------|------------------|--------------------------|
| Mean Score                                        | mean             | 4.2772                   |
| Min All Ratings                                   | min rating       | 2                        |
| Max All Ratings                                   | max rating       | 5                        |
| Widest Range Single Report                        | max range report | 3 (Lot: LOT-1028)        |
| Widest Range Single Pass                          | max range pass   | 3 (Pass: p1, p2, p3 ...) |
| StdDev of Report Means                            | sd report means  | 0.7042                   |
| StdDev of Pass Means                              | sd pass means    | 0.0991                   |
| Mean Pairwise Percent Agreement of Report Ratings | pairwise %_agree | 79.10%                   |
| Krippendorff's Alpha                              | alpha            | 0.8970                   |
==================================================================================================
```

### Step 5.2: Compute Summary & Reliability Metrics Across All 19 Criteria
```bash
cd ~/autoraters
python3 -m eval_harness.all_criteria_analysis \
  --run_geap_eval_out runs/geap_calls_and_responses_for_analysis.json
```

**Verified Output Table:**
```text
=== GEAP Evaluation Metrics Summary ===
| criterion_id           | criterion_name                                | mean   | min    | max    | max range | max range | sd report | sd pass | pairwise | alpha |
|                        |                                               |        | rating | rating | report    | pass      | means     | means   | %_agree  |       |
| ---------------------- | --------------------------------------------- | ------ | ------ | ------ | --------- | --------- | --------- | ------- | -------- | ----- |
| analyst-insight        | Deep Analyst Insights                         | 4.7294 |      4 |      5 |         1 |         1 |     0.257 |  0.0576 |   71.59% | 0.639 |
| core-specifications    | Explicit Plot Core Specifications             | 4.3623 |      4 |      5 |         1 |         1 |    0.2648 |  0.0765 |   65.05% | 0.621 |
| cost-per-acre          | Acquisition Cost-Per-Acre Calculation         | 1.3771 |      1 |      3 |         2 |         2 |    0.2163 |  0.0511 |   58.97% | 0.565 |
| data-density           | Information and Data Density                  | 4.9979 |      4 |      5 |         1 |         1 |    0.0118 |  0.0081 |   99.58% | 0.499 |
| document-formatting    | Document Readability & Formatting             | 4.8361 |      4 |      5 |         1 |         1 |    0.1488 |  0.0487 |   75.22% | 0.548 |
| environmental-esa      | Environmental Status (Phase I ESA) Evaluation | 4.2772 |      2 |      5 |         3 |         3 |    0.7042 |  0.0991 |   79.10% | 0.897 |
| flood-zone-flagging    | Precise Flood Zone Flagging                   | 4.8447 |      3 |      5 |         1 |         2 |    0.3357 |  0.0256 |   94.41% | 0.897 |
| general-tone           | General Tone Suitability                      |      5 |      5 |      5 |         0 |         0 |         0 |       0 |  100.00% |     1 |
| logical-flow           | Logical Narrative Flow                        |      5 |      5 |      5 |         0 |         0 |         0 |       0 |  100.00% |     1 |
| monthly-tax-holding    | Monthly Tax Holding Cost Calculation          | 2.2764 |      2 |      5 |         1 |         3 |    0.8872 |  0.0228 |   99.17% | 0.997 |
| neighboring-flood-risk | Neighboring Property Flood Risk Analysis      | 1.0316 |      1 |      2 |         0 |         1 |    0.1768 |  0.0009 |  100.00% |     1 |
| persuasiveness         | Document Persuasiveness                       |  4.884 |      4 |      5 |         1 |         1 |    0.2022 |  0.0395 |   86.51% | 0.672 |
| professional-tone      | Professional Tone                             | 4.9958 |      4 |      5 |         1 |         1 |    0.0164 |   0.011 |   99.17% | 0.498 |
| professionalism        | Analyst Professionalism                       |      5 |      5 |      5 |         0 |         0 |         0 |       0 |  100.00% |     1 |
| readability            | General Reading Ease                          |      5 |      5 |      5 |         0 |         0 |         0 |       0 |  100.00% |     1 |
| report-length          | Report Detail and Length                      | 4.7622 |      4 |      5 |         1 |         1 |    0.1715 |  0.0721 |   67.20% | 0.549 |
| risk-assessment        | Comprehensive Risk Assessment                 | 4.9532 |      3 |      5 |         2 |         2 |    0.1843 |  0.0306 |   96.25% | 0.803 |
| utility-topography     | Utility & Topography Evaluation               | 4.7756 |      4 |      5 |         1 |         1 |    0.3081 |  0.0695 |   82.57% | 0.751 |
| vocabulary-choice      | Advanced Real Estate Vocabulary               | 4.9349 |      4 |      5 |         1 |         1 |    0.1365 |  0.0391 |   90.83% | 0.623 |
======================================================================================================================================================================
```

### Step 5.3: Write Top 3 Reliable Criteria & Upload to Cloud Storage
Why `monthly-tax-holding`, `environmental-esa`, and `flood-zone-flagging` are chosen:
- Avoids the **Homogeneity Trap** (`general-tone`, `logical-flow`, `professionalism`, and `readability` have `sd report means = 0` because every single report gets a `5`).
- Avoids the **Floor Effect Trap** (`neighboring-flood-risk` gives `1` to 31 out of 32 baseline reports, `sd report means = 0.1768`).
- Achieves the highest **Krippendorff's Alpha ($\alpha \ge 0.897$)** combined with strong report-to-report discrimination (`sd report means` of `0.8872`, `0.7042`, and `0.3357`).

```bash
cd ~/autoraters
echo -n "monthly-tax-holding,environmental-esa,flood-zone-flagging" > best_criteria_submission.txt
gcloud storage cp best_criteria_submission.txt gs://$(gcloud config get-value project)-bucket/best_criteria_submission.txt
```
- Click **Check my progress** for **Task 5 (`AT ID: 1 Choose reliable criteria`)**.

---

## Task 6. Hill Climb Agent Skills to Raise Autorater Criteria Scores (`AT ID: 2`)

### Step 6.1: Execute Baseline Hill-Climbing Run (`hillclimb_1`)
```bash
cd ~/autoraters
python3 -m eval_harness.run_hillclimb
```

**Baseline Output (`hillclimb_1`):**
```text
=========================================
📊 Hill Climbing Results: hillclimb_1
=========================================
 • cost-per-acre             Mean Score: 1.1000
 • neighboring-flood-risk    Mean Score: 1.0000
=========================================
```

### Step 6.2: Diagnose Baseline Failures Using `explain_criteria_results.py`
```bash
cd ~/autoraters
python3 -m eval_harness.explain_criteria_results \
   --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
   --criterion cost-per-acre \
   --max_score_to_show 4.0

python3 -m eval_harness.explain_criteria_results \
   --run_geap_eval_out runs/hillclimb_1/geap_calls_and_responses_hillclimb_1.json \
   --criterion neighboring-flood-risk \
   --max_score_to_show 4.0
```
- **`cost-per-acre` Diagnosis:** The baseline skills never instruct the agent to compute $\text{Last Sale Price} / \text{Acreage}$, show the explicit derivation formula, or highlight/benchmark it as a core valuation metric.
- **`neighboring-flood-risk` Diagnosis:** Each lot record in `tools_data/lots_database.json` includes an `"adjacent_parcels"` list (e.g., `["LOT-8471", "LOT-8473"]`), but the baseline `analyze-flood-risk` skill neither declares `get_lot_data` in `adk_additional_tools` nor instructs the agent to look up neighboring parcels' FEMA flood zones and analyze shared drainage/runoff impacts.

### Step 6.3: Refine Agent Skills (`re_analyst/skills/*/SKILL.md`)

#### 1. Update `re_analyst/skills/analyze-opportunity/SKILL.md`
```bash
cat << 'EOF' > ~/autoraters/re_analyst/skills/analyze-opportunity/SKILL.md
---
name: analyze-opportunity
description: Investigates a land parcel and its adjacent neighboring parcels for real estate investment opportunity, core cost-per-acre valuation, and hydrological risk analysis.
metadata:
  adk_additional_tools:
    - get_lot_data
    - get_tax_assessment_data
---
# Analyze Opportunity Skill

When asked to investigate a property or analyze an investment opportunity for a lot, you MUST follow all steps below thoroughly:

1. **Retrieve Subject Parcel Data**:
   - Use the `get_lot_data` tool to lookup the subject property attributes by its lot ID (e.g., `LOT-8472`).
   - Review all returned attributes, including `acreage`, `zoning`, `last_sale_price`, `flood_zone`, `topography`, `soil_classification`, `easement_type`, `environmental_status`, `parcel_apn`, and `adjacent_parcels`.

2. **Retrieve All Neighboring / Adjacent Parcels Data (MANDATORY)**:
   - Inspect the `adjacent_parcels` array from the subject lot's record.
   - You MUST call `get_lot_data` for **every single lot ID** listed in `adjacent_parcels` so you can investigate their exact FEMA `flood_zone` classifications, `topography`, `soil_classification`, `easement_type`, `acreage`, and `last_sale_price`.

3. **Calculate and Highlight Acquisition Cost-Per-Acre as a Core Valuation Metric (MANDATORY)**:
   - Derive and state the exact mathematical **Acquisition Cost-Per-Acre** based on the subject parcel's `last_sale_price` and `acreage` using the explicit formula:
     $$\text{Acquisition Cost-Per-Acre} = \frac{\text{Last Sale Price}}{\text{Acreage}}$$
   - State the exact formula, the substituted numbers, and the exact resulting dollar figure per acre (e.g., `$\$450,000 / 1.2\text{ acres} = \$375,000.00\text{ per acre}$`). Double-check all arithmetic to ensure 100% mathematical accuracy (round repeating decimals to two decimal places, e.g., `$520,000 / 1.5 acres = $346,666.67/acre`, `$680,000 / 4.4 acres = $154,545.45/acre`, `$380,000 / 2.6 acres = $146,153.85/acre`, `$450,000 / 1.7 acres = $264,705.88/acre`).
   - **Highlight and emphasize Acquisition Cost-Per-Acre as a CORE VALUATION METRIC** in **both** the `## Executive Summary` and a dedicated subsection under `## Acquisition Basis and Holding Costs`.
   - **Benchmark** the subject parcel's Acquisition Cost-Per-Acre against:
     1. The county **Assessed Land Value Per Acre** ($\text{Assessed Land Value} / \text{Acreage}$) from the tax assessment records, and
     2. The **Acquisition Cost-Per-Acre of the Neighboring/Adjacent Parcels** ($\text{Neighbor Last Sale Price} / \text{Neighbor Acreage}$) to evaluate relative land valuation and acquisition basis competitiveness.

4. **Consult Specialized Flood Risk and Tax Skills**:
   - Consult the `analyze-flood-risk` skill to perform an exhaustive analysis of both the subject parcel's flood zone and all neighboring/adjacent properties' FEMA flood zone classifications, drainage, runoff, and shared hydrological impacts on the subject parcel's development feasibility.
   - Consult the `analyze-tax-implications` skill and call `get_tax_assessment_data` using the subject lot's `parcel_apn` to investigate official property tax holding costs (annual and monthly) and assessed valuations.
EOF
```

#### 2. Update `re_analyst/skills/analyze-flood-risk/SKILL.md`
```bash
cat << 'EOF' > ~/autoraters/re_analyst/skills/analyze-flood-risk/SKILL.md
---
name: analyze-flood-risk
description: Investigates subject and neighboring property FEMA flood zone designations, drainage, runoff, and shared hydrological risks for real estate investment opportunity analysis.
metadata:
  adk_additional_tools:
    - get_lot_data
---
# Analyze Flood Risk Skill

When evaluating environmental, flood, or hydrological risks for a subject lot, you MUST evaluate **both** the subject parcel AND all neighboring/adjacent parcels:

1. **Subject Parcel Flood & Environmental Investigation**:
   - Check the exact FEMA `flood_zone` classification (e.g., `Zone AE (High Risk)`, `Zone VE (Coastal High Hazard)`, `Zone A (100-year Floodplain)`, or `Zone X (Minimal Risk)`), `topography` (slope grade), `soil_classification`, `easement_type`, `tree_canopy_coverage_pct`, and `environmental_status` (Phase I / Phase II ESA) of the subject lot.

2. **Neighboring / Adjacent Properties Flood Investigation (MANDATORY)**:
   - Check the `adjacent_parcels` field from the subject parcel's data.
   - Call `get_lot_data` for **every lot ID** listed in `adjacent_parcels` if not already retrieved.
   - Record the exact FEMA `flood_zone` designation, `topography`, `soil_classification`, and `easement_type` for each neighboring property.

3. **Exhaustive Neighboring Flood Risk & Shared Hydrological Analysis (MANDATORY IN REPORT)**:
   - In the `## Risk Assessment` section (and summarized in `## Executive Summary` and `## Recommendation`), include a dedicated subsection on **Neighboring Property Flood Zones & Shared Hydrological Impact Analysis**.
   - **Explicitly state the specific FEMA flood zone classifications of every neighboring/adjacent parcel by Lot ID** (including a side-by-side comparison table of Subject vs. Neighboring Lots showing Lot ID, FEMA Flood Zone, Topography/Slope, Soil Classification, and Easements).
   - **Critically analyze potential drainage, stormwater runoff, and shared hydrological impacts on the subject parcel's development feasibility**:
     - **Topographic & Cross-Boundary Runoff Dynamics**: Analyze how the slope grades (`topography`) of the subject lot and its neighbors interact (e.g., upslope runoff sheet flow onto flatter downslope parcels, or ponding/staging on 0-2% flat grades).
     - **Soil Permeability & Subsurface Hydrology**: Analyze how `soil_classification` (e.g., `Urban Land-Inwood complex`, `Clay loam (Moderate shrink-swell potential)`, `Sandy loam (Good drainage)`, `Bedrock outcrop / Shallow rocky soil`) across the contiguous lots affects infiltration rates, water table elevation, hydrostatic foundation pressure, and stormwater retention performance.
     - **Floodplain Connectivity & Backwater/Encroachment Effects**: Even if the subject parcel is in `Zone X (Minimal Risk)`, critically evaluate whether adjacent parcels in high-risk zones (`Zone AE`, `Zone VE`, `Zone A`) create boundary floodwater encroachment, stormwater outfall tailwater/backwater lock, or coastal velocity/surge exposure. Conversely, if the subject lot is in `Zone AE`, `Zone VE`, or `Zone A`, analyze shared floodplain storage displacement, FEMA "No Adverse Impact" (NAI) / zero-rise certification requirements (ensuring site fill or podium elevation does not divert floodwater onto adjacent lots), and regional drainage easements.
     - **Impact on Development Feasibility & Engineering Mitigation**: Detail how these shared hydrological conditions impact the subject parcel's buildable footprint, grading plan, foundation engineering, perimeter drainage infrastructure (interceptor swales, retention/detention basins, French drains), permitting timeline, and overall development feasibility.
EOF
```

#### 3. Update `re_analyst/skills/analyze-tax-implications/SKILL.md`
```bash
cat << 'EOF' > ~/autoraters/re_analyst/skills/analyze-tax-implications/SKILL.md
---
name: analyze-tax-implications
description: Investigates property tax holding costs, assessment records, and acquisition cost-per-acre valuation metrics for a parcel.
metadata:
  adk_additional_tools:
    - get_tax_assessment_data
    - get_lot_data
---
# Analyze Tax Implications Skill

When investigating financial carrying costs, acquisition valuation basis, or tax implications for a lot:

1. **Retrieve Official Tax Assessment Data**:
   - Locate the `parcel_apn` from the lot property records.
   - Use the `get_tax_assessment_data` tool with that APN to retrieve official county tax assessment details (`annual_tax_amount`, `assessed_land_value`, `assessed_improvement_value`, `total_assessed_value`, `tax_year`, `payment_status`).
   - Calculate the exact **monthly tax holding cost** ($\text{Official Annual Tax Amount} / 12$) and compare the official annual tax against the preliminary `estimated_annual_taxes` from the lot record.

2. **Derive, Highlight, and Benchmark Acquisition Cost-Per-Acre as a Core Valuation Metric (MANDATORY)**:
   - Derive and state the exact mathematical **Acquisition Cost-Per-Acre** using the subject lot's `last_sale_price` and `acreage`:
     $$\text{Acquisition Cost-Per-Acre} = \frac{\text{Last Sale Price}}{\text{Acreage}}$$
   - Explicitly show the mathematical derivation formula and exact calculation in your report (e.g., `$\$450,000 / 1.2\text{ acres} = \$375,000.00\text{ per acre}$`).
   - Also compute the **Assessed Land Value Per Acre**:
     $$\text{Assessed Land Value Per Acre} = \frac{\text{Assessed Land Value}}{\text{Acreage}}$$
   - **Highlight and emphasize Acquisition Cost-Per-Acre as a CORE VALUATION METRIC** in both the `## Executive Summary` and `## Acquisition Basis and Holding Costs` sections, benchmarking the subject lot's Acquisition Cost-Per-Acre against its Assessed Land Value Per Acre and the Acquisition Cost-Per-Acre of its adjacent neighboring parcels (`adjacent_parcels`).
EOF
```

### Step 6.4: Execute the Hill-Climbed Evaluation Run (`hillclimb_2`)
```bash
cd ~/autoraters
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PYENV_ROOT/shims:$PATH"
pyenv global 3.13
export GOOGLE_CLOUD_PROJECT=$(gcloud config get-value project)

python3 -m eval_harness.run_hillclimb
```

**Verified Output (`hillclimb_2`):**
```text
📂 Created versioned run directory: /home/student_03_e7878e38169e/autoraters/runs/hillclimb_2

🤖 Running agent 're_analyst' over target lots to generate traces...

🚀 Executing 're_analyst' over 10 properties to populate evaluation traces...
-----------------------------------------------------------------
⚡ Starting concurrent execution across 10 properties (max concurrency: 10)...
✅ [5/10] Completed trace for LOT-1001
✅ [10/10] Completed trace for LOT-1006
✅ [3/10] Completed trace for LOT-8473
✅ [7/10] Completed trace for LOT-1003
✅ [1/10] Completed trace for LOT-8472
✅ [9/10] Completed trace for LOT-1005
✅ [2/10] Completed trace for LOT-8471
✅ [6/10] Completed trace for LOT-1002
✅ [8/10] Completed trace for LOT-1004
✅ [4/10] Completed trace for LOT-1000

Cleaning up resources...

☁️ Dispatching single-pass (k=1) evaluation for criteria: ['cost-per-acre', 'neighboring-flood-risk']...
🧪 Vertex AI Experiment Name: hillclimb-20261029-093936
  ☁️ Dispatching EvalTask pass 1/1 to Vertex AI...
All 20 metric requests are successfully computed.
Evaluation Took:12.122431017000054 seconds
✅ Evaluation results saved to -> /home/student_03_e7878e38169e/autoraters/runs/hillclimb_2/geap_calls_and_responses_hillclimb_2.json

=========================================
📊 Hill Climbing Results: hillclimb_2
=========================================
 • cost-per-acre             Mean Score: 5.0000
 • neighboring-flood-risk    Mean Score: 5.0000
=========================================
```

- Click **Check my progress** for **Task 6 (`AT ID: 2 Perform hill-climbing automation`)**.
