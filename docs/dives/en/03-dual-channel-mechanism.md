# The Dual-Channel Mechanism: How Belief and Preference Calibration Works

> Prerequisite: [Theoretical Foundations](01-theoretical-foundations.md) explains *why* the channels must be separated. This document explains *how* they work.

## The Fundamental Problem: An Irremediable Gap

Parameter search is an offline process: fix a dataset of historical logs, sample thousands of parameter combinations, evaluate each one's Influence distribution, pick the best. This assumes the offline data represents online reality.

**That assumption is never fully true.** Offline data is a historical snapshot. Online traffic is alive -- user behavior shifts, competitors change, seasonal patterns evolve. Parameters that show I_gmv uplift of +12% offline might yield only +3% online, or even go negative.

If search is "planning a route on a map," the offline-online gap is "the map doesn't quite match the terrain." Traditional practice: search, deploy, check results, manually adjust, repeat. The entire feedback loop depends on a scarce resource -- an engineer who can read A/B data, understand the gap, and make correct adjustments. Such engineers handle maybe two or three rounds per day, and their judgment stays in their heads, never becoming system state.

Sortify automates this feedback loop through two parallel correction channels that operate on orthogonal dimensions.

## Belief Channel: Correcting "How Much Can We Trust Offline Predictions?"

### The Linear Migration Model

For each offline-online metric pair, the system maintains a linear mapping:

```
online_uplift ~ slope * offline_uplift + intercept
```

Example: the `GMV~I_gmv` relation has slope=0.6, intercept=-0.05. Meaning: "For every 1% I_gmv uplift offline, expect ~0.6% GMV uplift online, minus a 5% systematic discount."

After each round, the system collects the actual online result and updates this mapping.

### Why Only Adjust Intercept, Not Slope?

A key design decision: slope changes slowly (via LMS regression), but intercept is the primary adjustment target.

The reasoning: the **marginal relationship** between offline and online metrics is relatively stable in the short term. If 1% offline I_gmv maps to 0.6% online GMV today, then 2% offline probably maps to ~1.2% online -- the proportionality holds. But the **absolute level** drifts constantly due to external factors (promotions, seasonality, competition).

This is analogous to a thermometer: if it reads 2 degrees too high, every reading is biased by the same constant. The relationship between actual temperature changes is preserved -- the scale factor is correct, but the zero point is off. Correcting the intercept (zero point) fixes the bias without disturbing the scale factor.

In control theory, this is **System Identification** -- refining the model of the plant. In reinforcement learning, it parallels **Sim-to-Real Transfer**.

### LMS: The Steady Hand

The LMS (Least Mean Squares) algorithm updates slope and intercept each round:

```
error = observed_online - predicted_online
slope  += eta * error * offline_value
intercept += eta * error
```

with learning rate eta=0.2. This means each round corrects only 20% of the residual error.

Properties:
- **Unconditional**: updates every round, whether the observation is signal or noise
- **Fixed step**: no distinction between a large structural shift and a small random fluctuation
- **Convergent but slow**: if the intercept needs to move by 0.06, it takes approximately 15 rounds (over two days) to converge to 95%

### LLM: The Selective Jump

When the underlying distribution shifts abruptly (e.g., a promotion ends, a competitor launches), LMS is painfully slow. The LLM Meta-Controller can make selective intercept jumps of up to +/- 0.1 in a single round, compressing what would take 10+ rounds into 1-2.

Critically, the LLM can also choose to do nothing. When data is noisy or evidence is ambiguous, it returns `delta_intercept = 0` with an explicit reason: "outcomes are high-variance around the mapped value; belief shift is not robust." In high-variance scenarios, inaction is often the optimal strategy.

The two mechanisms form a **dual-rate system**:

| | LMS | LLM |
|---|---|---|
| Step size | eta * error (small, ~0.01) | delta_intercept (large, up to +/- 0.1) |
| Frequency | Every round, unconditional | Every round, but may choose zero |
| Input | Current 1 observation pair | Last 20 episodes + 30 update records |
| Capability | Gradient descent, no filtering | Pattern recognition, selective intervention |
| Analogy | Cruise control | Pilot override |

Their effects are additive within each round: `final_intercept = initial + LMS_correction + LLM_correction`.

## Preference Channel: Correcting "How Tightly Should We Enforce Constraints?"

### Penalty Weights as Lagrange Multipliers

Each constraint has a penalty weight lambda (analogous to a Lagrange multiplier). For example, if a constraint's lambda is on the order of 10^4, then exceeding the allowed range by 1% costs on the order of `lambda * 0.01^2` objective points.

### Multiplicative Updates

The system updates lambda multiplicatively:

```
lambda_new = lambda_old * exp(step)
```

- Constraint violated online: step > 0 -> lambda increases -> next search penalizes violations more heavily
- Constraint safe and main objective met: step < 0 -> lambda slowly decays -> search gets more exploration room

### Why Multiplicative, Not Additive?

Different constraints have lambda values spanning orders of magnitude (1,000 to 80,000). Additive updates (`lambda += alpha * violation`) would overcorrect small lambdas and undercorrect large ones. Multiplicative updates are **scale-invariant**: the same pressure produces the same proportional adjustment whether lambda is currently 1,000 or 80,000.

In log-space, multiplicative updates become additive steps, which is mathematically equivalent to classical Lagrangian dual ascent performed in the natural geometry of the multiplier space.

### Built-In Loss Aversion

One typical configuration makes tightening steps meaningfully larger than loosening steps:

```
delta_up   = +0.25  (tightening, when constraint is violated)
delta_down = -0.08  (loosening, when constraint is satisfied)
```

Tightening steps are intentionally larger than loosening steps. This encodes a fundamental business asymmetry: undershooting a metric (e.g., orders drop below target) damages business trust far more than overshooting a constraint (e.g., being slightly more conservative than necessary). The system is deliberately loss-averse.

## Orthogonality: Why This Decomposition Works

### Geometric Independence

From the search engine's perspective:

- **Belief** adjusts `target_range` -- the **position** of constraint boundaries. It shifts the optimization landscape's feasible region.
- **Preference** adjusts `penalty_weight` -- the **hardness** of constraint boundaries. It changes the cost of being outside the feasible region without moving the region itself.

One acts on the objective function, the other on the constraint surface. In the geometry of the optimization problem, these are naturally orthogonal -- adjusting one does not interfere with the other's correction.

### Complementary Update Rhythms

| | Belief | Preference |
|---|---|---|
| Update speed | Slow (eta=0.2, small steps) | Fast (pressure-driven, immediate violation response) |
| Persistence | Written to Memory DB, permanently accumulated | Written to config, overridable each round |
| Conflict prevention | No freeze | Frozen for 1 round after major Belief update |
| Character | Learning (cognitive sediment) | Reflex (stress response) |

The freeze mechanism deserves explanation: if both channels make large adjustments simultaneously, the next round's results cannot distinguish which adjustment caused which effect. By freezing Preference for one round after a major Belief update, the system preserves **attributability** -- you can trace each outcome change to a specific cause.

## The Correction Decision Tree

When online results disappoint, the system follows this diagnostic logic:

```
Online metric underperforms
    |
    +-- Was the offline prediction accurate?
    |       |
    |       +-- NO  --> Belief error: adjust intercept (don't touch penalty)
    |       |
    |       +-- YES --> Was the penalty weight sufficient?
    |                       |
    |                       +-- NO  --> Preference error: increase penalty (don't touch intercept)
    |                       |
    |                       +-- YES --> Constraint spec error: escalate to Layer 1 (human)
```

This tree is not just a debugging tool -- it is the actual logic the system executes. Every round, the Belief channel corrects mapping errors, and the Preference channel corrects penalty errors. They never cross-contaminate.

## In Control Theory Terms

The complete picture maps cleanly onto established control theory concepts:

| Sortify Component | Control Theory Equivalent |
|---|---|
| Belief channel (intercept calibration) | System Identification / Model Adaptation |
| Preference channel (penalty scaling) | Gain Scheduling / Dynamic Controller Tuning |
| LMS + LLM dual-rate | Dual-rate adaptive control |
| Freeze after major update | Excitation/observation separation |
| Memory DB persistence | Recursive state estimation |
| Three-layer architecture | Hierarchical control with separation of concerns |
