# VLM Agent 初步架构

## 1. 设计原则

### Harness first

模型训练只是 executor 的一种。核心系统先解决输入契约、可恢复执行、证据、数据 lineage、预算、安全和比较，避免所有逻辑绑定到某一个 VLA 仓库。

### Typed boundary

所有跨模块输入输出使用可版本化 schema。LLM 可以提出候选，但不能绕过 schema、预算和安全 gate 直接生成并执行任意命令。

### Evidence before recommendation

没有完整 baseline、评测覆盖或数据 lineage 时，系统应输出 blocked/insufficient evidence，而不是给出高置信度优化建议。

### Frozen research

训练轮次不临时联网查询论文。论文、代码、组件、许可证和 checkpoint 元数据在轮次开始前构成 hash 固定的 research snapshot。

### Simulation before real robot

任何改变动作分布的 candidate 先经过 replay、仿真、扰动和安全评测。真机执行需要显式 adapter、workspace 限制、急停和人工批准。

## 2. 核心领域模型

建议使用 Pydantic 定义以下 schema，并为每个 schema 增加 `schema_version`。

### `TaskSpec`

- task id、自然语言指令、成功条件、task family、对象、场景和 horizon。
- 允许的 embodiment、所需传感器、评测次数、seed、阈值和预算。

### `EmbodimentSpec`

- robot、关节、末端执行器、相机和控制频率。
- observation keys、shape、dtype、坐标系和时间同步规则。
- action space、单位、范围、chunk length、控制模式、归一化和安全 clamp。

### `EpisodeManifest`

- episode id、dataset version、task、scene、robot、operator 和时间。
- frame/chunk 数、采样频率、模态完整性和文件 hash。
- success、reward、termination、intervention、failure labels、split group 和近重复 cluster。

### `ModelSpec`

- architecture、checkpoint、revision、license 和 source。
- observation/action requirements、processor、action decoder、precision 和 runtime requirements。

### `RetrievalRecord`

- memory id、episode id、时间区间和模态引用。
- visual/text/action embedding 版本。
- task、scene、object、robot、outcome、split、source hash 和过期策略。

### `ExperimentNode`

- parent、changed variables、hypothesis 和 expected evidence。
- model/dataset/research snapshot hash、executor、budget、安全级别和依赖。

### `ExecutionResult` 与 `EvidenceBundle`

- typed command、状态、资源、stdout/stderr、checkpoint、指标和 failure type。
- dataset/training/evaluation/retrieval/safety/cost evidence 的完整性、freshness 和可比较性。

## 3. 模块边界

### Core

- `LoopState`：stage 状态、attempt、blocked reason 和 artifacts。
- `StageContract`：requires/provides/evidence/retry/budget/safety。
- `EvidenceStore`：typed evidence、完整性检查和 lineage。
- `ArtifactManifest`：文件 hash、producer、parent 和 retention。
- `EventLog`：append-only JSONL。
- `DecisionLedger`：proposal、prompt hash、gate 和最终执行映射。
- `ExecutionQueue`：可恢复队列、lease、heartbeat 和幂等 key。
- `ExperimentGraph`：baseline、candidate、ablation 和 promotion 关系。

### Adapters

- model：`prepare_batch`、`predict_action`、`save/load`、`capabilities`。
- dataset：`scan`、`validate`、`iter_episodes`、`convert`。
- env：`reset`、`step`、`success`、`safety_events`、`render`。
- robot：observation/action conversion、rate control、clamp、estop。
- tracker：可选接入 W&B/MLflow，但本地 artifact 始终是 source of truth。

### Harness

- orchestrator 只推进状态与调用 stage runner。
- stage runner 校验 contract，调用 service/executor，写入 evidence。
- deterministic gate 负责可执行性、预算、安全和晋级。
- LLM agent 只负责解释、归因、候选提议和报告，不直接持有执行权。

## 4. 推荐状态机

| Stage | 主要输出 | 典型阻塞条件 |
|---|---|---|
| `init` | run context | spec 非法 |
| `validate_environment` | capability report | 依赖或硬件缺失 |
| `profile_dataset` | dataset evidence | schema、split 或模态错误 |
| `build_research_snapshot` | frozen snapshot | hash 或许可证审计失败 |
| `build_retrieval_index` | index manifest | split 污染或 embedding 不完整 |
| `run_baseline` | checkpoint/runtime evidence | smoke 失败或预算不足 |
| `evaluate_baseline` | evaluation evidence | coverage 不足 |
| `diagnose_failures` | failure facts | 事实不可复现 |
| `generate_candidates` | experiment graph | 没有可否证 hypothesis |
| `safety_review` | safety decision | 动作范围或真机风险不明 |
| `smoke_train` | smoke evidence | OOM、NaN、schema mismatch |
| `full_train` | candidate checkpoint | budget gate 失败 |
| `rollout_eval` | rollout evidence | env/robot 不可用 |
| `compare_and_promote` | promotion decision | evidence 不完整或安全回退 |
| `mine_episodes` | mining plan | consent/质量不满足 |
| `dataset_promote` | dataset version | split leakage 或质量失败 |
| `report` | report | 缺失项必须显式标注 |
| `next_round` | delta plan | 达到停止条件 |

## 5. 视觉检索子系统

### 索引单位

- frame：物体状态、局部视觉和短期匹配。
- clip：1–5 秒动作前后文，用于 temporal retrieval。
- episode/skill：完整任务和结果，用于 plan-level memory。

每条记录只保存 artifact URI 与 hash，不在向量库内复制不可追踪的原始数据。

### 表征与查询

第一版定义可替换的 visual、language、proprio/action encoder contract。查询链为：

```text
current observation + instruction + embodiment
  -> metadata compatibility filter
  -> visual/text ANN top-k
  -> temporal and outcome rerank
  -> diversity selection
  -> leakage/safety gate
  -> RetrievalContext
```

`RetrievalContext` 记录 query hash、index version、top-k、score、来源 split、episode 和时间区间。

### 注入风险等级

1. 只提供给 failure analyst 和 candidate planner。
2. 生成显式高层提示或技能选择。
3. memory embedding 作为 policy adapter 的额外 token。
4. retrieval-conditioned action decoder。

每一级必须有无检索和随机检索对照，避免把数据规模或额外 token 误判为检索收益。

## 6. 强化学习子系统

### Reward contract

统一记录 sparse success、shaped progress、safety penalty、collision、workspace violation、intervention、reset cost、smoothness 和 time-to-completion。奖励版本必须进入 checkpoint lineage。

### Executor 层级

- `SFTExecutor`：监督微调 baseline。
- `OfflineRLExecutor`：固定 episode 上的 Q/advantage/preference 训练。
- `SimRolloutExecutor`：并行环境采样和 outcome reward。
- `OnlineRLExecutor`：策略更新，默认仅允许 simulation。
- `HumanInterventionExecutor`：受控真机采样，需显式审批与日志。

### RL gate

- baseline 可复现且 seed 方差已知。
- action decoder、log-prob 或训练目标满足算法需求。
- simulator success 与安全指标达到阈值。
- reward 经过回放审计，rollout budget 和停止条件已配置。
- 真机具有 clamp、watchdog、急停和人工操作员。

## 7. 评测与故障分类

VLA failure taxonomy 至少包括：

- perception grounding：目标、属性、空间关系错误。
- instruction：忽略约束、顺序或条件。
- planning：子目标顺序、长时依赖和恢复失败。
- control：抓取、接触、振荡、超调和动作饱和。
- temporal：反应延迟、chunk 边界和观测过期。
- embodiment：坐标系、归一化、关节或夹爪不兼容。
- retrieval：错误近邻、过期 memory、跨 split 泄漏。
- safety：碰撞、越界、异常力或 watchdog 触发。

报告按 task、scene、object、robot、seed 和 failure type 分层，不能只给全局平均 success。

## 8. 从 YOLO-Agent 迁移

不要把整个仓库复制后全局替换 `yolo` 为 `vla`。建议顺序：

1. YAML/JSON IO、schema 基类和 utils。
2. loop state、stage contract、event log 和 artifact manifest。
3. evidence store、decision ledger 和 execution queue。
4. command/executor、experiment graph 和 run context。
5. orchestrator、stage runner、report 和 resume。
6. dataset versioning，随后重写为 episode-aware。
7. budget/ASHA/Pareto 等确定性策略。
8. research snapshot 与 reproduction pipeline。

每一步独立测试、无 Ultralytics/COCO import、dry-run 可执行，并保留许可证要求。

## 9. 建议 CLI

```text
vlm-agent doctor
vlm-agent run init|stage|resume
vlm-agent dataset profile|version|diff|promote
vlm-agent retrieval build|query|audit
vlm-agent train sft|offline-rl|online-rl
vlm-agent eval replay|sim|real|compare
vlm-agent research sync|build-snapshot|queue-reproduction
vlm-agent report run|compare|next-round
```

CLI 只构造 typed command，不把任意 shell 字符串直接传给 executor。

## 10. MVP 定义

- run 能从 init 推进到 report 并支持 resume。
- LeRobot episode 可验证、版本化并防止 group leakage。
- SmolVLA baseline 可通过 typed executor 启动或 dry-run。
- 指标可导入 EvidenceStore，并与 baseline 做确定性比较。
- failure clip 可写入 visual index 并带 lineage 地检索。
- 缺失 evidence、安全配置或预算时 loop 会阻塞。
