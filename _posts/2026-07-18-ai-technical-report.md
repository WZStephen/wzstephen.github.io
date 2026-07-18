---
layout: post
title: 'SearchOS 多 Agent 信息检索框架、长上下文微调 16GB 显存跑 13 万 token、GUI Agent 计划驱动交互'
date: 2026-07-18 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期聚焦 Agent 工程化与推理效率：SearchOS 将开放域信息检索建模为关系模式补全，通过上下文外化（Frontier Task、Evidence Graph、Coverage Map、Failure Memory）实现多 Agent 协作的鲁棒状态管理，在 WideSearch 和 GISA 上领先所有基线；Plover 提出计划驱动的 GUI Agent 交互范式，将任务计划外化为可检查、可编辑的持久化制品，使自主失败可结构化修复；长上下文微调新方案 HGA 在 16GB 显存上实现 131K token 推理，通过分层全局注意力和分段反向传播突破显存瓶颈；PRISM 用隐藏状态探针区分文本危险与物理危险，为具身 Agent 安全提供新范式；vLLM v0.25 移除 PagedAttention、Model Runner V2 成默认、Transformers 后端追平原生性能；SGLang v0.5.15 在 Blackwell 上调优 GLM-5.2 NVFP4 达 500+ tok/s/user，Spec V2 零开销调度；TensorRT-LLM v1.3.0rc21 宣布 AutoDeploy 后端废弃；llama.cpp 最新构建聚焦 OpenCL MoE 内核和 ggml 0.17.0 版本升级。

---

## 🔥 今日看点

1. **7 月 17 日** — arXiv 2607.15257：**SearchOS 多 Agent 开放域信息检索框架**。将信息检索建模为关系模式补全，Agent 发现实体、填充属性、锚定引用证据。核心创新 Search-Oriented Context Management (SOCM) 将搜索状态外化为 Frontier Task、Evidence Graph、Coverage Map 和 Failure Memory 四类持久化制品。Pipeline-parallel 调度机制重叠子 Agent 执行，持续用未覆盖缺口任务填充空闲槽位。在 WideSearch 和 GISA 基准上全面领先单 Agent 和多 Agent 基线。[arXiv:2607.15257](https://arxiv.org/abs/2607.15257)

2. **7 月 17 日** — arXiv 2607.15193：**Plover 计划驱动的 GUI Agent 交互系统**。针对 GUI 自动化中 Agent 因动态布局和意外对话框偏离用户意图的问题，提出 planner-executor 架构，将任务计划和重计划外化为可检查、可编辑的持久化制品。支持截图锚定干预和局部化修正，保留已完成进度。基准失败案例修复实验证明：当计划可见且干预局部化时，多数自主 GUI Agent 失败可结构化修复。[arXiv:2607.15193](https://arxiv.org/abs/2607.15193)

3. **7 月 17 日** — arXiv 2607.15105：**HGA 长上下文微调在 16GB 显存上跑 131K token**。结合分层全局注意力 (HGA)、分段反向传播和分层 KV 存储，仅活跃段保留在显存中参与微分计算，旧 KV 卸载到 RAM 或 NVMe。在 Qwen3-8B + 4-bit QLoRA + PG19 上，16GB Quadro RTX 5000 密集训练到 2K 即溢出，HGA 达到 16K 训练长度（15.28GB 峰值显存），同一适配器推理跑满 131K token。2K 训练长度下 HGA 与密集训练困惑度几乎一致（2.7405 vs 2.7383 nat）。[arXiv:2607.15105](https://arxiv.org/abs/2607.15105)

4. **7 月 17 日** — arXiv 2607.15218：**PRISM 隐藏状态探针区分文本危险与物理危险**。针对 LLM 作为具身 Agent 规划器时语言安全但物理危险的问题，发现内容危险 (CD) 和物理危险 (PD) 在 LLM 隐藏状态中形成可分离信号。PRISM 用单层 L2 正则化 logistic 探针在 SafeAgentBench 上达 86-88% 准确率，在 PhysicalSafetyBench-1K 上达 99.6% 准确率和 0.7% FPR，而同规模 LLM 评判器误拒 67.8% 的安全任务。[arXiv:2607.15218](https://arxiv.org/abs/2607.15218)

5. **7 月 17 日** — arXiv 2607.15247：**AutoSynthesis 端到端多 Agent 自动元分析系统**。给定自然语言研究问题，自动构建搜索策略、检索文献、筛选研究、评估全文资格、提取定量统计、计算标准化效应量、执行随机效应元分析。支持异质性分析和偏倚风险评估，输出符合 PRISMA 指南的透明报告。筛选 28+ 研究、提取 20+ 定量结论，汇总效应量与专家手动元分析高度一致。[arXiv:2607.15247](https://arxiv.org/abs/2607.15247)

6. **7 月 14 日** — **vLLM v0.25.0/v0.25.1 重大架构更新**。PagedAttention 正式移除（V1 后端标准化）；Model Runner V2 成为所有密集模型默认路径，支持 EVS、实时嵌入、Mamba 混合模型前缀缓存、动态推测解码兼容完整 CUDA graphs；Transformers 建模后端性能追平原生 vLLM，新增 FP8 MoE 支持；新 Streaming Parser Engine 统一工具调用/推理解析框架。v0.25.1 补丁修复 TorchCodec FFmpeg 依赖和混合精度 allreduce 融合导致的输出损坏问题。[GitHub Release](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)

7. **7 月 10-14 日** — **SGLang v0.5.15 GLM-5.2 NVFP4 Blackwell 生产调优**。GLM-5.2 NVFP4 在 8x B300 上达到 500+ tok/s/user（bs=1），4x GB300 达 450 tok/s；Spec V2 成为默认调度器，CUDA-graphable DSA draft-extend 实现零开销调度，端到端 TPS 提升 11%；IndexShare MTP 复用 indexer top-k 跨 draft 步骤，长上下文 draft 步成本降低 1.9 倍；TopK V2 融合 top-k 选择与 page-table 变换，运行时 k 上限 2048。v0.5.15.post1 修复 GLM 5.2 IndexShare 在 PD 分离和上下文并行的稳定性。[GitHub Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.15)

8. **7 月 15-17 日** — **TensorRT-LLM v1.3.0rc21 与 llama.cpp 构建更新**。TensorRT-LLM 重要变更：AutoDeploy 后端正在废弃，转向 PyTorch 后端加速新模型支持。llama.cpp 最新构建（master-fff0e0e）聚焦 OpenCL 平台：新增 MoE Q6_K F32 GEMM 内核、向量化 128-bit 本地内存读写、q4_K 转置缩放因子合并读取，ggml 升级到 0.17.0。[TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc21) | [llama.cpp](https://github.com/ggerganov/llama.cpp)

---

## 💡 深度解读

### 1️⃣ SearchOS：多 Agent 信息检索的操作系统级状态管理

**问题背景：**
随着 Tool-Integrated LLM 的进步，web 搜索已成为信息检索 Agent 的核心能力。但当交互历史增长时，Agent 越来越难以追踪任务进度。当搜索尝试未能产出有用证据时，当前单 Agent 和多 Agent 系统容易陷入重复循环，浪费搜索预算并最终损害输出的质量和完整性。核心问题是：如何让多 Agent 协作搜索具有鲁棒的状态管理，避免重复失败和预算耗尽？

**核心思路/原理：**
SearchOS 将开放域信息检索建模为**关系模式补全**（relational schema completion）：Agent 发现实体、跨链接表填充属性、每个值锚定到源证据。核心创新是 Search-Oriented Context Management (SOCM)，将搜索状态外化为四类持久化制品：(1) Frontier Task——当前待解决的任务边界；(2) Evidence Graph——已发现的实体-属性-证据图结构；(3) Coverage Map——已覆盖和未覆盖的属性空间；(4) Failure Memory——失败搜索模式的记录，避免重复尝试。在此之上，pipeline-parallel 调度机制重叠子 Agent 执行，持续用未覆盖缺口任务填充空闲槽位。Search Tool Middleware Harness 拦截模型和工具交互，记录引用证据并响应停滞或预算耗尽。

**数据与证据：**
- 在 WideSearch 和 GISA 两个基准上，SearchOS 在所有评估指标上领先所有单 Agent 和多 Agent 基线
- Pipeline-parallel 调度显著提升利用率和吞吐量
- 层次化技能系统（strategy skills + access skills）跨运行复用成功搜索模式

来源：
- [SearchOS-V1: arXiv:2607.15257](https://arxiv.org/abs/2607.15257)

**工程启示：**
1. **Agent 状态外化是鲁棒性的关键**：将搜索进度从隐式（存在于 Agent 上下文窗口中）转为显式持久化制品，使多 Agent 系统具备可审计、可恢复、可重入的特性。这一设计模式可推广到代码生成、数据分析等其他需要长程探索的 Agent 任务。
2. **Failure Memory 避免重复错误**：记录失败搜索模式并跨运行复用，是降低 API 成本和提高效率的关键机制。在生产环境中，类似机制可用于记录所有工具调用的失败模式，构建组织级"经验库"。
3. **Pipeline-parallel 调度提升资源利用率**：将搜索 Agent 视为"流水线工人"，空闲时立即用未覆盖缺口任务填充，最大化并行搜索效率。这一思路对 RAG 系统的查询并行化有直接借鉴价值。

---

### 2️⃣ HGA 长上下文微调：16GB 显存上的 131K token 推理

**问题背景：**
参数高效微调 (PEFT) 如 QLoRA 降低了模型和优化器内存，但密集注意力仍使长训练序列的显存成本极高。在 16GB 消费级 GPU 上，Qwen3-8B + 4-bit QLoRA 的密集训练在 4K token 时即溢出，严重限制了长上下文模型的微调可及性。

**核心思路/原理：**
HGA (Hierarchical Global Attention) 结合三项技术突破显存瓶颈：(1) **分段反向传播**——仅活跃段保留在显存中参与微分计算；(2) **分层 KV 存储**——旧段 KV 从显存卸载到 RAM 或 NVMe，HGA 为每个查询块加载有界的精确历史 token 集；(3) **分层全局注意力**——通过块摘要 (chunk summaries) 提供全局上下文，同时保持对关键历史 token 的精确访问。这三者协同使得显存占用与序列长度近似解耦——显存非常量但随块摘要温和增长，RAM 和 NVMe 容量决定实际长度上限。

**数据与证据：**
- 16GB Quadro RTX 5000 上密集训练到 2K 即溢出，HGA 达到 16K 训练长度（15.28GB 峰值显存）
- 同一适配器推理跑满 131,072 token
- 2K 训练长度下 HGA 与密集训练困惑度几乎一致（2.7405 vs 2.7383 nat），而原始模型为 2.9541
- HGA 训练速度在 2K 长度已略快于密集训练（217.75 vs 207.02 tokens/s），且随上下文增长优势扩大

来源：
- [Long-Context Fine-Tuning with Limited VRAM: arXiv:2607.15105](https://arxiv.org/abs/2607.15105)

**工程启示：**
1. **消费级 GPU 的长上下文微调变得可行**：16GB 显存即可微调 8B 模型的 131K 上下文能力，极大降低了长上下文微调的硬件门槛。这对个人研究者和中小团队尤其重要。
2. **训练-推理显存解耦的设计思路**：训练时显存占用由活跃段决定，推理时通过 KV 卸载支持超长上下文。这一思路可推广到其他需要长上下文的生产场景，如长文档摘要、代码库分析等。
3. **质量无损的关键**：在共享 2K 训练长度下，HGA 与密集训练的困惑度几乎一致，证明显存优化不以质量为代价。但论文指出生产级 serving 实现仍在开发中，工程化部署需等待优化版本。

---

### 3️⃣ PRISM 与 Plover：Agent 安全的新维度

**问题背景：**
LLM 越来越多地作为具身 Agent 的高级规划器，但语言层面安全的指令在物理世界落地后可能变得不安全。例如，"帮我整理实验台"在语言上完全安全，但如果实验台上有危险化学品，执行动作可能导致事故。现有的 LLM 安全机制主要关注文本层面的内容危险检测，无法覆盖物理世界的接地危险。同时，GUI Agent 在执行过程中可能因动态界面偏离用户意图，需要可干预的机制。

**核心思路/原理：**
**PRISM** 发现内容危险 (CD) 和物理危险 (PD) 在 LLM 隐藏状态中形成可分离信号，通过单层 L2 正则化 logistic 探针在表示层面检测物理危险，而非依赖文本层面的内容审核。这绕过了 LLM 评判器的高误拒率问题。**Plover** 则从交互设计角度解决 GUI Agent 的可控性问题：将任务计划外化为可检查、可编辑的持久化制品，支持截图锚定干预和局部化修正。两者分别从安全检测和交互控制两个维度提升 Agent 的可靠性。

**数据与证据：**
- PRISM 在 SafeAgentBench 上达 86-88% 准确率（FPR 11.7-13.7%），而同规模 LLM 评判器误拒率高达 24.7-39.0%
- 在 PhysicalSafetyBench-1K 上 PRISM 达 99.6% 准确率和 0.7% FPR，LLM 评判器误拒 67.8% 的安全任务
- Plover 的基准失败案例修复实验证明多数 GUI Agent 失败在计划可见时可结构化修复
- PRISM 在 Qwen2.5-3B/7B/14B/32B、Phi-3.5 和 SmolLM2 上均有效

来源：
- [PRISM: arXiv:2607.15218](https://arxiv.org/abs/2607.15218)
- [Plover: arXiv:2607.15193](https://arxiv.org/abs/2607.15193)

**工程启示：**
1. **具身 Agent 安全需要表示级检测**：文本安全 ≠ 物理安全。部署具身 Agent 时，应在隐藏状态层面部署物理危险检测探针，而非仅依赖文本层面的内容审核。PRISM 的 99.6% 准确率和 0.7% FPR 使其成为可行的生产安全组件。
2. **GUI Agent 的可干预性设计**：Plover 证明将计划外化为可见、可编辑的制品，可将多数自主失败转化为可修复的结构化问题。这对 RPA 和 GUI 自动化产品的 UX 设计有直接指导意义——"可检查的计划"优于"黑盒执行"。
3. **安全与可控性的双重保障**：在生产 Agent 系统中，应同时部署 PRISM 类的表示级安全探针和 Plover 类的计划可视化机制，形成"安全检测 + 人工干预"的双层保障。

---

## 🔧 开源工具动态

1. **vLLM** — **v0.25.0/v0.25.1（7 月 11-14 日）重大架构更新**。PagedAttention 正式移除，V1 后端标准化为唯一路径。Model Runner V2 成为所有密集模型默认，新增 EVS、实时嵌入、Mamba 混合模型前缀缓存、多模态双向注意力、动态推测解码兼容完整 CUDA graphs。Transformers 建模后端性能追平原生 vLLM，新增 FP8 MoE 支持。新 Streaming Parser Engine 统一工具调用/推理解析。新模型支持 LLaVA-OneVision-2、MOSS-Transcribe-Diarize、GLM-5/DeepSeek-V3.2、MiniMax-M3 流水线并行和 NVFP4。v0.25.1 修复 TorchCodec FFmpeg 和混合精度 allreduce 融合导致的输出损坏。**生产建议**：v0.25 是架构标准化里程碑，建议生产环境升级到 v0.25.1 并验证模型兼容性，特别注意 PagedAttention 移除后自定义 attention 实现的迁移。

2. **SGLang** — **v0.5.15/v0.5.15.post1（7 月 10-14 日）Blackwell 生产调优**。GLM-5.2 NVFP4 在 8x B300 上达 500+ tok/s/user（bs=1），4x GB300 达 450 tok/s。Spec V2 成为默认调度器，CUDA-graphable DSA draft-extend 零开销调度，端到端 TPS +11%。IndexShare MTP 复用 indexer top-k 跨 draft 步骤，长上下文 draft 成本降低 1.9 倍。TopK V2 融合 top-k 与 page-table 变换，k 上限 2048。Indexer prologue 融合 12 kernel→4，bs=1 decode 快 8%。v0.5.15.post1 修复 GLM 5.2 在 PD 分离和上下文并行的稳定性。**与 vLLM 互补**：SGLang 在推测解码和 Blackwell 优化上持续领先，vLLM 在架构标准化和模型覆盖面上更广。

3. **TensorRT-LLM** — **v1.3.0rc21（7 月 15 日）AutoDeploy 后端废弃**。重要变更：AutoDeploy 后端正在废弃，转向 PyTorch 后端加速新模型支持（已用该方法在 Minimax M3 发布首周实现功能支持）。新增 DeepSeek V4 完整支持、Cosmos3 推理器和音频输出、Minimax M3 MXFP8/NVFP4、Gemma 4 12B 统一多模态、Qwen3.5-VL MoE/Dense。注意 v1.3.0rc20 是最后支持 TensorRT 后端的版本，下一版本将移除 TensorRT 后端。**生产建议**：关注后端迁移路径，PyTorch 后端将成为主要支持方向。

4. **llama.cpp** — **最新构建 master-fff0e0e（7 月 17 日）OpenCL 平台优化**。聚焦 OpenCL 后端：新增 MoE Q6_K F32 GEMM 内核 (`kernel_gemm_moe_q6_k_f32_ns`)；MoE dp4a 激活 tile 向量化 128-bit 本地内存读写；q4_K noshuffle 缩放因子转置合并读取。ggml 升级到 0.17.0。llama.cpp 已切换为 nightly 发布模式（master-XXXXXXX tag 格式），无 semver release。**CPU/开放平台推理**：OpenCL 优化持续改善 AMD/Intel GPU 和移动平台的推理性能，MoE 模型支持增强。

5. **MLC LLM** — **无近期发布**。最近 release 仍为 v0.1.dev0（2023 年 4 月）。项目仍在活跃开发中，但发布节奏较慢。**端侧部署**：MLC LLM 在端侧部署和内存优化方面仍有独特价值，但生产环境建议关注 vLLM 和 llama.cpp 的活跃进展。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 18 日*
