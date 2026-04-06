---
name: metaopt-experiment-selection
description: "Use when the ml-metaoptimization orchestrator needs to select an experiment from the proposal pool. Ranks proposals by expected impact and selects exactly one winner for the next iteration. Keywords: experiment selection, proposal ranking, synthesis, winner selection, metaoptimization worker."
---

# metaopt-experiment-selection

## Overview

Rank eligible proposals from the current proposal pool and select exactly one winning proposal for the next experiment iteration. This is the critical synthesis step that determines what experiment the campaign runs next.

This skill is a leaf worker operating in the **synthesis** auxiliary lane. It receives a frozen proposal pool from the orchestrator and returns a single winner with supporting rationale. It does not generate new proposals, modify existing ones, or implement code changes.

**Lane:** Auxiliary slot — `synthesis`
**Model class:** `strong_reasoner` (prefer a strong reasoning model like Opus 4.6 fast, fallback to a capable general model like GPT-5.4)

## Input Contract

The orchestrator provides the following via the subagent prompt:

### Standard Envelope (provided by orchestrator on every dispatch)

| Field | Type | Description |
|-------|------|-------------|
| `campaign_id` | string | Campaign identifier |
| `current_iteration` | integer | Current iteration number |
| `slot_id` | string | The slot ID dispatching this worker |
| `attempt` | integer | Attempt number for this dispatch (1-indexed) |

### Campaign Context

| Field | Type | Description |
|-------|------|-------------|
| `goal` | string | The campaign's optimization goal |
| `metric` | string | The target metric name (e.g. `val_loss`, `accuracy`) |
| `direction` | `"minimize"` or `"maximize"` | Whether lower or higher metric values are better |
| `aggregation_method` | string | How per-dataset scores roll up into the aggregate (e.g. `mean`, `weighted_mean`) |
| `aggregation_weights` | object or null | Per-dataset weights when method is `weighted_mean`; `null` otherwise |
| `aggregate_baseline` | number | The current authoritative aggregate campaign score |
| `per_dataset_baselines` | object | Map of dataset IDs to their current numeric baseline values |
| `current_proposals` | array | The frozen pool of candidate proposals eligible for selection |
| `key_learnings` | array | Learnings accumulated from prior iterations |
| `completed_experiments` | array | Prior experiments with their outcomes and deltas |
| `proposal_policy` | object | Policy constraints governing proposal eligibility and selection |

Each proposal in `current_proposals` is a full proposal record containing:

**Orchestrator-owned fields:**
- `proposal_id`: non-empty string, unique within the campaign
- `source_slot_id`: non-empty string identifying the originating slot
- `creation_iteration`: positive integer
- `created_at`: ISO 8601 timestamp

**Worker-provided fields:**
- `title`: concise name (≤ 12 words)
- `rationale`: why this change is expected to improve the metric
- `expected_impact`: object with `direction` and `magnitude`
- `target_area`: what part of the system the proposal modifies

## Output Contract

Return a response containing:

### Required

| Field | Type | Description |
|-------|------|-------------|
| `winning_proposal` | object | The complete, unmodified proposal object selected from the pool |
| `ranking_rationale` | string | Explanation of why this proposal was chosen, referencing baselines, learnings, and campaign goal |

The `ranking_rationale` must include:
- Why this proposal was chosen over alternatives
- What prior evidence or learnings support the choice
- Expected impact assessment relative to the current aggregate baseline

### Optional

| Field | Type | Description |
|-------|------|-------------|
| `ranked_candidates` | array | Ordered ranking of top-N candidates, each with a brief note explaining placement |

Each entry in `ranked_candidates` contains:
- `proposal_id`: the proposal's identifier
- `rank`: integer position (1 = winner)
- `note`: brief explanation of ranking placement

## Behavioral Rules

1. **Exactly one winner.** Must select exactly one proposal — never zero, never multiple.

2. **Justify with evidence.** Selection must reference baselines, key learnings, and the campaign goal. Do not select based on surface-level appeal alone.

3. **Target the largest opportunity.** Prefer proposals that address the largest remaining improvement opportunity relative to the current aggregate baseline and per-dataset baselines.

4. **Avoid repeating failures.** Do not select proposals similar to recently failed experiments unless the rationale explicitly explains why this attempt will differ and why the prior failure does not apply.

5. **No modifications.** Select proposals as-is from the pool. Do not edit, merge, or rewrite any proposal content.

6. **No new proposals.** The proposal pool is frozen at this point. Do not generate, suggest, or append new proposals.

7. **No code changes.** This skill performs ranking and selection only. It does not implement, debug, or design experiments.

8. **Respect direction.** When evaluating expected impact, use `objective.direction` to determine whether improvement means increasing or decreasing the metric.

   > Note: `objective.direction` (from the campaign config) uses `"maximize"` or `"minimize"`. The `expected_impact.direction` field in the proposal schema uses `"improve"`, `"neutral"`, or `"worsen"` — these are different field systems. When evaluating proposals, map `"maximize"` → prefer `expected_impact.direction == "improve"` and vice versa.

9. **Use aggregation rules.** When reasoning about multi-dataset impact, apply `aggregation_method` (and `aggregation_weights` when applicable) rather than ad-hoc comparison.

10. **Consider proposal diversity.** When two proposals have similar expected impact, prefer the one exploring a less-tested hypothesis to maximize information gain.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Selecting multiple proposals or hedging with "either A or B" | Commit to exactly one winner |
| Ignoring `objective.direction` when assessing improvement | Check whether the metric should be minimized or maximized |
| Picking a proposal similar to a recently failed experiment without justification | Explain specifically why the new attempt differs from the prior failure |
| Modifying or merging proposals before returning them | Return the winning proposal object unchanged |
| Generating new proposal ideas during selection | The pool is frozen — select only from what is provided |
| Providing a vague rationale like "this seems promising" | Reference specific baselines, deltas, learnings, or experiment outcomes |
| Ignoring per-dataset baselines | Identify which datasets have the most room for improvement |
| Comparing raw per-dataset scores without applying aggregation rules | Use `aggregation_method` and `aggregation_weights` to evaluate expected aggregate impact |
| Selecting a low-risk incremental proposal when large gaps remain | Prefer proposals targeting the largest remaining improvement opportunity |

## References

- `ml-metaoptimization/references/worker-lanes.md` — authoritative lane contract for the synthesis slot
- `ml-metaoptimization/references/contracts.md` — state file, slot, and proposal pool field definitions
