---
title: Model Compression
date: 6/5/2026
category: AI Infra
tags: [model-compression, quantization, pruning, distillation]
---

# 从 10~30B 到 1B：压缩结论与能力取舍

> 需求：10~30B --> 1B

以 Qwen3-o (30B) 作为source，根据调研分析以及实际需求，预期能够将30B模型压缩至3B大小（~10%）。

压缩将会损失一定的长链推理、长尾知识、上下文理解（知识记忆）、世界模型理解能力。上下文理解和知识记忆能力下降，可以通过外置知识实现超100%补偿。实际场景因对长链推理、长尾知识、世界模型理解能力的依赖不大，这些能力下降对最终实现效果影响不大。

![知识蒸馏简图](../Distillation.drawio.png)

## 1. 核心判断

模型压缩必然带来能力损失，尤其是长链推理、复杂规划和世界模型完整性。知识蒸馏更适合保留语言表达、指令跟随和固定任务能力；领域知识和实时信息更适合通过高质量数据、RAG 和工具调用补偿。

[Capacity Gap（容量间隙）](https://aclanthology.org/2025.acl-long.1097.pdf)：小模型受到参数规模、层数和宽度等物理容量限制，在部分任务上存在能力上限。教师模型和学生模型差距越大，学生模型越容易出现“学不会”或只能学习表层行为的情况。

可以概括为：蒸馏容易保留的是结果，难保留的是产生结果的内部机制。

## 2. 能力取舍

压缩后的能力变化可以分成两类：

- 记忆与知识检索能力：包括事实记忆、领域知识、RAG 问答和工具使用等。
- 推理与组合能力：包括长链推理、复杂规划、泛化和 In-context Learning 等。

| 能力                    | 压缩后能否保留 |
| --------------------- | ------- |
| 语言流畅性                    | 基本可以    |
| 指令跟随                      | 基本可以    |
| 特定领域知识                  | 可以      |
| 固定任务能力                  | 可以      |
| Tool Calling                 | 可以      |
| RAG 问答                      | 可以      |
| 长链推理                      | 部分损失    |
| 复杂规划能力                  | 明显下降    |
| In-context Learning 能力      | 明显下降    |
| 世界模型完整性                 | 明显下降    |


世界模型完整性指模型对真实世界知识、关系和常识结构的综合建模能力，这部分通常需要较大的模型容量。RAG 问答能力指压缩后的小模型仍然能够利用外部知识库完成检索增强问答。

## 3. 容易保留的能力

知识蒸馏后较容易保留的是结构清晰、边界明确、训练数据覆盖充分的能力：

- 语言流畅性和基础表达能力
- 指令跟随能力
- 固定任务能力
- 特定领域内的事实问答
- Tool Calling 和 RAG 问答流程

## 4. 可补偿的能力

小模型的知识覆盖和实时性不足，可以通过外部数据和系统设计进行补偿，但这类提升更多依赖工程方案，而不是模型参数本身。

在特定知识库或垂直场景中，通过高质量蒸馏数据、合成数据和持续训练，小模型在事实问答上的准确率有机会接近教师模型。领域知识不足可以通过数据质量、训练时长和外部知识库进行补偿。

主要补偿方式：

- 高质量蒸馏数据
- 合成数据
- 外挂知识库
- 工具调用
- Test-Time Compute（增加推理时长）

## 5. 难以保留的能力

泛化与未见任务能力：

- 小模型由于缺乏足够抽象的底层表征能力，会弱于大模型。

长链推理能力：

- 当问题复杂度超过一定阈值后，模型可能出现中间步骤错误累积，最终导致结果崩塌。

复杂规划能力：

- 需要同时维护多个目标、约束和中间状态时，小模型更容易丢失上下文或选择短视策略。

In-context Learning 能力：

- 小模型从少量上下文示例中抽象新规律的能力通常弱于大模型。

核心原因是：推理和组合能力依赖模型内部形成复杂的计算结构。小模型的层数、宽度和注意力头数量有限，能够表达的函数复杂度也更有限，因此很难完整继承大模型的抽象推理能力。

## 6. 推荐实现方案 

实现技术：通过将30B模型稀疏化到3B，再通过知识蒸馏进行提升其通用能力。

理想情况下（例如 30B -> 1B），可以将目标拆成：保留部分通用能力，通过数据和知识库增强领域能力，再通过工具调用补齐外部能力。

## 7. 学术实验支撑

[《What Do Compressed Deep Neural Networks Forget?》](https://arxiv.org/pdf/1911.05248)

- 实验模型：ImageNet 上的 ResNet50、CelebA 上的 CNN 分类器。
- 压缩方法：量化和稀疏化。
- 结论：模型压缩后整体准确率可能变化不大，但会损失部分能力。类比 LLM，可理解为更容易损失稀有知识或长尾知识，基础分类和短问答变化相对较小。

[《Quantization Hurts Reasoning? An Empirical Study on Quantized Reasoning Models》](https://arxiv.org/abs/2504.04823v1)

- 实验模型：DS-R1-Distill-Qwen-1.5B、7B、14B、32B。
- Benchmark：AIME24、MATH500、GPQA、LiveCodeBench。
- 压缩方法：Weight-only Quantization、KV Cache Quantization、Weight-Activation Quantization。
- 结论：小参数模型的长链推理能力在量化后下降更明显。

[Phase transitions in large language model compression](https://www.nature.com/articles/s44387-026-00072-8)

- 实验模型：LLaMA-7B 为主，补充 Qwen2.5/7B/14B/72B、Gemma、LLaMA3.1-8B。
- 压缩方法：Structured Pruning、Unstructured Pruning、Quantization、Low Rank、联合压缩。
- 指标：PPL（perplexity）。
> meaning the model can be compressed to ~10% of its original size without significant performance degradation.

- 结论：论文指出在多种压缩方式组合下，理论估计可达到约 10% 压缩率；同时模型压缩存在临界值，超过临界值后性能会发生明显变化。

## 8. 压缩技术背景

理解性分类：

- 量化
- 蒸馏
- 剪枝

技术性分类：

- 数值量化（Data Quantization）
- 模型稀疏化（Model sparsification）
- 知识蒸馏（Knowledge Distillation）
- 轻量化网络设计（Lightweight Network Design）
- 张量分解（Tensor Decomposition）

[Unified Scaling Laws for Compressed Representations](https://arxiv.org/abs/2506.01863)：不同压缩方式（量化、稀疏、蒸馏）最终都在减少模型有效容量（Effective Capacity）。
