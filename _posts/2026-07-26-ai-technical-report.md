---
layout: post
title: '推测解码军备竞赛 DSPARK 384 tok/s、Inkling 975B MoE Day-0、Agent 训练新范式、上下文管理生命周期'
date: 2026-07-26 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期覆盖 2026 年 7 月 26 日 AI 推理与 Agent 领域的重要进展。开源推理框架迎来重大更新：vLLM v0.26.0 与 SGLang v0.5.16 同日发布公告，SGLang 引入 DSPARK 置信驱动推测解码在 DeepSeek-V4-Pro 上达到 383.7 tok/s（accept length ~5），vLLM 完成 PagedAttention 到 V1/MRv2 的迁移并新增 Inkling 模型家族支持；NVIDIA Inkling 975B 多模态 MoE 成为两大框架 Day-0 支持焦点，混合滑动窗口、全注意力与 Mamba2 线性注意力架构支持 1M-token 上下文。TensorRT-LLM v1.3.0rc22 新增 Laguna DFlash 与 DSPARK 推测解码 drafters，llama.cpp b10107 引入 NVFP4 量化与推测解码自动检测。论文方面，OpenForgeRL 提出端到端训练 harness-native Agent 的开源框架；AREX 基于发现-验证不对称性实现递归自我改进 Deep Research；PATS 策略感知训练脚手架解决长时域 Agent RL 弱策略问题；Agentic Context Management 将上下文管理重新定义为生命周期与架构问题；Euclid-MCP 通过 MCP 协议引入 Prolog 确定性逻辑推理；MIRROR 实现多模态推理跨视角一致性。

---

## 🔥 今日看点

1. **7 月 25 日** — SGLang v0.5.16：DSPARK 推测解码与 Inkling 975B Day-0 支持。DSPARK 通过置信度驱动的块半自回归起草与自适应验证窗口，在 DeepSeek-V4-Pro TP8 B300 上达到 383.7 tok/s（accept length ~5）；Inkling 975B 多模态 MoE 支持 1M-token 上下文，混合滑动窗口、全注意力与 Mamba2 线性注意力，Blackwell TP8 上 71.7k tok/s 输入、171.0 tok/s per-user decode；UnifiedRadixTree 成为 SWA/Mamba/DSA 模型默认，GLM-5.2 DSA cache layer split 降低每 rank KV 内存 ~74%（[GitHub Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.16)）。

2. **7 月 25 日** — vLLM v0.26.0：PagedAttention 正式移除，V1/MRv2 成为唯一路径。411 commits 来自 212 位贡献者（61 位新加入）。新增 Inkling 模型家族完整支持（piecewise CUDA graphs、Hopper FA4 relative attention、MTP=1 speculative decoding、LoRA、NVFP4 量化）；DeepSeek-V4 专用路由内核（2.94% E2E TPOT 改善）、fused_topk_bias（1.5-2× 内核加速）；fp32 lm_head 提升生成精度；Rust 前端支持多模态视频/音频；KV offloading 与分层存储成熟（[GitHub Release](https://github.com/vllm-project/vllm/releases/tag/v0.26.0)）。

3. **7 月 22 日** — TensorRT-LLM v1.3.0rc22：新增 Laguna DFlash 与 DSPARK 推测解码 drafters。DeepSeek-V4-Pro 精选配置、Qwen3-VL 混合图像+视频模态；disaggregated coordinator 与 multi-process orchestrator 支持大规模 serving；FP4 KV cache 与非 FP4 Mamba state 支持；SM121 MLA cache reuse；ModelExpress checkpoint 加载集成。Breaking changes：legacy C++ TensorRT backend 模块移除（[GitHub Release](https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc22)）。

4. **7 月 24 日** — llama.cpp b10107：NVFP4 W4A4 激活量化与推测解码自动检测。32-byte loads、nvfp4x4 intrinsics、融合 per-channel amax/量化内核；DeepSeek V4 模板修复、APE tensor op、drop_reasoning flag；推测解码类型自动检测（从 HuggingFace repo sidecars 推断 mtp-/dflash-/eagle3-）；load-mode 重构（mlock/mmap/directio 合并为 --load-mode，breaking change）；WebGPU depthwise conv2d、CUDA GET_ROWS k-quant、Hexagon GEGELU microkernel（[GitHub](https://github.com/ggml-org/llama.cpp/releases/tag/b10107)）。

5. **7 月 25 日** — OpenForgeRL：端到端训练 Harness-native Agent 的开源框架。现有 SFT/RL 栈无法原生表达 Claude Code、Codex 等复杂 Agent 推理 harness 的有状态多进程特性。OpenForgeRL 通过统一训练-推理 harness 抽象、轻量代理记录 harness 模型调用作为训练数据、Kubernetes 编排可扩展 rollout，在 ClawEval（31.7 pass³）和 OSWorld-Verified（37.7）上达到领先水平。关键发现：不同 harness 的学习难度差异显著，RL 提升 Agent 可靠性但错误恢复仍弱（[arXiv:2607.21557](https://arxiv.org/abs/2607.21557)）。

6. **7 月 24 日** — AREX：递归自我改进的 Deep Research Agent。基于发现-验证不对称性原理——发现满足多约束答案成本高，但验证候选答案可分解为逐项检查——AREX 不仅延长搜索时间，而是通过内部研究循环（收集证据、构建临时答案）与外部自我改进循环（审计约束、识别未解决声明、发起针对性跟进）递归改进。学习自主 context-update 工具压缩增长交互历史为紧凑改进状态。4B dense 与 122B-A10B MoE 模型在 BrowseComp、HLE 上大幅超越基线（[arXiv:2607.21461](https://arxiv.org/abs/2607.21461)）。

7. **7 月 24 日** — PATS：策略感知训练脚手架解决长时域 Agent RL。弱策略在长 horizon Agent RL 中反复产生相似失败的无信息量 rollout。PATS 将 rollout groups 转化为"证据卡"并动态调整后续 rollout 上下文——具体引导帮助弱策略完成任务，随策略提升修剪冗余上下文。在 ALFWorld/WebShop 上超越强基线达 18.6%，同时减少 32.1% prompt tokens。脚手架在部署时丢弃（[arXiv:2607.21419](https://arxiv.org/abs/2607.21419)）。

8. **7 月 24 日** — Agentic Context Management：将上下文管理重新定义为架构问题。生产环境 Agent 失败更多源于无法管理推理上下文——对话历史膨胀、token 成本二次增长、跨会话记忆缺失——而非推理能力不足。ACM 提出 5 个原语的生命周期纪律：architecting、ingesting、scoping、anticipating、compacting/consolidation。验证的压缩方案实现线性 token 成本并保持保真度，Maximem Synap 参考实现在 LongMemEval 达到 92%（[arXiv:2607.21503](https://arxiv.org/abs/2607.21503)）。

9. **7 月 24 日** — Euclid-MCP：通过 MCP 协议为 LLM Agent 提供确定性逻辑推理。LLM 在多步逻辑推理上不可靠，语义 RAG 从根本上不适合规则执行。Euclid-MCP 将 SWI-Prolog 确定性逻辑引擎封装为标准化 MCP Server，引入引擎无关中间表示 Euclid-IR 支持 Horn 子句逻辑，LLM 委托推理同时保留完整证明轨迹。在 IT 安全/合规领域评估显示 LLM 在大型知识库上系统性幻觉，Euclid-MCP 提供精确答案且延迟更低（[arXiv:2607.21412](https://arxiv.org/abs/2607.21412)）。

10. **7 月 24 日** — MIRROR：多模态推理跨视角一致性学习。视觉语言模型在几何等值文本/图表视图上表现不同行为与失败模式。MIRROR 通过 RL 在所有视图下评估模型，选择最佳视图作为 teacher，用 reverse-KL 训练其他视图向其对齐。超越标准 RL 并产生更一致的跨模态行为（[arXiv:2607.21552](https://arxiv.org/abs/2607.21552)）。

---

## 💡 深度解读

### 1️⃣ 推测解码军备竞赛：DSPARK、Laguna、MTP、EAGLE3 全面开花

**问题背景：**
推测解码（Speculative Decoding）通过将大模型的自回归解码拆分为"小模型草稿 + 大模型验证"两步并行，在不改变输出分布的前提下显著降低推理延迟。2026 年下半年，这一领域进入白热化竞争阶段：SGLang 的 DSPARK、TensorRT-LLM 的 Laguna DFlash、vLLM 的 MTP=1、llama.cpp 的 EAGLE3 自动检测——四大开源框架同日或近期密集发布推测解码增强，标志着该技术从"实验性优化"进入"生产标配"阶段。

**核心思路/原理：**
SGLang 的 DSPARK 采用置信度驱动的块半自回归起草策略：小模型不再逐 token 串行起草，而是以块为单位并行生成，同时根据每个位置的置信度自适应调整验证窗口——高置信度区域扩大验证窗口减少验证开销，低置信度区域缩小窗口保证接受率。这打破了传统"固定 draft length"的 trade-off。TensorRT-LLM 的 Laguna DFlash 则通过 flash-style attention 优化草稿-目标模型间的数据流，减少 HBM 访问。llama.cpp 的自动检测机制从 HuggingFace repo sidecars 推断 draft model 类型（mtp-/dflash-/eagle3-），降低用户配置门槛。

**数据与证据：**
- SGLang DSPARK 在 DeepSeek-V4-Pro TP8 B300 bs=1 上达到 383.7 tok/s，accept length ~5，较标准自回归解码提升约 3-4×
- vLLM DeepSeek-V4 路由内核贡献 2.94% E2E TPOT 改善，fused_topk_bias 实现 1.5-2× 内核加速
- TensorRT-LLM v1.3.0rc22 新增 FP4 KV cache 支持，进一步降低推测解码内存开销
- llama.cpp NVFP4 W4A4 量化在消费级硬件上实现 4-bit 激活量化，为端侧推测解码铺路

来源：
- [SGLang v0.5.16 Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.16)
- [vLLM v0.26.0 Release](https://github.com/vllm-project/vllm/releases/tag/v0.26.0)
- [TensorRT-LLM v1.3.0rc22 Release](https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc22)
- [llama.cpp b10107](https://github.com/ggml-org/llama.cpp/releases/tag/b10107)

**工程启示：**
1. 推测解码正从"可选优化"变为"默认配置"——生产环境部署应优先评估 draft model 选型与 accept length 调优
2. 置信度驱动的自适应验证（DSPARK 思路）比固定 draft length 更优，未来可能成为标准范式
3. 自动检测机制（llama.cpp）大幅降低用户配置成本，其他框架预计跟进
4. FP4 量化与推测解码的结合将进一步降低内存瓶颈，使更大模型的实时推理成为可能

---

### 2️⃣ Inkling 975B MoE：超长上下文多模态 MoE 的工程挑战

**问题背景：**
NVIDIA Inkling 是 975B 参数的多模态 MoE 模型，支持 1M-token 上下文长度，采用混合注意力架构（滑动窗口 + 全注意力 + Mamba2 线性注意力）。这类超大规模模型的 Day-0 部署支持对推理框架提出了全方位挑战：MoE 路由效率、超长上下文的内存管理、多模态输入的处理流水线、以及异构注意力机制的 CUDA kernel 适配。vLLM 与 SGLang 均在最新版本中实现 Day-0 支持，展示了开源推理生态的快速响应能力。

**核心思路/原理：**
Inkling 的混合注意力架构设计精妙：滑动窗口注意力处理局部依赖（O(n) 复杂度），全注意力处理全局依赖但仅限于关键层，Mamba2 线性注意力提供 O(n) 的长程状态压缩。三者协同使 1M-token 上下文的实际计算与内存开销可控。MoE 部分采用 NVFP4 量化，在保持精度的同时将模型体积压缩至 FP16 的约 1/4。SGLang 的 UnifiedRadixTree 成为 SWA/Mamba/DSA 模型默认，通过统一的 radix tree 管理不同注意力机制的 KV cache，避免碎片化。vLLM 则通过 piecewise CUDA graphs 将 MoE 路由、attention、FFN 等子图分别捕获与调度，Hopper FA4 relative attention 利用硬件特性加速长序列处理。

**数据与证据：**
- SGLang 在 Blackwell TP8 上实现 Inkling 71.7k tok/s 输入吞吐、171.0 tok/s per-user decode
- SGLang GLM-5.2 DSA cache layer split 在 prefill CP 下降低每 rank KV 内存 ~74%（0.77→0.20 GB/rank）
- vLLM 为 Inkling 提供 MTP=1 speculative decoding、LoRA、NVFP4 量化全栈支持
- TensorRT-LLM 新增 FP4 KV cache 与非 FP4 Mamba state 支持，SM121 MLA cache reuse

来源：
- [SGLang v0.5.16 Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.16)
- [vLLM v0.26.0 Release](https://github.com/vllm-project/vllm/releases/tag/v0.26.0)

**工程启示：**
1. 1M-token 上下文的 MoE 模型部署已具备工程可行性，但需要 TP8 Blackwell 级别硬件——消费级部署仍需等待量化与稀疏化进一步成熟
2. 混合注意力架构（SWA + Full + Mamba2）可能成为下一代超长上下文模型的标配，框架需要持续适配异构 KV cache 管理
3. Day-0 支持的速度表明开源推理框架的模块化架构已足够成熟，新模型适配从"月级"缩短到"天级"
4. NVFP4 量化在 MoE 模型中的成功应用验证了 4-bit 精度在超大模型上的可用性

---

### 3️⃣ Agent 训练范式革新：从 Harness-Aware RL 到上下文生命周期管理

**问题背景：**
当前最先进的 AI Agent（Claude Code、Codex、OpenClaw）依赖复杂的推理 harness 驱动多轮推理、工具调用与外部系统访问。这些 harness 使得端到端训练极为困难——现有开源 SFT/RL 栈无法原生表达有状态多进程 Agent 推理。同时，Agent 在实际部署中的失败更多源于无法管理推理上下文——对话历史膨胀导致 token 成本二次增长、跨会话记忆缺失、关键信息在长对话中被淹没——而非推理能力本身不足。OpenForgeRL、PATS、AREX 和 ACM 从不同角度系统性地解决这些问题。

**核心思路/原理：**
OpenForgeRL 通过统一训练-推理 harness 抽象解决"如何训练"：轻量代理记录 harness 模型调用作为训练数据，Kubernetes 编排支持可扩展 rollout，使策略梯度可直接穿过整个 Agent 执行轨迹。PATS 解决"如何高效训练"：将 rollout groups 转化为证据卡，动态调整后续 rollout 上下文——弱策略获得更多引导，随策略提升修剪冗余，部署时脚手架丢弃。AREX 解决"如何递归改进"：基于发现-验证不对称性，内部研究循环收集证据构建临时答案，外部自我改进循环审计约束识别未解决声明，学习自主 context-update 工具压缩交互历史。ACM 解决"如何管理上下文"：将上下文视为具有生命周期（创建、活跃、归档、检索）的结构化对象，5 个原语实现线性 token 成本。

**数据与证据：**
- OpenForgeRL 在 ClawEval 达到 31.7 pass³，OSWorld-Verified 37.7，匹配或超越更大模型
- PATS 在 ALFWorld/WebShop 上超越强基线 18.6%，同时减少 32.1% prompt tokens
- AREX 4B dense 与 122B-A10B MoE 在 BrowseComp、HLE 上大幅超越基线
- ACM 参考实现 Maximem Synap 在 LongMemEval 达到 92%，验证压缩实现线性 token 成本

来源：
- [OpenForgeRL: arXiv:2607.21557](https://arxiv.org/abs/2607.21557)
- [PATS: arXiv:2607.21419](https://arxiv.org/abs/2607.21419)
- [AREX: arXiv:2607.21461](https://arxiv.org/abs/2607.21461)
- [Agentic Context Management: arXiv:2607.21503](https://arxiv.org/abs/2607.21503)

**工程启示：**
1. Agent 训练正从"先推理后微调"向"端到端 harness-aware 训练"演进，训练基础设施将决定 Agent 能力上限
2. PATS 的 32% token 减少直接降低训练成本——对大规模 Agent RL 训练的经济性至关重要
3. AREX 的学习型上下文压缩工具与 ACM 的生命周期管理可结合：压缩工具决定什么保留，生命周期管理决定如何保留
4. Euclid-MCP 的确定性逻辑推理补充了 Agent 工具链的可靠性缺口——对于需要形式化保证的场景（安全、合规），语义 RAG 不可靠，Prolog 引擎是必要补充

---

## 🔧 开源工具动态

1. **vLLM** — v0.26.0（7 月 25 日）为重大版本更新。PagedAttention 在 v0.25.0 正式移除，V1/MRv2 成为唯一路径。新增 Inkling 模型家族完整支持（piecewise CUDA graphs、Hopper FA4、MTP=1 spec decode、LoRA、NVFP4）；DeepSeek-V4 专用路由内核（2.94% TPOT 改善）；fp32 lm_head 提升生成精度；Rust 前端支持多模态视频/音频；KV offloading 与分层存储成熟（object-store secondary tier、DP-replica-aware tiering）。Transformers 5.13.0 后端迁移持续推进。**生产环境建议：** v0.26.0 为 Day-0 Inkling 部署的首选，但需验证现有模型在新版本下的兼容性——PagedAttention 移除后 V1 路径的 KV cache 行为可能有差异。

2. **SGLang** — v0.5.16（7 月 25 日）574 PRs 来自 169 位贡献者。DSPARK 推测解码为最大亮点（383.7 tok/s on DeepSeek-V4-Pro）；Inkling 975B Day-0 支持展示快速响应能力；UnifiedRadixTree 成为 SWA/Mamba/DSA 默认简化部署；GLM-5.2 DSA cache layer split 降低 KV 内存 74% 对大 batch serving 意义重大。新增 LongCat 2.0 FP8、JetBrains Mellum v2、Pi0.5（VLA）、LongLive 2.0（diffusion）支持。**与 vLLM 互补关系：** SGLang 在推测解码与 radix tree 管理上领先，vLLM 在模型生态广度与 Rust 前端稳定性上占优，两者共同推动开源推理前沿。

3. **TensorRT-LLM** — v1.3.0rc22（7 月 22 日）仍为 release candidate。新增 Laguna DFlash 与 DSPARK 推测解码 drafters；DeepSeek-V4-Pro 精选配置；disaggregated coordinator 支持大规模 serving 部署；FP4 KV cache 降低内存开销。**Breaking changes 注意：** legacy C++ TensorRT backend 模块移除，依赖旧后端的部署需迁移。已知问题包括 torch.compile 崩溃、DeepSeek V3.2 multi-GPU 精度、Mixtral FP8 MoE + multi-LoRA 兼容性，生产环境部署需谨慎评估。

4. **llama.cpp** — b10107（7 月 24 日）保持 near-daily 发布节奏。NVFP4 W4A4 量化改进（32-byte loads、nvfp4x4 intrinsics）；推测解码类型自动检测从 HuggingFace repo sidecars 推断 draft model 类型，大幅降低配置门槛；load-mode 重构（mlock/mmap/directio 合并为 --load-mode，breaking change）；DeepSeek V4 模板修复；WebGPU depthwise conv2d、CUDA GET_ROWS k-quant、Hexagon GEGELU microkernel、Metal f16 leaky relu、PowerPC on AIX 支持。**CPU 推理建议：** NVFP4 量化使消费级硬件上的 4-bit 推理质量进一步提升，适合本地 Agent 部署与隐私敏感场景。

5. **MLC LLM** — v0.20.0（7 月 7 日）通过 git tag 发布，无 GitHub release notes。作为端侧部署首选框架，MLC LLM 持续优化内存占用与移动端推理性能。建议关注[官方文档](https://llm.mlc.ai/)与[提交历史](https://github.com/mlc-ai/mlc-llm/commits/v0.20.0)获取更新详情。**端侧部署建议：** MLC LLM 在 iOS/Android/WebGPU 上的内存优化持续领先，适合 Agent 端侧推理场景。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 26 日*
