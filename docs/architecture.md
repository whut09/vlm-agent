# Native Grounding VLM Agent 架构

## 1. 系统边界

系统只研究视觉语言模型的视觉理解、目标识别与定位能力。主路径是一个原生支持 grounding/pointing 的 VLM，不依赖外部 YOLO 或开放词汇检测器完成最终框预测。

系统可以借鉴 YOLO-Agent 的实验工程能力，但模型前向、输出协议、数据集和评测必须针对生成式 VLM 重写。

## 2. 设计原则

### Model agnostic

Qwen3-VL、GLM-V、Molmo2 等通过统一 `GroundingModelAdapter` 接入。内部 contract 不出现某个模型专用 token。

### Structured first

自然语言解释是辅助 evidence，目标、框、点、关系和置信必须进入稳定 schema。原始响应始终保留。

### Coordinates are data contracts

像素、0–1、0–1000、特殊 token 和 resized-image 坐标不能混用。所有转换记录原尺寸、预处理尺寸、padding/crop 和逆变换。

### Evidence before optimization

没有 baseline、标注版本、评测覆盖和原始输出时，Agent 不生成高置信度优化结论。

### One changed variable

模型、prompt、解码、分辨率、训练数据、LoRA 配置和检索上下文分别作为实验变量，避免一次修改多项后无法归因。

## 3. 能力协议

`TaskType` 建议包含：

- `recognition`：识别图中目标和属性。
- `grounding`：按自然语言描述输出一个或多个框。
- `pointing`：输出代表目标的点。
- `open_vocabulary_detection`：按开放类别列表检测。
- `referring_expression`：理解复杂指代、属性和关系。
- `counting`：计数并可选返回每个实例位置。
- `ocr_grounding`：定位并读取文字。
- `spatial_relation`：目标间上下左右、包含、邻近等关系。
- `region_understanding`：给定框解释区域，或从描述返回区域。
- `video_grounding`：后续扩展，输出时间片段和空间位置。

## 4. 核心 Schema

### `GroundingTaskSpec`

- task id、task type、query、language、output schema。
- image/video artifact、frame selection 和 image transform。
- coordinate space、允许的 region 类型和最大目标数。
- ontology、属性、关系与 negative constraints。
- latency、token、显存和成本预算。

### `ModelSpec`

- provider、model id、revision、license、quantization。
- native coordinate protocol、特殊 tokens、图像尺寸策略。
- inference backend、precision、context、最大视觉 token。
- LoRA/全参训练能力和已知限制。

### `GroundingPrediction`

- `label`、`query_span`、`box`、`point`、`mask_ref`。
- `confidence`、`evidence_text`、`attributes`、`relations`。
- 原始坐标、标准坐标、parser version 和 provenance。

### `GroundingEvidence`

- task、dataset version、split、image hash。
- raw response、parsed prediction、validation errors。
- ground truth matching、IoU、FP/FN、latency、tokens、VRAM。
- prompt、decode、model、adapter 和 code revision。

### `ExperimentNode`

- baseline/parent、hypothesis、changed variable。
- expected evidence、budget、executor 和 promotion gate。
- dataset/research/retrieval snapshot hash。

## 5. 推理链

```text
GroundingTaskSpec
  -> Image Preprocessor
  -> Prompt Compiler
  -> GroundingModelAdapter
  -> RawResponse Artifact
  -> Model-specific Parser
  -> Coordinate Normalizer
  -> Schema Validator
  -> Geometry Validator
  -> Prediction Deduplicator
  -> GroundingEvidence
```

### Prompt Compiler

负责：

- task-specific system/user template。
- 明确输出 schema 和坐标协议。
- 类别 ontology、属性、关系和 negative constraints。
- 可选的视觉/文本 few-shot context。
- 模型专用 chat template，但不泄漏到通用 task schema。

### Parser

采用 adapter-specific parser + generic validator：

- 识别特殊 box/point token、JSON、XML 或普通文本坐标。
- 允许受控 repair，但 repair 前后均保存 artifact。
- 无法确定坐标语义时返回 parse failure，不猜测。
- 同时校验 label 与 query 是否对应。

### Geometry Validator

- `x1 < x2`、`y1 < y2`。
- 坐标在允许范围内。
- resize、crop、letterbox、rotation 可逆。
- point 位于目标 box 内时可作为一致性证据。
- 重复框、异常全图框和退化小框标记为 warning/error。

## 6. Model Adapter

```text
GroundingModelAdapter
  capabilities()
  compile_prompt(task, context)
  preprocess(media)
  generate(request)
  parse(raw_response, transform)
  training_capabilities()
  checkpoint_metadata()
```

首批 adapter 建议：

1. `Qwen3VLAdapter`：默认 baseline 候选。
2. `GLMVAdapter`：精确 grounding 与 reasoning 对照。
3. `Molmo2Adapter`：pointing 和视频扩展对照。
4. `OpenAICompatibleAdapter`：用于兼容 vLLM/SGLang 或远程服务，但仍需指定坐标协议。

模型选择通过 capability matrix 和 benchmark，不通过硬编码优先级。

## 7. Dataset Harness

统一 `GroundingSample`：

- image/video URI 和 hash。
- query、task type、language、ontology。
- boxes、points、masks、phrases、attributes、relations。
- negative queries、difficult/ignore、source license。
- group id、duplicate cluster 和 split。

Adapters：COCO、LVIS、Objects365、OpenImages、RefCOCO family、Flickr30k Entities、Visual Genome、YOLO-to-grounding、自定义 JSONL。

YOLO 数据转换不能只把类别名拼成一句话。至少生成：

- 类别查询：`定位所有 {class}`。
- 属性/关系模板（存在标注时）。
- 正样本、无目标负样本和易混类别 hard negative。
- 单目标、多目标、计数和全量场景查询。

切分必须按 source image/video/group，禁止同图不同 query 跨 train/val/test。

## 8. 评测 Harness

### 定位

- AP/AP50/AP75、Recall、mean IoU。
- phrase grounding Acc@0.5/0.75。
- point-in-box accuracy、point distance。
- small/medium/large 和单目标/密集目标分层。

### 开放词汇

- base/novel、seen/unseen query。
- rare/common/frequent 类别。
- 同义词、中文/英文、属性和关系表达鲁棒性。

### 可靠性

- hallucinated object rate。
- empty-result precision/recall。
- duplicate prediction rate。
- invalid schema/coordinate rate。
- explanation 与定位的一致性。

### 效率

- 首 token、总延迟、吞吐、输出 token。
- GPU 显存、量化影响、图像分辨率成本。
- API 单图成本与失败重试成本。

## 9. Error Taxonomy

- `recognition_error`：类别或属性识别错误。
- `grounding_miss`：理解正确但框错位或漏框。
- `hallucination`：查询目标不存在但仍输出。
- `language_mismatch`：忽略指代、关系、否定或数量。
- `coordinate_protocol`：坐标尺度、顺序或 resize 逆变换错误。
- `format_error`：JSON/token/字段无效。
- `duplicate_error`：同一实例重复输出。
- `small_dense_failure`：小目标或密集场景失败。
- `ocr_grounding_error`：文字读取或对应区域错误。
- `reasoning_localization_conflict`：解释正确但位置错误，或反之。

错误事实进入 candidate planner，而不是让 LLM 直接查看所有原始日志自由发挥。

## 10. Visual Retrieval

Visual retrieval 是增强上下文和错误分析的组件：

```text
query image + task text
  -> metadata filter
  -> image/region embedding retrieval
  -> hard-negative rerank
  -> split leakage guard
  -> RetrievalContext
  -> Prompt Compiler
```

索引记录 image/region hash、类别、属性、query、标注、错误类型、split 和 encoder version。检索结果进入 prompt 时必须有 token/image budget，并记录 RetrievalTrace。

首轮消融：

- no retrieval。
- random few-shot。
- text-only similar query。
- visual positive examples。
- visual positive + hard negatives。

## 11. 训练与后训练

### Prompt/Decode Optimization

先调坐标协议、输出 schema、temperature、max tokens、图像分辨率和 few-shot，不训练模型。

### LoRA SFT

训练目标同时覆盖 reasoning text 与 box/point tokens。数据混合比例、坐标格式和负样本必须版本化。

### Preference Optimization

构造同一输入的正确/错误框、有效/无效格式、简洁/冗长解释偏好，优先提升输出稳定性。

### Reward-based Post-training

Reward contract 可包含：

- schema/coordinate validity。
- Hungarian matching 后的 IoU/coverage。
- FP、FN、duplicate、hallucination penalty。
- empty-query correctness。
- explanation-grounding consistency。
- latency/token cost。

Reward 需要防止“只输出一个大框”“永远输出空”“省略困难目标”等投机策略。

## 12. Loop State

推荐 stage：

```text
init
validate_environment
profile_dataset
build_research_snapshot
run_baseline
evaluate_grounding
diagnose_errors
build_visual_index
generate_candidates
smoke_inference
optimize_prompt_or_train
evaluate_candidate
compare_and_promote
mine_hard_samples
dataset_promote
report
next_round
```

晋级要求：定位收益达到阈值、开放词汇不回退、结构化输出错误不增加、幻觉不恶化、延迟和显存满足预算、evidence 完整。

## 13. 建议目录

```text
vlm_agent/
  core/
  harness/
  adapters/
    models/
    datasets/
    inference/
  prompting/
  parsing/
  geometry/
  retrieval/
  training/
  evaluation/
  research/
  agents/
  reports/
  visualization/
```

## 14. MVP

- toy grounding dataset 可 profile/version/split。
- 一个 native grounding VLM adapter 可 dry-run 或本地推理。
- 同一任务输出 raw response、标准 JSON 和可视化框。
- parser/coordinate/geometry 有独立测试。
- 支持 RefCOCO 风格 Acc@IoU 和 COCO 风格 AP 导入。
- failure facts 可生成下一轮 prompt 或 LoRA candidate。
- 缺失坐标协议、标注、预算或 evidence 时 loop 阻塞并可 resume。
