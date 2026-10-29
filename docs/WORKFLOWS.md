# Step-by-Step Workflows — Evaluating and Hill Climbing a Gen AI Agent Using Autoraters (L400-APEX006)

> **Lab URL:** [https://explore.qwiklabs.com/classrooms/21504/labs/115876](https://explore.qwiklabs.com/classrooms/21504/labs/115876)  
> **Lab Code:** `L400-APEX006`

---

## 1. Master End-to-End Lab Workflow (Tasks 1–6)

```mermaid
flowchart LR
    T1["Task 1: Environment Setup<br/>- ADC Auth<br/>- Copy gs://.../autoraters<br/>- Install Python 3.13 via pyenv<br/>- Configure .env"]
    T2["Task 2: Interactive Agent Test<br/>- Launch adk web (:8000)<br/>- Test LOT-8472 report<br/>- Inspect SkillToolset & Tools"]
    T3["Task 3: Batch Trace Generation<br/>- Run create_re_agent_evalset.py<br/>- 10 target lots concurrently<br/>- Output: hillclimb_lots.evalset.json"]
    T4["Task 4: LLM-as-a-Judge (GEAP)<br/>- Step 1: Dry run (--dry-run --k 3)<br/>- Step 2: Live run (--k 3)<br/>- Inspect Vertex AI Experiments"]
    T5["Task 5: Intra-Rater Reliability<br/>- Analyze 32 lots x 19 criteria x 15 passes<br/>- Filter out Homogeneity Trap<br/>- Submit top 3 reliable criteria to GCS"]
    T6["Task 6: Hill Climbing Loop<br/>- Run hillclimb_1 (Baseline: 1.1 / 1.0)<br/>- Diagnose via explain_criteria_results.py<br/>- Refine 3 SKILL.md files<br/>- Run hillclimb_2 (Final: 5.0 / 5.0)"]

    T1 --> T2 --> T3 --> T4 --> T5 --> T6
```

---

## 2. Task 1 Workflow: Environment Provisioning & Authentication

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Developer (Cloud Shell)
    participant GCS as Cloud Storage Bucket
    participant Pyenv as pyenv (Python 3.13)
    participant Env as autoraters/.env

    Dev->>Dev: gcloud config set project qwiklabs-gcp-01-b40696192cf2
    Dev->>Dev: gcloud auth application-default login
    Dev->>GCS: gcloud storage cp -r gs://...-bucket/autoraters .
    GCS-->>Dev: Downloads re_analyst/, eval_harness/, tools_data/, runs/
    Dev->>Pyenv: Install libreadline-dev, pyenv, Python 3.13
    Dev->>Pyenv: pyenv global 3.13 && pip install -r autoraters/requirements.txt
    Dev->>Env: Write GOOGLE_GENAI_USE_VERTEXAI=TRUE, PROJECT, LOCATION=global, MODEL=gemini-3.5-flash
```

---

## 3. Task 2 & Task 3 Workflow: ADK Agent Execution & EvalSet Generation

```mermaid
sequenceDiagram
    autonumber
    participant Harness as create_re_agent_evalset.py
    participant Runner as ADK Runner (max_concurrency=10)
    participant Agent as re_opportunity_analyst (gemini-3.5-flash)
    participant Skills as SkillToolset (SKILL.md)
    participant Tools as lot_tool / tax_tool
    participant EvalSet as hillclimb_lots.evalset.json

    Harness->>Harness: Read 10 Lot IDs from eval_harness/hillclimb_lots.txt
    loop Concurrently for each Lot ID (LOT-8472 .. LOT-1006)
        Harness->>Runner: run_async("Please evaluate investment suitability for lot {lot_id}...")
        Runner->>Agent: Invoke root_agent with SYSTEM_INSTRUCTION
        Agent->>Skills: Load analyze-opportunity, analyze-flood-risk, analyze-tax-implications
        Skills-->>Agent: Return domain investigation instructions
        Agent->>Tools: get_lot_data(plot_id=subject_lot)
        Tools-->>Agent: Return parcel JSON (acreage, last_sale_price, flood_zone, adjacent_parcels, parcel_apn)
        opt Hill-Climbed Agent (Task 6)
            Agent->>Tools: get_lot_data(plot_id=neighbor_lot) for each ID in adjacent_parcels
            Tools-->>Agent: Return neighboring parcel JSON (FEMA flood_zone, topography, soil, easements)
        end
        Agent->>Tools: get_tax_assessment_data(parcel_apn)
        Tools-->>Agent: Return official 2025 tax assessment JSON
        Agent-->>Runner: Final Markdown Site Evaluation Memorandum
        Runner->>EvalSet: Convert session.events -> ADK 2.0 EvalInvocation
    end
```

---

## 4. Task 4 Workflow: Multi-Pass LLM-as-a-Judge Evaluation (`run_geap_eval.py`)

```mermaid
sequenceDiagram
    autonumber
    participant CLI as Operator / CLI
    participant Geap as GeapEvalRunner (run_geap_eval.py)
    participant Rubrics as eval_harness/rubrics.json
    participant Vertex as Vertex AI Gen AI Eval Service (us-central1)
    participant Out as runs/baseline/geap_calls_and_responses.json

    CLI->>Geap: Step 1: --dry-run --k 3
    Geap->>Rubrics: Load 19 criteria & 1-5 rating scales
    Geap->>Geap: Build PointwiseMetricPromptTemplate & assemble(response=report)
    Geap->>Out: Save rendered prompts with "is_dry_run": true

    CLI->>Geap: Step 2: Live execution (--k 3, no --dry-run)
    Geap->>Vertex: vertexai.init(location="us-central1")
    loop eval_pass_idx = 1 to k (k=3)
        Geap->>Vertex: EvalTask(dataset=dataset_df, metrics=pointwise_metrics).evaluate()
        Note over Vertex: Internally samples Judge LLM 4x<br/>and takes majority vote per metric
        Vertex-->>Geap: metrics_table ({criterion_id}/score, {criterion_id}/explanation)
    end
    Geap->>Out: Assemble prompts + responses_from_service (k=3 passes per lot x criterion)
```

---

## 5. Task 5 Workflow: Intra-Rater Reliability & Homogeneity Trap Filtering

```mermaid
flowchart TD
    Input[("runs/geap_calls_and_responses_for_analysis.json<br/>(N=32 Lots x 19 Criteria x k=15 Passes)")]
    AllCrit["Run all_criteria_analysis.py<br/>Compute Mean, Min/Max, Max Range Report,<br/>SD Report Means, Pairwise % Agree, Alpha"]
    
    Filter1{"Check SD Report Means > 0.30<br/>and Min < Max?"}
    Trap["Discard Homogeneity / Floor Traps:<br/>- general-tone (SD=0, all 5s)<br/>- logical-flow (SD=0, all 5s)<br/>- professionalism (SD=0, all 5s)<br/>- readability (SD=0, all 5s)<br/>- data-density (SD=0.0118)<br/>- professional-tone (SD=0.0164)<br/>- neighboring-flood-risk (SD=0.1768, 31/32=1)"]

    Filter2{"Check Krippendorff's Alpha > 0.85?"}
    Noisy["Discard Low/Moderate Agreement Criteria:<br/>- cost-per-acre (Alpha=0.565)<br/>- document-formatting (Alpha=0.548)<br/>- report-length (Alpha=0.549)<br/>- core-specifications (Alpha=0.621)<br/>- vocabulary-choice (Alpha=0.623)<br/>- analyst-insight (Alpha=0.639)<br/>- persuasiveness (Alpha=0.672)<br/>- utility-topography (Alpha=0.751)<br/>- risk-assessment (Alpha=0.803, SD=0.1843)"]

    Winners["Select Top 3 Reliable Criteria:<br/>1. monthly-tax-holding (Alpha=0.997, SD=0.8872, Agree=99.17%)<br/>2. flood-zone-flagging (Alpha=0.897, SD=0.3357, Agree=94.41%)<br/>3. environmental-esa (Alpha=0.897, SD=0.7042, Agree=79.10%)"]

    Submit["Write to best_criteria_submission.txt<br/>and upload to gs://...-bucket/best_criteria_submission.txt"]

    Input --> AllCrit --> Filter1
    Filter1 -->|"No (Zero/Near-Zero Variance)"| Trap
    Filter1 -->|"Yes (True Report Discrimination)"| Filter2
    Filter2 -->|"No (Alpha <= 0.85)"| Noisy
    Filter2 -->|"Yes (Alpha > 0.85)"| Winners --> Submit
```

---

## 6. Task 6 Workflow: Iterative Hill-Climbing Automation

```mermaid
flowchart TB
    Start["Step 1: Run Baseline Hill Climb<br/>python3 -m eval_harness.run_hillclimb<br/>Creates runs/hillclimb_1/"]
    BaseScore["Baseline Scores (hillclimb_1):<br/>- cost-per-acre: 1.1000<br/>- neighboring-flood-risk: 1.0000"]
    Explain["Step 2: Diagnose Deficiencies<br/>python3 -m eval_harness.explain_criteria_results<br/>--criterion cost-per-acre / neighboring-flood-risk<br/>--max_score_to_show 4.0"]
    InspectRubric["Inspect eval_harness/rubrics.json<br/>& tools_data/lots_database.json<br/>Discover 'adjacent_parcels' field & 5-point rubric rules"]
    EditSkills["Step 3: Refine Agent Skills (re_analyst/skills/):<br/>1. analyze-opportunity/SKILL.md<br/>2. analyze-flood-risk/SKILL.md<br/>3. analyze-tax-implications/SKILL.md"]
    Run2["Re-run Hill Climb Harness<br/>python3 -m eval_harness.run_hillclimb<br/>Creates runs/hillclimb_2/"]
    CheckScore{"Both Mean Scores > 4.0?"}
    Pass["Final Scores (hillclimb_2):<br/>- cost-per-acre: 5.0000 / 5.0<br/>- neighboring-flood-risk: 5.0000 / 5.0<br/>STOP & Click 'Check my progress'"]

    Start --> BaseScore --> Explain --> InspectRubric --> EditSkills --> Run2 --> CheckScore
    CheckScore -->|"Yes (5.0000 & 5.0000)"| Pass
    CheckScore -->|"No"| Explain
```
