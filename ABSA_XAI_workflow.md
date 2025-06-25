# End-to-End ABSA Workflow Overview

This document provides a high-level architectural and data-flow overview of the **Tier-1 V2** extraction pipeline and the **Tier-2** escalation & diagnostic system. Together they form a continuous, self-healing workflow that moves from raw dataset rows all the way to human-ready executive summaries and evidence bundles.

---
## 1. Tier-1 V2 – Aspect-Based Sentiment Analysis

**Purpose:** Extract aspect categories (ATE) from raw text reviews, self-check the extraction quality, and repair errors before results are considered *final*.

### 1.1 Core Actors
1. **Loader Factory** – Reads `config.yaml` and instantiates a dataset-specific loader (`HFLoader`, CSV loader, …).  Normalises each record and injects a category taxonomy.
2. **DecisionAgent V2** – Performs zero-shot extraction of `(aspect, opinion)` pairs.
3. **ReviewAgent V2** – Compares Decision output with ground-truth labels using `utils/json_diff.py` and marks mismatches.
4. **ReviseAgent V2** – Fixes only the mismatched predictions, producing `final_revised_aspects`.
5. **Tier-1 Orchestrator** – Async controller that runs **Decision → Review → Revise** while respecting global rate limits.
6. **RateLimitController** – Global semaphore / rolling-window quota checker used by every LLM call.

### 1.2 Data Flow (SQLite)
```
results          ← initial predictions, review verdicts, revisions
run_logs         ← one row per LLM call (token usage, errors)
(batches table)  ← batch-level status: in_progress → revision_complete
```

### 1.3 End-State & Handoff
• When the orchestrator marks a batch `revision_complete`, Tier-1 is finished.
• The orchestrator (or a CLI wrapper) now calls:
```python
run_escalation(db_file, config_file, window_size=N, ...)
```
This triggers **Tier-2** analysis inside the same process and on the **same DB file**.

---
## 2. Tier-2 – Escalation, Diagnosis & Summary

**Purpose:** Detect systemic issues in Tier-1 output, diagnose root causes, propose fixes, and generate executive summaries for human stakeholders.

### 2.1 Core Components
1. **EscalationAnalyzer** (`analyze_and_escalate.py`)
   • Evaluates trigger rules (low F1, high variance, delta drops, etc.).
   • Writes rich evidence bundles to `escalation_cases` (status `pending`).
2. **Tier-2 Orchestrator** (`tier2_orchestrator.py`)
   • Async daemon that pulls `pending` cases, launches EscalationAgent tasks, and shows a live dashboard.
   • Self-heals by resetting stale `in_progress` tasks.a
3. **EscalationAgent**
   • **Triage** → **Plan-Execute-Reflect** loop → **Failure handling**.
   • Logs every cognitive step to `agent_run_steps`.
4. **SummaryAgent**
   • Runs when a case reaches a *terminal* status (e.g. `triaged`, `diagnosis_pending_review`).
   • Aggregates artefacts and writes an executive summary to `case_summaries`.
5. **Wrapper Utilities** (`tier2_wrapper.py`, `evidence_to_csv.py`)
   • Provide one-command execution, evidence export, and CSV conversion for reviewers.

### 2.2 Data Tables (Added by Tier-2 Migrations)
```
escalation_cases  ← trigger reason, evidence, current status
agent_run_steps   ← per-LLM/tool invocation trace
execution_plans   ← multi-step plans created by EscalationAgent
art_proposals     ← gap-analysis proposals for new tools
diagnoses         ← final diagnosis objects
case_summaries    ← executive summaries (interim + final)
```

### 2.3 Typical Case Lifecycle
1. `escalation_cases`: **pending**  → picked up by orchestrator.
2. EscalationAgent runs, writing to `agent_run_steps`, `diagnoses`, etc.
3. Case status updated to `triaged` or `diagnosis_pending_review`.
4. SummaryAgent generates `case_summaries` row.
5. Wrapper copies artefacts to `logs/tier2_run_<ts>/…` for offline audit.

---
## 3. Big-Picture Sequence Diagram (Text)
```
Loader → DecisionAgent → ReviewAgent → ReviseAgent → EscalationAnalyzer
      results & run_logs                      │
                        (same SQLite file)    │
                                               ↓
Tier2Orchestrator → EscalationAgent ↔ agent_run_steps / diagnoses / plans
                                               ↓
                                       SummaryAgent → case_summaries
```

---
## 4. Observability & Resilience Highlights
• **Structured Logging** – All LLM interactions captured (`run_logs`, `agent_run_steps`).
• **Non-destructive Migrations** – `workflow_models.py` auto-creates/updates tables without data loss.
• **Rate-Limit & Retry** – Global controller + automatic back-off for 429/RateLimitError.
• **Self-Healing Orchestrator** – Detects & resets hung tasks.
• **Prompt Hardening** – Full Python source of ART tools injected into prompts; validation aligned with Pydantic schemas.

---
## 5. Running the Entire Workflow
1. **Tier-1 only** (dataset key `restaurant` example)
```bash
uv run tier1_runner.py --dataset_key restaurant --sample_size 200
```
2. **Tier-1 + Tier-2 end-to-end**
```bash
uv run tier1_runner.py --dataset_key restaurant --sample_size 200 \
                      --run_tier2  # orchestrator flag calls run_escalation()
```
3. **Tier-2 standalone** (analyse existing DB, then process)
```bash
# Generate escalation cases
python -m classification_agents.classification_mas.analyze_and_escalate --db path/to/db.sqlite

# Run the Tier-2 orchestrator
uv run tier2_orchestrator.py --db path/to/db.sqlite
```

---
## 6. Detailed Reference – Tier-1 V2

### 6.1 Pipeline Phases
1. **Decision** – Zero-shot extraction of `(aspect, opinion)` pairs.
2. **Review** – Automated QA that diff-checks Decision output against ground-truth and flags mismatches.
3. **Revision** – Targeted repair pass that rewrites only the erroneous predictions.
4. **Validation & Metrics** – Batch marked `revision_complete`; macro-F1 recalculated for analytics.

### 6.2 Key Files
* `tier1_v2/decision_agent_v2.py`
* `tier1_v2/review_agent_v2.py`
* `tier1_v2/revise_agent_v2.py`
* `tier1_v2/orchestrator.py`
* `classification_mas/config.yaml` (dataset definitions)

### 6.3 Schema Touch-Points
* `results` – stores all predictions (`ate_extracted_aspects`, `final_revised_aspects`).
* `run_logs` – one row per LLM call with `error_code`, `error_details`, token counts.
* `batches` – orchestrator state machine (`in_progress`, `review_complete`, `revision_complete`).

### 6.4 Resilience & QoL Features
* Global **RateLimitController** prevents API quota overruns.
* `llm_io.py` adds JSON-block stripping and automatic back-off on 429.
* Idempotent migrations keep legacy DBs working.
* Rich CLI tables (via `rich`) provide live progress & token metrics.

### 6.5 CLI Entry Points
```bash
uv run tier1_runner.py --dataset_key <dataset> --sample_size <N>
python -m classification_agents.classification_mas.decision_agent_v2  # (legacy single-agent mode)
```

### 6.6 ART (Agent Reasoning Tool) Catalogue – Tier-1 V2

| Agent | Key ARTs | Purpose |
|-------|----------|---------|
| **DecisionAgent V2** | `add_aspect`, `check_aspects`, `remove_aspect`, `end_extraction` | Interactive extraction loop: add candidate aspect, optionally prune, finalise list. |
| **ReviewAgent V2** | `single_instance_eval`, `parse_prev_results` | Evaluate extracted aspects against ground-truth; generate actionable feedback. |
| **ReviseAgent V2** | Re-uses `add_aspect`, `check_aspects`, `remove_aspect`, `end_extraction` | Guided repair pass that amends the aspect list based on Review feedback. |

> Note These Tier-1 ARTs are embedded directly in the agent prompts (see `ate_app_prompt.txt`, `review_prompt.txt`, `revision_prompt.txt`). They follow the same declarative pattern as Tier-2 ARTs: the function signature + docstring define the contract, and the LLM "calls" exactly one tool per step within a `tool_calls` JSON block.

### 6.7 Database Tables & Columns of Interest (Tier-1 → Tier-2)

| Table | Columns (non-exhaustive) | Why It Matters for Tier-2 |
|-------|-------------------------|---------------------------|
| `batches` | `batch_id`, `dataset_name`, `status`, `categories_used`, `validation_metrics_json`, `validation_summary_text` | EscalationAnalyzer scans **completed** batches (`revision_complete`) and parses `validation_metrics_json` / `validation_summary_text` to compute trigger rules (low F1, high variance, etc.). |
| `results` | `result_id`, `batch_id`, `review_text`, `ate_extracted_aspects`, `final_revised_aspects`, `actual_aspects` | Source of per-review predictions & ground-truth that Tier-2 uses when building illustrative examples in an evidence package. |
| `run_logs` | `log_id`, `result_id`, `stage_name`, `error_code`, `error_details`, `raw_output_json`, `timestamp` | Enables deep-dive debugging; Tier-2 can surface underlying LLM errors or anomaly patterns when diagnosing systemic failures. |
| *(Tier-2 creates)* `escalation_cases` | (see Section 7) | Populated by EscalationAnalyzer when a trigger fires; serves as the entry-point for the Tier-2 pipeline. |

> Tip If you need to run ad-hoc SQL, the `batches` and `results` tables are your primary window into Tier-1 performance, while `run_logs` helps explain *why* a particular prediction failed.

---
## 7. Detailed Reference – Tier-2 Escalation Pipeline

### 7.1 System Pillars
1. **Escalation Triggering** – Data-driven detection of systemic issues (`analyze_and_escalate.py`).
2. **Concurrent Orchestration** – Async task manager with live dashboard (`tier2_orchestrator.py`).
3. **Cognitive Processing** – EscalationAgent performing triage, planning, execution, reflection.
4. **Post-Processing** – SummaryAgent generating executive summaries and artefact bundles.

### 7.2 EscalationAgent Flow (Happy Path)
```
TRIAGE (single prompt)
   └── solved? → write diagnosis, status=triaged, END
PLAN-EXECUTE-REFLECT LOOP (max_iter)
   ├── ExecutionPlanner_ART → plan
   ├── Execute plan step via ART → reflect
   └── confidence ≥ threshold? → write diagnosis, END
FAILURE HANDLING
   └── GapToART_Specification_ART | RequestHumanInput_ART
```

### 7.3 ART (Agent Reasoning Tool) Catalogue
* `ExecutionPlanner_ART` – creates multi-step plans.
* `GapToART_Specification_ART` – proposes new tools when capability gap detected.
* `RequestHumanInput_ART` – asks for human help with clear action items.

### 7.4 Database Tables & Relations
* `escalation_cases 1─∞ agent_run_steps`
* `escalation_cases 1─1 diagnoses`
* `escalation_cases 1─∞ execution_plans`
* `escalation_cases 1─1 art_proposals`
* `escalation_cases 1─1 case_summaries`

### 7.5 Observability & Debugging Aids
* **Rich Dashboard** – live task view (`tier2_orchestrator.py`).
* **Step Traceability** – every LLM/tool call logged in `agent_run_steps`.
* **Evidence Bundles** – written to `logs/tier2_run_<ts>/evidence/` for offline inspection.
* **Fallback Validation** – `_execute_art_turn` coerces near-miss JSON into valid schema objects.

---
**Last updated:** <!-- KEEP-AUTOMATED-DATE --> 
