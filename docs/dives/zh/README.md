# Sortify 架构解析

本目录包含一组架构解析文章，解释 Sortify 架构背后的**为什么**。每篇文章自成体系，但在概念上逐层递进。

## 阅读导引

| # | 文档 | 核心问题 | 前置阅读 |
|---|------|----------|----------|
| 1 | [理论基础](01-theoretical-foundations.md) | 为什么信念和偏好必须分离？ | -- |
| 2 | [影响力份额](02-influence-share.md) | 如何量化每个因子对排序的贡献？ | -- |
| 3 | [双通道机制](03-dual-channel-mechanism.md) | 信念通道和偏好通道具体如何运作？ | #1 |
| 4 | [LLM 元控制器](04-llm-meta-controller.md) | LLM 做什么，以及它刻意不做什么？ | #1, #3 |
| 5 | [搜索引擎](05-search-engine.md) | Optuna 如何在约束下找到最优参数？ | #2 |
| 6 | [流水线与运维](06-pipeline-and-operations.md) | 单轮如何运行，多轮如何链接为持续运行？ | #3, #4, #5 |
| 7 | [记忆架构](07-memory-architecture.md) | 为什么持久化记忆是其他一切的前提？ | #3, #6 |
| 8 | [元反思](08-meta-reflection.md) | AI 智能体构建 AI 智能体意味着什么？ | 全部 |

> **建议路径：** 先读 [理论基础](#1) 理解驱动整个设计的公理，再读 [影响力份额](#2) 了解量化骨架。之后 [双通道机制](#3) 和 [LLM 元控制器](#4) 解释核心自适应循环。想了解端到端运行逻辑时读 [流水线与运维](#6)。最后 [元反思](#8) 是关于更广泛意义的收尾论述。

## 核心数据流

```
线上 A/B 结果
      |
      v
Episodes（离线-线上配对记录）
      |
      +---> LMS 更新 prior_relations（斜率/截距）     <-- 信念通道
      |
      +---> LLM 生成 Proposal                         <-- 元控制器
      |       +-- delta_intercept  --> 调整 target_range
      |       +-- penalty_multiplier --> 调整 penalty_weight
      |
      v
Optuna TPE 搜参（受 target_range + penalty_weight 约束）
      |
      v
最优参数（小型参数向量）
      |
      v
配置发布 --> 线上流量 --> 新的 A/B 结果 --> 下一轮
```
