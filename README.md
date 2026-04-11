<div align="center">

# Sortify

**Let the Agent Steer: Closed-Loop Ranking Optimization via Influence Exchange**

*The first autonomous LLM agent that continuously optimizes live production ranking —<br>observing, reasoning, deciding, and self-correcting without human intervention.*

[English](README.md) | [中文](README-zh.md)

<br>

<table width="100%">
<tr>
<td align="center" width="50%">
<h3><a href="https://arxiv.org/abs/2603.27765">Technical Report</a></h3>
</td>
<td align="center" width="50%">
<h3><a href="docs/dives/en/">Architecture Dives</a></h3>
</td>
</tr>
</table>

</div>

---

> [!IMPORTANT]
> **100% Agent-Generated.** The Sortify system itself — ~100K lines of production code — and every artifact in this repository (technical reports, architecture dives, figures, and demo video) were generated entirely by AI agents orchestrated by **[yin.cheng](https://www.zhihu.com/people/cheng-yin-36)**. Zero lines of code, zero sentences of prose, and zero pixels of diagrams were produced by human hand.

> [!NOTE]
> This repository is intended solely as a companion to the Sortify technical report. It reorganizes, explains, and guides readers through the core material already disclosed in the paper. It should not be read as separate product documentation, internal implementation documentation, or business operations documentation.

Sortify is the **first LLM-driven autonomous agent that takes full control of ranking optimization in a live production recommender system** — observing real-time metrics, reasoning about multi-objective trade-offs, making parameter decisions, and self-correcting from its own online outcomes, all without human intervention.

But what makes Sortify truly singular is its **meta-recursive nature**: the entire system — ~100K lines of production code, 78 modules, 7-table memory database, dual-channel calibration engine, and this very document — was built *from scratch by AI coding agents* orchestrated by a single human. **Zero lines of production code were written by a human hand; zero sentences in the technical report were drafted by a human.** The human served solely as *architect and orchestrator*: defining objectives, decomposing problems, reviewing outputs, and steering agents toward coherent execution.

This creates a recursive closed loop: **AI agents built an AI agent that autonomously operates a live production system.** The pattern at the development level mirrors the pattern inside Sortify itself — a higher intelligence (human) does not perform low-level optimization (writing code), but adjusts *framework-level parameters* (task specifications, design constraints) to guide agent execution. The same **"steer, don't row"** principle that powers Sortify's autonomous optimization is what made single-person development possible.

The implication is a **paradigm reconstruction, not an efficiency gain**. What used to separate "having a good idea" from "a system running in production" was a vast implementation chasm — months of infrastructure, data plumbing, and debugging. That cost has collapsed to near zero. The ceiling is no longer team size or API fluency, but the depth of one person's *understanding of the problem*, their *architectural taste*, and their *art of orchestrating agents*.

<p align="center">
  <img src="figures/abs-overview.png" alt="Sortify overview" width="100%"/>
</p>

<p align="center">
  <em>(a) Three-layer architecture &nbsp;|&nbsp; (b) Online metric trajectory across rounds &nbsp;|&nbsp; (c) LLM correction convergence</em>
</p>

<br>

## Key Results

<div align="center">

| Market | Metric | Result |
|--------|--------|--------|
| Market A (hot start) | GMV | **-3.6% &rarr; +9.2%** over 7 rounds |
| Market A (hot start) | Orders | **+12.5%** peak |
| Market B (cold start) | GMV/UU | **+4.15%** (7-day A/B) |
| Market B (cold start) | Ads Revenue | **+3.58%** (7-day A/B) |

</div>

- Fully autonomous: **~6 rounds/day**, zero human intervention after launch
- LLM cost per iteration: **~$0.03 – $0.10 USD**
- LLM corrections converge from **5 &rarr; 2 items**, demonstrating learned stability

<br>

## The Problem

Traditional ranking optimization suffers from three structural problems:

<p align="center">
  <img src="figures/intro-paradigm.png" alt="Traditional vs Sortify loop" width="85%"/>
</p>

1. **Offline-online gap** — Parameters optimized offline systematically misjudge online performance. Different metrics exhibit asymmetric biases (some optimistic, others pessimistic), and no single calibration factor can fix both simultaneously.

2. **Diagnostic entanglement** — When a metric underperforms, is the world model wrong (the offline prediction missed) or is the objective wrong (we didn't penalize the violation enough)? These require opposite corrections but produce identical symptoms.

3. **No learning across rounds** — Each optimization cycle starts from scratch. History is discarded, and hard-won calibration insights are lost.

<br>

## Architecture

Sortify addresses these with a **three-layer architecture** grounded in [Savage's Subjective Expected Utility (SEU) theory](https://en.wikipedia.org/wiki/Subjective_expected_utility), which mandates separating beliefs about the world from preferences over outcomes.

<p align="center">
  <img src="figures/arch-overview.png" alt="Three-layer architecture" width="70%"/>
</p>

<div align="center">

| Layer | Responsibility | Updated by |
|-------|---------------|------------|
| **Layer 1: Human / Config** | Business objectives, constraints, initial parameters | Human (infrequent) |
| **Layer 2: LLM + Algorithm** | Belief calibration, preference adaptation | Dual-channel auto-calibration |
| **Layer 3: Optuna TPE Search** | Find optimal 7-dimensional parameter vector | 5,000 trials &times; 25 workers |

</div>

### Influence Share: A Decomposable Ranking Metric

<p align="center">
  <img src="figures/arch-influence-share.png" alt="Influence Share calculation" width="85%"/>
</p>

Sortify introduces **Influence Share** — a novel metric that decomposes ranking decisions into per-factor percentage contributions. Unlike Kendall Tau, Influence Share is:

- **Decomposable**: each factor's exact contribution is measurable
- **Summable**: factor shares always total 100%
- **Monotonic**: increasing a factor's parameter weight always increases its share
- **Actionable**: enables precise trade-off reasoning ("move 5% influence from Orders to GMV")

### Dual-Channel: Belief &times; Preference

The core of Layer 2 is an **orthogonal dual-channel mechanism** that separately corrects two fundamentally different error types:

<div align="center">

| | Belief Channel | Preference Channel |
|-|---------------|-------------------|
| **Question** | "How do offline metrics map to online reality?" | "How severely should we enforce constraints?" |
| **Controls** | `target_range` (constraint boundary position) | `penalty_weight` (constraint boundary hardness) |
| **Update rule** | LMS regression (smooth) + LLM intercept jump (selective) | Asymmetric multiplicative scaling |
| **Error type** | Epistemic — wrong world model | Axiological — miscalibrated values |
| **Analogy** | Adjusting a map to match the terrain | Adjusting how much you care about detours |

</div>

The two channels are geometrically orthogonal: Belief moves constraint edges horizontally, Preference scales violation costs vertically. Corrections on one axis do not interfere with the other.

### LLM Meta-Controller

<p align="center">
  <img src="figures/arch-llm-controller.png" alt="LLM Meta-Controller flow" width="85%"/>
</p>

The LLM operates as a **framework-level meta-controller** — it never directly tunes bottom-layer parameters. Instead, it adjusts two high-level knobs:

- **`delta_intercept`** ∈ [-0.1, +0.1] — nudges the offline-to-online mapping (Belief)
- **`penalty_multiplier`** ∈ [0.5, 2.0] — scales constraint strictness (Preference)

It receives structured context from the last 20 episodes and 30 calibration updates, reasons about patterns, and returns JSON proposals with mandatory evidence citations. When confidence is low, it returns an empty proposal — safety by design.

### Persistent Memory

A 7-table SQLite database accumulates experience across rounds:

<div align="center">

| Table | Purpose |
|-------|---------|
| `episodes` | Complete offline/online paired records |
| `prior_relations` | Current migration slopes and intercepts (6 metrics) |
| `prior_update_history` | All belief updates (LMS + LLM), 30-record window |
| `penalty_weights` | Current constraint penalty values |
| `penalty_weight_update_history` | Preference channel updates, 30-record window |
| `llm_proposals` | LLM recommendations with evidence references |
| `evidence_links` | Traceability between proposals and episodes |

</div>

This eliminates the "Groundhog Day" problem — each round inherits and builds upon all prior learning.

<br>

## Continuous Operation: The YOLO Loop

<p align="center">
  <img src="figures/train-01.png" alt="YOLO continuous operation state machine" width="90%"/>
</p>

Sortify runs as a three-state machine:

1. **Coldstart** — Bootstrap initial parameters, anchor the time window
2. **Initial** — Execute the full pipeline once with optional human confirmation
3. **YOLO** — Infinite autonomous loop: wait 3.5h for data &rarr; pull A/B report &rarr; run 10-step pipeline &rarr; publish &rarr; repeat

Each YOLO cycle executes a **10-step pipeline**:

<p align="center">
  <img src="figures/arch-data-flow.png" alt="One YOLO cycle data flow" width="70%"/>
</p>

> Pull A/B data &rarr; Record to Memory DB &rarr; LMS calibration &rarr; Assemble LLM context &rarr; LLM proposal &rarr; Apply updates &rarr; Derive target ranges &rarr; Update penalties &rarr; Optuna search &rarr; Publish to Redis

<br>

## Evaluation

### Market A: Hot Start (7 Rounds, 25 Hours)

<p align="center">
  <img src="figures/eval-01.png" alt="GMV and Orders uplift trajectory" width="70%"/>
</p>

GMV recovered from **-3.6%** to **+9.2%** in 7 rounds, with 4 consecutive rounds of positive GMV. Orders peaked at **+12.5%**. The system demonstrated genuine self-correction — not just random walk, but directed improvement driven by calibrated belief updates.

### Offline-Online Calibration

<p align="center">
  <img src="figures/eval-02.png" alt="Offline vs online calibration trajectory" width="65%"/>
</p>

The trajectory from R2 to R7 shows the system learning to translate offline predictions into online outcomes. R2's offline +18.2% I_gmv mapped to online -3.6% GMV (severe optimistic bias). By R7, the calibrated model correctly predicted that offline +41.6% would yield online +9.2%.

### Parameter Evolution

<p align="center">
  <img src="figures/eval-03.png" alt="Parameter heatmap across rounds" width="65%"/>
</p>

The heatmap shows 7 parameters evolving across rounds. Most parameters stabilize; `ps_ads_wo` shows the highest oscillation (4.5× ratio), reflecting the structural tension between ad visibility and organic relevance — a fundamental trade-off that single-objective optimization cannot fully resolve.

### LLM Convergence

<p align="center">
  <img src="figures/eval-04.png" alt="LLM correction convergence" width="65%"/>
</p>

The number of non-zero LLM corrections decreases from 5 (R2) to 2 (R7), and maximum correction magnitude drops exponentially. The LLM's role transitions from aggressive initial framework resetting to gentle residual error correction — evidence of genuine system-level learning.

<br>

## Architecture Dives

The [Architecture Dives](docs/dives/) series serves as a guided reading companion to the technical report. These eight self-contained articles reorganize and explain the paper's core ideas from foundations to philosophy:

<div align="center">

<table width="100%">
<thead>
<tr><th>#</th><th>Document</th><th>Core Question</th></tr>
</thead>
<tbody>
<tr><td>1</td><td><a href="docs/dives/en/01-theoretical-foundations.md">Theoretical Foundations</a></td><td>Why must belief and preference be separated?</td></tr>
<tr><td>2</td><td><a href="docs/dives/en/02-influence-share.md">Influence Share</a></td><td>How do we measure each factor's contribution to ranking?</td></tr>
<tr><td>3</td><td><a href="docs/dives/en/03-dual-channel-mechanism.md">Dual-Channel Mechanism</a></td><td>How do the Belief and Preference channels actually work?</td></tr>
<tr><td>4</td><td><a href="docs/dives/en/04-llm-meta-controller.md">LLM as Meta-Controller</a></td><td>What does the LLM do, and what does it deliberately not do?</td></tr>
<tr><td>5</td><td><a href="docs/dives/en/05-search-engine.md">Search Engine</a></td><td>How does Optuna find optimal parameters under constraints?</td></tr>
<tr><td>6</td><td><a href="docs/dives/en/06-pipeline-and-operations.md">Pipeline and Operations</a></td><td>How does a single cycle run, and how do cycles chain continuously?</td></tr>
<tr><td>7</td><td><a href="docs/dives/en/07-memory-architecture.md">Memory Architecture</a></td><td>Why is persistent memory the prerequisite for everything else?</td></tr>
<tr><td>8</td><td><a href="docs/dives/en/08-meta-reflection.md">Meta-Reflection</a></td><td>What does it mean that AI agents built an AI agent?</td></tr>
</tbody>
</table>

</div>

Start with #1 and #2 for foundations, then #3 and #4 for the core loop.

<br>

## Design Philosophy

Sortify's architecture is not a collection of engineering heuristics — it is a necessary consequence of **Savage's SEU axioms**. The key philosophical commitments:

- **Belief-Preference separation is mandatory**, not optional. Mixing "what I think will happen" with "what I want to happen" produces wishful thinking (optimistic beliefs inflate) or sour grapes (preferences warp to match poor predictions). The dual-channel architecture is the structural enforcement of this axiom.

- **The LLM controls the framework, not the parameters.** Bottom-layer parameter optima are data-dependent one-time facts — they change every round and are best found by numerical search. Framework parameters (migration models, penalty weights) encode cross-round structural patterns — exactly the kind of knowledge LLMs excel at recognizing.

- **Asymmetric loss aversion is built in.** Violating a constraint tightens it 3× faster than satisfying it loosens it (δ_up = 0.25 vs δ_down = 0.08). This reflects the real-world asymmetry: undershipping a metric damages business trust far more than marginally overspending on a constraint.

- **Memory is not optimization history — it is accumulated judgment.** The 7-table Memory DB doesn't just store past parameters; it stores calibrated world models, validated trade-offs, and evidence-linked decisions. Each round starts smarter than the last.

<br>

## Project Scale

<div align="center">

| Dimension | Value |
|-----------|-------|
| System codebase | 98,000 lines across 78 modules, 7 subsystems |
| Experiment rounds | 30+ across 2 markets |
| Longest continuous run | 25 hours (7 rounds, fully autonomous) |
| Cycle time | ~4 hours (3.5h data + 25–50min processing) |
| LLM cost | ~$0.03–$0.10/round |
| Human-written code | **0 lines** — entirely AI-agent generated |
| Foundation | [ParaDance](https://github.com/yinsn/ParaDance) — started May 2023; 20 months of core iteration per tech report; 101K+ PyPI downloads |

</div>

<br>

## Sortify vs. autoresearch

> [!NOTE]
> **Automated comparison by `codex-gpt-5.4-xhigh`.** [`autoresearch`](https://github.com/karpathy/autoresearch) and Sortify were completed at nearly the same time — a natural opportunity for a side-by-side architectural analysis.

**Key finding:** `autoresearch` is a remarkably strong "minimum viable harness": it compresses the research problem into a single-file mutable surface, single-metric adjudication, and fixed-time-budget local autonomy loop. Sortify turns the harness itself into the product: it engineers not only how the agent experiments, but how the world model is corrected, how preferences are revised, how memory accumulates, how autonomy recovers, and how decisions are audited.

- `autoresearch` focuses on **letting the agent iterate at high frequency inside a clean experiment sandbox**
- Sortify focuses on **letting the agent continuously make correct decisions in a messy, drifting, constrained, production, auditable real world**

| Dimension | [`autoresearch`](https://github.com/karpathy/autoresearch) | Sortify | Verdict |
|---|---|---|---|
| Problem type | Local single-machine, single-metric, short-feedback training optimization | Online ranking with multi-objective constraints and offline-online gap | Sortify faces a high-entropy world; architectural requirements are an order of magnitude higher |
| Agent action site | Directly modifies `train.py` | Never touches bottom-layer parameters; only calibrates the search framework | Sortify acts as a "bounded advisor"; `autoresearch` acts as a "hands-on hacker" |
| Mutable surface | Single file `train.py` | target range, penalty weight, search config, online parameter publishing | `autoresearch` is a strongly constrained minimal surface; Sortify is a layered mutable surface |
| Evaluator | Fixed `evaluate_bpb` in `prepare.py`, 5-min wall-clock | Offline Influence objective + online A/B + transfer calibration | Sortify's evaluator is not a point evaluation but a cross-round reality calibration system |
| Search method | Agent uses natural-language heuristic local search | Optuna TPE with 5,000 trials / 25 workers for numerical search | Sortify separates "thinking" from "searching" |
| Memory | Git + `results.tsv` + `run.log` + notebook | 7-table Memory DB + proposal/evidence/update history + state files | Memory is a first-class citizen in Sortify; in `autoresearch` it is more like lab notes |
| Autonomy loop | `program.md` constrains an external agent in an infinite loop | System-internal one-shot / YOLO pipeline | `autoresearch` is protocol-first; Sortify is runtime-first |
| Calibration | Almost no explicit calibration; directly reads measured `val_bpb` | Belief / Preference dual-channel calibration | This is the most fundamental structural watershed between the two |
| Safety & governance | Fixed evaluator, single-file mutation, no dependency additions, timeout kill | Clamp, fallback, freeze, audit, outer governance loop | Sortify is significantly closer to production-grade autonomy |
| Explainability | Change descriptions + frontier chart | Episode evidence chain + proposal reason + evidence link + update history | Sortify's diagnostic granularity is far higher |

<br>

## Resources

<p align="center">
<video src="docs/sortify-demo.mp4" width="80%" controls>
  <a href="docs/sortify-demo.mp4">Download Demo Video</a>
</video>
</p>
<p align="center"><sub>Visual walkthrough accompanying the report</sub></p>

<div align="center">

<table width="100%">
<tr>
<td align="center" width="50%">
<h3><a href="https://arxiv.org/abs/2603.27765">Technical Report</a></h3>
<sub>Full architecture, evaluation, and discussion</sub>
</td>
<td align="center" width="50%">
<a href="docs/dives/en/"><strong>Architecture Dives (EN)</strong></a><br>
<sub>8-part guide to the report's ideas</sub>
</td>
</tr>
</table>

</div>

<br>

## Citation

If you find Sortify useful for your research or work, please cite:

> Cheng et al., "Let the Agent Steer: Closed-Loop Ranking Optimization via Influence Exchange," arXiv:2603.27765, 2026.

```bibtex
@misc{cheng2026lettheagent,
  title   = {Let the Agent Steer: Closed-Loop Ranking Optimization via Influence Exchange},
  author  = {Yin Cheng and Liao Zhou and Xiyu Liang and Dihao Luo and Tewei Lee and Kailun Zheng and Weiwei Zhang and Mingchen Cai and Jian Dong and Andy Zhang},
  year    = {2026},
  eprint  = {2603.27765},
  archivePrefix = {arXiv},
  primaryClass  = {cs.AI}
}
```

## License

This repository contains documentation, figures, and reader-oriented companion materials for the Sortify technical report. All rights reserved.
