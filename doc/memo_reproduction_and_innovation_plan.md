# MEMO 复现与轻量创新工作计划

## 1. 当前结论

第一篇小论文建议以 MEMO 为复现和改造底座。原因是 MEMO 有官方代码，任务是 TextArena 小型多智能体游戏，memory 模块位置清楚，baseline 和 ablation 都比较自然，适合做“轻量改进 + 小规模复现”。

推荐题目暂定为：

> Feedback-Guided Dual-Pool Memory Retrieval for Multi-Agent LLM Games

核心定位：不重做 MEMO，而是在 MEMO 的 memory bank 与 retrieval 逻辑上做一个小而清楚的改进。

## 2. 候选论文角色分工

| 论文 | 建议角色 | 主要用途 |
|---|---|---|
| MEMO | 主复现底座 | 多智能体游戏、self-play、memory bank、prompt evolution、replay、TrueSkill |
| ACON | 机制参考 | compression、filtering、cost-aware token budget |
| MemoryArena | 评价思想参考 | multi-session memory-agent-environment loop，强调 memory 对后续行为的影响 |
| MemEvolve | taxonomy 参考 | encode / store / retrieve / manage 的 memory architecture 拆分 |
| EvolveMem / DecentMem / MemMA | related work 与灵感 | feedback-driven memory configuration、multi-agent memory 相关叙事 |

第一阶段不建议完整复现 ACON、MemoryArena 或 MemEvolve。

## 3. 代码阅读清单

优先读 MEMO 代码，目标是能跑实验、定位 memory 接口、完成最小改造。

| 模块 | 阅读重点 |
|---|---|
| `README.md` | 安装、Quick Start、支持任务、关键实验参数 |
| `mpr/self_play_prompt_evolution_memory.py` | 主流程、`run_evolution`、memory update、evaluation、CLI 参数 |
| `mpr/memory/trajectory_memory_system.py` | memory schema、insight 生成、merge、`MemoryEnhancedAgent`、retrieval |
| `mpr/prompts/prompt_evolution_engine.py` | `memory_guided_ratio`、`insight_sampling_mode`、`_generate_memory_guided_prompt` |
| `mpr/tournament/tournament.py` | tournament、baseline/eval、best candidate evaluation |
| `mpr/evaluation/simple_memory_eval.py` | 具体游戏运行、trajectory 保存、replay 注入、统计 |
| `mpr/replaybuffer/replaybuffer.py` | replay state 采样，可用于 stage/state-aware retrieval |

## 4. 创新点设计

主创新保留一个方向：

**Feedback-Guided Dual-Pool Memory Retrieval**

在 MEMO 原有 single memory bank 基础上，将 insight 分成两个池：

| Pool | 内容来源 | 预期作用 |
|---|---|---|
| Exploitation pool | 胜局、高 TrueSkill prompt、低错误率 trajectory 的成功经验 | 提供可靠策略，提升稳定胜率 |
| Exploration pool | 失败局、draw、格式错误、无效动作、低置信反思 | 提供纠错和探索信号，避免过早收敛 |

检索策略：

| 策略 | 说明 |
|---|---|
| fixed dual-pool | 固定比例检索，例如 exploitation:exploration = 0.7:0.3 |
| stage-aware dual-pool | 根据 opening / midgame / endgame 动态调整两个 pool 权重 |
| cost-controlled top-k | 限制每次注入的 insight 数量，例如 `top_k=3/5/8` |

第一版实现应优先改 memory schema 和 retrieval，不改 TextArena，不重写 tournament，不大改 prompt evolution 主流程。

## 5. 最小接口设计

建议新增 CLI 参数：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--memory-pool-mode` | `single` | `single` 保持 MEMO 原版；`dual` 启用双池 |
| `--retrieval-policy` | `all` | `all`、`fixed_dual`、`stage_dual` |
| `--exploitation-weight` | `0.7` | fixed dual-pool 下 exploitation pool 权重 |
| `--memory-top-k` | `5` | 每次最多注入的 insight 数量 |

memory JSON 建议保留原字段以兼容 MEMO：

```json
{
  "format": "simple",
  "insights": [],
  "exploitation_insights": [],
  "exploration_insights": [],
  "pool_stats": {}
}
```

## 6. 阶段安排

| 阶段 | 目标 | 产出 |
|---|---|---|
| 1 | 跑通 MEMO 原版 smoke test | 可复现命令、logs、summary、trajectory |
| 2 | 整理复现基线 | no-memory 与 MEMO original 的小规模结果 |
| 3 | 实现 dual-pool | 双池 memory JSON、fixed dual-pool retrieval |
| 4 | 实现 stage-aware retrieval | stage 判断、动态 pool 权重、检索日志 |
| 5 | 完成实验与论文材料 | 主表、ablation、成本曲线、论文初稿素材 |

## 7. 默认假设

- 不训练本地模型，不做 LoRA，不需要 GPU。
- API 成本优先可控，因此实验规模小但实验矩阵完整。
- 如果效果提升不稳定，论文叙事转为：dual-pool 在部分任务或低 token budget 下更稳，并用 cost-efficiency 与 ablation 支撑。
