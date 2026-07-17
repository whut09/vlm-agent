# Native Grounding VLM Research Map

本文件用于建立初始模型、数据集、评测和训练组件地图。正式实验前应冻结模型 revision、代码 revision、许可证、数据许可和评测配置。

## 1. 首批模型候选

| 模型 | 关注能力 | 项目角色 |
|---|---|---|
| [Qwen3-VL](https://github.com/QwenLM/Qwen3-VL) | 开放词汇 2D grounding、pointing、OCR、视觉推理、多尺寸权重 | 默认 baseline 候选 |
| [GLM-V](https://github.com/zai-org/GLM-V) | 复杂描述 grounding、规范化坐标、视觉推理和后训练工具 | 精确定位对照 |
| [Molmo2](https://github.com/allenai/molmo2) | pointing、图像/视频 grounding、开放训练生态 | 点定位和视频扩展 |
| [InternVL](https://github.com/OpenGVLab/InternVL) | 多尺寸视觉语言模型和微调生态 | 通用能力与成本对照，grounding 需实测 |

模型进入 capability matrix 前记录：

- 原生 box/point 输出协议。
- 是否支持多目标、无目标、复杂指代和中文查询。
- 本地权重、许可证、量化、Transformers/vLLM/SGLang 支持。
- LoRA/全参训练方案和显存需求。
- 已知 parser、resize、tokenizer 或 chat-template 限制。

## 2. Grounding 与开放词汇基础工作

这些工作用于理解任务和评测，不表示项目需要把相应检测器接入主路径：

| 工作 | 研究重点 |
|---|---|
| [GLIP](https://arxiv.org/abs/2112.03857) | detection 与 phrase grounding 的统一预训练 |
| [Grounding DINO](https://github.com/IDEA-Research/GroundingDINO) | 开放集合目标检测与语言 grounding |
| [OWL-ViT](https://arxiv.org/abs/2205.06230) / [OWLv2](https://arxiv.org/abs/2306.09683) | image-text pretraining 到开放词汇检测 |
| [YOLO-World](https://github.com/AILab-CVC/YOLO-World) | 实时开放词汇检测，用作效率指标参考 |
| [Florence-2](https://huggingface.co/microsoft/Florence-2-large) | 统一 prompt-based vision tasks 和定位表示 |

## 3. 数据集

### 检测与长尾

- [COCO](https://cocodataset.org/)
- [LVIS](https://www.lvisdataset.org/)
- [Objects365](https://www.objects365.org/)
- [Open Images](https://storage.googleapis.com/openimages/web/index.html)
- [ODinW](https://github.com/microsoft/GLIP#odinw--object-detection-in-the-wild)

### Referring Expression 与 Phrase Grounding

- RefCOCO、RefCOCO+、RefCOCOg。
- [Flickr30k Entities](https://bryanplummer.com/Flickr30kEntities/)。
- [Visual Genome](https://visualgenome.org/)。

### OCR 与文档定位

- [TextOCR](https://textvqa.org/textocr/)。
- [DocLayNet](https://github.com/DS4SD/DocLayNet)。
- 自有票据、包装、界面和工业文档数据。

### 自有 YOLO 数据转换

保留图像和 boxes，新增 query、语言、同义词、属性、关系、负样本和 ontology。转换后的 dataset version 必须能追踪回原 YOLO 数据版本。

## 4. Benchmark Matrix

| 能力 | 主要指标 |
|---|---|
| closed/open vocabulary localization | AP、Recall、base/novel AP |
| referring expression | Acc@0.5/0.75、mean IoU |
| pointing | point-in-box、normalized distance |
| counting | exact match、MAE、实例定位覆盖 |
| negative query | empty precision/recall、hallucination rate |
| format | JSON valid、box parse、coordinate valid |
| reliability | duplicate、miss、FP、calibration |
| efficiency | latency、tokens、VRAM、throughput、cost |

评测必须按 small/medium/large、稀有类别、密集目标、遮挡、分辨率、语言和 query 复杂度分层。

## 5. Visual Retrieval

研究目标不是用检索模型替代定位，而是回答：视觉示例能否提升原生 VLM 对长尾类别、领域术语、复杂关系和 hard negative 的定位？

需要的组件：

- image/region encoder 与版本化索引。
- query-image、query-text 和 metadata 联合检索。
- positive、hard negative、same-scene 和 cross-domain 策略。
- retrieval trace、split leakage、near-duplicate audit。
- few-shot image/token budget 和消融。

## 6. 训练研究

### SFT

- 多任务混合：grounding、pointing、counting、OCR grounding、relations。
- 多坐标格式应统一为一个训练协议，避免模型学到互相冲突的表达。
- 加入 no-object、易混类别、复杂否定和重复框清理数据。

### Preference Optimization

- 正确框优于偏移框。
- 完整多目标优于只返回显著目标。
- 正确空结果优于幻觉目标。
- 有效简洁 JSON 优于不可解析长解释。

### Reward-based Post-training

可研究 GLM-V 一类基于可验证视觉 reward 的后训练思路。Reward 必须由可重复计算的 schema、IoU、coverage、FP/FN、重复、幻觉和成本构成，并保留每项分量。

## 7. Research Snapshot

```yaml
model_or_paper_id: qwen3-vl
source_url: ...
revision: ...
license: ...
native_tasks: [grounding, pointing, ocr]
coordinate_protocol: ...
inference_backends: [...]
training_support: ...
claims:
  - benchmark: ...
    metric: ...
    value: ...
components:
  - name: box_parser
    maturity: experimental
reproduction:
  status: queued
  budget: ...
  acceptance: [...]
```

## 8. 组件进入实验的门控

1. 模型或论文 revision 固定。
2. 权重、代码和数据许可证已记录。
3. 坐标协议和预处理可复现。
4. claim 可定位到官方配置、论文或模型卡。
5. changed variable 可隔离，并存在 baseline。
6. 预算、数据 split 和预期 evidence 完整。
7. research snapshot hash 已冻结。
