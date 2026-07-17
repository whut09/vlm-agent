# Codex 分段实施 Prompts

以下 prompts 按顺序执行。每一段都要求 Codex 先检查仓库、`AGENTS.md` 和测试，只修改当前阶段需要的文件；完成后运行最小相关测试并汇报改动、测试和风险。不要一次粘贴全部 prompts。

## Prompt 0：仓库审计与迁移清单

```text
你正在实现 VLM Agent。先不要写业务代码。

任务：
1. 阅读 README.md、docs/architecture.md、docs/research-map.md。
2. 检查仓库结构、AGENTS.md、git status 和测试配置。
3. 只读分析 ../YOLO-Agent-reference；若不存在则提示，不要盲猜。
4. 输出直接复用、泛化复用、必须重写、暂不迁移四类清单。
5. 为每项标注源文件、目标文件、依赖、风险和最小测试。
6. 只生成 docs/migration-plan.md。

约束：不复制 Ultralytics、COCO 或 detector-specific 代码；不声称 VLA 训练已实现；遵守 MIT 许可证。
验收：覆盖 core、harness、adapters、retrieval、training、evaluation、research、reports，并给出迁移顺序。
```

## Prompt 1：Python 工程骨架

```text
根据 docs/migration-plan.md 创建最小 Python package。

任务：
1. 创建 pyproject.toml，包名 vlm-agent，Python >=3.10。
2. 创建 vlm_agent/、tests/、CLI 和 `python -m vlm_agent` 入口。
3. 默认依赖仅 pydantic、PyYAML；训练依赖放 optional extras。
4. 实现 `vlm-agent --help` 和不下载模型的 `vlm-agent doctor`。
5. 添加最小测试。

约束：不把 torch/transformers/lerobot 设为默认依赖；不创建大量空 placeholder；返回码可测试。
验收：package 可安装，help/doctor 成功，相关 pytest 通过。
```

## Prompt 2：Core Contracts

```text
实现与具体模型无关的 core contracts。

任务：
1. 用 Pydantic 实现 TaskSpec、BudgetSpec、SafetySpec、EmbodimentSpec、ModelSpec。
2. 实现 EpisodeManifest、DatasetManifest、RetrievalRecord。
3. 实现 ExperimentNode、ExecutionResult、EvidenceBundle。
4. 所有 schema 包含 schema_version，支持稳定 JSON/YAML round-trip。
5. 校验 action range、shape、control frequency、episode id 和版本字符串。
6. 添加 focused tests 和示例配置。

约束：Path 序列化跨平台；不导入具体 VLA SDK；扩展字段集中在 metadata。
验收：非法 action range、空 episode id、非法版本失败；round-trip 语义一致。
```

## Prompt 3：状态机、证据和恢复

```text
参考 YOLO-Agent 的通用设计，实现 VLM Agent 可恢复核心。

任务：
1. 实现 LoopState、LoopStageState、StageContract 和默认 stage order。
2. 实现 append-only EventLog、ArtifactManifest、DecisionLedger。
3. 实现 EvidenceStore 的 record/load/completeness validation。
4. 实现 ExecutionQueue 的状态、幂等 key、lease/heartbeat 基础接口。
5. 实现 run context、save/load、blocked、resume 和 retry。
6. 测试正常推进、缺失 evidence、失败、恢复和重复执行。

约束：requires 不满足时禁止继续；artifact 记录 hash/producer/parent；resume 不重复已完成幂等工作。
验收：dry-run loop 可 init -> report；补齐缺失 evidence 后可 resume。
```

## Prompt 4：Episode-aware Dataset Harness

```text
实现第一版 episode-aware 数据系统，不训练模型。

任务：
1. 定义 DatasetAdapter protocol。
2. 实现 toy/filesystem adapter 与 LeRobot adapter 的可选依赖边界。
3. 实现 scan、schema、modality completeness 和 statistics。
4. 实现按 task/scene/robot/operator/time 的 group split。
5. 实现 episode manifest、dataset version、diff、promote。
6. 增加 near-duplicate guard 接口和简单 perceptual hash baseline。
7. 添加 dataset profile/version/diff CLI 与测试。

约束：禁止按 frame 随机切分；未安装 lerobot 时 core tests 仍可运行；默认不复制大数据。
验收：相同 group 不跨 split；文件修改会改变 manifest hash 和 diff。
```

## Prompt 5：Model Adapter 与 SmolVLA

```text
实现模型适配协议和 SmolVLA 第一版 adapter。

任务：
1. 定义 ModelAdapter：capabilities、prepare_batch、predict_action、train_plan、checkpoint metadata。
2. 定义 observation/action compatibility report。
3. 实现 SmolVLA adapter，重依赖延迟导入。
4. 实现 DryRunExecutor 和 typed SFT CommandSpec。
5. 支持生成训练计划；依赖可用时支持最小 smoke。
6. 记录 checkpoint、dataset、normalization、precision 和 git lineage。
7. 添加 mock tests，CI 不下载 checkpoint。

约束：不泄漏 token；adapter 不修改 LoopState；action 不兼容时提前失败。
验收：dry-run 输出稳定 typed plan；缺少可选依赖时错误可操作。
```

## Prompt 6：Replay 与 LIBERO 评测

```text
实现统一评测层。

任务：
1. 定义 EvaluationAdapter、RolloutRecord 和评测 evidence schema。
2. 实现 offline replay evaluator：action error、chunk consistency、latency。
3. 实现 LIBERO adapter 的可选依赖边界和 dry-run。
4. 记录 task/seed/scene 粒度 success、partial success、episode length。
5. 实现 failure taxonomy 与关键失败 clip artifact。
6. 实现 baseline/candidate 可比较性检查和 mock env tests。

约束：不只输出全局平均；timeout/env crash/policy failure 分开；配置不匹配时不晋级。
验收：固定 mock policy 评测可复现；failure clip 可回溯 episode 和时间区间。
```

## Prompt 7：视觉与轨迹检索 MVP

```text
实现可审计的视觉/轨迹检索。

任务：
1. 定义 Encoder、VectorIndex、Reranker protocol。
2. 支持 frame、clip、episode 三类 RetrievalRecord。
3. 实现本地 numpy 索引和可选 faiss backend。
4. 实现 metadata filter、top-k、temporal rerank、diversity selection。
5. 实现 split leakage 和 near-duplicate guard。
6. 每次 query 写 RetrievalTrace：query hash、index version、来源和 score。
7. 添加 retrieval build/query/audit CLI 与 synthetic tests。

约束：val/test episode 不能进入训练 memory；原始图像不复制进索引；encoder 版本变化产生新索引版本。
验收：相似片段可召回；跨 split 或近重复污染被 audit 阻止。
```

## Prompt 8：Retrieval-aware Candidate

```text
把检索接入 planner，但暂不直接修改底层动作策略。

任务：
1. 实现 RetrievalContextBuilder。
2. failure analyst 检索相似成功、失败和干预片段。
3. candidate planner 输出 hypothesis、changed variable、expected evidence。
4. 生成 no-retrieval、random-retrieval、top-k retrieval 消融。
5. 实现 deterministic RecipeCritic：兼容性、泄漏、预算、安全、可否证性。
6. 记录 prompt hash、retrieval trace 和 gate 结果。

约束：LLM 输出不能直接进 shell；检索必须保留 lineage；没有对照时不得声称 RAG 有效。
验收：冻结输入产生可追踪计划；污染结果在 gate 被拒绝。
```

## Prompt 9：OpenVLA-OFT Adapter

```text
在通用 ModelAdapter 上实现 OpenVLA-OFT adapter。

任务：
1. 审计上游代码、许可证并固定支持 revision。
2. 映射 image/history、language、proprio 和 continuous action chunk。
3. 实现 normalization 与 action denormalization contract。
4. 实现 SFT dry-run、smoke plan、checkpoint import、capability report。
5. 复用统一 dataset/evaluation/evidence，不创建第二套 loop。
6. 添加 mock tests 和可选 LIBERO smoke 配置。

约束：不复制大型上游源码；未知机器人不标记兼容；代码/checkpoint/data stats revision 必须绑定。
验收：SmolVLA 与 OpenVLA-OFT 可在同一 capability matrix 比较；不兼容 schema 提前失败。
```

## Prompt 10：预算调度与晋级

```text
实现 VLA 实验预算和确定性晋级。

任务：
1. 泛化 successive halving、ASHA、Pareto 和 budget optimizer。
2. 预算支持 GPU hours、steps、episodes、sim rollouts、real robot minutes、interventions。
3. smoke 淘汰 OOM、NaN、低吞吐和 schema 错误。
4. promotion 同时考虑 success、safety、latency、coverage 和 cost。
5. 输出 rejection reason、可重试建议和边界测试。

约束：LLM 不决定最终 promotion；safety regression 不能被 success 抵消；不同 coverage 不直接排序。
验收：安全回退候选必拒绝；超预算候选不进队列。
```

## Prompt 11：RL Harness

```text
只实现强化学习 harness 和 contract，不进行真机在线训练。

任务：
1. 定义 RewardSpec、RolloutSpec、RLAlgorithmSpec、InterventionRecord。
2. 实现 OfflineRLExecutor 与 SimRolloutExecutor protocol/dry-run。
3. 为 ConRFT 风格 offline/online 阶段建立 recipe schema。
4. 为 SimpleVLA-RL 风格 outcome reward、并行 rollout、GRPO 建立 adapter boundary。
5. 实现进入 online RL 前的 deterministic gate。
6. 记录 reward version、policy lineage、rollout seed 和 safety events。
7. 添加 mock rollout tests。

约束：默认禁止 real-robot online RL；缺 simulator/reward audit/baseline evidence 时 blocked；算法字段不用无类型 dict。
验收：dry-run 生成完整 RL plan；任一安全前置缺失会阻止 online stage。
```

## Prompt 12：论文快照与复现队列

```text
实现 research-to-experiment pipeline。

任务：
1. 实现 paper registry、dedup、classification 和 local metadata import。
2. 校验 paper/code/checkpoint/license 字段。
3. 实现 component contract、compatibility review、reproduction state。
4. 构建包含输入副本和 SHA256 的 frozen ResearchSnapshot。
5. 训练 run 只引用 snapshot hash，不在训练轮次联网。
6. 从 paper claim 生成 reproduction queue 和 ablation plan。
7. 添加 snapshot tamper tests。

约束：无法确认的 claim 标记 unknown；snapshot 变化后 hash 必变；recipe 仍通过预算、安全和兼容性 gate。
验收：篡改 snapshot 后 resume 明确失败；复现项能追踪论文 claim 和本地 evidence。
```

## Prompt 13：端到端 MVP

```text
整合已有模块，完成无需下载大模型、可在 CI 运行的端到端 MVP。

场景：toy episode dataset + mock VLA + mock simulation environment。

任务：
1. init、profile dataset、build retrieval index。
2. baseline、evaluation、failure mining。
3. 生成 no-retrieval 与 retrieval candidate。
4. 执行 smoke、rollout、compare、promote。
5. 生成 report、decision ledger、next-round plan。
6. 中途 blocked 后补齐 artifact 并 resume。
7. 添加端到端测试和 quickstart。

约束：CI 不联网、不下载 checkpoint、不要求 GPU；所有 artifact 有 schema/hash/producer；mock 指标不得描述为真实结果。
验收：单命令运行端到端测试；run 目录展示 state/events/queue/evidence/retrieval/report/lineage。
```

## Prompt 14：真实训练前审计

```text
不要新增功能。执行首次真实训练前 readiness audit。

任务：
1. 检查模型、数据、环境、机器人、安全、许可证和 secrets。
2. 检查 CLI 是否只接受 typed command。
3. 检查 split leakage、checkpoint lineage、reward version、retrieval trace。
4. 检查 resume、OOM、NaN、env crash、disk full、timeout 路径。
5. 运行单元、集成、端到端 dry-run 测试。
6. 生成 docs/readiness-audit.md，按 blocker/high/medium/low 分类。

约束：测试绿色不等于真机安全；blocker 未清零时不给真实 online RL 命令；不顺手大范围重构。
验收：每个 blocker 有证据、影响、最小修复和验证命令。
```
