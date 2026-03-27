# Theoretical Foundations: Why Belief and Preference Must Be Separated

## The Central Axiom

Sortify's architecture is strongly informed by L.J. Savage's **Subjective Expected Utility (SEU) theory** (1954):

> In Savage's representation framework, if a decision-maker's choices satisfy a set of consistency axioms, those choices can be represented by two components: a subjective probability over world states (belief) and a utility over outcomes (preference). In engineering terms, this is a useful distinction between "what is likely to happen" and "how much we care about the result."

In formula form, the optimal action is:

```
a* = argmin_a  SUM_theta  P(theta | D)  x  L(a, theta)
                           -----------     -----------
                             belief         preference
```

This is not merely an engineering taste. The practical lesson from SEU here is: **if a system compresses world-model judgments and value judgments into the same adjustment knob, it becomes much harder to interpret, diagnose, and calibrate.**

## What Happens When You Violate This

In practice, ranking optimization systems routinely violate this separation. When online GMV drops, the typical response is to "tighten the constraint" -- a single, blunt adjustment that mixes two fundamentally different questions:

1. **Was our prediction wrong?** (The offline model said GMV would rise, but it didn't)
2. **Was our tolerance wrong?** (GMV dropped by an acceptable amount, but it turns out even that amount was unacceptable)

These questions demand opposite corrections, yet produce identical symptoms. When they are handled by a single mechanism, two cognitive biases emerge:

- **Wishful Thinking**: The system wants a target to be met so badly that it distorts its belief about how offline metrics translate to online outcomes. It inflates the mapping to make predictions look better than they are.
- **Sour Grapes**: The system's predictions keep failing, so it reduces the importance of the constraint -- "I can't hit this target, so it must not matter that much."

Both biases are self-reinforcing feedback loops. Once a system starts confusing "what I think will happen" with "what I want to happen," there is no internal mechanism to detect the corruption.

## Sortify's Operationalization

Sortify enforces the SEU separation at the architecture level:

| | Belief Channel | Preference Channel |
|---|---|---|
| **Savage mapping** | P(theta\|D) -- probability over world states | L(a, theta) -- loss function over outcomes |
| **System question** | "How do offline metrics translate to online reality?" | "How much does it hurt when a constraint is violated?" |
| **What it controls** | `target_range` -- the position of constraint boundaries | `penalty_weight` -- the cost of crossing those boundaries |
| **Error it corrects** | Epistemic error: the world model is wrong | Axiological error: the value function is miscalibrated |

The channels are not merely conceptually separate -- they operate on geometrically orthogonal dimensions of the optimization landscape. Belief moves constraint edges horizontally (changing *where* the boundary is). Preference scales violation costs vertically (changing *how much* crossing the boundary hurts). A correction on one axis has zero effect on the other.

## Two Types of Error, Two Opposite Corrections

The deepest insight in Sortify's design is that an agent can be wrong in two orthogonal directions, and confusing them leads to strictly worse outcomes.

### Epistemic Error: "I thought X would happen, but it didn't"

The system predicts that an offline I_order uplift of +10% will produce a positive online order uplift. Instead, orders drop. The world behaved differently than the model expected.

**Correct response**: Update the belief. Adjust the migration function's intercept via LMS regression and/or LLM jump. The system acknowledges: "My mapping from offline to online was biased. I need to recalibrate."

**Incorrect response**: Increase the penalty weight on orders. This makes the search more conservative, but the underlying mapping is still wrong -- the system is now searching harder in a distorted coordinate system.

### Axiological Error: "It happened, but I didn't realize it would hurt this much"

The system accurately predicts a slight dip in ad revenue. The dip occurs as predicted. But this small dip triggers a cascade of business alerts -- the constraint was set too loosely.

**Correct response**: Increase the penalty weight. The mapping was correct; the value assessment was wrong. The system learns: "This constraint is harder than I thought. I need to treat violations more severely."

**Incorrect response**: Adjust the intercept. The prediction was accurate -- shifting the mapping would make it *less* accurate, not more.

## Dual Timescale: Accumulation vs. Reaction

The two channels also operate on different time horizons, forming a complementary rhythm:

| | Belief Channel | Preference Channel |
|---|---|---|
| **Update speed** | Slow (LMS: eta=0.2 per round) | Fast (multiplicative: can tighten materially in one round) |
| **Persistence** | Written to Memory DB, permanently accumulated | Written to config, adjustable each round |
| **Character** | Learning -- gradual cognitive refinement | Reflex -- immediate response to pain |

Belief is like calibrating a scientific instrument: each measurement refines the calibration slightly, and the refinement persists forever. Preference is like adjusting pain sensitivity: when you touch a hot stove, the reaction is immediate and asymmetric (the penalty for touching it again increases far faster than it decreases after you've been safe for a while).

This dual-timescale design gives the system both **long-term stability** (belief changes slowly, so a single noisy round cannot cause wild swings) and **short-term responsiveness** (preference reacts immediately when a constraint is violated, so the system doesn't keep making the same mistake).

## The Three-Layer Architecture as a Natural Result

The separation of belief and preference, combined with the distinction between what an LLM can judge and what requires numerical search, produces the three-layer architecture:

```
Layer 1 (Human/Config):  Define objectives and constraints  --  "the constitution"
         |
Layer 2 (LLM + Algorithm):  Calibrate belief and preference  --  "the coaching staff"
         |
Layer 3 (Optuna TPE):  Find optimal parameters within the calibrated framework  --  "the athletes"
```

- **Layer 1** is static: humans define what to optimize (GMV influence) and what to protect (orders, ads revenue). This changes infrequently.
- **Layer 2** is adaptive: every round, it automatically recalibrates the constraint landscape based on online feedback. This is where accumulated experience lives.
- **Layer 3** is reactive: given a constraint landscape, it finds the best parameters for the current dataset. Its output is a one-time fact, not reusable knowledge.

Each layer handles exactly the type of problem it is suited for. No layer attempts to do another layer's job.

## Why This Matters for Practitioners

The practical consequence of this design is **diagnosability**. When an online metric underperforms:

1. Check the Belief channel: Was the offline-to-online mapping accurate? If not, the intercept needs adjustment.
2. Check the Preference channel: Was the penalty weight sufficient? If the mapping was accurate but the constraint was still violated, the penalty needs to increase.
3. Check the search: Did Optuna find a good solution within the given constraints? If both channels are correctly calibrated but results are poor, the constraint specification itself (Layer 1) may need human revision.

Without the separation, you face a single black box: "something went wrong, but we don't know whether it's the prediction, the penalty, or the search." With the separation, each failure mode has a specific diagnostic and a specific fix.
