# Polar Sky — Offline Eval + Human Feedback Loop (Interview Homework)

## High-level problem
Polar Sky analyzes document permissions and produces remediation recommendations -- concrete actions like removing a user's access, downgrading a group from editor to viewer, or restricting an organization-wide sharing link. These recommendations are surfaced to reviewers (typically the file owner or, for shared drives, the organizers) through Teams or Slack notifications. Reviewers see a list of impacted principals: the specific users and groups whose access would change if the recommendation is applied. For each principal, the reviewer makes a binary decision -- apply the recommendation (proceed with the access change) or retain access (override the AI and keep current permissions). Every decision is captured as structured feedback tied to the original recommendation snapshot, enabling us to measure recommendation quality, identify patterns in overrides, and improve future classifications.

Reviewers (typically the file owner/organizer) see the impacted principals and decide whether to **apply the recommendation** or **retain access**.

Design an **offline evaluation system** that uses **real-world human feedback captured during normal product usage** to:
- catch regressions over time, and
- enable offline experimentation (try different models/prompts/planner variants and see whether they improve outcomes).

Assume you can use an AI agent (Claude/Codex/etc.) while working. This is expected.

Time expectation: **60–120 minutes max**.

## Deliverables
### 1) A single Markdown “agent spec”
Write one self-contained Markdown spec that you could give to an agentic coding tool to implement the eval system (we are **not** asking you to implement it). Your spec should be precise enough that an implementation could be built with minimal follow-ups.

Your spec should include:
- **Scope**: goals, non-goals, and explicit tradeoffs.
- **Metrics**: exact metric definitions (formulas) mapped to the data model below.
- **Labeling logic**: how you turn human feedback into evaluation labels (append-only “latest wins” resolution, missing feedback, and how you map decisions back to actions/principals).
- **System architecture**: proposed components, interfaces, and expected artifacts.
- **Reproducibility**: how results are pinned to evaluated versions and cohorts.
- **AI usage notes (as part of your spec)**: which tools you used, what you delegated, and what you reviewed/changed manually.

If you want, add a short section proposing changes you’d make to the runtime (data model, instrumentation, workflow changes) to make offline evaluation more reliable or informative.

### 2) Submission format
Submit:
- `SPEC.md` — your single Markdown “agent spec” (including your AI usage notes section)

## Requirements and constraints
- **Offline eval only**: evaluation runs without any new human labeling at eval time.
- **Use human feedback**: metrics are grounded in the `authz_recommendation_feedback` table below (you may optionally incorporate legacy “sentiment” feedback if you choose).
- **Supports experimentation**: it should be straightforward to run offline comparisons for candidate changes (new model versions, prompt variants, planner changes) against a baseline on a pinned cohort.
- **Reproducible**: every eval result is attributable to:
  - the evaluated system versions (`model_version`, `planner_version`)
  - the eval runner version (you define how)
  - a pinned cohort definition (time window, inclusion/exclusion rules, filters)
- **Practical**: assume this runs on a schedule (e.g., daily) and produces trendable metrics.

### What the eval system should do
Your spec should define an offline eval system that:
1) Builds an evaluation-ready dataset from recommendation snapshots (`authz_recommendation_events`) joined to the latest human decision(s) from `authz_recommendation_feedback`.
2) Computes one or more well-defined metrics over time and by slice (e.g., by `model_version`, `planner_version`, action type, access level, inheritance depth, sensitivity).
3) Produces stable, reviewable artifacts (e.g., JSON + Markdown summary) suitable for regression detection and experiment comparison.

### Metrics
Metrics should be precisely defined and explicitly mapped to the data model.

Define **no more than two primary metrics**. If you include additional metrics, justify why the primary ones are insufficient.

#### Metric failure case
Describe one realistic scenario where your metrics improve while the system’s real-world quality worsens.

Examples (you can define different/better ones):
- **Principal-level acceptance rate**: among principals impacted by recommendations, what fraction are approved (`apply_recommendation`)?
- **False-positive removal proxy**: among `REMOVE_PERMISSION` principals, what fraction are retained (`retain_access`)?
- **Acceptance by slice**: acceptance by `current_access` and `inheritance_depth` (direct vs inherited).

### Feedback-to-label mapping
Your spec should define:
- The “latest wins” resolution query/key.
- What happens when feedback is missing (no decision recorded).
- How you map feedback back to the underlying snapshot (via `recommendation_id`, `principal_id`, `selected_action_id`, and/or `impacted_principals`).
- How you handle disagreements between multiple reviewers (if applicable) and whether you weight/choose a canonical reviewer.

### Reproducibility/versioning
Your spec should define:
- How eval output is keyed to `model_version`, `planner_version`, and an eval runner version.
- How the cohort is pinned (time window + inclusion/exclusion rules).
- How you store enough raw inputs/aggregations to debug “why did this metric change?” later.

## Context: data model (self-contained)
All IDs are UUIDs unless noted otherwise. PostgreSQL is used for persistence.

### Data model diagram (ASCII, simplified)
```
document_permission_assessments
  id, assessed_at, workflow_version, integration_id, semantic_envelope_id, source_file_id
            |
            v
assessed_permissions
  assessment_id, principal, classification, permission_level, inheritance_depth, reason, created_at

semantic_envelopes
  id, created_at, summary_text, summary_topics, sensitivity_score, sensitivity_reasons
            ^
            |
authz_recommendation_events
  id, tenant_id, integration_id, model_version, planner_version, created_at,
  document_permission_assessment_id, semantic_envelope_id,
  trusted_principals, recommended_actions, impacted_principals
            |
            v
authz_recommendation_feedback
  id, tenant_id, integration_id, recommendation_id, responder_user_id,
  principal_id, principal_type, decision, selected_action_id, grant_paths, comment, created_at
```

### System overview
At a high level, the system behaves like:
1) Build a snapshot of effective access to a document (who can access it and why).
   - Persisted on the recommendation snapshot as `authz_recommendation_events.access_graph_digest`.
   - The assessment itself is anchored by `document_permission_assessments` (time + workflow).
2) Decide who *should* have access (“trusted principals”), using model-driven reasoning.
   - Persisted on the recommendation snapshot as `authz_recommendation_events.trusted_principals`.
   - Related per-principal classifications can be inspected via `assessed_permissions` (allowed vs overprovisioned, inheritance depth, reasons).
3) Generate recommended remediation actions to align actual access with the desired access.
   - Persisted as `authz_recommendation_events.recommended_actions` (with stable `action_id` values).
4) Present the impacted principals/actions to the document owner/organizer for review.
   - Persisted in UI-ready form as `authz_recommendation_events.impacted_principals` (also keyed by `action_id`).
5) Record what the reviewer chose (apply vs retain) as human feedback.
   - Persisted as append-only rows in `authz_recommendation_feedback` linked to the snapshot via `recommendation_id` (and optionally `selected_action_id`).

The eval system you design should use the persisted snapshots + human decisions to measure how often recommendations match human judgment, and to compare outcomes across versions/slices and candidate changes.

### Tables and key fields (minimal)
```
+------------------------------+            +---------------------+
| document_permission_         |            | semantic_envelopes  |
| assessments                  |            |---------------------|
|------------------------------|            | id                  |
| id                           |            | created_at          |
| assessed_at                  |            | summary_text        |
| workflow_version             |            | summary_topics      |
| integration_id               |            | sensitivity_score   |
| semantic_envelope_id         |<-----------| sensitivity_reasons |
| source_file_id               |            +---------------------+
+------------------------------+
              |
              | assessment_id
              v
+------------------------------+
| assessed_permissions         |
|------------------------------|
| assessment_id                |
| principal                    |
| classification               |
| permission_level             |
| inheritance_depth            |
| reason                       |
| created_at                   |
+------------------------------+

+------------------------------+
| authz_recommendation_events  |
|------------------------------|
| id                           |
| tenant_id                    |
| integration_id               |
| model_version                |
| planner_version              |
| created_at                   |
| document_permission_         |
|   _assessment_id             |
| semantic_envelope_id         |
| trusted_principals           |
| recommended_actions          |
| impacted_principals          |
+------------------------------+
              |
              | recommendation_id
              v
+------------------------------+
| authz_recommendation_        |
| feedback                     |
|------------------------------|
| id                           |
| tenant_id                    |
| integration_id               |
| recommendation_id            |
| responder_user_id            |
| principal_id                 |
| principal_type               |
| decision                     |
| selected_action_id           |
| grant_paths                  |
| comment                      |
| created_at                   |
+------------------------------+
```

You can assume the following logical tables exist, with the listed fields being the ones that matter for this exercise.

#### `authz_recommendation_events` (immutable recommendation snapshot)
One row per recommendation snapshot emitted by the system (“what the system recommended at time T”).
- Keys and partitioning: `id`, `tenant_id`, `integration_id`
- Versioning: `model_version`, `planner_version`, `created_at`
- Join anchors: `document_permission_assessment_id`, `semantic_envelope_id`
- Snapshot payloads (JSON): `trusted_principals`, `recommended_actions`, `impacted_principals`

#### `authz_recommendation_feedback` (append-only human decisions)
One row per reviewer decision about one principal within one recommendation snapshot. Append-only; reviewers can change their mind.
- Keys and join: `id`, `tenant_id`, `integration_id`, `recommendation_id` → `authz_recommendation_events.id`
- Who decided: `responder_user_id`
- What the decision is about: `principal_id` (string), `principal_type` (`user` | `group`)
- The decision: `decision` (`apply_recommendation` | `retain_access`), `selected_action_id` (UUID; set when applying)
- Optional context: `grant_paths`, `comment`
- Timing: `created_at`

Important semantics:
- “Latest wins” resolution is part of the problem: define exactly how you collapse multiple feedback rows into labels (the grouping key + tie-breaking).
- Feedback can be missing (no decision recorded); define how metrics treat unlabeled rows.

#### `document_permission_assessments` (assessment anchor)
An operational record that an assessment occurred (useful for cohort windows and joins).
- Keys and timing: `id`, `assessed_at`, `workflow_version`
- Joins: `integration_id`, `semantic_envelope_id`, `source_file_id` (document id)

#### `assessed_permissions` (per-principal classification output)
Per-principal classification output for an assessment (debugging and slicing).
- Join: `assessment_id` → `document_permission_assessments.id`
- Classification: `classification` (`overprovisioned` | `allowed`)
- Permission level: `permission_level` (e.g., `read` | `write` | `owner`)
- Optional ids: `external_user_id`, `external_group_id`, `display_name`
- Provenance: `inheritance_depth` (0=direct, >0=inherited), `reason`, `created_at`

#### `semantic_envelopes` (document semantic snapshot)
Document summary/sensitivity snapshot for slicing.
- Keys: `id`, `created_at`
- Fields: `summary_text`, `summary_topics`, `sensitivity_score` (typically 1–10), `sensitivity_reasons`

### Snapshot payloads (high-level; for metric mapping)
The recommendation snapshot includes JSON payloads. You can assume:
- `recommended_actions` is a list of actions, each with:
  - `action_id` (UUID), `action_type` (e.g., remove/downgrade/tighten link/move), and `affected_principals` (principal `{type,id,display_name}` plus access + provenance)
- `impacted_principals` is a flattened, UI-ready list, each with:
  - principal `{type,id,display_name}`, current/proposed access, `recommended_outcome`, and `action_id`

If anything is ambiguous, state your assumptions explicitly and proceed.
