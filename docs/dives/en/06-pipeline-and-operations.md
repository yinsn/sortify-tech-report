# Pipeline and Operations: From Single Cycle to Continuous Autonomy

> Prerequisite: [Dual-Channel Mechanism](03-dual-channel-mechanism.md), [LLM as Meta-Controller](04-llm-meta-controller.md), [Search Engine](05-search-engine.md).

## The Single Cycle: 10 Steps

Each optimization cycle follows a fixed 10-step pipeline. The logic is: **reflect on what happened -> calibrate the framework -> search for better parameters -> deploy**.

### Overview

```
Phase 1: Online Update (Steps 1-10)
    Reflect on online feedback, calibrate both channels, prepare search framework

Phase 2: Offline Search
    Optuna TPE finds optimal parameters within the calibrated framework

Phase 3: Publish
    Deploy parameters through the online configuration channel, write handoff state for the next cycle
```

Two optional human checkpoints bracket the process:
- **Checkpoint 1** (after Step 8): Review the LLM proposal before committing to search
- **Checkpoint 2** (after search): Review search results before publishing to the online configuration channel

In autonomous mode (`--yes`), both checkpoints are auto-confirmed.

### Step-by-Step

**Steps 1-4: Data Collection**

| Step | Action | Output |
|---|---|---|
| 1 | Pull A/B experiment report for the current time window | Raw metric deltas (GMV, Orders, Ads Revenue, etc.) |
| 2 | Initialize Memory DB if this is the first cycle | Persistent state store |
| 3 | Record offline data: the search parameters and predicted metrics from last cycle | New episode record with offline fields populated |
| 4 | Record online data: parse A/B deltas and attach to the episode | Episode record completed with online outcomes |

**Step 5: LMS Belief Update**

The LMS algorithm reads the latest (offline_prediction, online_actual) pair and updates the slope and intercept of all 6 migration models. This is unconditional -- it fires every round, adjusting 20% of the residual error.

```
For each of the 6 relations:
    error = online_actual - (slope * offline_value + intercept)
    slope     += 0.2 * error * offline_value
    intercept += 0.2 * error
```

**Steps 6-8: LLM Meta-Controller**

| Step | Action | Detail |
|---|---|---|
| 6 | Export LLM context | Assemble JSON from Memory DB: 20 recent episodes, 30 calibration updates, 6 prior relations, penalty weight history |
| 7 | Generate LLM proposal | Call LLM API with structured context; receive JSON with delta_intercept and penalty_multiplier per relation |
| 8 | Apply LLM proposal | Write intercept adjustments to Memory DB; scale penalty weights in next-cycle config |

If the LLM returns an empty proposal (insufficient evidence), the system continues without framework adjustment.

**Steps 9-10: Algorithmic Refinement**

| Step | Action | Detail |
|---|---|---|
| 9 | Derive target ranges | Use updated migration models (slope, intercept) to calculate what offline metric ranges correspond to acceptable online outcomes |
| 10 | Update penalty weights | Apply the Preference channel's asymmetric multiplicative rule based on whether each constraint was satisfied or violated online |

After Step 10, the complete search configuration is ready: target ranges define the feasible region, penalty weights define the violation costs.

### Pipeline Flow Diagram

```
 [1] A/B Data Pull
       |
       v
 [2] Episode -> Memory DB
       |
       v
 [3] LMS: update prior_relations  ----> [4] Assemble LLM Context
                                              |
                                              v
                                         [5] LLM Proposal
                                              |
                                              v
                                         [6] Apply LLM
                                              |
 [7] Many-Trial     <--- [8] Target Ranges + ----+
     Search               Penalty Weights -> Optuna
       |
       v
 [9] Best Params -> Online Config
       |
       v
 [10] Generate New A/B Data -> Next Cycle
```

## Three Operating Modes

The 10-step pipeline is the building block. Three operating modes determine how it is invoked:

### Mode 1: Coldstart (Bootstrap)

**When**: A brand-new experiment with no history -- no Memory DB, no prior results, no handoff state.

**What it does**:
1. Push a conservative initial parameter set through the online configuration channel
2. Anchor the time window origin to the next 15-minute boundary
3. Write an initial state snapshot

**What it does NOT do**: No Memory DB creation, no LLM, no search. It simply gets parameters live so the experiment can start accumulating A/B data.

### Mode 2: Initial (First Full Cycle)

**When**: The experiment is running and has accumulated enough A/B data (typically >= 3.5 hours), but the pipeline has never executed.

**What it does**:
1. Execute the full 10-step pipeline
2. Create and initialize Memory DB with baseline prior relations
3. Establish a handoff snapshot for the next cycle
4. Pause at both checkpoints for human review (unless `--yes` is specified)

This creates the foundation that all subsequent cycles build upon.

### Mode 3: YOLO (Continuous Autonomous Loop)

**When**: The baseline is established. The system enters fully autonomous operation.

**What it does**:
```
while true:
    wait until (time_since_last_push >= 30min AND data_window >= 3.5h)
    align time window to 15-minute boundary
    attempt A/B data pull (retry up to 5 times, 5-min intervals)
    execute full 10-step pipeline (--yes auto-confirm)
    update the state snapshot
    append to the audit log
```

**Design goals**:
- **Unattended**: All confirmations auto-approved
- **Resumable**: State is persisted to a snapshot; a crash and restart picks up from the last completed round
- **Self-healing**: A/B data fetch failures trigger automatic retries; only non-A/B errors cause an immediate exit
- **Auditable**: Every round produces a detailed audit-log entry

### Mode Progression

```
Coldstart -----> Initial -----> YOLO
 (bootstrap)    (first run)    (infinite loop)
   |               |               |
   v               v               v
 Initial publish  Persistent state   Continuous
 + time anchor    + handoff snapshot self-improvement
```

Each mode's output is the next mode's prerequisite. You can also skip ahead if the prerequisites are already met (e.g., jump straight to YOLO if a previous session left valid state files).

## Time Window Management

Each YOLO cycle needs a data window -- a time range over which the A/B report is computed. Sortify manages this automatically:

**15-minute alignment**: All window boundaries snap to 15-minute marks (e.g., 14:00, 14:15, 14:30). This matches the A/B platform's reporting granularity.

**Automatic chaining**: Each cycle's window starts where the previous cycle's window ended. If round N covered 10:00-13:30, round N+1 starts from 13:30 (aligned to the next 15-minute boundary).

**Minimum accumulation**: At least 3.5 hours of data must accumulate before a cycle runs. This is an operational heuristic for improving signal stability; whether it is sufficient still depends on sample size, variance, and effect size.

**Minimum gap**: At least 30 minutes must elapse between online publishes. This prevents the system from deploying changes faster than the A/B platform can measure them.

## State Snapshot

The autonomous loop persists a compact state snapshot across restarts, containing fields such as:

```text
round=<N>
last_push_time=<timestamp>
last_online_from=<timestamp>
last_online_to=<timestamp>
prev_params_snapshot={...}
```

On restart, the loop reads the latest snapshot and resumes from the last completed online window.

## Audit Log: The Flight Recorder

Each autonomous round appends a structured audit entry summarizing:

- the deployed parameter snapshot
- the observed A/B outcome summary
- the LLM proposal summary
- the next parameter snapshot

This log serves as the audit trail for autonomous operation. Operators review it to monitor trends, diagnose issues, and validate that the system's decisions are reasonable.

## Fault Handling

| Failure Type | Response |
|---|---|
| A/B data fetch failure | Retry up to 5 times at 5-minute intervals. If all retries fail, wait for the next cycle. |
| Non-A/B failure (code error, DB corruption) | Immediate exit. Manual restart required after diagnosis. |
| LLM API failure | Fall back to empty proposal (no framework adjustment this round). |
| Optuna crash | Resume from the search backend's checkpoint. |
| Process kill/crash | Restart reads the latest state snapshot and resumes from the last completed round. |

The design philosophy: **A/B data unavailability is transient and worth retrying. Everything else is structural and should not be retried blindly.**
