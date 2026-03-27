# Memory Architecture: Why Persistence Is the Prerequisite

> Prerequisite: [Dual-Channel Mechanism](03-dual-channel-mechanism.md), [Pipeline and Operations](06-pipeline-and-operations.md).

## The Groundhog Day Problem

Without persistent memory, every optimization cycle starts from scratch. The LMS corrections learned in round 5 are lost by round 6. The LLM receives a single episode of context instead of 20. The system cannot distinguish "this is the first time GMV predictions were too optimistic" from "this is the fifth consecutive round of optimistic bias."

This is not a marginal issue -- it is the difference between a learning system and a stateless loop. A system that discards its history after each cycle is condemned to repeat the same mistakes, like the protagonist of *Groundhog Day* who wakes up each morning with no memory of yesterday.

Sortify's Memory DB is not an optimization: it is the architectural foundation that makes dual-channel calibration, LLM meta-control, and cross-round learning possible at all.

## The Logical Schema

The Memory DB can be described as seven logical entities. In practice, these could be implemented as relational tables or another persistent storage format; what matters here is their role in the learning loop.

### 1. `episodes`

The central record. Each row is a complete offline-online cycle:

| Field | Content |
|---|---|
| episode_key | Stable cycle / experiment identifier |
| offline_params | The deployed parameter vector |
| offline_metrics | Predicted Influence values from search |
| offline_target_ranges | Constraint boundaries used during search |
| offline_penalty_weights | Penalty weights used during search |
| online_uplifts | Actual A/B metric deltas |
| status | pending / completed / failed |
| timestamps | Creation and completion times |

This table is the single source of truth for "what we predicted vs. what happened." Every other table derives its updates from episode data.

### 2. `prior_relations`

Current state of the 6 offline-to-online migration models:

| Field | Content |
|---|---|
| relation_key | e.g., "GMV~I_gmv" |
| slope | Current migration slope |
| intercept | Current migration intercept |
| eta | LMS learning rate (0.2) |
| metadata | Human-readable description |

This is the Belief channel's persistent state. Across rounds, the slope and intercept converge toward the true offline-to-online relationship.

### 3. `prior_update_history`

Rolling window (30 most recent entries) of all slope/intercept changes:

| Field | Content |
|---|---|
| relation_key | Which model was updated |
| old_slope, new_slope | Before and after |
| old_intercept, new_intercept | Before and after |
| update_method | "lms" or "llm" |
| episode_key | Which episode triggered the update |

This table serves two purposes:
- **LLM context**: The LLM sees 30 recent updates to understand whether corrections are converging or diverging
- **Audit trail**: Every model change is attributable to a specific episode and method

### 4. `penalty_weights`

Current penalty weight for each constraint:

| Field | Content |
|---|---|
| constraint_key | e.g., "I_order" |
| weight | Current lambda value |

### 5. `penalty_weight_update_history`

Rolling window (30 entries) of penalty weight changes:

| Field | Content |
|---|---|
| constraint_key | Which constraint |
| old_weight, new_weight | Before and after |
| trigger | "violated" or "satisfied" |
| episode_key | Triggering episode |

The Preference channel's audit trail. Shows how constraint strictness has evolved.

### 6. `llm_proposals`

Every LLM output, whether applied or not:

| Field | Content |
|---|---|
| proposal_id | Unique identifier |
| proposals_json | Full JSON output from the LLM |
| applied | Whether the proposal was actually used |
| episode_key | The episode that prompted the proposal |

### 7. `evidence_links`

Traceability between proposals and the episodes they cite:

| Field | Content |
|---|---|
| proposal_id | Which proposal |
| evidence_episode_key | Which episode was cited as evidence |

This table makes it possible to answer: "When the LLM decided to lower the GMV intercept in round 7, which specific episodes did it base that decision on?"

## How Tables Interact During a Cycle

```
A/B results arrive
      |
      v
episodes: new row created (offline + online data)
      |
      +---> prior_relations: slope/intercept updated by LMS
      |           |
      |           v
      |     prior_update_history: LMS update logged
      |
      +---> LLM reads: episodes (20), prior_update_history (30),
      |                 penalty_weight_update_history (30), prior_relations (6)
      |           |
      |           v
      |     llm_proposals: LLM output stored
      |     evidence_links: cited episodes linked
      |           |
      |           v
      |     prior_relations: intercept updated by LLM
      |     prior_update_history: LLM update logged
      |
      +---> penalty_weights: updated by multiplicative rule
                |
                v
          penalty_weight_update_history: update logged
```

Every modification is logged. No state change is silent.

## Why 30-Record Windows?

The update history tables cap at 30 entries. This is a deliberate design choice:

- **Recency**: Older corrections reflect environmental conditions that may no longer hold. A correction made 30 rounds ago (5 days) is likely stale.
- **LLM context size**: 30 records provide enough trajectory to identify trends (convergence, oscillation, drift) without overwhelming the LLM's reasoning capacity.
- **Storage**: Bounded tables prevent unbounded growth in long-running deployments.

The episodes table is unbounded -- all historical records are preserved. But the LLM only receives the 20 most recent episodes as context, balancing completeness with relevance.

## What Memory Enables

### 1. Cross-Round Learning

Without memory: Each round's LMS correction is lost. The system oscillates around the true intercept forever.

With memory: LMS corrections accumulate. After 7 rounds, the intercept has converged close to the true value. The correction magnitude naturally decreases as the model improves.

### 2. Trend Detection

Without memory: The LLM sees one episode. It cannot distinguish "GMV was negative this one time" from "GMV has been negative for 5 consecutive rounds."

With memory: The LLM sees 20 episodes and recognizes structural patterns: "4 of the last 6 rounds show optimistic GMV bias -- this is not noise, it's a systematic mapping error."

### 3. Convergence Monitoring

Without memory: No way to know if the system is improving or degrading over time.

With memory: The update history shows that LLM corrections decreased from 5 items in round 2 to 2 items in round 7, with decreasing magnitude. This is direct evidence of system-level learning -- the framework is stabilizing.

### 4. Hot Start / Cold Start Transfer

Without memory: A new market deployment starts from scratch, with no knowledge of how offline metrics relate to online outcomes.

With memory: A Memory DB from a similar market can seed the new deployment's prior relations, giving it a "warm start" that avoids the worst early-round mistakes.

### 5. Post-Hoc Auditing

Every decision the system made -- every LMS update, every LLM proposal, every penalty adjustment -- is recorded with its inputs and the evidence that drove it. When an online metric moves unexpectedly, operators can trace the causal chain backward:

```
Online GMV dropped
  -> Round 6 used a certain GMV-related multiplier setting
    -> Search used a corresponding guardrail boundary
      -> That boundary was derived from a prior intercept calibration
        -> The intercept was last adjusted by LLM proposal P-005
          -> The LLM cited episodes E-003, E-004, E-005
            -> Those episodes showed [specific offline-online pairs]
```

In practice, this kind of traceability is one of the most useful tools for understanding system behavior during debugging.

## The "Accumulated Judgment" Perspective

A common misconception is that the Memory DB stores "optimization history" -- a log of past parameters and results. It does store those, but its primary value is different.

The Memory DB stores **calibrated world models** (migration functions that have been refined by dozens of rounds of evidence), **validated trade-off decisions** (penalty weights that have been tested against real online outcomes), and **evidence-linked reasoning** (LLM proposals with explicit citations).

Each round, the system doesn't just inherit parameters from the previous round -- it inherits judgment. Round 10's search operates in a constraint landscape that has been shaped by the accumulated wisdom of rounds 1-9. The Memory DB is not a filing cabinet; it is the institutional memory of a learning organization.

## Implementation Notes

The implementation needs four properties: durable storage, single-writer consistency during each cycle, schema evolution over time, and recoverable snapshots at cycle boundaries. The exact storage technology is an implementation choice rather than a conceptual requirement.
