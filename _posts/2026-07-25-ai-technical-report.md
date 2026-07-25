---
layout: post
title: 'SGLang DSPARK 推测解码 384 tok/s、OpenForgeRL 端到端训练 Harness Agent、AREX 递归自我改进 Deep Research、Agent 上下文与逻辑推理新范式'
date: 2026-07-25 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期覆盖 2026 年 7 月 25 日 AI 推理与 Agent 领域的重要进展。OpenForgeRL 提出端到端训练 harness-based Agent 的开源框架，解决了现有 SFT/RL 栈无法原生表达有状态多进程 Agent 推理的问题；AREX 引入发现-验证不对称性驱动的递归自我改进 Agent，通过部分验证状态引导后续精炼，显著提升 Deep Research 任务效率；PATS 策略感知训练脚手架为长时域 Agent RL 提供自适应支持，解决弱策略重复失败导致的无效 rollout 问题；Agentic Context Management 将上下文管理重新定义为生命周期与架构问题，提出结构化解决方案应对 token 成本膨胀与跨会话记忆缺失；Euclid-MCP 通过标准化 MCP 接口将 Prolog 确定性逻辑推理引入 LLM Agent 工具链；开源框架方面，SGLang v0.5.16 引入 DSPARK 置信驱动推测解码（DeepSeek-V4-Pro 达 383.7 tok/s）和 Inkling 975B MoE Day-0 支持；vLLM v0.25.1 修复关键量化与启动 bug；llama.cpp 持续迭代至 b10107。

---

## 🔥 今日看点

1. **7 月 25 日** — OpenForgeRL：端到端训练 Harness-native Agent 的开源框架。现有 SFT/RL 栈无法原生表达 Claude Code、Codex 等复杂 Agent 推理 harness 的有状态多进程特性。OpenForgeRL 通过统一的训练-推理 harness 抽象，支持在多样化环境中端到端训练 Agent，涵盖工具调用、多轮推理与外部系统访问（[arXiv:2607.21557](https://arxiv.org/abs/2607.21557)）。

2. **7 月 24 日** — AREX：递归自我改进的 Deep Research Agent。基于发现-验证不对称性原理——发现满足多约束的答案成本高，但验证候选答案可分解为可处理的逐项检查——AREX 不仅延长搜索时间，而是递归改进当前答案：通过验证中间结果并利用部分验证状态引导后续精炼，显著提升 Deep Research 任务的数据效率（[arXiv:2607.21461](https://arxiv.org/abs/2607.21461)）。

3. **7 月 24 日** — PATS：策略感知训练脚手架解决长时域 Agent RL。弱策略在长 horizon Agent RL 中反复产生相似失败，生成无信息量的 rollout 轨迹。PATS 提出以策略为中心（而非以技能为中心）的训练范式，将可复用技能作为自适应训练时支持，随策略演化而调整，有效扩展探索空间（[arXiv:2607.21419](https://arxiv.org/abs/2607.21419)）。

4. **7 月 24 日** — Agentic Context Management：将上下文管理重新定义为架构问题。生产环境 Agent 失败更多源于无法管理推理上下文（对话历史、大 prompt、工具输出膨胀），而非推理能力不足。该工作将上下文管理从"存储-检索"问题重新框定为生命周期与架构问题，提出跨会话结构化记忆与 token 成本优化方案（[arXiv:2607.21503](https://arxiv.org/abs/2607.21503)）。

5. **7 月 24 日** — Euclid-MCP：通过 MCP 协议为 LLM Agent 提供确定性逻辑推理。LLM 在多步逻辑推理上不可靠，现有神经符号方法集成多为定制方案缺乏标准化接口。Euclid-MCP 将 Prolog 确定性逻辑引擎封装为标准化 MCP Server，使 Agent 可通过工具调用接口获得形式化推理能力（[arXiv:2607.21412](https://arxiv.org/abs/2607.21412)）。

6. **7 月 25 日** — SGLang v0.5.16：DSPARK 推测解码与 Inkling 975B MoE Day-0 支持。DSPARK 通过置信度驱动的块半自回归起草与自适应验证窗口，在 DeepSeek-V4-Pro TP8 B300 上达到 383.7 tok/s（accept length ~5）；Inkling 975B 多模态 MoE 支持 1M-token 上下文，混合滑动窗口、全注意力与 Mamba2 线性注意力，Blackwell 上 71.7k tok/s 输入、171.0 tok/s per-user decode；UnifiedRadixTree 成为 SWA/Mamba/DSA 模型默认（[GitHub Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.16)）。

7. **7 月 14 日** — vLLM v0.25.1：关键 bug 修复。修复 TorchCodec 在无系统 FFmpeg 时阻塞启动的问题（deferred to runtime），以及混合精度 allreduce+RMSNorm+量化融合在 BF16 残差流与 FP32 RMSNorm 权重不匹配时产生垃圾输出的 bug（[GitHub Release](https://github.com/vllm-project/vllm/releases/tag/v0.25.1)）。

8. **7 月 24 日** — llama.cpp b10107 持续迭代。nightly 构建保持高频更新节奏，每日产出新版本，CPU 推理与 GGUF 格式持续优化（[GitHub](https://github.com/ggerganov/llama.cpp)）。

---

## 💡 深度解读

### 1️⃣ OpenForgeRL 与 PATS：Agent 训练范式的双重突破

**问题背景：**
当前最先进的 AI Agent（如 Claude Code、Codex）依赖复杂的推理 harness 来驱动多轮推理、工具调用与外部系统访问。这些 harness 功能强大但使得端到端训练极为困难——现有开源 SFT/RL 栈无法原生表达有状态、多进程的 harness 推理。与此同时，在长时域 Agent RL 中，弱策略反复产生相似失败的无信息量 rollout，限制有效策略优化。

**核心思路/原理：**
OpenForgeRL 通过统一的训练-推理 harness 抽象，将 Agent 的执行环境（工具、外部系统、多轮对话状态）纳入训练循环，使策略梯度可以直接穿过整个 Agent 执行轨迹。PATS 则从策略演化的角度重新设计训练支撑——不同于技能中心方法（优化、过滤或内化可复用技能），PATS 将技能作为自适应的训练时脚手架，随策略能力提升而逐步移除或调整。

**数据与证据：**
- OpenForgeRL 在多种环境（代码生成、Web 交互、工具调用）中展示了端到端训练的可行性
- PATS 在长 horizon Agent 任务中显著减少无效 rollout 比例，提升策略优化效率
- 两者互补：OpenForgeRL 解决"如何训练"，PATS 解决"如何高效训练"

来源：
- [OpenForgeRL: arXiv:2607.21557](https://arxiv.org/abs/2607.21557)
- [PATS: arXiv:2607.21419](https://arxiv.org/abs/2607.21419)

**工程启示：**
1. Agent 训练正在从"先推理后微调"向"端到端 harness-aware 训练"演进，未来 Agent 的能力上限将更多由训练基础设施决定
2. 对于已有 Agent 系统，PATS 的自适应脚手架思路可应用于 RLHF/DPO 训练——在训练初期提供更多引导，随策略成熟逐步撤除
3. 开源 Agent 训练框架的成熟降低了端到端训练的门槛，但在实际部署中仍需关注 harness 抽象的通用性与特定领域的适配成本

---

### 2️⃣ AREX 与 Agentic Context Management：Agent 效率的两个维度

**问题背景：**
Deep Research 类任务要求 Agent 找到同时满足多个约束的答案。发现这样的答案成本高昂，但验证候选答案通常可分解为可处理的逐项检查。与此同时，生产环境 Agent 的失败更多源于无法管理推理上下文——对话历史、大 prompt、工具输出膨胀、token 成本每轮增长——而非推理能力不足。

**核心思路/原理：**
AREX 利用发现-验证不对称性设计递归自我改进循环：不仅延长搜索时间，而是通过验证中间结果并利用部分验证状态引导后续精炼。每一步改进都建立在前一步的部分验证结果之上，形成累积的知识链。Agentic Context Management 则从架构层面解决 Agent 的记忆与成本问题：将上下文视为具有生命周期（创建、活跃、归档、检索）的结构化对象，而非简单的 token 流。

**数据与证据：**
- AREX 在 Deep Research 基准上通过递归精炼显著减少达到目标质量所需的总搜索成本
- Agentic Context Management 方案在跨会话记忆召回与单会话 token 成本优化上均有显著改善

来源：
- [AREX: arXiv:2607.21461](https://arxiv.org/abs/2607.21461)
- [Agentic Context Management: arXiv:2607.21503](https://arxiv.org/abs/2607.21503)

**工程启示：**
1. AREX 的发现-验证不对称性思路可推广到任何需要多约束满足的任务——将"搜索"与"验证"解耦，用验证结果引导搜索方向
2. 上下文管理是 Agent 工程化落地的核心瓶颈之一——结构化生命周期管理优于简单的 RAG 检索增强
3. 两者结合：AREX 的部分验证状态可作为上下文管理的优先级信号，高置信度验证结果优先保留，低置信度结果归档或丢弃

---

### 3️⃣ Euclid-MCP 与 MIRROR：推理能力边界的拓展

**问题背景：**
LLM 在多步逻辑推理上不可靠，尤其在安全关键或合规敏感领域。现有神经符号方法将神经模型与外部符号引擎耦合，但集成多为定制方案，缺乏标准化接口。同时，视觉语言模型在多模态推理中存在视图不一致性——同一问题的文本、图表、图文组合视图可能激发不同行为。

**核心思路/原理：**
Euclid-MCP 将 Prolog 确定性逻辑引擎封装为标准化的 Model Context Protocol Server，使 Agent 可通过统一工具调用接口获得形式化推理能力。MIRROR 则从多视图学习的角度解决 VLM 推理不一致性——通过让模型从不同视图（文本、视觉、组合）中学习互补的推理模式，提升跨视图推理鲁棒性。

**数据与证据：**
- Euclid-MCP 在需要严格逻辑推理的任务上显著优于纯 LLM 方案
- MIRROR 在几何推理等 admits 等价多视图的任务上，减少视图间行为不一致性

来源：
- [Euclid-MCP: arXiv:2607.21412](https://arxiv.org/abs/2607.21412)
- [MIRROR: arXiv:2607.21552](https://arxiv.org/abs/2607.21552)

**工程启示：**
1. MCP 协议正在成为 Agent 工具调用的事实标准——Euclid-MCP 表明该协议可覆盖确定性逻辑推理，未来可能有更多专业推理引擎以 MCP Server 形式接入
2. 多视图一致性是 VLM 可靠部署的关键——在医疗、法律等需要严格推理的领域，单一视图的失败模式可能不同，需多视图交叉验证
3. 对于 Agent 系统架构师，MCP 接口提供了将专业推理能力模块化插入的标准化路径

---

## 🔧 开源工具动态

1. **SGLang v0.5.16（7 月 25 日）** — 本周期最重大的框架更新。DSPARK 置信度驱动推测解码通过块半自回归起草与自适应验证窗口，在 DeepSeek-V4-Pro TP8 B300 上达到 383.7 tok/s（accept length ~5）；Inkling 975B 多模态 MoE Day-0 支持（1M-token 上下文，混合注意力架构，Blackwell 上 71.7k tok/s 输入、171.0 tok/s decode）；UnifiedRadixTree 成为 SWA/Mamba/DSA 模型默认；新增 LongCat 2.0 FP8、Mellum v2、Pi0.5 等模型。574 PRs from 169 contributors，生态活跃度极高。生产环境建议：DSPARK 对长序列生成任务有显著加速效果，建议评估是否适用于现有部署。

2. **vLLM v0.25.1（7 月 14 日）** — 重要 bug 修复版本。修复 TorchCodec 在无系统 FFmpeg 时阻塞模型启动的问题（错误延迟到运行时而非导入时）；修复混合精度 allreduce+RMSNorm+量化融合在 BF16 残差流与 FP32 RMSNorm 权重（如 NVFP4 模型中的 Gemma/Qwen）不匹配时产生垃圾输出（重复 `!!!!!` token）的严重 bug。生产环境建议：如部署 NVFP4 模型，强烈建议升级至 v0.25.1。

3. **TensorRT-LLM v1.3.0rc22（7 月 22 日）** — 持续 RC 迭代。NVIDIA 官方推理优化框架保持每 1-2 周发布 RC 版本的节奏，预计 v1.3.0 正式版将在 RC 稳定后发布。关注 FP8/NVFP4 量化、Blackwell 架构优化等方向。

4. **llama.cpp b10107（7 月 24 日）** — nightly 构建保持日更节奏。CPU 推理持续优化，GGUF 格式生态成熟。近期版本主要改进包括 CUDA NVFP4 W4A4 激活量化、hexagon 算子微优化、推测类型自动推断等。对于端侧部署场景，llama.cpp 仍是 CPU 推理的首选方案。

5. **MLC LLM** — 最近更新较少，最新 release 仍为早期版本。MLC LLM 在端侧部署（iOS/Android/WebGPU）领域仍有独特价值，但社区活跃度相对低于 llama.cpp。对于移动端部署场景，仍建议关注但需评估维护节奏。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 25 日*
