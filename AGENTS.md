# AGENTS.md

本文件是 MEMO 复现与轻量创新项目的项目级规则入口。详细工程规范、研究计划和实验方案放在 `doc/` 下。开发、实验和分析时必须先遵守本文件，再查阅对应规范。

## 项目目标

基于 MEMO 官方代码构建一个可复现、可扩展、可记录的多智能体记忆实验工程。当前目标是完成 MEMO 小规模复现，并实现面向 TextArena 多智能体游戏的轻量创新：

> Feedback-Guided Dual-Pool Memory Retrieval for Multi-Agent LLM Games

第一阶段重点：

- 跑通 MEMO 在小型 TextArena 游戏上的复现。
- 设计并实现 dual-pool memory retrieval。
- 对比 no-memory、MEMO original、fixed dual-pool、stage-aware dual-pool。
- 控制 API 成本，并保证命令、结果、日志和分析可追溯。

## 核心工程规则

- 项目采用 Harness-style research engineering 管理方式，详见 `doc/HARNESS_PROJECT_MANAGEMENT.md`。
- 所有非平凡变更必须说明 Intent、Interface、Implementation、Verification、Record。
- 不得只依赖聊天上下文保存关键决策；重要分析必须沉淀到 `doc/`。
- 优先保持 MEMO 原始行为兼容，新增能力必须通过显式参数开启。
- `single` memory 模式必须尽量保持 MEMO 原版行为。
- 新增 memory schema 时必须兼容原始 `insights` 字段。
- API 成本相关实验必须先小规模 smoke test，再扩大运行。
- 不得随意修改 TextArena 子模块，除非任务明确要求环境级改动。
- 不得大规模重构主流程，除非已有复现基线稳定。

## 代码边界规则

- 主实验流程集中在 `mpr/self_play_prompt_evolution_memory.py`。
- memory schema、merge、retrieval 相关改动集中在 `mpr/memory/trajectory_memory_system.py`。
- prompt evolution 相关改动集中在 `mpr/prompts/prompt_evolution_engine.py`。
- tournament / evaluation 相关改动集中在 `mpr/tournament/` 与 `mpr/evaluation/`。
- replay 相关改动集中在 `mpr/replaybuffer/`。
- 研究方案、实验设计、命令记录和结果分析必须写入 `doc/`。

## 文档索引

- 文档入口：`doc/README.md`
- Harness 工程管理规范：`doc/HARNESS_PROJECT_MANAGEMENT.md`
- MEMO 复现与创新计划：`doc/memo_reproduction_and_innovation_plan.md`
- 智能体实验对比设计：`doc/agent_experiment_design.md`

建议后续新增：

- 实验日志：`doc/experiment_log.md`
- 实现笔记：`doc/implementation_notes.md`
- 结果汇总：`doc/results_summary.md`

## 验证规则

- 修改 Python 代码后，至少运行与变更相关的 smoke test 或静态检查；无法运行时必须在最终说明中写明原因。
- 修改 CLI 参数、memory schema 或实验设置后，必须更新 `doc/` 中对应说明。
- 修改 memory / retrieval 行为后，必须验证 single-memory 兼容路径仍可运行。
- 运行会消耗 API token 的实验前，先使用最小参数验证流程。
- 关键实验必须记录命令、模型、任务、参数、输出目录和主要结果。

## 当前默认实验方向

- 主底座：MEMO。
- 主任务：`TicTacToe-v0`、`SimpleNegotiation-v0-short`。
- 可选任务：`KuhnPoker-v0-short`。
- 主创新：dual-pool memory retrieval + optional stage-aware weighting。
- 不做本地训练、不做 LoRA、不引入 GPU 依赖。

