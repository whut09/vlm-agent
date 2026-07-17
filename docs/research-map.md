# VLA Research Map

本文件是初始研究索引，不代替逐篇复现与许可证审计。加入 production research snapshot 前，应保存论文版本、代码 revision、checkpoint、许可证、数据许可和 claim evidence。

## 1. VLA 基础路线

| 工作 | 建议关注 | 项目用途 |
|---|---|---|
| [RT-1](https://arxiv.org/abs/2212.06817) | 大规模真实机器人数据、tokenized action | action representation 基线 |
| [RT-2](https://arxiv.org/abs/2307.15818) | web-scale VLM 知识到动作迁移 | VLM-to-VLA 设计谱系 |
| [PaLM-E](https://arxiv.org/abs/2303.03378) | embodied multimodal language model | 高层语义与具身输入 |
| [Open X-Embodiment / RT-X](https://robotics-transformer-x.github.io/) | 跨机构、跨本体数据统一 | dataset/embodiment contract |
| [Octo](https://octo-models.github.io/) | 开源 generalist robot policy | 多任务、多本体 adapter 参考 |
| [OpenVLA](https://openvla.github.io/) | 开源 VLA、离散动作 token | 主流开源 baseline |
| [OpenVLA-OFT](https://github.com/moojink/openvla-oft) | action chunk、continuous action、优化微调 | 第一主研究 adapter |
| [π0](https://www.physicalintelligence.company/blog/pi0) / [openpi](https://github.com/Physical-Intelligence/openpi) | flow matching、通用 robot policy | 连续动作与大模型扩展 |
| [π0.5](https://www.physicalintelligence.company/blog/pi05) | open-world generalization | 跨环境泛化研究 |
| [SmolVLA](https://huggingface.co/blog/smolvla) | 小型开源 VLA 与 LeRobot 工具链 | 默认 MVP 模型 |
| [GR00T N1](https://arxiv.org/abs/2503.14734) / [Isaac-GR00T](https://github.com/NVIDIA/Isaac-GR00T) | humanoid、dual-system、合成数据 | humanoid/Isaac 扩展 |
| [Gemini Robotics](https://deepmind.google/discover/blog/gemini-robotics-brings-ai-into-the-physical-world/) | embodied reasoning 与动作模型 | 闭源能力参考 |
| [Helix](https://www.figure.ai/news/helix) | humanoid visuomotor policy | 长时、双臂和部署参考 |

## 2. 数据集与基准

| 数据/基准 | 重点 | Harness 需求 |
|---|---|---|
| [LeRobot](https://github.com/huggingface/lerobot) | dataset/model/robot 工具链 | 默认 dataset contract |
| [LIBERO](https://libero-project.github.io/) | lifelong manipulation、任务套件 | baseline 与遗忘评测 |
| [Open X-Embodiment](https://robotics-transformer-x.github.io/) | RLDS、多本体、多任务 | embodiment mapping、数据过滤 |
| [DROID](https://droid-dataset.github.io/) | 大规模真实世界操作数据 | 场景与操作者 group split |
| [BridgeData V2](https://rail-berkeley.github.io/bridgedata/) | 多任务真实机器人轨迹 | 跨任务迁移 |
| [RoboMimic](https://robomimic.github.io/) | imitation learning 数据和基准 | offline replay 与算法测试 |
| [RoboTwin](https://robotwin-platform.github.io/) | 双臂仿真、合成数据与评测 | RL rollout 和跨任务评估 |

优先研究动作/坐标统一、时间同步、相机缺帧、语言质量、成功标签、相邻帧泄漏、同场景跨 split 泄漏和数据许可证。

## 3. 强化学习与后训练

| 工作 | 核心启发 | 建议实验 |
|---|---|---|
| [ConRFT](https://arxiv.org/abs/2502.05450) | offline + online reinforced fine-tuning、一致性策略 | 小数据接触任务、人工干预日志 |
| [SimpleVLA-RL](https://arxiv.org/abs/2509.09674) | outcome reward、并行 rollout、VLA RL scaling | OpenVLA-OFT + LIBERO/RoboTwin executor |
| [Diffusion Policy](https://diffusion-policy.cs.columbia.edu/) | 视觉运动策略与 action sequence | 连续动作 decoder 对照 |
| [ACT](https://tonyzhaozh.github.io/aloha/) | action chunking、低成本双臂数据 | chunk length 与 temporal aggregation |

RL 组件契约至少包含：是否需要 log-prob、value/Q、reward 类型、on/off-policy、rollout 环境、action decoder 兼容性、显存、并行策略和安全条件。

## 4. 视觉 RAG、检索与记忆

| 工作 | 核心启发 | 项目映射 |
|---|---|---|
| [Embodied-RAG](https://arxiv.org/abs/2409.18313) | 分层非参数 embodied memory | semantic/episodic memory 层级 |
| [EmbodiedRAG](https://arxiv.org/abs/2410.23968) | 动态 3D scene graph retrieval | 场景过滤与 planning context |
| [RAG-Modulo](https://arxiv.org/abs/2405.16506) | 检索、生成与 verifier 闭环 | retrieval verifier 思路 |
| [RoboRAG](https://arxiv.org/abs/2511.21652) | demonstration retrieval 用于机器人策略 | trajectory retrieval 入口 |

第一阶段聚焦可验证的 episodic retrieval：

- query：当前关键帧、指令、proprio、robot/task metadata。
- corpus：训练 split 的成功、失败和人工干预片段。
- result：带来源、时间区间、outcome 和 action summary 的 typed context。
- audit：禁止检索 validation/test episode 或 near duplicate。
- evaluation：除 Recall@K 外，比较 task success、negative transfer、latency 和 safety。

## 5. Research Snapshot Schema

```yaml
paper_id: openvla-oft
title: Fine-Tuning Vision-Language-Action Models
paper_url: ...
code_url: ...
paper_revision: ...
code_revision: ...
license: ...
checkpoint_license: ...
datasets: [...]
embodiments: [...]
action_representation: ...
claims:
  - metric: ...
    benchmark: ...
    value: ...
components:
  - name: action_chunk_decoder
    maturity: experimental
    compatibility: [...]
reproduction:
  status: queued
  budget: ...
  acceptance: [...]
```

## 6. 论文到实验的门控

1. claim 可定位到论文表格或代码配置。
2. 代码、checkpoint 和数据许可证已记录。
3. observation/action/embodiment compatibility 明确。
4. changed variable 可隔离，存在 baseline 与消融。
5. 预算和预期 artifact 完整。
6. 真机风险通过安全审查。
7. research snapshot hash 已冻结。
