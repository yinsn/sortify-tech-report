# The Search Engine: Constrained Optimization with Optuna TPE

> Prerequisite: [Influence Share](02-influence-share.md) defines the metrics that the search optimizes.

## What the Search Engine Does

The search engine sits at Layer 3 of Sortify's architecture. Given a fixed offline dataset and a calibrated constraint framework (target ranges and penalty weights from Layer 2), it finds the parameter vector that maximizes the primary Influence objective while respecting all business constraints.

```
Input:  offline dataset + target_ranges + penalty_weights
Output: best parameter vector (several values)
Method: thousands of trials, parallel workers, Optuna TPE
```

The search is purely numerical. It does not reason about offline-online gaps, does not learn across rounds, and does not need to. Its job is to find the best answer to a well-posed optimization problem.

## The Objective Function

Each trial evaluates a candidate parameter vector by computing a single scalar score:

```
objective = positive_score - penalty_sum
```

### Positive Score: What We Want to Maximize

```
positive_score = 10.0 * uplift(I_gmv)
```

where the relative uplift is:

```
uplift(I_gmv) = (I_gmv_new - I_gmv_base) / |I_gmv_base|
```

The baseline `I_gmv_base` is computed with all parameters set to 1.0 (no adjustment). Using relative uplift rather than absolute values ensures comparability across metrics with different scales.

### Penalty Sum: What We Cannot Break

Each constraint applies a **quadratic penalty** when violated. In this public design write-up, the exact production thresholds and weights are intentionally abstracted away; the important idea is that every guardrail defines:

- an allowed range, describing how much movement is acceptable
- a penalty weight, describing how expensive a violation should be
- a business rationale, describing what the guardrail is protecting

The penalty formula:

```
if uplift is within allowed range:
    penalty = 0

if uplift exceeds range:
    violation = distance beyond the boundary
    penalty = weight * violation^2
```

### Why Quadratic Penalties?

Two properties make quadratic penalties superior to linear ones:

1. **Graceful near the boundary**: A 0.1% violation costs `weight * 0.001^2` = almost nothing. A 10% violation costs `weight * 0.1^2` = 10,000x more. This encourages solutions that are "slightly over multiple constraints" rather than "massively over one constraint" -- the former is usually recoverable, the latter is catastrophic.

2. **Continuous objective shaping**: Compared with a hard cutoff, the quadratic penalty increases continuously with violation distance. That gives the sampler richer score differences between "near the boundary" and "far outside it."

### A Concrete Example

Baseline: I_gmv_base = 0.40, I_order_base = 0.35

**Good trial** (parameters A):
```
I_gmv = 0.44 -> uplift = +10% -> positive_score = 10 * 0.10 = 1.0
I_order = 0.348 -> uplift = -0.57% -> within [-1%, +100%] -> penalty = 0
Objective = 1.0
```

**Bad trial** (parameters B):
```
I_gmv = 0.46 -> uplift = +15% -> positive_score = 10 * 0.15 = 1.5
I_order = 0.30 -> uplift = -14.3% -> exceeds -1% by 13.3%
penalty = 10000 * 0.133^2 = 176.89
Objective = 1.5 - 176.89 = -175.39
```

Parameters B produce a higher GMV uplift (+15% vs +10%), but at the cost of a 14.3% collapse in order influence. A sufficiently large quadratic penalty annihilates the 1.5-point positive score. The search engine correctly eliminates this parameter combination.

### Why Upper Bounds Are Generous

Most constraints have upper bounds set to +100% -- effectively uncapped. This is because *increasing* influence for orders or ad revenue is generally welcome. The business risk is in the *decrease* direction. The tight lower bounds (-1%, -2%, -3%) protect against regression while leaving room for serendipitous improvement.

## Optuna TPE: The Sampling Strategy

### Why Not Grid Search?

Even a modest grid over multiple parameters explodes combinatorially. Each evaluation recomputes all item scores and Influence metrics across the full dataset, so exhaustive search is usually infeasible.

Random search is feasible but wasteful: late trials are as "blind" as early ones, learning nothing from history.

### Tree-structured Parzen Estimator (TPE)

TPE is Optuna's default Bayesian optimization algorithm:

1. **Exploration phase** (first N trials): Sample broadly across parameter space, similar to random search.

2. **Split**: Rank all completed trials by objective score. The top gamma fraction (default gamma=0.1, i.e., top 10%) become the "good" group; the rest become the "bad" group.

3. **Model**: Fit probability densities via kernel density estimation:
   - l(x) = P(x | good) -- where good parameters tend to be
   - g(x) = P(x | bad) -- where bad parameters tend to be

4. **Sample**: The next trial point is chosen to maximize l(x) / g(x) -- parameters that are common among good trials and rare among bad ones.

5. **Iterate**: Each completed trial updates the density models.

The behavioral trajectory is: **broad exploration -> identify promising regions -> progressive focus -> fine-grained search within promising regions**. In moderate-dimensional problems, a few thousand trials often produce strong candidate solutions.

### Parallel Execution: Constant Liar

With multiple parallel workers, a naive approach would have workers blindly duplicating each other's exploration. The **Constant Liar** strategy solves this:

When Worker A starts evaluating parameter x_A (result pending), Worker B treats x_A as if it had returned the median score of all completed trials. This "lie" nudges Worker B's density model to avoid x_A's neighborhood, naturally spreading exploration across the parameter space.

Workers coordinate through a shared study backend. The liar values update as real results come in, keeping the density models increasingly accurate.

### Configuration

Trial budget, degree of parallelism, seeding, and the study backend are implementation choices rather than conceptual requirements. The design only needs three properties: enough search budget to explore meaningfully, enough parallelism to keep latency reasonable, and resumable storage so long searches can recover from interruption.

## Derived Metrics: Beyond Influence

The search engine computes additional metrics for monitoring, even when they are not directly optimized:

| Metric | Computation | Purpose |
|---|---|---|
| `ads_top10_rate` | Weighted fraction of ad items in top-10 positions (exponential decay by rank) | Monitor ad visibility in prominent positions |
| `ratio_ecpm` | Mean ratio of ecpm_term to total score | Track ad bid dominance in ranking |
| `TopKOverlap` | Fraction of top-10 items unchanged after parameter adjustment | Measure ranking stability |
| `CR_order_gmv` | Fraction of pairs where order and GMV point in opposite directions | Quantify factor tension -- higher conflict means more room for exchange |

## How Calibration Affects Search

The search engine does not know or care about the offline-online gap. It optimizes within whatever constraint framework it receives. The magic is that Layer 2 continuously calibrates this framework:

**Without calibration** (round 1): Target ranges are based on initial guesses. The search finds "optimal" parameters, but the target ranges are wrong -- they assume an offline-online relationship that doesn't hold. The deployed parameters underperform.

**With calibration** (round 7): After 6 rounds of LMS corrections and LLM jumps, the target ranges are much closer to the real offline-online relationship than the initial guess. The search engine operates in a better-calibrated space, so its "optimal" solution has a better chance of performing well online.

The search engine's output quality is only as good as the framework it's given. Sortify's dual-channel calibration is meant to move that framework progressively closer to reality over successive rounds.

## Example Outcome Pattern

A typical successful run looks like this qualitatively:

- the primary objective improves materially
- hard guardrails end up near, but not far beyond, their boundaries
- ad-visibility and revenue-related guardrails remain stable
- the resulting parameter vector reallocates decision power rather than simply maximizing one signal at all costs
