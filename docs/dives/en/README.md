# Sortify Architecture Dives

This directory contains a series of articles that explain the **why** behind Sortify's architecture. Each article is self-contained but builds on ideas introduced in earlier ones.

## Reading Guide

| # | Document | Core Question | Depends On |
|---|----------|--------------|------------|
| 1 | [Theoretical Foundations](01-theoretical-foundations.md) | Why must belief and preference be separated? | -- |
| 2 | [Influence Share](02-influence-share.md) | How do we measure each factor's contribution to ranking? | -- |
| 3 | [Dual-Channel Mechanism](03-dual-channel-mechanism.md) | How do the Belief and Preference channels actually work? | #1 |
| 4 | [LLM as Meta-Controller](04-llm-meta-controller.md) | What does the LLM do, and what does it deliberately not do? | #1, #3 |
| 5 | [Search Engine](05-search-engine.md) | How does Optuna find optimal parameters under constraints? | #2 |
| 6 | [Pipeline and Operations](06-pipeline-and-operations.md) | How does a single cycle run, and how do cycles chain into continuous operation? | #3, #4, #5 |
| 7 | [Memory Architecture](07-memory-architecture.md) | Why is persistent memory the prerequisite for everything else? | #3, #6 |
| 8 | [Meta-Reflection](08-meta-reflection.md) | What does it mean that AI agents built an AI agent? | All |

**Recommended first reading**: Start with [Theoretical Foundations](#1) to understand the axiom that drives the entire design, then [Influence Share](#2) for the quantitative backbone. After that, [Dual-Channel Mechanism](#3) and [LLM as Meta-Controller](#4) explain the core adaptive loop. Read [Pipeline and Operations](#6) when you want to understand how everything runs end-to-end. Use [Meta-Reflection](#8) as a closing essay on the broader implications.

## Core Data Flow

```
Online A/B Results
      |
      v
Episodes (offline-online paired records)
      |
      +---> LMS updates prior_relations (slope/intercept)     <-- Belief Channel
      |
      +---> LLM generates Proposal                            <-- Meta-Controller
      |       +-- delta_intercept  --> adjusts target_range
      |       +-- penalty_multiplier --> adjusts penalty_weight
      |
      v
Optuna TPE Search (constrained by target_range + penalty_weight)
      |
      v
Best parameters (compact parameter vector)
      |
      v
Configuration publish --> Online traffic --> New A/B results --> Next cycle
```
