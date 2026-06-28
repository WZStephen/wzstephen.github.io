---
layout: post
title: 'OpenAI Jalapeño 推理专用芯片发布、Codex Agent 采用曲线深度数据、Meta TLX Block Attention Blackwell 块稀疏注意力 Kernel'
date: 2026-06-28 09:00:00 +0800
categories: [ai-technical-report]
---

> 过去 48 小时 AI 工程领域三条主线：**OpenAI 与 Broadcom 联合发布 Jalapeño——首个从零设计的 LLM 推理专用 Intelligence Processor（6 月 24 日）——工程样品已在实验室以生产目标频率和功耗运行 GPT-5.3-Codex-Spark，每瓦性能大幅优于当前 state-of-the-art，架构围绕 kernel、内存移动、网络和服务模式设计，计划与 Microsoft 等数据中心合作伙伴在 GW 级规模部署**；**OpenAI 发布"How agents are transforming work"白皮书（6 月 25 日）——首次系统性披露 Codex 内部采用数据：80.6% 个人用户发起超 30 分钟人类工作量的请求，70.2% 超 1 小时，25.6% 超 8 小时；Codex 占 OpenAI 内部 99.8% 的输出 Token；非开发者采用增速 137x；P99 用户日均发起 60+ 小时 Agent 工作**；**Meta Ads AI 开源 TLX Block Attention（PyTorch 博客 5 月 26 日发布，近期开源代码库）——基于 TLX（Triton Language Extensions）的 Blackwell SM100 warp-specialized 块稀疏注意力 kernel，利用编译时块对角注意力模式消除 online softmax correction 和 logsumexp 存储，B200 上前向加速 ~1.85x、反向加速 ~2.50x，融合 rotary embeddings 后反向联合加速 ~3.5x**。下面逐一拆解。

---

## 🔥 今日看点

1. **6 月 24 日** — OpenAI 与 Broadcom 发布 Jalapeño，首个从零设计的 LLM 推理专用 Intelligence Processor。工程样品已在实验室以生产目标频率和功耗运行 ML 工作负载（包括 GPT-5.3-Codex-Spark）。架构减少数据移动，平衡计算、内存和网络资源，使实际利用率接近理论峰值。Broadcom 提供 Tomahawk 网络芯片，Celestica 负责机架系统集成。计划多代路线图，GW 级规模部署（[OpenAI](https://openai.com/index/openai-broadcom-jalapeno-inference-chip/)）
2. **6 月 25 日** — OpenAI 发布"How agents are transforming work"。首次披露 Codex 内部采用曲线：80.6% 个人用户发起超 30 分钟人类工作量的请求，70.2% 超 1 小时，25.6% 超 8 小时。Codex 占 OpenAI 内部 99.8% 周输出 Token。非开发者采用增长 137x（个人）、189x（组织）、12x（内部）。P99 用户日均 60+ 小时 Agent 工作。Legal、Finance、Recruiting 在 2026 年 4 月转向 Codex 为主要 AI 工具（[OpenAI](https://openai.com/index/how-agents-are-transforming-work/)）
3. **5 月 26 日** — PyTorch 博客发布 Meta Ads AI 的 TLX Block Attention——Blackwell SM100 上的 warp-specialized 块稀疏注意力 kernel。核心 insight：当注意力模式在编译时已知（块对角，固定 64-token 块），可以消除 FlashAttention 的 online softmax correction、logsumexp 存储和多 tile 迭代。B200 上前向 ~1.85x、反向 ~2.50x 加速（vs FlashAttention v2），融合 rotary 后反向联合 ~3.5x。代码已在 GitHub 开源（[PyTorch Blog](https://pytorch.org/blog/tlx-block-attention-a-warp-specialized-blackwell-kernel-for-fixed-block-sparse-self-attention/)，[GitHub](https://github.com/facebookresearch/ads_model_kernel_library)）
4. **6 月 15 日** — vLLM v0.23.0 发布。408 commits，200 贡献者。核心更新：Multi-tier KV cache offloading（object-store 二级存储 + HMA 默认启用）；Model Runner V2 扩展到 Llama/Mistral 稠密模型；Rust frontend 新增 streaming generate、动态 LoRA endpoints；DeepSeek-V4 全面硬化（TRTLLM-gen attention、EPLB Mega-MoE、选择性 prefix-cache）；Gemma 4 Unified encoder-free 支持；Transformers v5 兼容（[GitHub: vLLM v0.23.0](https://github.com/vllm-project/vllm/releases)）
5. **6 月 26 日** — Hugging Face 发布"Run a vLLM Server on HF Jobs in One Command"——一行命令在 HF Jobs 上启动 vLLM 推理服务器，降低开源模型部署门槛（[Hugging Face Blog](https://huggingface.co/blog/vllm-jobs)）
6. **6 月 25 日** — Hugging Face 发布"Which tokens does a hybrid model predict better?"——研究混合架构（Mamba + Attention）在不同 Token 类型上的预测质量差异，为混合模型设计提供实证依据（[Hugging Face Blog](https://huggingface.co/blog/allenai/hybrid-token-prediction)）
7. **6 月 24 日** — Hugging Face 发布"Accelerating Transformers Fine-Tuning with NVIDIA NeMo AutoModel"——使用 NVIDIA NeMo AutoModel 加速 Transformer 微调流程（[Hugging Face Blog](https://huggingface.co/blog/nvidia/accelerating-fine-tuning-nvidia-nemo-automodel)）
8. **6 月 22 日** — OpenAI 发布"Codex-maxxing for Long-Running Work"白皮书——Jason Liu 分享将 Codex 作为持久工作空间的实践策略：如何将长任务分解为可验证步骤、跨工作流保持上下文连续性、何时委托执行 vs 何时人类监督（[OpenAI](https://openai.com/index/codex-maxxing-long-running-work/)）
9. **6 月 22 日** — OpenAI 发布 Daybreak 安全工具和"Patch the Planet"开源维护者支持计划——面向全球组织的网络安全工具，以及支持开源维护者的新倡议（[OpenAI](https://openai.com/index/daybreak-securing-the-world/)）
10. **6 月 24 日** — OpenAI 与 Broadcom 发布 Jalapeño 的同一天，Greg Brockman 阐述了全栈推理基础设施的飞轮逻辑：更好的基础设施 → 更高计算效率 → 更好训练和服务 → 更强模型 → 更好产品 → 更多用户和收入 → 更多基础设施投资。Jalapeño 是这个飞轮的物理层延伸（[OpenAI](https://openai.com/index/openai-broadcom-jalapeno-inference-chip/)）

---

## 💡 深度解读

### 1️⃣ OpenAI Jalapeño：从零设计的 LLM 推理专用芯片意味着什么

**问题背景：**
当前 LLM 推理主要依赖 NVIDIA 通用 GPU（H100/B200/GB300）。GPU 的架构设计初衷是图形渲染和通用并行计算，并非专门为 LLM 推理的 memory-bound 特性优化。LLM 推理的核心瓶颈是 KV cache 的内存带宽和 MoE 架构的 expert 路由效率——这些在通用 GPU 上存在大量计算单元闲置。OpenAI 选择与 Broadcom 合作从零设计推理专用芯片，是对这一结构性问题的长期回应。

**核心思路/原理：**
Jalapeño 的关键设计哲学是"blank-slate design for modern LLM inference"——不是把旧架构适配推理，而是围绕推理工作负载的物理约束设计架构：

- **减少数据移动**：LLM 推理中，数据在 SRAM、HBM 和芯片间移动的能耗远超计算本身。Jalapeño 的架构重新平衡了计算、内存和网络资源的比例，使实际利用率更接近理论峰值——这意味着更少的"空转"周期
- **围绕 kernel 和 serving 模式设计**：OpenAI 的独特优势在于它同时运营 ChatGPT、Codex、API 和未来的 Agent 产品，对推理工作负载有第一手数据。Jalapeño 的架构是围绕这些真实 serving 模式（continuous batching、prefix caching、speculative decoding、MoE routing）设计的，而不是围绕 benchmark 数字
- **全栈飞轮**：模型 → kernel → serving 系统 → 芯片架构的垂直整合，使每一层都能针对同一目标优化。这与 NVIDIA 的"通用 GPU + CUDA 生态"路线形成对比——OpenAI 选择了类似 Google TPU 的"自研芯片 + 自用模型"路径，但更激进地针对 LLM 推理优化

**数据与证据：**
- 工程样品已在实验室以生产目标频率和功耗运行 GPT-5.3-Codex-Spark
- 早期测试显示每瓦性能"substantially better than current state-of-the-art"
- 详细性能报告将在未来几个月发布
- Broadcom 提供 Tomahawk 网络芯片，Celestica 负责机架集成
- 计划从 2026 年开始与 Microsoft 等合作伙伴在 GW 级数据中心部署

来源：
- [OpenAI: OpenAI and Broadcom unveil LLM-optimized inference chip](https://openai.com/index/openai-broadcom-jalapeno-inference-chip/)

**工程启示：**
1. **推理专用芯片是 MaaS 成本结构长期变化的信号**——如果 Jalapeño 的每瓦性能确实大幅优于通用 GPU，那么使用专用芯片的 MaaS 提供商将在成本上获得结构性优势。对推理工程师来说，这意味着未来选择推理平台时，"是否使用专用芯片"将成为重要的成本维度
2. **全栈垂直整合正在成为头部 AI 公司的标配**——OpenAI（Jalapeño）、Google（TPU）、Meta（MTIA）、Amazon（Trainium/Inferentia）都在做自研芯片。这对 NVIDIA 的长期垄断地位构成挑战，但短期内 CUDA 生态的护城河仍然深厚。对 MaaS 工程师来说，关注多后端兼容（vLLM/SGLang 的硬件抽象层）是降低锁定风险的务实策略
3. **GW 级部署规模说明推理需求的增速远超预期**——Jalapeño 计划"GW 级规模部署"，这意味着数百亿瓦级的数据中心专门为推理服务。这个数字在两年前是不可想象的

---

### 2️⃣ Codex Agent 采用曲线：从数据看 Agent 工程的真实拐点

**问题背景：**
"Agent 是未来"是 2025-2026 年 AI 工程领域最常见的论断之一。但大多数关于 Agent 采用的讨论都缺乏硬数据——有多少用户真的在用 Agent？用在什么场景？任务复杂度如何？OpenAI 的"How agents are transforming work"首次提供了来自生产环境的系统性采用数据，对 MaaS/Agent 工程师来说这是难得的基准参考。

**核心思路/原理：**
OpenAI 披露的数据揭示了四个关键趋势：

- **任务复杂度持续上移**：从短交互到长时自主任务。80.6% 的用户发起超 30 分钟的请求，70.2% 超 1 小时，25.6% 超 8 小时。这意味着 Agent 不再是"问答机器人"，而是"长时工作委托系统"
- **P99 用户日均 60+ 小时 Agent 工作**：这是最令人震惊的数字。最重度的用户不是在"使用 AI"，而是在"编排 AI 工作流"——同时运行多个并行 Agent，分配不同子任务
- **非开发者采用增速超过开发者**：非开发者用户增长 137x（个人）、189x（组织），而开发者增长相对温和。Legal、Finance、Recruiting 在 2026 年 4 月转向 Codex 为主要工具。这说明 Agent 的价值不仅在代码生成，更在"技术执行的民主化"——非技术人员通过 Agent 获得编程和数据转换能力
- **Codex 占 OpenAI 内部 99.8% 输出 Token**：这几乎意味着 ChatGPT 作为"主要 AI 工具"在 OpenAI 内部已经被 Codex 取代

**数据与证据：**
- 80.6% 个人用户发起超 30 分钟人类工作量的请求
- 70.2% 超 1 小时，25.6% 超 8 小时
- Codex 占 OpenAI 内部 99.8% 周输出 Token
- 非开发者采用增长：个人 137x、组织 189x、内部 12x
- P99 用户日均 60+ 小时 Agent 工作（截至 2026 年 6 月）
- Legal、Finance、Recruiting 在 2026 年 4 月转向 Codex 为主要 AI 工具
- Research 部门 Token 使用中位数增长 56x（vs 2025 年 11 月），Customer Support 32x，Engineering 27x

来源：
- [OpenAI: How agents are transforming work](https://openai.com/index/how-agents-are-transforming-work/)
- [OpenAI: Codex-maxxing for Long-Running Work](https://openai.com/index/codex-maxxing-long-running-work/)

**工程启示：**
1. **Agent 基础设施的核心挑战从"能不能用"转向"能不能可靠地长时间运行"**——当 P99 用户日均发起 60+ 小时 Agent 工作时，系统的上下文管理、错误恢复、成本控制变得至关重要。这对 MaaS 平台来说意味着：长时 Agent 任务的可靠性（而非单次推理速度）将成为核心竞争力
2. **非开发者采用增速超过开发者——这改变了 Agent 产品的设计方向**——如果 Agent 的主要用户不是工程师，那么 Agent 的交互设计、错误处理、可解释性需要大幅简化。对 Agent 框架开发者来说，"面向非技术用户的 Agent 产品"是一个巨大的未开发市场
3. **"委托式工作"模式正在替代"对话式 AI"模式**——Codex 占 99.8% 输出 Token 意味着 OpenAI 内部已经从"问 AI 一个问题"转向"委托 AI 一个任务"。对 MaaS 工程师来说，这意味着推理平台的 API 设计需要从"单轮请求-响应"转向"长时任务委托-进度回报-结果交付"

---

### 3️⃣ Meta TLX Block Attention：编译时知识消除 runtime 开销的 kernel 设计方法论

**问题背景：**
FlashAttention v2 是当前 LLM 推理和训练中最广泛使用的注意力 kernel。它通过 online softmax correction 和 logsumexp 存储来处理任意长度的因果注意力。但当注意力模式在编译时已知（如块对角模式），这些机制变成纯开销。Meta Ads AI 的 TLX Block Attention 展示了如何利用编译时知识来消除这些开销。

**核心思路/原理：**
核心 insight 极其优雅：**当每个 Q tile 只 attend 到一个对应的 K/V tile 时（块对角模式），FlashAttention 的整个复杂性框架变成冗余。**

具体简化级联：
- **无多 tile 迭代**：S = Q · Kᵀ ∈ ℝ^{64×64} 在一次 GEMM 后完成，无需循环维护状态
- **无 online softmax correction**：单 tile 的行最大值和求和立即全局正确，correction factor α = exp(m_old - m_new) 恒等于 1
- **无 logsumexp 存储**：FlashAttention 将 per-row logsumexp L 存到 HBM 供反向使用；单 tile 下 L 可以直接从 S 重新计算，无需存储
- **融合 rotary embeddings**：将 rotary 计算融入注意力 epilogue，消除独立 kernel launch

技术实现基于 TLX（Triton Language Extensions）——Triton 编译器的底层扩展，提供对 warp specialization、异步 tensor core 操作和 Blackwell 内存层次结构的硬件级控制。这弥合了 Triton 的高生产力和 CUDA/CUTLASS 的细粒度硬件控制之间的差距。

**数据与证据：**
- NVIDIA B200 上：前向加速 ~1.85x，反向加速 ~2.50x（vs FlashAttention v2）
- 融合 rotary embeddings 后：反向联合加速 ~3.5x
- 生产工作负载：batch size 1152，序列长度 ~4k tokens，head dim 64/128，~70% 稀疏度
- 代码开源：[github.com/facebookresearch/ads_model_kernel_library](https://github.com/facebookresearch/ads_model_kernel_library)

来源：
- [PyTorch Blog: TLX Block Attention](https://pytorch.org/blog/tlx-block-attention-a-warp-specialized-blackwell-kernel-for-fixed-block-sparse-self-attention/)
- [GitHub: ads_model_kernel_library](https://github.com/facebookresearch/ads_model_kernel_library)

**工程启示：**
1. **"编译时知识消除 runtime 开销"是一个可推广的 kernel 设计方法论**——不仅限于块对角注意力。任何已知注意力模式（滑动窗口、稀疏模式、prefix caching 场景）都可以用同样的思路优化。对 kernel 工程师来说，问"我的场景中哪些 runtime 机制在编译时可以消除？"是一个高价值的优化方向
2. **TLX 是 Blackwell 上 Triton 生态的重要演进**——它提供了接近 CUDA 的硬件控制但保持 Triton 的开发效率。对做 Blackwell 优化的团队来说，TLX 值得纳入技术栈评估
3. **Meta 开源 Ads/Recsys kernel 库是一个值得关注的信号**——这说明头部公司的推荐系统已经在用高度定制化的 GPU kernel，而不是依赖通用 attention 实现。对做推荐系统推理的团队来说，这个库可以直接复用

---

## 🔧 开源工具动态

1. **vLLM v0.23.0（6 月 15 日）** — 408 commits，200 贡献者。核心更新：Multi-tier KV cache offloading（object-store 二级存储 + HMA 默认启用 + per-request offloading 策略）；Model Runner V2 扩展到 Llama/Mistral 稠密模型（FlashInfer sampler、breakable CUDA graphs、pipeline-parallel bubble 消除）；Rust frontend 新增 streaming generate、动态 LoRA endpoints、server-router hook；DeepSeek-V4 全面硬化（TRTLLM-gen attention、EPLB Mega-MoE、选择性 prefix-cache retention）；Gemma 4 Unified encoder-free + MTP；Transformers v5 兼容。新模型：Step-3.7-Flash、Cosmos3 Reasoner、JetBrains Mellum v2、Granite Speech Plus、Cohere Mini Code（[GitHub](https://github.com/vllm-project/vllm/releases)）

2. **SGLang v0.5.14 Speculative Decoding 更新（6 月 26 日）** — 在 v0.5.14 基础上新增 EAGLE draft-extend 的 sync-free fast_prefill_plan 和 FlashInfer CUDA graph 支持；MTP rejection sampling；GLM-4.7-Flash MTP 支持（NPU）；sliding window attention draft layer。这些更新进一步增强了 SGLang 的 speculative decoding 能力（[GitHub](https://github.com/sgl-project/sglang/releases)）

3. **Hugging Face: vLLM on HF Jobs（6 月 26 日）** — 一行命令在 HF Jobs 上启动 vLLM 推理服务器。降低开源模型部署门槛，适合快速原型验证和小规模部署（[Hugging Face Blog](https://huggingface.co/blog/vllm-jobs)）

4. **Meta Ads Model Kernel Library（开源）** — Meta Ads AI 开源的高性能 GPU kernel 库，包含 GDPA（Generalized Dot Product Attention）、TLX Block Attention、TLX Multi-CTA Norm Fusion、TLX GDPA Megakernel。目标 Blackwell SM100，Apache 2.0 许可（[GitHub](https://github.com/facebookresearch/ads_model_kernel_library)）

---

## 结语

今天三条主线交汇出一个清晰趋势：**AI 推理正在从"通用 GPU 上的通用框架"走向"全栈垂直优化"**。OpenAI Jalapeño 代表芯片层的专用化，Meta TLX Block Attention 代表 kernel 层对特定注意力模式的编译时优化，而 Codex 采用数据则证明推理需求的增长速度已经超出了"通用方案"能承载的范围——P99 用户日均 60+ 小时 Agent 工作需要的是可靠、高效、低成本的长时推理基础设施，而不是更快的单次推理。对 MaaS 工程师来说，关注硬件多样性适配（vLLM/SGLang 的多后端支持）、kernel 层的编译时优化机会、以及 Agent 长时任务的可靠性工程，是接下来最有价值的三个方向。
