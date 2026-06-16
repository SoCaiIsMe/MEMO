# 智能体实验对比设计

## 1. 实验要回答的问题

围绕四个问题设计实验：

1. memory 是否比 no-memory 有效？
2. dual-pool 是否比 MEMO 原版 single memory bank 有效？
3. stage-aware retrieval 是否比固定比例 retrieval 有效？
4. 额外 memory 注入带来的 token 成本是否划算？

## 2. 主对比实验

| 组别 | Memory | Retrieval | 目的 |
|---|---|---|---|
| G1 No Memory | 不使用 memory | none | 最基础下限 |
| G2 MEMO Original | single memory bank | 原始 insight sampling | 官方方法复现 |
| G3 Dual-Pool Fixed | exploitation + exploration | 固定比例，例如 0.7/0.3 | 验证双池是否有效 |
| G4 Dual-Pool Stage-Aware | exploitation + exploration | 按游戏阶段动态调权 | 验证 stage-aware 是否有效 |

这四组是论文主表格的核心。

## 3. 消融实验

优先级从高到低：

| 消融 | 对比目的 |
|---|---|
| only exploitation pool | 检查只用成功经验是否过度保守 |
| only exploration pool | 检查只用失败反思是否噪声过大 |
| fixed dual-pool vs stage-aware dual-pool | 证明动态调权不是装饰 |
| no replay vs replay | 检查 replay buffer 对 memory 质量的贡献 |
| no merge / simple add / basic merge | 检查 memory 更新策略的影响 |

如果预算有限，至少做前三个。

## 4. 检索与成本实验

| 设置 | 目的 |
|---|---|
| `top_k=3` | 低成本、少量 memory 注入 |
| `top_k=5` | 默认主设置 |
| `top_k=8` | 更多 memory，检查收益递减或干扰 |
| random retrieval | 检查 proposed retrieval 是否优于随机选 insight |

成本指标建议同时报告：

- total tokens
- self-play tokens
- optimization tokens
- injected memory length
- win-rate improvement per 1k tokens

## 5. 任务选择

| 任务 | 角色 |
|---|---|
| `TicTacToe-v0` | smoke test，规则简单，快速验证 pipeline |
| `SimpleNegotiation-v0-short` | 主实验，更能体现多智能体交互、策略和记忆 |
| `KuhnPoker-v0-short` | 可选，不完全信息场景，适合作补充泛化 |

正式小论文结果优先保证 `TicTacToe-v0` 和 `SimpleNegotiation-v0-short`。

## 6. 推荐最小实验矩阵

| 组别 | 设置 |
|---|---|
| G1 | no memory |
| G2 | MEMO original |
| G3 | dual-pool fixed 0.7/0.3 |
| G4 | dual-pool stage-aware |
| G5 | only exploitation |
| G6 | only exploration |

默认参数：

| 参数 | smoke test | 正式小实验 |
|---|---:|---:|
| `generations` | 1 | 3-5 |
| `population-size` | 2-4 | 4-6 |
| `tournament-rounds` | 1-3 | 10-20 |
| `eval-rounds` | 1-3 | 10-20 |
| repeats / seeds | 1 | 至少 3 |

## 7. 主要指标

| 指标 | 说明 |
|---|---|
| best candidate win rate | 最核心结果 |
| average population win rate | 观察整体进化质量 |
| TrueSkill | 与 MEMO 官方逻辑一致 |
| draw rate / loss rate | 防止只看胜率造成误判 |
| invalid move rate | memory 是否减少非法动作 |
| format error rate | memory 是否改善输出格式 |
| token usage | 成本 |
| win-rate per 1k tokens | 成本效率 |

## 8. 预期论文结论形态

理想结论：

- MEMO original 优于 no-memory。
- dual-pool fixed 优于 MEMO original。
- stage-aware dual-pool 优于 fixed dual-pool。
- `top_k=5` 在效果和成本之间较均衡。

如果提升不稳定，可以改写为更稳妥的结论：

- dual-pool 在 negotiation 等策略交互更强的任务上收益更明显。
- exploration pool 单独使用噪声较大，但与 exploitation pool 组合后能提升鲁棒性。
- stage-aware retrieval 在低 token budget 下更有优势。
