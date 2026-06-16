# MEMO 复现与轻量创新工作计划

## 1. 总体目标

以 **MEMO** 为第一篇小论文复现底座，完成三件事：

- 跑通 MEMO 官方代码在 TextArena 小型多智能体游戏上的最小复现。
- 基于现有 memory / replay / prompt evolution 结构，加入一个轻量创新：**feedback-guided dual-pool memory retrieval**。
- 设计小规模但完整的实验，比较 no-memory、MEMO 原版、dual-pool、stage-aware dual-pool 的效果与成本。

默认题目暂定为：

> Feedback-Guided Dual-Pool Memory Retrieval for Multi-Agent LLM Games

核心定位：不是重做 MEMO，而是在 MEMO 的 memory bank 与 retrieval 逻辑上做小而清楚的改进。

## 2. 论文与代码阅读计划

第一优先级读 **MEMO**，目标是能改代码、跑实验、写复现部分。

重点读：

- [README.md](D:/study/python_code/memo/MEMO/README.md)：安装、Quick Start、实验参数、支持的 TextArena 任务。
- [self_play_prompt_evolution_memory.py](D:/study/python_code/memo/MEMO/mpr/self_play_prompt_evolution_memory.py)：主流程，重点看 `run_evolution`、memory update、evaluation、CLI 参数。
- [trajectory_memory_system.py](D:/study/python_code/memo/MEMO/mpr/memory/trajectory_memory_system.py)：memory schema、insight 生成、merge、`MemoryEnhancedAgent` 和 retrieval。
- `mpr/prompts/prompt_evolution_engine.py`：prompt evolution 策略，重点看 `memory_guided_ratio`、`insight_sampling_mode`、`_generate_memory_guided_prompt`。
- `mpr/tournament/tournament.py` 与 `mpr/evaluation/simple_memory_eval.py`：tournament、baseline/eval、trajectory 输出、TrueSkill/winrate 统计。
- `mpr/replaybuffer/replaybuffer.py`：replay state 如何采样，后续可借来做 stage/state 相关 retrieval。

第二优先级读 **ACON**，只读思想，不作为主复现底座：

- history compression / filtering 怎么控制 token budget。
- cost-aware 指标如何定义。
- 可迁移到 MEMO 的部分：memory 注入数量限制、insight 压缩、token-cost vs win-rate 曲线。

第三优先级读 **MemoryArena / MemEvolve / EvolveMem / DecentMem / MemMA**：

- MemoryArena：用于 motivation 和 evaluation philosophy，强调 memory 对跨 session 行为的影响。
- MemEvolve：用于把 memory 拆成 encode / store / retrieve / manage 的 taxonomy。
- EvolveMem / DecentMem：用于借鉴 Evaluate → Diagnose → Propose → Guard 或 feedback-driven configuration tuning，但第一版不完整复现。
- MemMA：用于 related work，说明多智能体 memory 的上下文。

## 3. 创新点设计

主创新只保留一个清晰方向：

**Feedback-Guided Dual-Pool Memory Retrieval**

在 MEMO 原有 single memory bank 基础上，把 insight 分成两个 pool：

- **Exploitation pool**：来自胜局、高 TrueSkill prompt、低错误率 trajectory 的成功经验。
- **Exploration pool**：来自失败局、draw、格式错误、无效动作、低置信策略反思，保留可探索或可纠错经验。

新增 retrieval 策略：

- 固定比例 retrieval：例如 exploitation:exploration = 0.7:0.3。
- stage-aware retrieval：根据游戏阶段动态调整比例。
  - opening：更多 exploration，避免过早收敛。
  - midgame：平衡两类记忆。
  - endgame / critical state：更多 exploitation。
- cost-controlled top-k：限制每次注入的 insight 数量，例如 `top_k=3/5/8`，记录 token 成本。

实现上优先改 memory schema 和 retrieval，不改 TextArena、不重写 tournament、不大改 prompt evolution 主流程。

最低可行接口：

- 新增 CLI 参数：
  - `--memory-pool-mode single|dual`
  - `--retrieval-policy all|fixed_dual|stage_dual`
  - `--exploitation-weight 0.7`
  - `--memory-top-k 5`
- memory 文件中新增：
  - `exploitation_insights`
  - `exploration_insights`
  - 保留原 `insights` 字段，兼容 MEMO 原版逻辑。

## 4. 实验设计

优先选两个任务：

- `TicTacToe-v0`：快速 smoke test，验证流程和代码正确性。
- `SimpleNegotiation-v0-short`：主实验任务，更能体现 memory、策略、交互和反思价值。

实验组：

| Setting | 说明 |
|---|---|
| No-memory baseline | `memory_guided_ratio=0`，不使用 memory-guided prompt |
| MEMO original | 原始 single memory bank + 原始 insight sampling |
| Dual-pool fixed | exploitation/exploration 固定比例检索 |
| Dual-pool stage-aware | 按游戏阶段动态调整检索比例 |
| Optional cost-aware | 在 best variant 上比较 `top_k=3/5/8` |

默认小规模参数：

- smoke test：`generations=1`，`population-size=2~4`，`tournament-rounds=1~3`，`eval-rounds=1~3`。
- 正式小实验：`generations=3~5`，`population-size=5`，`tournament-rounds=10~20`，`eval-rounds=10~20`。
- 每个 setting 至少跑 3 个 seed 或 3 次重复。
- 模型优先用同一个 API 模型，避免模型差异污染结果。

主要指标：

- best candidate win rate。
- average population win rate。
- TrueSkill。
- draw / loss / invalid move / format error。
- token usage：total / self-play / optimization。
- memory size：insight 数量、注入 token 长度。
- cost-efficiency：win-rate improvement per 1k tokens 或 per dollar。

## 5. 阶段安排

**阶段 1：代码跑通与复现基线**

- 配置 `.env` 和依赖。
- 跑 `TicTacToe-v0` smoke test。
- 跑 `SimpleNegotiation-v0-short` 小规模 MEMO 原版。
- 保存命令、日志目录、summary JSON、trajectory JSON。

**阶段 2：复现分析**

- 整理 MEMO 原版的运行曲线。
- 确认 memory 文件结构、insight 更新逻辑、replay buffer 是否启用。
- 记录原版在小规模设置下的均值和方差。

**阶段 3：实现 dual-pool**

- 在 memory update 阶段根据 outcome / format_errors / agent performance 给 insight 分池。
- 保持原 `insights` 兼容字段。
- 在 retrieval 或 memory-guided prompt generation 处支持 `fixed_dual`。
- 添加日志：每轮选了多少 exploitation / exploration insight。

**阶段 4：实现 stage-aware retrieval**

- 先用轻量 heuristic 判断阶段：
  - turn 数少：opening。
  - 中间轮次：midgame。
  - 接近终局、出现关键胜负描述：endgame。
- 不额外调用 LLM 做 stage classifier，避免成本上升。
- 根据 stage 调整 pool 权重。

**阶段 5：实验与论文材料**

- 完成实验矩阵。
- 画 win rate / TrueSkill / token cost 曲线。
- 做 ablation：single vs dual、fixed vs stage-aware、top-k。
- 写论文结构：Introduction、Related Work、Method、Experiments、Analysis、Limitations。

## 6. 测试与验收标准

代码验收：

- smoke test 能完整跑完并生成 logs。
- 原 MEMO single memory 模式结果不受影响。
- dual-pool 模式能生成包含两个 pool 的 memory JSON。
- stage-aware 模式日志中能看到 stage、pool 权重、选中 insight 数量。
- `memory-top-k` 能限制注入内容长度。

实验验收：

- 至少有 2 个 TextArena 任务结果。
- 至少 4 个 setting 对比。
- 每个 setting 至少 3 次重复或明确说明 API 成本限制。
- 表格包含 win rate、TrueSkill、token cost。
- 图表包含 generation-wise performance curve。

## 7. 默认假设

- 第一阶段不复现 ACON / MemoryArena / MemEvolve，只读论文思想并写入 related work。
- 第一版创新不做复杂 meta-evolution，只做 dual-pool + stage-aware retrieval。
- 不训练本地模型，不做 LoRA，不需要 GPU。
- API 成本优先可控，因此实验规模小但设置完整。
- 若提升不稳定，论文叙事转为：dual-pool 在部分任务或低 token budget 下更稳，并用 cost-efficiency 与 ablation 支撑。
