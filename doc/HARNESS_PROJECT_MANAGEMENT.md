# Harness-Style Research Engineering 规范

本文档定义 MEMO 复现与轻量创新项目的工程管理方式。`AGENTS.md` 是规则入口，本文件负责展开具体工作方法。

## 1. 五件套工作法

每个非平凡任务都按五件套推进：

| 项 | 要回答的问题 |
|---|---|
| Intent | 这个改动回答什么研究问题或工程问题？ |
| Interface | 改了哪些 CLI 参数、配置、JSON schema 或公开函数行为？ |
| Implementation | 最小实现路径是什么，涉及哪些核心模块？ |
| Verification | 用什么 smoke test、脚本或实验验证？ |
| Record | 哪些决策、命令、结果和风险需要写入 `doc/`？ |

任务完成时，至少要能说明 Verification 和 Record。不能验证时，必须写清楚阻塞原因。

## 2. 变更分层

优先在既有 MEMO 结构内做小步扩展。

| 层级 | 允许改动 | 注意事项 |
|---|---|---|
| Research docs | 方案、实验矩阵、结果分析 | 所有关键判断都应落文档 |
| Scripts | 新增小规模实验脚本 | 不要覆盖官方 baseline 脚本 |
| CLI / config | 新增显式参数 | 默认值必须兼容原版 |
| Memory | schema、merge、retrieval | 保留 `insights` 字段兼容性 |
| Evolution | memory-guided prompt 生成 | 避免改变非 memory 组行为 |
| Tournament / eval | 指标、日志、统计 | 不改变胜负统计语义 |
| TextArena | 环境规则 | 第一阶段默认不改 |

## 3. 推荐实现边界

当前创新点应优先围绕以下接口实现：

- `--memory-pool-mode single|dual`
- `--retrieval-policy all|fixed_dual|stage_dual`
- `--exploitation-weight 0.7`
- `--memory-top-k 5`

建议 memory JSON 兼容结构：

```json
{
  "format": "simple",
  "insights": [],
  "exploitation_insights": [],
  "exploration_insights": [],
  "pool_stats": {}
}
```

其中：

- `insights` 继续服务 MEMO 原始路径。
- `exploitation_insights` 存胜局、高 TrueSkill、低错误率经验。
- `exploration_insights` 存失败反思、draw、格式错误、无效动作和探索策略。
- `pool_stats` 记录每代分池数量、检索数量和注入长度。

## 4. 实验记录规范

每个有意义的实验都应记录：

- 日期
- 实验目的
- git 状态或 commit
- 运行命令
- 模型名
- TextArena 任务
- generations / population-size / tournament-rounds / eval-rounds
- memory 设置
- replay 设置
- 输出目录
- 主要指标
- 异常、失败和下一步

建议后续新建 `doc/experiment_log.md`，按时间追加。

## 5. 最小验证梯度

任何新功能先按以下梯度验证：

1. 参数解析或纯函数级检查。
2. `TicTacToe-v0` 极小 smoke test。
3. `SimpleNegotiation-v0-short` 小规模实验。
4. 多 seed / 多 setting 对比。

API 成本敏感时，先使用：

- `generations=1`
- `population-size=2-4`
- `tournament-rounds=1-3`
- `eval-rounds=1-3`

## 6. 结果可信度要求

论文实验至少应尽量满足：

- 主任务不少于 2 个。
- 主 setting 包含 no-memory、MEMO original、dual-pool fixed、dual-pool stage-aware。
- 每组至少 3 次重复；若预算不足，明确说明限制。
- 同时报告效果指标和成本指标。
- 保留原始日志和 summary JSON 的路径。

## 7. 交接要求

任何 agent 结束一轮较大工作时，应在最终说明中给出：

- 改了什么。
- 没改什么。
- 如何验证。
- 相关文档位置。
- 下一步最合理动作。

如果工作涉及实验，必须说明是否消耗 API token，以及实验规模。
