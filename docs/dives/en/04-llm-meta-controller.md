# The LLM as Meta-Controller

> Prerequisite: [Theoretical Foundations](01-theoretical-foundations.md) and [Dual-Channel Mechanism](03-dual-channel-mechanism.md).

## What the LLM Does and Does Not Do

The LLM occupies Layer 2 of the three-layer architecture -- between human-defined objectives (Layer 1) and numerical parameter search (Layer 3). Its role is precisely scoped:

- It **does not** search for ranking parameters. That is Optuna's job.
- It **does not** define business objectives. That is the human's job.
- It **does** adjust the framework within which the search operates, based on patterns it recognizes in historical data.

This is what "Meta-Controller" means: it controls the controller. It does not optimize parameters directly; it optimizes the *conditions under which parameters are optimized*.

### The Coach-Athlete Analogy

The LLM is a coach reviewing game tape. It sees how the team performed across 20 recent games (episodes) and the trajectory of all tactical adjustments (30 calibration updates). Based on patterns -- "our defensive line has been too soft in the last 5 games" -- it adjusts the tactical framework for the next game.

The search engine is the athlete. Given a tactical framework, it uses large-scale trial and evaluation to find the best execution plan for the current opponent (dataset). The athlete doesn't need game-tape analysis; the coach doesn't need to run sprints.

## Two Orthogonal Control Knobs

The LLM's entire output is a JSON proposal containing adjustments along two dimensions:

### Knob 1: `delta_intercept` -- Belief Correction

- Range: [-0.1, +0.1]
- Target: the intercept of the offline-to-online migration function
- Effect: shifts `target_range` -- the position of constraint boundaries in search space

Positive value: "The current mapping is too pessimistic. Online reality is better than we predict. Raise the intercept."

Negative value: "The current mapping is too optimistic. Online reality is worse than we predict. Lower the intercept."

Zero: "Evidence is insufficient or noisy. Don't move."

### Knob 2: `penalty_multiplier` -- Preference Correction

- Range: [0.5, 2.0]
- Target: scales the penalty weight of a specific constraint
- Effect: changes how severely the search penalizes constraint violations

Greater than 1.0: "This constraint has been repeatedly violated. Increase penalty to make the search more cautious."

Less than 1.0: "This constraint is over-tightened, wasting optimization space. Relax it."

Equal to 1.0: "No adjustment needed."

### Why These Two and Nothing Else?

These knobs map directly to the two types of error in SEU theory. The LLM cannot propose values for bottom-layer search parameters directly because those values depend on the current dataset in ways that cannot be determined by reasoning alone -- they require numerical search over the actual data.

Framework parameters, by contrast, encode **cross-round structural knowledge**: "offline predictions of GMV have been consistently too optimistic" or "an order-related constraint has been persistently too soft." This is exactly the type of pattern that LLMs excel at recognizing.

## What the LLM Sees: The Context Structure

The LLM receives a single structured JSON context, assembled from the Memory DB in Pipeline Step 6. It contains:

### Prior Relations Table

The system maintains several offline-to-online migration relationships, for example:

- Offline GMV influence -> online GMV
- Offline order influence -> online orders
- Offline ad influence -> online ad revenue / impressions / clicks
- Offline advertiser-value proxy -> online advertiser value

The LLM sees the current slope/intercept estimates for these relations, plus how those estimates have evolved over time.

### Recent Episodes (up to 20)

Each episode contains a complete offline-online round:
- Offline search results (best parameters, predicted metric values, target ranges, penalty weights)
- Online A/B outcomes (actual uplift for each business metric)

The LLM can compare predicted vs. actual outcomes across many rounds, identifying systematic biases.

### Update History (up to 30 records)

The trajectory of all LMS and LLM corrections to slopes and intercepts. This lets the LLM see whether corrections are converging (system stabilizing) or diverging (system lost).

### Penalty Weight Evolution (up to 30 records)

How each constraint's penalty weight has changed over time, enabling the LLM to reason about whether constraints are being persistently violated or consistently satisfied.

## An Illustrative Proposal, Annotated

Below is an abstracted example proposal:

```json
{
  "proposals": [
    {
      "relation_key": "primary_gmv_relation",
      "delta_intercept": -0.01,
      "penalty_multiplier": 1.10,
      "reason": "Recent online primary-metric outcomes have repeatedly underperformed offline predictions, even after several rounds of tightening, suggesting the current transfer relation remains mildly optimistic.",
      "evidence_episode_keys": ["ep_a", "ep_b", "ep_c", "ep_d"]
    }
  ]
}
```

Reading proposal 1:
- **delta_intercept = -0.01**: A small downward correction. The LLM judges that the mapping is still mildly optimistic, so it nudges the intercept rather than making a dramatic jump.
- **penalty_multiplier = 1.10**: Slightly increase the penalty on the primary guardrail. In addition to the belief correction, the LLM makes the search a bit more cautious.
- **evidence_episode_keys**: Several episodes are cited. This is not optional -- the prompt requires evidence references for every proposal, forcing the LLM to ground its reasoning in concrete observations rather than plausible-sounding but unfounded claims.

## How LLM and LMS Cooperate

Within each round, both mechanisms update the intercept sequentially:

```
Step 5 (LMS):   intercept += eta * error                  // always fires, small step
Step 8 (LLM):   intercept += delta_intercept              // may or may not fire, potentially large step

Final intercept = initial + LMS_correction + LLM_correction
```

Their complementary strengths:

| Scenario | LMS Alone | LLM Alone | LMS + LLM |
|---|---|---|---|
| Gradual drift | Handles well | Overkill; wastes LLM capacity on noise | LMS handles it; LLM stays quiet |
| Abrupt structural shift | Takes ~15 rounds to converge | Jumps in 1-2 rounds | LLM makes the jump; LMS fine-tunes |
| High-variance noise | Overcorrects (adjusts 20% of noise) | Can choose not to move | LMS moves slightly; LLM holds still |
| Converged state | Maintains with tiny adjustments | Recognizes stability, stays quiet | System remains stable |

The key insight: **the LLM is not a "better algorithm" -- it is a pilot who knows when not to touch the controls.**

## Safety Architecture

### Magnitude Bounds

No matter how confident the LLM is, its corrections are clamped:

| Knob | Maximum Single-Round Effect |
|---|---|
| `delta_intercept` | +/- 0.1 (even if the LLM thinks the intercept is off by 0.5) |
| `penalty_multiplier` | 0.5x to 2.0x (penalty can at most double or halve per round) |

Even a completely wrong LLM judgment cannot cause catastrophic drift in a single round. The next round has a chance to correct.

### Empty Proposal Fallback

If the LLM returns an empty proposals array (judging evidence too weak), the pipeline continues with a conservative default:

```json
{"delta_intercept": 0.0, "penalty_multiplier": 1.0, "reason": "bootstrap_fallback"}
```

The system runs without framework adjustment rather than inventing one.

### Freeze After Major Belief Update

When the Belief channel makes a large adjustment, the Preference channel freezes for one round. Without this, the next round's A/B results would reflect both changes simultaneously, making it impossible to attribute the outcome to either one. The freeze preserves signal clarity.

### Human Audit

In attended mode, every LLM proposal is displayed for human review before the search begins. In autonomous mode, all proposals are written to an audit log for post-hoc review. The system never makes an unrecorded decision.

## The Deeper Rationale: What Counts as "Experience"?

The reason the LLM adjusts framework parameters but not bottom-layer parameters comes down to a question about knowledge:

**What kind of observation from round N is still useful in round N+1?**

Bottom-layer parameter optima are **data-dependent one-time facts**. "A certain lower-level multiplier was best on this dataset" -- change the dataset and the optimal value changes. These are not reusable; they are archive entries, not experience.

Framework parameters encode **cross-round structural regularities**. "Offline GMV predictions have been optimistic for several consecutive rounds" -- this tells you something about the environment that will likely persist into the next round. This is experience: knowledge that transfers.

The LLM processes experience. The search engine processes data. Asking the LLM to choose parameter values would be like asking a coach to run the 100-meter dash. Asking the search engine to diagnose structural trends would be like asking a sprinter to design training programs. The separation is not a compromise -- it is a recognition of fundamentally different problem types.
