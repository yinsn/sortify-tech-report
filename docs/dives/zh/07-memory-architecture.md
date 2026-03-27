# 记忆架构：为什么持久化是一切的前提

> 前置阅读：[双通道机制](03-dual-channel-mechanism.md)、[流水线与运维](06-pipeline-and-operations.md)。

## 土拨鼠之日问题

没有持久化记忆，每轮优化都从零开始。第 5 轮学到的 LMS 修正在第 6 轮就丢了。LLM 只能看到 1 个 episode 而非 20 个。系统无法区分"这是 GMV 预测第一次偏乐观"和"这是连续第五轮偏乐观"。

这不是锦上添花——而是学习系统与无状态循环之间的根本分界。一个每轮都丢弃历史的系统注定重复同样的错误，如同电影《土拨鼠之日》里每天醒来都忘记昨天的主角。

Sortify 的 Memory DB 不是优化手段——它是使双通道校准、LLM 元控制和跨轮次学习成为可能的架构基础。

## 逻辑 Schema

Memory DB 在逻辑上可以分成七类实体；实现上既可以是关系型表，也可以是其他持久化存储结构。它们各自在系统的学习循环中承担特定角色。

### 1. `episodes`

中央记录表。每行是一轮完整的离线-线上周期：

| 字段 | 内容 |
|------|------|
| episode_key | 唯一标识（稳定的轮次 / 实验键） |
| offline_params | 部署的参数向量 |
| offline_metrics | 搜索预测的影响力值 |
| offline_target_ranges | 搜索使用的约束边界 |
| offline_penalty_weights | 搜索使用的惩罚权重 |
| online_uplifts | 实际 A/B 各指标 delta |
| status | pending / completed / failed |

所有其他表的更新都源自 episode 数据。

### 2. `prior_relations`

6 条迁移模型的当前状态（relation_key、slope、intercept、eta）。这是信念通道的持久化状态。

### 3. `prior_update_history`

滚动窗口（最近 30 条）的所有斜率/截距变更记录，标注更新方法（lms 或 llm）和触发的 episode。

双重用途：
- **LLM 上下文**：LLM 借此判断修正是在收敛还是发散
- **审计轨迹**：每次模型变更都可归因到具体的 episode 和方法

### 4. `penalty_weights`

各约束的当前惩罚权重。

### 5. `penalty_weight_update_history`

滚动窗口（30 条）的惩罚权重变更记录（trigger: violated 或 satisfied）。偏好通道的审计轨迹。

### 6. `llm_proposals`

每次 LLM 输出（无论是否被应用）：proposal_id、完整 JSON、是否应用、触发的 episode。

### 7. `evidence_links`

提案与其引用的 episode 之间的可追溯链接。可以回答："LLM 在第 7 轮决定下调 GMV 截距时，依据了哪些具体 episode？"

## 表间交互

```
A/B 结果到达
      |
      v
episodes: 新建记录（离线 + 线上数据）
      |
      +---> prior_relations: LMS 更新斜率/截距
      |           |
      |           v
      |     prior_update_history: 记录 LMS 更新
      |
      +---> LLM 读取: episodes(20), prior_update_history(30),
      |                penalty_weight_update_history(30), prior_relations(6)
      |           |
      |           v
      |     llm_proposals: 存储 LLM 输出
      |     evidence_links: 链接引用的 episodes
      |           |
      |           v
      |     prior_relations: LLM 更新截距
      |     prior_update_history: 记录 LLM 更新
      |
      +---> penalty_weights: 乘性规则更新
                |
                v
          penalty_weight_update_history: 记录更新
```

每次修改都被记录。没有沉默的状态变更。

## 为什么是 30 条窗口？

- **时效性**：30 轮前（约 5 天）的修正反映的是可能已不成立的环境条件
- **LLM 上下文容量**：30 条记录足以识别趋势（收敛、振荡、漂移），又不会溢出 LLM 的推理能力
- **存储**：有界表防止长时间运行时无限增长

episodes 表是无界的——所有历史记录都保留。但 LLM 只接收最近 20 个 episode 作为上下文。

## 记忆使什么成为可能

### 1. 跨轮次学习

无记忆：每轮的 LMS 修正丢失，系统永远围绕真实截距振荡。
有记忆：LMS 修正累积。7 轮后截距已接近真实值，修正幅度自然衰减。

### 2. 趋势识别

无记忆：LLM 看 1 个 episode，无法区分"GMV 这一次为负"和"连续 5 轮为负"。
有记忆：LLM 看 20 个 episode，识别出结构性模式。

### 3. 收敛监控

无记忆：无从知道系统在改善还是恶化。
有记忆：更新历史显示 LLM 修正从第 2 轮的 5 项降至第 7 轮的 2 项，幅度递减——系统级学习的直接证据。

### 4. 热启动/冷启动迁移

类似市场的 Memory DB 可以作为新市场部署的种子，提供"温启动"，避免最初几轮的严重失误。

### 5. 事后审计

系统做的每个决策——每次 LMS 更新、每个 LLM 提案、每次惩罚调整——都带有输入和驱动证据。当线上指标异常移动时，运维人员可以向后追溯因果链：

```
线上 GMV 下降
  → 第 6 轮采用了某个 GMV 相关乘子配置
    → 搜索使用了相应的约束边界
      → 该边界由某次截距校准推导
        → 截距上次被 LLM 提案 P-005 调整
          → LLM 引用了 episode E-003, E-004, E-005
            → 这些 episode 的具体离线-线上配对数据
```

## "积累的判断力"视角

Memory DB 存储的不仅是"优化历史"——过去参数和结果的日志。它的核心价值是不同的：

它存储的是**校准过的世界模型**（经数十轮证据打磨的迁移函数）、**经过验证的权衡决策**（对真实线上结果测试过的惩罚权重）、以及**有证据链的推理**（带明确引用的 LLM 提案）。

每一轮，系统继承的不仅是上一轮的参数——而是上一轮的判断力。第 10 轮的搜索在一个由前 9 轮积累智慧塑造的约束景观中运行。Memory DB 不是档案柜；它是一个学习型组织的机构记忆。
