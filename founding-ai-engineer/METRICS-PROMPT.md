# Polar Sky — Collaborative Metrics IA Exercise

## Context
Polar Sky analyzes document permissions and recommends remediation actions (for example: remove access, downgrade permissions, tighten sharing links). Reviewers see these recommendations and make decisions, and we log those outcomes.

Today, in `polar-eval` E2E runs, we already capture core ACL-quality metrics:
- **Detection Precision**
- **Detection Recall**
- **Detection F1**

We also capture additional E2E signals:
- **Approval Precision / Recall / F1**
- **Under-Mitigation Rate**
- **Extraction Error Rate**
- **Avg Total Latency (ms)** (plus related request/response size and polling diagnostics)

`polar-eval` also supports scenario/plugin runs for targeted subsystem evaluation (for example, semantic-summary and knowledge-class scorecards). Those scenario metrics are useful, but we do **not** currently roll them up into a unified metrics view with E2E.

Two targeted scenarios we currently assess:
- **`semantic_summary`**: evaluates how well we summarize a document's **purpose** while stripping unnecessary sensitive specifics.
  - Example intuition: for an LOI, we need the summary to make clear that the document is an LOI (purpose), but we do not need to include company names, dollar amounts, or other extra specifics to assess sensitivity.
  - Representative scorecard metrics today include: `avg_candidate_purpose_clarity`, `avg_candidate_abstraction`, `hard_gate_fail_rate`, `candidate_vs_baseline_candidate_superior_rate`, and `candidate_vs_neg_overall_delta`.
- **`knowledge_class`**: evaluates a classifier that assigns a document to the right knowledge class.
  - Representative scorecard metrics today include: `taxonomy_total_classes`, `taxonomy_documents_in_other`, `holdout_assignments_confident`, `holdout_avg_similarity`, and `holdout_similarity_p50/p90`.

Current reporting flow is split: E2E produces an HTML report (`polar_eval_report.html`) and uploads metrics/artifacts to MLflow. In practice, MLflow is not always the easiest interface for day-to-day evaluation review.

## What we want to do together
This is a collaborative working session to figure out the right **metrics information architecture (IA)** for Polar Sky evaluation.

We're trying to figure out:
- which metrics should be top-line health signals
- which metrics should be diagnostic
- which guardrails we need so top-line metrics are not misleading

## Discussion
Assume we are building an evaluation dashboard for weekly model/planner iteration. How would you structure the metrics IA so it helps us answer:
- Are we actually getting better?
- If we regress, where in the system is the regression coming from?
- Which issues should we prioritize first?

## Areas we think are important (examples, not requirements)

### 1) Recommendation quality beyond today’s detection precision / recall / F1
Some possible directions:
- document-level exact match
- cost-weighted error (false positives vs false negatives)
- calibration/confidence quality
- coverage/abstention behavior

### 2) Justification / rationale quality
We generate rationales for recommendations. We likely need metrics for things like:
- faithfulness to evidence
- completeness
- contradiction/hallucination rate
- usefulness/actionability to reviewers

### 3) Specific component-model quality
We also want subsystem visibility, for example metrics on a **sensitivity classifier**:
- core classifier quality (by bucket/class)
- drift or calibration over time
- relationship between classifier performance and end-to-end recommendation outcomes

### 4) Slices and drill-downs
What dimensions should always be sliceable in the dashboard?
Examples:
- action type
- access level
- inheritance depth
- sensitivity bucket
- tenant/integration segment
- model/planner version

## Practical constraints
- Offline evaluation only (no new human labeling during eval runs).
- Use existing logged product signals; propose extra instrumentation only if lightweight and clearly valuable.
- Keep it practical for regular scheduled runs.

## Data signals you can assume exist
- recommendation snapshots (with actions, impacted principals, model/planner versions)
- reviewer decisions per recommendation/principal/action
- assessed permission metadata (including access level + inheritance depth)
- semantic/sensitivity outputs (including sensitivity scores)
- generated recommendation rationales

If you need extra assumptions, call them out explicitly.
