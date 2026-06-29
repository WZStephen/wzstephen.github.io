---
layout: post
title: 'OpenAI GPT-5.6 Sol 发布、SGLang DeepSeek-V4 GB300 吞吐量提升 5 倍、MoE 推理负载均衡新方案 Waterfill 与 LPLB'
date: 2026-06-29 09:00:00 +0800
categories: [ai-technical-report]
---

> 过去 48 小时 AI 工程领域三条主线：**OpenAI 发布 GPT-5.6 Sol 限量预览（6 月 26 日）——引入全新 max reasoning effort 和 ultra mode（通过 subagent 编排加速复杂工作），在 Terminal-Bench 2.1、GeneBench v1 和 ExploitBench² 上刷新 SOTA，网络安全能力跨越关键阈值但未达到 Cyber Critical 级别，配套最强安全防护栈**；**SGLang 发布重大更新（6 月 26 日）——DeepSeek-V4 在 GB300  disaggregated 场景下吞吐量从 Day-0 的 ~2,200 tok/s/GPU 提升至 ~11,200 tok/s/GPU（5x），核心优化包括 MHC 深度融合、KV Compression V2、W4A4 MegaMoE 和 breakable CUDA graph；同时引入 Waterfill 和 LPLB 两种 dispatch-time MoE 负载均衡算法，在 DeepSeek-V3/V4 上实现 +1.5%~+7.3% 的吞吐提升**；**HP Inc. 宣布与 OpenAI 扩大 Frontier 战略合作（6 月 28 日）——一名工程师在数周内用 OpenAI 模型处理 43 个项目的 122 个 pull request，安全团队用 AI 在一天内修复了原本需要一个月的软件漏洞，展示 Codex 在企业级软件工程中的实际落地**。下面逐一拆解。

---

## 🔥 今日看点

1. **6 月 26 日** — OpenAI 发布 GPT-5.6 Sol 限量预览。引入全新 max reasoning effort 和 ultra mode——后者通过 subagent 编排突破单 agent 能力上限。Terminal-Bench 2.1（命令行工作流）SOTA；GeneBench v1（长时基因组学分析）优于 GPT-5.5 且使用更少 token；ExploitBench² 上以 ~1/3 输出 token 达到 Mythos Preview 水平。未跨越 Cyber Critical 阈值。配套最强安全防护栈，分阶段发布（[OpenAI](https://openai.com/index/previewing-gpt-5-6-sol/)）
2. **6 月 26 日** — SGLang 发布重大更新。DeepSeek-V4 在 GB300 disaggregated 8K/1K 场景下实现 ~11,200 tok/s/GPU（vs Day-0 ~2,200，5x 提升）。核心优化：MHC 深度融合（mhc_pre 迁移到 DeepGEMM 流程 + RMSNorm 融合 + 专用 fused hc_head kernel）；KV Compression V2（c4/c128/online c128 压缩 kernel）；W4A4 MegaMoE；breakable CUDA graph 扩展到 prefill 路径。Blackwell Ultra aggregated 场景下 no-MTP 峰值吞吐提升 6x+（[PyTorch Blog](https://pytorch.org/blog/serving-deepseek-v4-on-gb300-with-sglang-5x-higher-throughput-at-the-same-interactivity-since-day-0/)）
3. **6 月 26 日** — LMSYS 发布 Waterfill 和 LPLB——SGLang 中两种 dispatch-time MoE 负载均衡算法。Waterfill 将 shared expert 作为可调度 slot 分配到低负载 rank，DeepSeek-V4 上吞吐提升 +4.92%（49,253→51,677 tok/s）。LPLB 基于线性规划为冗余 expert 副本做 per-layer dispatch 优化，吞吐提升 +0.84%~+7.34%。均已在 SGLang 主分支合入（[LMSYS Blog](https://www.lmsys.org/blog/2026-06-26-waterfill-lplb)）
4. **6 月 28 日** — HP Inc. 宣布扩大与 OpenAI 的 Frontier 战略合作。一名工程师数周内处理 43 个项目 122 个 pull request；安全团队一天内修复原本需要一个月的漏洞。Frontier 作为统一平台管理 agent 的上下文、权限和评估。覆盖定价/合作伙伴支持、WXP 设备运维、网络安全等工作流（[OpenAI](https://openai.com/index/hp-frontier-partnership/)）
5. **6 月 26 日** — SGLang 新增 KDA CuteDSL prefill kernel（Blackwell SM100），面向 Kimi-Linear（KDA），比 Triton 路径快 1.08-1.52x，通过可复用 scratch workspace 实现。同时引入 linear-attention prefix-cache 内存优化——int8 checkpoint pool 将循环状态紧凑存储在 Mamba radix cache 中，大幅增加 prefix-cache 容量（[GitHub](https://github.com/sgl-project/sglang/releases)）
6. **6 月 26 日** — SGLang 新增 NVFP4 MoE 量化路径（DeepSeek-V4 on Blackwell），通过 `--moe-runner-backend flashinfer_trtllm_routed` 启用。FP8 group quantization 现在直接从 per-token group-quant kernel 输出 power-of-two（UE8M0）scales，省去单独 rounding pass。MLA decode q-heads pad 到 64 以使用更便宜的 FlashMLA head64 kernel（[GitHub](https://github.com/sgl-project/sglang/releases)）
7. **6 月 26 日** — SGLang 新增模型支持：GLM-5.2、LiquidAI LFM2.5、Kimi-K2.7-Code、Poolside Laguna-M.1、DiffusionGemma、Zyphra ZAYA1、MiMo-V2-ASR。Speculative decoding 更新：EAGLE draft-extend sync-free fast_prefill_plan + FlashInfer CUDA graph 支持；MTP rejection sampling；GLM-4.7-Flash MTP（NPU）（[GitHub](https://github.com/sgl-project/sglang/releases)）
8. **6 月 23 日** — Hugging Face 发布 IBM Research 的 CUGA——轻量级 agent harness，提供两打可运行的 agentic app 示例。展示如何用最小框架搭建真实 agent 应用（[Hugging Face Blog](https://huggingface.co/blog/ibm-research/cuga-apps)）
9. **6 月 25 日** — Hugging Face 发布 Allen AI 的混合模型 token 预测质量研究——研究 Mamba + Attention 混合架构在不同 token 类型上的预测质量差异，为混合模型设计提供实证依据（[Hugging Face Blog](https://huggingface.co/blog/allenai/hybrid-token-prediction)）
10. **6 月 24 日** — Hugging Face 发布 NVIDIA NeMo AutoModel 加速 Transformer 微调——使用 NVIDIA NeMo AutoModel 加速微调流程，降低工程摩擦（[Hugging Face Blog](https://huggingface.co/blog/nvidia/accelerating-fine-tuning-nvidia-nemo-automodel)）

---

## 💡 深度解读

### 1️⃣ GPT-5.6 Sol：Ultra Mode 与 Subagent 编排意味着什么

**问题背景：**
当前 LLM 的能力提升主要集中在单模型维度——更大的 context、更好的推理、更强的代码能力。但真实世界的复杂工作往往超出单个 agent 的能力范围：需要并行执行多个子任务、跨工具协调、在长时运行中保持上下文一致性。OpenAI 在 GPT-5.6 Sol 中引入的 ultra mode 首次在产品层面尝试通过 subagent 编排来突破单 agent 能力上限。

**核心思路/原理：**
GPT-5.6 Sol 的关键创新点：

- **Ultra mode（subagent 编排）**：不再依赖单个 agent 串行执行所有步骤，而是将复杂工作分解为多个子任务，由 subagent 并行处理。这类似于软件工程中的"分而治之"——主 agent 负责规划和协调，subagent 负责具体执行。对推理系统来说，这意味着需要支持多路并发推理、跨 session 上下文传递和结果聚合
- **Max reasoning effort**：新增的最高级别推理努力，给 Sol 更多时间进行深度推理。与现有的 low/medium/high reasoning effort 形成四级梯度。对推理基础设施来说，更长的推理时间意味着更多的 compute per request，需要在成本和质量之间做更好的 tradeoff
- **分层安全防护**：GPT-5.6 Sol 配备 OpenAI 迄今最强的安全防护栈。在 ExploitBench² 上，Sol 能识别 bug 和 exploitation primitives，但不能自主生成完整的 full-chain exploit。这使其跨越了之前的模型未达到的能力阈值，但未达到 Cyber Critical 级别。OpenAI 选择分阶段发布（先小范围 trusted partners），而非直接全面开放

**数据与证据：**
- Terminal-Bench 2.1 SOTA（命令行工作流，需要规划、迭代和工具协调）
- GeneBench v1 优于 GPT-5.5，使用更少 token（长时基因组学分析）
- ExploitBench² 上以 ~1/3 输出 token 达到 Mythos Preview 水平
- ExploitGym 上 Sol/Terra/Luna 均展示随 reasoning effort 提升的网络安全能力增长
- Terra 性能与 GPT-5.5 相当，成本降低 2x；Luna 提供最低成本选项
- 未跨越 Cyber Critical 阈值（Chromium/Firefox 测试中能识别 bug 但不能自主生成 full-chain exploit）

来源：
- [OpenAI: Previewing GPT-5.6 Sol](https://openai.com/index/previewing-gpt-5-6-sol/)
- [GPT-5.6 Preview System Card](https://deploymentsafety.openai.com/gpt-5-6-preview)

**工程启示：**
1. **Subagent 编排对推理基础设施提出新要求**——当 ultra mode 需要同时运行多个 subagent 时，推理平台需要支持：多路并发请求的调度、跨 session 的上下文共享、结果聚合和错误恢复。这对 MaaS 平台来说是一个新的系统挑战——不只是"更快的单次推理"，而是"可靠的多路并发推理编排"
2. **推理成本的梯度定价将变得更重要**——max reasoning effort 意味着最高 compute per request 的选项。对推理平台来说，需要为不同 reasoning effort 级别提供差异化的定价和 SLA。Terra（2x cheaper）和 Luna（lowest cost）的分级进一步丰富了定价维度
3. **分阶段发布正在成为前沿模型的标准做法**——GPT-5.6 Sol 先给 trusted partners，再逐步扩大。这对推理平台的弹性提出要求：需要在新模型发布初期快速扩容小范围部署，然后在全面发布时迅速扩展到全球规模

---

### 2️⃣ SGLang DeepSeek-V4 GB300 吞吐量 5x 提升：推理系统持续优化的工程方法论

**问题背景：**
DeepSeek-V4 是 2026 年 4 月发布的最新 MoE 大模型，使用 FP4 推理和稀疏 expert 激活。SGLang 在 Day-0 就提供了功能性支持，但 Day-0 的栈只是起点。从 4 月到 6 月，SGLang 团队协调了一系列 kernel、runtime 和硬化改进，在 GB300 disaggregated 场景下实现了 5x 的吞吐量提升——这是一个典型的"从能跑到跑好"的工程优化案例。

**核心思路/原理：**
5x 提升来自多个层面的协同优化，而非单一突破：

- **MHC 深度融合**：将 mhc_pre 路径迁移到 DeepGEMM 支持的流程，RMSNorm 融合进 MHC 路径（不再是独立的 stage boundary），新增专用 fused hc_head kernel 和 fused mhc_fused_post_pre kernel。减少了中间 tensor 流量和 scheduler 可见的 plumbing 开销——这是 prefill 路径中最昂贵的部分之一
- **KV Compression V2**：新增 c4、c128 和 online c128 压缩 kernel，更新 compressor 管道和 fused norm/rope V2 组件。KV cache 压缩直接影响内存带宽利用率——MoE 模型的内存瓶颈比稠密模型更突出
- **W4A4 MegaMoE**：4-bit weight + 4-bit activation 的 MoE 量化路径，在保持质量的前提下大幅提升 expert 计算吞吐
- **Breakable CUDA graph 扩展到 prefill**：之前只在 decode 侧支持，现在扩展到 prefill 路径。这消除了 prefill 阶段因 graph break 导致的 fallback 开销
- **Disaggregated decode admission 改进**：更好的 decode worker 准入策略，在高并发下保持更稳定的吞吐

**数据与证据：**
- GB300 disaggregated 8K/1K（DeepSeek-V4 Pro, FP4, ISL=8192, OSL=1024）：
  - Day-0（4 月）no-MTP：~2,200 tok/s/GPU
  - 6 月 no-MTP：在 40 tok/s/user 下 sustain 2.1x more throughput
  - 6 月 MTP：在 80 tok/s/user 下 sustain 2.6x more throughput
  - 6 月 MTP 峰值：~11,200 tok/s/GPU（5x vs Day-0）
- Blackwell Ultra aggregated 8K/1K：
  - no-MTP 峰值吞吐提升 6x+
  - 30 tok/s/user 下 2.91x 提升，90 tok/s/user 下 2.85x 提升

来源：
- [PyTorch Blog: Serving DeepSeek-V4 on GB300 with SGLang](https://pytorch.org/blog/serving-deepseek-v4-on-gb300-with-sglang-5x-higher-throughput-at-the-same-interactivity-since-day-0/)
- [SGLang GitHub Releases](https://github.com/sgl-project/sglang/releases)

**工程启示：**
1. **"从 Day-0 到 production-ready"的系统性优化是 MaaS 竞争力的核心**——5x 提升不是来自单一突破，而是 kernel、runtime、quantization、CUDA graph、admission 等多个层面的协同优化。对推理团队来说，这意味着"Day-0 模型支持"只是起点，真正的竞争力在于后续的持续优化速度
2. **GB300 disaggregated serving 正在成为生产级部署的标准模式**——5x 提升是在 disaggregated 模式下实现的，说明 prefill-decode 分离架构在 GB300 上已经成熟。对做大规模推理部署的团队来说，disaggregated serving + SGLang 是当前最有前景的技术栈之一
3. **FP4/MoE 量化路径正在快速成熟**——W4A4 MegaMoE + NVFP4 MoE + FP8 group quantization with UE8M0 scales，这些量化路径的组合使 DeepSeek-V4 在 Blackwell 上实现了接近理论峰值的吞吐。对推理工程师来说，FP4 正在从"实验性"转向"production-ready"

---

### 3️⃣ Waterfill 与 LPLB：MoE 推理中 dispatch-time 负载均衡的两个工程切入点

**问题背景：**
MoE 模型（DeepSeek-V3/R1/V4）使用 Expert Parallelism（EP）将 expert 分布到多个 GPU 上。但 router 不会生成完美均衡的 expert 流量——某些 expert 收到更多 token，导致对应 GPU 成为瓶颈，其他 GPU 等待。静态的 expert 放置（如 EPLB）能改善长期均衡，但单个 batch 内仍有残余不均衡。Dispatch-time 负载均衡在 runtime 决定每个 token 或 shared expert 请求由哪个物理 rank 处理，填补静态放置的空白。

**核心思路/原理：**
两种算法解决同一问题的不同维度：

- **Waterfill（shared expert 负载均衡）**：核心 insight 是将 shared expert 从"每个 rank 都本地计算"变为"可调度到轻负载 rank 的 dispatchable slot"。算法计算每个 rank 的 routed expert 负载，设定一个水位线 H = ⌈(ΣLr + N) / R⌉，将 shared expert 工作分配到水位线以下的 rank。关键设计选择：通信保守模式（只在 token 已经访问的 rank 中选择）vs 全 rank 模式（更多平衡自由度但可能增加 all-to-all 通信）
- **LPLB（冗余 expert 副本负载均衡）**：为 EPLB 引入的冗余 expert 副本提供 per-layer 线性规划优化。每个 token 的 routed expert 选择不变（语义不变），但选择哪个物理副本处理由 LP 求解器决定。通过 `--ep-dispatch-algorithm=lp` 启用

**数据与证据：**
- Waterfill：DeepSeek-V4 上最佳吞吐从 49,253 tok/s 提升到 51,677 tok/s（+4.92%）；DeepSeek-V3/R1 上 MMLU/GPQA/GSM8K 提升 +1.48%~+4.66%
- LPLB：MMLU/GPQA/GSM8K 吞吐提升 +0.84%~+7.34%
- 两种算法可以组合使用（分别优化 shared expert 和 routed expert 的负载均衡）

来源：
- [LMSYS Blog: Waterfill & LPLB](https://www.lmsys.org/blog/2026-06-26-waterfill-lplb)
- [SGLang PR #19290 (Waterfill)](https://github.com/sgl-project/sglang/pull/19290)
- [SGLang PR #24515 (LPLB)](https://github.com/sgl-project/sglang/pull/24515)

**工程启示：**
1. **Dispatch-time 负载均衡是 MoE 推理的"免费午餐"**——Waterfill 和 LPLB 都不改变模型语义，只改变物理执行路径。对使用 DeepSeek-V3/V4 的推理团队来说，这是几乎零成本的吞吐提升。Waterfill 的通信保守模式尤其值得注意——它在几乎不增加通信开销的前提下实现了有意义的负载均衡
2. **线性规划在推理系统中的应用是一个值得关注的方向**——LPLB 将 per-layer dispatch 建模为 LP 问题并求解。这种"用数学规划优化推理调度"的思路可以推广到更多场景：TP/PP 的负载均衡、disaggregated serving 的 admission 控制等
3. **MoE 推理的系统优化空间远未耗尽**——从静态 EPLB 到 dispatch-time Waterfill/LPLB，再到 SGLang 的 NVFP4 MoE 和 KV Compression V2，DeepSeek-V4 的推理优化仍在快速推进。对 MaaS 工程师来说，跟踪 SGLang/vLLM 的 MoE 相关更新是保持竞争力的关键

---

## 🔧 开源工具动态

1. **SGLang 重大更新（6 月 26 日）** — DeepSeek-V4 GB300 5x 吞吐提升；Waterfill + LPLB MoE 负载均衡；KDA CuteDSL prefill kernel（Blackwell SM100，1.08-1.52x）；linear-attention prefix-cache 内存优化（int8 checkpoint pool）；NVFP4 MoE for DeepSeek-V4；MSCCL++ 集成 + MNNVL allreduce fusion；Nemotron DP attention + MTP；AMD breakable CUDA graph on ROCm/HIP。新模型：GLM-5.2、LiquidAI LFM2.5、Kimi-K2.7-Code、Poolside Laguna-M.1、DiffusionGemma（[GitHub](https://github.com/sgl-project/sglang/releases)）

2. **vLLM v0.23.0（6 月 15 日）** — 408 commits，200 贡献者。Multi-tier KV cache offloading（object-store 二级存储 + HMA 默认启用）；Model Runner V2 扩展到 Llama/Mistral 稠密模型；Rust frontend 新增 streaming generate、动态 LoRA endpoints；DeepSeek-V4 全面硬化（TRTLLM-gen attention、EPLB Mega-MoE、选择性 prefix-cache）；Gemma 4 Unified encoder-free 支持；Transformers v5 兼容（[GitHub](https://github.com/vllm-project/vllm/releases)）

3. **Hugging Face: IBM CUGA Agentic Apps（6 月 23 日）** — 轻量级 agent harness，提供两打可运行的 agentic app 示例。展示如何用最小框架搭建真实 agent 应用，适合 agent 开发入门和快速原型（[Hugging Face Blog](https://huggingface.co/blog/ibm-research/cuga-apps)）

4. **Meta Ads Model Kernel Library（开源持续更新）** — TLX Block Attention（Blackwell SM100 块稀疏注意力，前向 ~1.85x、反向 ~2.50x 加速）、GDPA、TLX Multi-CTA Norm Fusion、TLX GDPA Megakernel。目标 Blackwell SM100，Apache 2.0 许可（[GitHub](https://github.com/facebookresearch/ads_model_kernel_library)）

---

## 结语

今天的三条主线交汇出一个清晰的工程趋势：**AI 推理优化正在从"单点突破"走向"系统性协同"**。GPT-5.6 Sol 的 ultra mode 通过 subagent 编排突破单 agent 能力上限，对推理基础设施提出了"多路并发 + 上下文协调"的新要求；SGLang 在 DeepSeek-V4 上的 5x 提升不是来自单一 kernel 突破，而是 MHC 融合、KV 压缩、W4A4 量化、CUDA graph 扩展和 admission 策略的协同优化；Waterfill 和 LPLB 则展示了 MoE 推理中 dispatch-time 负载均衡这个被忽视的优化维度。对 MaaS 工程师来说，接下来的竞争力不在于"能不能跑最新模型"，而在于"能不能在真实硬件上把模型跑到接近理论峰值"——这需要 kernel、runtime、quantization、调度和系统层面的全栈协同能力。
