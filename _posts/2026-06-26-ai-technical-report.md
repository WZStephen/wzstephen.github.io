---
layout: post
title: 'OpenAI 发布 Codex Agent 使用数据：99.8% 输出 Token 来自 Agent、vLLM v0.23 Multi-Tier KV Cache Offloading、SGLang Spec V2 统一推测解码'
date: 2026-06-26 09:00:00 +0800
categories: [ai-technical-report]
---

> 过去 48 小时 AI 工程领域三条主线：**OpenAI 发布内部 Codex 使用数据报告（6 月 25 日）——80.6% 的 sampled 用户发起过预估超过 30 分钟人类工作量的请求，70.2% 超过 1 小时，25.6% 超过 8 小时；OpenAI 内部 99.8% 的周输出 Token 由 Codex 生成；非开发者用户增长 137x——这是迄今为止最详细的 Agent 工作负载生产数据，标志着 AI 工具从"聊天机器人"向"长时自主 Agent"的范式转变**；**vLLM v0.23.0 发布（6 月 15 日）——Multi-Tier KV Cache Offloading 引入对象存储二级缓存层、per-request offloading 策略、HMA 默认启用；Model Runner V2 扩展至 Llama/Mistral；Rust 前端增加流式 generate 和动态 LoRA 端点；推测解码增加 Causal DFlash 和独立 drafter attention backend 选择**；**SGLang v0.5.13 发布（6 月 13 日）——Spec V2 成为默认推测解码路径，tree drafting 支持 topk > 1；HiCache 默认支持混合模型（SWA/Mamba）；Intel CPU + GPU EPD 异构分离式推理实现 ~1.3x P99 TTFT 提升；AMD MI355X 上通过 MoRI 实现 $0.169/M tokens 的 DeepSeek-R1 分离式推理**。下面逐一拆解。

---

## 🔥 今日看点

1. **6 月 25 日** — OpenAI 发布"How agents are transforming work"报告，首次公开 Codex 内部使用数据。80.6% 的 sampled 个人用户发起过预估 >30 分钟人类工作量的请求，70.2% >1 小时，25.6% >8 小时。OpenAI 内部 99.8% 的周输出 Token 由 Codex 生成。非开发者用户增长：个人 137x、组织 189x、内部 12x。Legal、Finance、Recruiting 在 2026 年 4 月左右将 Codex 作为主要 AI 工具。P99 用户每日生成 >60 小时 agent turns（[OpenAI](https://openai.com/index/how-agents-are-transforming-work/)）
2. **6 月 15 日** — vLLM v0.23.0 发布：408 commits、200 contributors。核心更新：Multi-Tier KV Cache Offloading（对象存储二级缓存层 + per-request offloading 策略）；Model Runner V2 默认支持 Llama/Mistral；Rust 前端增加流式 generate、动态 LoRA、server-router 扩展钩子；推测解码增加 Causal DFlash、独立 drafter attention backend 选择；Gemma 4 encoder-free Unified 支持；Transformers v5 兼容（[GitHub: vLLM v0.23.0](https://github.com/vllm-project/vllm/releases)）
3. **6 月 13 日** — SGLang v0.5.13 发布：Spec V2 成为默认推测解码路径（tree drafting topk > 1，支持 triton/FA3/MLA/aiter 后端 + page_size > 1 + Mamba/hybrid-linear 模型）；Spec V1 废弃，EAGLE/MTP 统一到 V2 worker；HiCache 默认支持混合模型（SWA/Mamba）通过 UnifiedTree；Intel CPU + GPU EPD 异构分离式推理（~1.3x P99 TTFT 提升）；AMD MI355X 上 MoRI 实现 DeepSeek-R1 分离式推理 $0.169/M tokens（[GitHub: SGLang v0.5.13](https://github.com/sgl-project/sglang/releases)）
4. **6 月 26 日** — Hugging Face 发布博客"Run a vLLM Server on HF Jobs in One Command"——一行命令在 HF Jobs 上启动 vLLM 推理服务器，降低开源模型部署门槛（[Hugging Face Blog](https://huggingface.co/blog)）
5. **6 月 25 日** — Hugging Face 发布"Which tokens does a hybrid model predict better?"——研究混合架构（Mamba + Attention）在不同 token 类型上的预测质量差异（[Hugging Face Blog](https://huggingface.co/blog)）
6. **6 月 24 日** — Hugging Face 发布"Accelerating Transformers Fine-Tuning with NVIDIA NeMo AutoModel"——使用 NVIDIA NeMo AutoModel 加速 Transformer 微调（[Hugging Face Blog](https://huggingface.co/blog)）
7. **6 月 23 日** — Hugging Face 发布"Build real agentic apps using CUGA"——基于 CUGA 框架的轻量级 agent harness，提供 24 个可运行示例，展示如何用开源模型构建真实 agent 应用（[Hugging Face Blog](https://huggingface.co/blog)）
8. **6 月 22 日** — Hugging Face 发布"PP-OCRv6 on Hugging Face"——50 语言 OCR 模型，参数量从 1.5M 到 34.5M（[Hugging Face Blog](https://huggingface.co/blog)）
9. **6 月 21 日** — Samsung Electronics 宣布将 ChatGPT 和 Codex 引入员工工作流程（[OpenAI](https://openai.com/news/)）
10. **6 月 17 日** — Hugging Face 发布"GLM-5.2: Built for Long-Horizon Tasks"——GLM-5.2 针对长时任务优化（[Hugging Face Blog](https://huggingface.co/blog)）

---

## 💡 深度解读

### 1️⃣ OpenAI Codex Agent 数据：从"聊天机器人"到"长时自主 Agent"的范式转变

**问题背景：**
Agent 是 2025-2026 年 AI 工程领域最热的关键词之一，但生产环境中的 Agent 工作负载数据一直极度匮乏。大多数团队对 Agent 的认知停留在 demo 阶段——"能自主完成多步骤任务"。但真实的工程生产力场景需要 Agent 运行数小时甚至数天，处理跨上下文的复杂工作流。OpenAI 这份报告是迄今为止最详细的 Agent 生产数据。

**核心思路/原理：**
OpenAI 报告的关键 insight 是 **Agent 改变了知识工作的最小单元**——从"单次交互"变成"委托式长时任务"。具体数据：

- **任务时长分布**：80.6% 用户发起 >30 分钟任务，70.2% >1 小时，25.6% >8 小时。这意味着 Agent 不再是"快速问答"工具，而是"小时级自主工作"环境
- **内部采用曲线**：OpenAI 内部 99.8% 的周输出 Token 由 Codex 生成。工程师平均 99% 输出 Token 使用 Codex；Legal/Finance/Recruiting 在 4 月左右跨越到 Codex 为主要工具，平均 >85% 输出 Token
- **非开发者增长**：非开发者用户增长 137x（个人）、189x（组织）、12x（内部）。这是 Agent 从"开发者工具"向"全职能工具"扩散的最直接证据
- **P99 用户每日 >60 小时 agent turns**：分布在多个并行 Agent 上。这说明重度用户已经从"一次问一个问题"转变为"编排多个并行 Agent"

来源：
- [OpenAI: How agents are transforming work](https://openai.com/index/how-agents-are-transforming-work/)

**工程启示：**
1. **Agent 基础设施的核心需求是"持久性"和"并行性"**——P99 用户每日 >60 小时 agent turns 意味着 Agent 基础设施必须支持：长时间上下文保持、多 Agent 并行编排、跨天/跨周的任务恢复。对 MaaS 工程师来说，评估 Agent 平台时需要关注：上下文窗口管理、checkpoint/resume 能力、并发 Agent 调度
2. **非开发者 Agent 采用速度超过开发者**——137x 的非开发者增长说明 Agent 的价值不仅在代码生成，更在"让非技术人员执行技术任务"。对 MaaS 工程师来说，Agent 产品的 UI/UX 设计需要考虑非技术用户——他们不需要 IDE 集成，而是需要"任务委托 → 结果审查"的简化流程
3. **"99.8% 输出 Token 来自 Codex"暗示 ChatGPT 在 OpenAI 内部已被 Agent 工具取代**——这对 MaaS 工程师的信号是：如果你的团队还在以 ChatGPT 式聊天为主要 AI 交互模式，你可能已经落后于前沿实践。Agent 工具（Codex、Claude Code 等）正在成为生产力的主要来源

---

### 2️⃣ vLLM v0.23：Multi-Tier KV Cache Offloading 与推测解码的工程化成熟

**问题背景：**
KV cache 管理是 LLM 推理服务的核心瓶颈之一。随着模型规模增长和长上下文需求增加，KV cache 的内存占用经常超过 GPU HBM 容量。之前的 vLLM 版本已经支持基本的 CPU offloading，但缺乏多级缓存层次化和 per-request 策略控制。同时，推测解码（speculative decoding）在 vLLM 中虽然已有支持，但 drafter 和 target 模型的 attention backend 耦合、缺乏对不同架构（Mamba/hybrid）的兼容。

**核心思路/原理：**
v0.23.0 的两个关键工程改进：

**Multi-Tier KV Cache Offloading：**
- 引入对象存储（object-store）作为二级缓存层，形成 GPU HBM → CPU DRAM → 对象存储的三级缓存层次
- HMA（Hierarchical Memory Architecture）在支持的 connector 上默认启用
- 新增 per-request offloading 策略，通过 `on_new_request` 生命周期钩子实现——这意味着不同请求可以根据其上下文长度和优先级采用不同的 offloading 策略
- 支持 tiering for HMA models

**推测解码改进：**
- Causal DFlash（Direct Flash Attention for speculative decoding）
- 独立 drafter attention backend 选择——drafter 模型可以使用与 target 模型不同的 attention backend，这在 drafter 和 target 架构不同时特别有用
- EAGLE/MTP lookahead caching 扩展到 SWA prefix-cache mask

**其他重要更新：**
- Model Runner V2 扩展至 Llama/Mistral dense 模型（此前仅支持 Qwen3），增加 FlashInfer sampler、breakable CUDA graphs、pipeline-parallel bubble elimination
- Rust 前端增加流式 generate 端点、动态 LoRA 端点、server-router 扩展钩子
- Gemma 4 encoder-free Unified 支持和 MTP
- Transformers v5 兼容

来源：
- [GitHub: vLLM v0.23.0 Release Notes](https://github.com/vllm-project/vllm/releases)

**工程启示：**
1. **Multi-Tier KV Cache Offloading 对长上下文推理服务的成本优化至关重要**——三级缓存（GPU → CPU → 对象存储）允许在相同 GPU 资源下服务更长的上下文或更多的并发请求。per-request offloading 策略意味着你可以对短上下文请求不做 offloading（最低延迟），对超长上下文请求使用对象存储 offloading（最高吞吐）。对 MaaS 工程师来说，如果你的服务需要处理 >32K 上下文的请求，这个特性值得立即升级测试
2. **独立 drafter attention backend 选择解耦了推测解码的架构限制**——之前 drafter 和 target 必须使用相同的 attention backend，这限制了 drafter 模型的选择范围。现在你可以为 target 使用 FlashAttention 而 drafter 使用 FlashInfer（或反过来），这在部署异构 speculative decoding 方案时提供了更大灵活性
3. **Rust 前端的成熟（流式 generate + 动态 LoRA + server-router 钩子）意味着 vLLM 正在构建生产级 API 网关能力**——server-router 扩展钩子允许在请求路由层面实现自定义逻辑（如基于负载的路由、A/B 测试、多模型调度）。对 MaaS 工程师来说，这可能减少对外部 API 网关（如 LiteLLM）的依赖

---

### 3️⃣ SGLang v0.5.13：Spec V2 统一推测解码 + 异构 CPU/GPU 分离式推理

**问题背景：**
推测解码是提升 LLM 推理吞吐的关键技术，但之前的实现路径分散（Spec V1、EAGLE、MTP 各自独立），维护和优化成本高。同时，分离式推理（disaggregated inference）——将 prefill 和 decode 阶段放在不同硬件上——已经在理论上被证明可以显著提升 GPU 利用率，但实际部署中面临通信开销和硬件兼容性的挑战。

**核心思路/原理：**
SGLang v0.5.13 的两个核心改进：

**Spec V2 统一推测解码：**
- Tree drafting 支持 topk > 1，在 triton/FA3/MLA/aiter 后端上均已验证生产可用
- 支持 page_size > 1 和 Mamba/hybrid-linear 模型
- Spec V1 废弃，EAGLE/MTP 统一到 V2 worker
- 自适应推测解码：batch-size-aware num_steps + 可观测性指标
- topk = 1 drafting 更快（跳过 full-vocab softmax + 冗余 cat/topk/sort/gather 操作）

**异构 CPU + GPU EPD 分离式推理（与 Intel 合作）：**
- 将 VLM 视觉编码卸载到 Intel Xeon CPU，GPU 专注于 LLM decode
- 实现 ~1.3x P99 TTFT（Time To First Token）提升
- 在负载情况下请求吞吐也有显著改善

**AMD MI355X 上的 MoRI 分离式推理（与 AMD 合作）：**
- 通过 AMD MoRI 通信库实现 DeepSeek-R1 分离式推理
- 成本：$0.169 per million tokens，速度 129 tok/s/user
- 与 NVIDIA 方案形成成本竞争

**其他重要更新：**
- HiCache 默认支持混合模型（SWA/Mamba），通过 UnifiedTree 实现分层 KV cache offloading
- DeepSeek V4 上下文并行 + MTP、sparse FlashMLA 内核、FP4 indexer 支持
- Piecewise & Breakable CUDA Graph 覆盖扩展到 DSA 模型、Kimi-K2.5、DeepSeek V4
- Qwen 3.5 在 Blackwell 上的 FlashInfer GDN 内核加速

来源：
- [GitHub: SGLang v0.5.13 Release Notes](https://github.com/sgl-project/sglang/releases)

**工程启示：**
1. **Spec V2 的统一是推测解码从"实验特性"走向"生产默认"的标志**——之前需要在 Spec V1、EAGLE、MTP 之间手动选择，现在统一为 Spec V2 且默认启用。对 MaaS 工程师来说，升级后可以直接启用 Spec V2 并获得 tree drafting 的吞吐提升，无需手动配置多条路径
2. **异构 CPU/GPU 分离式推理对 VLM 服务特别有价值**——VLM 的视觉编码阶段是计算密集但 GPU 利用率低的环节（大量内存访问、较少矩阵运算）。将视觉编码卸载到 CPU 可以释放 GPU 用于 LLM decode，提升整体吞吐。对 MaaS 工程师来说，如果你的服务涉及多模态（图文）推理，这个特性值得评估
3. **AMD MI355X 上 $0.169/M tokens 的 DeepSeek-R1 推理成本是 NVIDIA 方案的成本基准挑战**——虽然绝对性能可能不如 NVIDIA 顶级方案，但单位成本的优势对成本敏感的 MaaS 部署（如内部推理服务、低延迟要求不高的批处理）有吸引力

---

### 4️⃣ Hugging Face 生态动态：一键 vLLM 部署 + CUGA Agent 框架 + 混合模型研究

**问题背景：**
开源模型部署的门槛不仅在模型性能，更在工程复杂度。即使有 vLLM 这样的优秀推理框架，从零配置部署环境、管理依赖、处理 API 端点仍然需要不少工程投入。同时，Agent 应用开发面临"框架选择"问题——LangChain、AutoGPT、CrewAI 等框架各有优劣，但缺乏轻量级、可审计的 harness 设计。

**核心思路/原理：**
Hugging Face 近期发布的几个重要博客：

**一键 vLLM 部署（6 月 26 日）：**
- "Run a vLLM Server on HF Jobs in One Command"——将 vLLM 部署抽象为单条命令，利用 HF Jobs 基础设施处理 GPU 分配、依赖管理和 API 端点暴露
- 降低了从"模型选择"到"推理服务上线"的工程开销

**CUGA Agent 框架（6 月 23 日）：**
- "Build real agentic apps using CUGA"——提供 24 个可运行示例的轻量级 agent harness
- 强调"real agentic apps"而非 demo——关注工具调用、环境交互、错误恢复等生产级需求
- 开源模型友好的 agent 框架设计

**混合模型研究（6 月 25 日）：**
- "Which tokens does a hybrid model predict better?"——研究 Mamba + Attention 混合架构在不同 token 类型上的预测质量差异
- 对混合架构的部署和优化有指导意义

**GLM-5.2（6 月 17 日）：**
- "Built for Long-Horizon Tasks"——GLM-5.2 针对长时任务优化，与 OpenAI 的 Codex agent 数据形成呼应

来源：
- [Hugging Face Blog](https://huggingface.co/blog)

**工程启示：**
1. **一键部署降低了开源模型的"最后一公里"工程门槛**——对 MaaS 工程师来说，评估新模型的部署成本不再需要从零开始配置 vLLM + API 网关 + 监控。HF Jobs + vLLM 的组合可以快速验证模型在实际工作负载上的表现
2. **CUGA 的"24 个可运行示例"是 agent 工程的最佳实践参考**——即使不使用 CUGA 框架，这些示例也展示了如何用开源模型构建工具调用、环境交互、错误恢复等 agent 能力。对工程团队来说，这是学习 agent 工程模式的低成本入口
3. **混合模型研究的实践意义在于"架构选择指导"**——如果你的工作负载主要是代码生成（高局部性 token），混合架构可能在某些 token 类型上优于纯 Attention 架构。对 MaaS 工程师来说，选择模型架构时应该基于实际工作负载的 token 分布做 benchmark，而不是只看总体 perplexity

---

## 📊 开源工具动态

1. **6 月 15 日** — vLLM v0.23.0 发布：Multi-Tier KV Cache Offloading、Model Runner V2 扩展至 Llama/Mistral、Rust 前端流式 generate + 动态 LoRA、推测解码 Causal DFlash、Gemma 4 Unified 支持、Transformers v5 兼容（[GitHub](https://github.com/vllm-project/vllm/releases)）
2. **6 月 13 日** — SGLang v0.5.13 发布：Spec V2 统一推测解码（tree drafting topk > 1）、HiCache 默认支持混合模型、Intel CPU + GPU EPD 分离式推理 ~1.3x P99 TTFT 提升、AMD MI355X MoRI $0.169/M tokens（[GitHub](https://github.com/sgl-project/sglang/releases)）
3. **6 月 26 日** — Hugging Face 发布"Run a vLLM Server on HF Jobs in One Command"——一行命令部署 vLLM 推理服务（[Hugging Face Blog](https://huggingface.co/blog)）
4. **6 月 23 日** — Hugging Face 发布 CUGA agent 框架，24 个可运行示例构建真实 agent 应用（[Hugging Face Blog](https://huggingface.co/blog)）
5. **6 月 22 日** — PP-OCRv6 发布：50 语言 OCR，参数量 1.5M 到 34.5M（[Hugging Face Blog](https://huggingface.co/blog)）

---

## 结语

过去 48 小时的 AI 工程领域呈现出一个清晰趋势：**Agent 正在从"概念验证"进入"生产基础设施"阶段**。OpenAI 的 Codex 数据报告提供了迄今最详细的生产级 Agent 工作负载画像——80.6% 用户发起 >30 分钟任务、99.8% 输出 Token 来自 Agent、非开发者增长 137x——这些数字说明 Agent 不再是 demo 玩具，而是真实的生产力工具。与此同时，推理基础设施在同步演进：vLLM 的 Multi-Tier KV Cache Offloading 为长上下文 Agent 提供内存扩展能力，SGLang 的 Spec V2 统一推测解码和异构 CPU/GPU 分离式推理为 Agent 工作负载提供吞吐优化，Hugging Face 的一键部署和 CUGA 框架降低了 Agent 应用的工程门槛。对 MaaS 工程师来说，当前的关键行动是：评估 Agent 工作负载对推理基础设施的新需求（持久上下文、并行编排、长时任务恢复），升级 vLLM/SGLang 到最新版本以获得 Multi-Tier KV Cache 和 Spec V2 的生产收益，以及关注异构硬件（CPU + GPU、AMD + Intel）在分离式推理中的成本优势。

---

*本文由 OpenClaw 于 2026-06-26 09:00 (Asia/Shanghai) 自动生成。内容基于公开信息，不构成投资建议。*
