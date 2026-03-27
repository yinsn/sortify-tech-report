# Influence Share: A Decomposable Ranking Metric

## The Problem with Existing Metrics

In a multi-factor ranking system, the final score is a sum of terms:

```
S_i = order_i + gmv_i + ecpm_i + other_i
```

The fundamental question is: **when the system decides that item A should rank above item B, how much of that decision was driven by each factor?**

The most common tool for this -- **Kendall Tau** -- cannot answer it. Kendall Tau measures correlation between two rankings, but it has four critical limitations for parameter search:

| Limitation | Why It Matters |
|-----------|---------------|
| **Not decomposable** | Kendall Tau tells you "GMV correlates 0.7 with final ranking" but cannot say "GMV contributed 40%, orders 35%, ads 18%." The per-factor tau values are independent -- they can all be high simultaneously and don't sum to anything meaningful. |
| **Blind to magnitude** | Every pair is binary: concordant or discordant. A pair where GMV difference is 10,000 and a pair where it's 1 contribute equally. But the first pair's ranking is almost entirely determined by GMV. |
| **Conflates direct and indirect influence** | If GMV and orders are correlated, setting GMV weight to zero still yields high tau(GMV, final_ranking) because orders indirectly lift high-GMV items. Kendall Tau cannot distinguish "GMV drives ranking" from "GMV happens to correlate with what drives ranking." |
| **Non-monotonic with parameters** | Increasing a factor's weight sometimes increases tau and sometimes decreases it. This makes tau unusable as an optimization target. |

Sortify needs a metric that is decomposable into mutually exclusive percentages, sensitive to contribution magnitude, immune to indirect correlation, and capable of providing a more directional optimization signal than Kendall Tau. Influence Share satisfies the first three directly and, in practice, is far more useful as search feedback.

## The Core Insight: Ranking Is Pairwise Comparison

A ranking is not a sequence of absolute scores -- it is a collection of pairwise decisions. When item A ranks above item B, it's because S_A > S_B. The decisive quantity is the **score difference**:

```
Delta_S(A,B) = Delta_order + Delta_gmv + Delta_ecpm + Delta_other
```

where Delta_order = order_A - order_B, etc.

This difference reveals the *forces* at play: each term pushes A ahead (if positive) or pulls it behind (if negative). The ranking outcome is determined by the net result of these competing forces.

## Share: Quantifying Each Force's Contribution

For a given pair (A, B), the **share** of each factor is its absolute contribution to the total force:

```
share_order(A,B) = |Delta_order| / (|Delta_order| + |Delta_gmv| + |Delta_ecpm| + |Delta_other|)
share_gmv(A,B)   = |Delta_gmv|   / (|Delta_order| + |Delta_gmv| + |Delta_ecpm| + |Delta_other|)
...
```

Why absolute values? Because a factor that *opposes* the ranking outcome is still exerting force. If A's order score is higher by 100 but B's GMV score is higher by 100, both factors contributed 50% of the decision-making force, even though they point in opposite directions.

The shares always sum to exactly 1, forming a complete partition: **each factor gets a percentage of the "ranking decision pie."**

### A Worked Example

```
Item A (rank 1): order=120, gmv=300, ecpm=0,  other=5
Item B (rank 2): order=100, gmv=350, ecpm=0,  other=3

Delta_order = +20    (pushes A forward)
Delta_gmv   = -50    (pushes B forward)
Delta_ecpm  = 0      (no influence)
Delta_other = +2     (slightly pushes A forward)

Total force = |20| + |50| + |0| + |2| = 72

share_order = 20/72  = 27.8%
share_gmv   = 50/72  = 69.4%   <-- GMV dominates this comparison
share_ecpm  = 0/72   = 0.0%
share_other = 2/72   = 2.8%
```

Reading: A ranks above B, but this outcome is primarily driven by their GMV difference (69.4%). If GMV's weight were reduced, B would likely overtake A.

## From Pairs to Global Influence

A single pair's share is local. To get a global metric, Sortify aggregates across all relevant pairs within each request, then across all requests.

### Which Pairs Matter?

Not all pairs are equally important. The ranking of item #50 vs. #51 is irrelevant to user experience -- only the top positions matter. Sortify selects pairs involving the **Top K + Buffer** items (e.g., Top 5 plus a buffer of 15 to capture items that might enter the top positions after parameter changes).

### Position-Weighted Aggregation

Head pairs (rank 1 vs. 2) matter more than tail pairs (rank 15 vs. 20). Sortify applies exponential decay weights:

```
weight(A,B) = exp(-min(rank_A, rank_B) / tau)
```

where tau controls the decay rate. With tau=10, a pair involving rank 1 gets weight 0.90, rank 5 gets 0.61, rank 15 gets 0.22. This focuses the metric on the positions users actually see.

The global **Influence** is the weighted average of per-pair shares across all pairs and all requests:

```
I_gmv = SUM(weight * share_gmv) / SUM(weight)
```

Crucially, the Influence values across all factors still sum to 1: **the total ranking decision power is always 100%, distributed as a pie among factors.**

## Key Mathematical Properties

**Summability**: I_order + I_gmv + I_ecpm + I_other = 1, always. This makes trade-off reasoning precise: "Move 5% of influence from orders to GMV" is a well-defined operation.

**Directionality**: For a single pair, holding other terms fixed, increasing a factor's raw contribution increases that factor's local share. For the fully aggregated Influence metric, strict global monotonicity does not hold in every configuration because Top-K pair selection and the ranking itself can change. Even so, Influence Share usually provides a much more direct search signal than Kendall Tau.

**Causal isolation**: If a factor's weight is set to zero, its Influence is strictly zero, regardless of correlations with other factors. Kendall Tau would still show high correlation due to indirect effects.

## When Factors Are Not Independent

Real ranking formulas have factors that are statistically correlated (high-GMV items tend to have high orders) or share sub-expressions (order and GMV both use the same base score multiplier). Does this break the decomposition?

**Statistical correlation**: The decomposition remains mathematically valid. Each pair's "GMV contributed X%" is still a precise statement. However, correlation compresses the *exchangeable* space -- if two factors are nearly collinear, it's hard to increase one's influence without proportionally affecting the other. This is a property of the data, not a flaw in the method.

**Shared sub-expressions**: Some parameters scale multiple factors together, making them "global knobs" that cannot redistribute influence between factors. Only parameter families that primarily affect one factor enable direct factor exchange. The decomposition still holds; what changes is the size of the exchangeable space.

## What "Quantitative Exchange" Means

The title phrase "influence exchange" refers to the controlled redistribution of the 100% ranking decision pie. The business formulation is:

> "Currently, GMV accounts for 40% of ranking decisions and orders account for 35%. We want to move GMV to 45% without dropping orders below 34%."

Influence Share turns this into a precise mathematical optimization problem. The search engine finds a parameter vector that achieves the desired redistribution within constraints. Without a decomposable, summable metric, this kind of targeted trade-off reasoning would be impossible.

## A Small Set of Adjustable Parameters

In the actual system, the theoretical "adjustment weight" becomes a small vector of tunable multipliers that control different aspects of the ranking formula:

| Parameter Category | What It Controls |
|-----------|-----------------|
| Ad-side order multiplier | Order emphasis within ad traffic |
| Ad-side GMV multiplier | GMV emphasis within ad traffic |
| Organic order multiplier | Order emphasis within organic traffic |
| Organic GMV multiplier | GMV emphasis within organic traffic |
| Shared scaling multiplier | Terms that scale multiple organic components together |
| Ad-bid pressure multiplier | How strongly bid-related terms shape ranking |
| Price-sensitivity multiplier | Nonlinear sensitivity to price-related features |

Separating ad and organic controls allows finer-grained trade-offs: you can boost GMV influence in one traffic slice without equally disturbing another. Some multipliers behave like global knobs; others are closer to targeted exchange knobs.

These parameters typically use 1.0 as the "no adjustment" baseline and are searched over a fairly wide positive range.
