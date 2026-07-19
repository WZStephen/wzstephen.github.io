---
layout: post
title: 'llama.cpp DFlash KV Cache 旋转优化、DeepSeek-V4 路由表量化修复、On-Policy Delta 蒸馏'
date: 2026-07-19 09:00:00 +0800
categories: [ai-technical-report]
---

> 注：今日为周日（7 月 19 日），arXiv 未更新新论文，本期论文均来自 cs.LG 等关联分类的近期论文池。llama.cpp 在周末持续发布三个新构建（b10066-b10068），聚焦 DFlash KV Cache 旋转注入、DeepSeek-V4 路由表量化排除和 OpenCL MoE 内核优化。vLLM v0.25.1 和 SGLang v0.5.15.post1 维持上周发布版本。本期深度解读聚焦三项主题：On-Policy Delta Distillation 提出教师-基模型差值信号替代直接分布模仿的蒸馏范式；MXsim 为 OCP MX 块浮点格式提供 MATLAB 仿真工具链，降低混合精度算法开发门槛；BTC 情绪分类器融合链上数据与社交文本解码市场情绪。以下内容基于 arXiv、GitHub Releases 等公开数据。

---

## 🔥 今日看点

1. **7 月 18 日** — llama.cpp b10068：**DFlash KV Cache 旋转注入优化**。针对 DFlash 模型使用量化 KV Cache 时的精度问题，新增 K/V 量化场景下的旋转注入机制。由 Georgi Gerganov 协同开发，优化了 dflash.cpp 中的 KV Cache 处理流程，改善长序列推理质量。[GitHub Release](https://github.com/ggml-org/llama.cpp/releases/tag/b10068)

2. **7 月 18 日** — llama.cpp b10067：**DeepSeek-V4 路由表量化排除修复**。DeepSeek-V4 的 ffn_gate_tid2eid 张量是 i32 类型的 token-id → expert-id 索引表，非权重张量，此前未被排除在量化流程之外。llama-quantize 错误地对其进行了量化处理，可能导致路由精度损失。本版本修复排除列表，确保索引表保持原始 i32 精度。[GitHub Release](https://github.com/ggml-org/llama.cpp/releases/tag/b10067)

3. **7 月 17 日** — llama.cpp b10066：**OpenCL MoE Q6_K GEMM 内核与 ggml 0.17.0**。新增 OpenCL 平台 MoE Q6_K F32 NS GEMM 内核（kernel_gemm_moe_q6_k_f32_ns），向量化 128-bit 本地内存读写，q4_K 转置缩放因子合并读取。ggml 核心库升级到 0.17.0 版本。[GitHub Release](https://github.com/ggml-org/llama.cpp/releases/tag/b10066)

4. **7 月 17 日** — arXiv 2607.15161：**On-Policy Delta Distillation**。提出以"delta 信号"（教师模型与基模型在指令微调前的输出分布差值）替代直接模仿教师输出分布的传统蒸馏方法。Delta 信号捕捉的是教师相对于基模型的增量知识，而非绝对分布，减轻了对教师模型绝对校准的依赖。在 RL 后训练场景中提供 token 级监督信号。[arXiv:2607.15161](https://arxiv.org/abs/2607.15161)

5. **7 月 15 日** — arXiv 2607.12915：**MXsim OCP MX 块浮点格式仿真器**。发布 MXsim v0.1 MATLAB 库，基于 CPFloat 自定义精度浮点仿真器构建，支持 OCP MX 块浮点算术和格式的仿真。面向混合精度算法开发社区，在缺乏最新 NVIDIA/AMD GPU 硬件时也能进行 MX 格式实验。对 FP8/FP4 量化研究有重要工具价值。[arXiv:2607.12915](https://arxiv.org/abs/2607.12915)

6. **7 月 17 日** — arXiv 2607.15258：**BTC 市场情绪链上解码分类器**。提出融合链上交易数据、BTC 历史价格和 Twitter 每日情绪分类的市场情绪分析方法。区别于价格预测模型，该工作聚焦于解释市场情绪：将情绪趋势与链上/金融指标归一化后构建数据集，用于细节级市场情绪解码。为加密货币市场的链上分析提供新范式。[arXiv:2607.15258](https://arxiv.org/abs/2607.15258)

7. **7 月 14 日** — **vLLM v0.25.1 补丁发布**。在 v0.25.0 基础上修复两个关键 bug：(1) 无系统 FFmpeg 时 TorchCodec 阻塞模型加载（#47888）；(2) 混合精度 allreduce 融合导致输出损坏。v0.25.0 核心变更保持不变：Model Runner V2 成默认、PagedAttention 移除、Transformers 后端追平原生性能。[GitHub Release](https://github.com/vllm-project/vllm/releases/tag/v0.25.1)

8. **7 月 14 日** — **SGLang v0.5.15.post1 GLM-5.2 稳定性补丁**。修复 GLM-5.2 IndexShare 在 PD 分离和上下文并行场景的稳定性，修复非 CUDA/HIP 设备上 DSA 模型启动，修复 CUDA 12 镜像上 flashinfer 依赖，修复 flashinfer trtllm FP4 MoE 内核在长输入时的 NaN 输出。GLM-5.2 NVFP4 在 8x B300 上维持 500+ tok/s/user。[GitHub Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.15.post1)

---

## 💡 深度解读

### 1️⃣ On-Policy Delta Distillation：从绝对分布模仿到增量知识蒸馏

**问题背景：**
On-policy 蒸馏是 RL 后训练中替代奖励模型约束的方法，通过教师模型提供 token 级监督信号。传统 on-policy 蒸馏直接模仿教师的输出分布（KL 散度最小化），但这种方法存在根本问题：教师模型的绝对分布可能受到其预训练偏差、过度自信等因素影响，直接模仿可能导致学生模型学到不准确的校准信息。核心问题是：如何从教师模型中提取"增量知识"而非"绝对知识"进行蒸馏？

**核心思路/原理：**
On-Policy Delta Distillation 提出以**delta 信号**作为蒸馏目标。Delta 信号定义为：教师模型在指令微调后的输出分布 减去 同一教师模型的基模型（指令微调前）的输出分布。这个差值捕捉的是教师相对于基模型学到的增量知识——即指令微调"教会"了教师什么。学生模型通过最大化与 delta 信号的一致性来学习，而非直接模仿教师的绝对分布。这种方法的优势在于：(1) 消除了教师模型预训练阶段引入的偏差；(2) 聚焦于指令微调带来的行为变化，更具针对性；(3) 对学生模型的绝对校准要求降低。

**数据与证据：**
- 论文系统研究了 delta 蒸馏在不同设置下的表现，包括不同教师-学生配对、不同 RL 训练阶段
- 相比传统 KL 散度蒸馏，delta 蒸馏在多个基准上展示了更稳定的性能提升
- Delta 信号的信噪比高于绝对分布信号，因为教师基模型的共性问题被差分消除

来源：
- [On-Policy Delta Distillation: arXiv:2607.15161](https://arxiv.org/abs/2607.15161)

**工程启示：**
1. **蒸馏目标的选择比教师模型的选择更重要**：传统蒸馏聚焦于选择更强的教师模型（更大、更强），但 delta 蒸馏表明，从同一教师中提取增量知识可能比选择更强教师更有效。这为生产环境中的模型压缩提供了新思路——不需要追求最大的教师模型，而是关注教师从基模型到指令模型的变化量。
2. **与 RLHF/DPO 流程的兼容性**：Delta 信号天然适配 RLHF 和 DPO 的后训练流程，因为它直接对应"指令微调带来的行为变化"这一核心目标。在生产 pipeline 中，可以将 delta 蒸馏作为 RLHF 的替代或补充，减少对奖励模型的依赖。
3. **降低教师模型部署成本**：由于 delta 蒸馏对学生模型的绝对校准要求更低，可以在更小的学生模型上实现更好的蒸馏效果，降低推理部署成本。

---

### 2️⃣ MXsim：OCP MX 块浮点格式的仿真工具链

**问题背景：**
随着 NVIDIA Blackwell 和 AMD MI300 系列 GPU 对 FP8/FP4 等低精度格式的原生硬件支持，混合精度训练和推理已成为大模型部署的核心技术路线。OCP（Open Compute Project）正在推进 MX 块浮点标准，定义了 E2M1、E3M0 等微格式和块缩放规则。然而，开发者在缺乏最新 GPU 硬件时难以进行 MX 格式的实验和算法开发，且现有仿真工具对 OCP MX 标准的支持有限。

**核心思路/原理：**
MXsim v0.1 是基于 CPFloat 自定义精度浮点仿真器构建的 MATLAB 库，专门针对 OCP MX 块浮点格式。核心功能包括：(1) 模拟 OCP MX 标准定义的各种微格式（E2M1、E3M0、E4M3 等）的算术运算；(2) 实现块缩放（block scaling）机制——多个元素共享一个缩放因子，降低存储开销但引入量化误差；(3) 支持自定义变体，允许研究者实验非标准的 MX 配置。MXsim 的 MATLAB 接口使其易于与现有的数值线性代数工具链集成。

**数据与证据：**
- MXsim v0.1 提供完整的 OCP MX 格式仿真，包括前向传播和反向传播的精度模拟
- 基于 CPFloat 核心引擎，支持 IEEE 754 标准格式和自定义格式的混合仿真
- MATLAB 实现使其易于集成到学术研究和原型开发流程中

来源：
- [MXsim: arXiv:2607.12915](https://arxiv.org/abs/2607.12915)

**工程启示：**
1. **降低量化研究的硬件门槛**：在 GPU 供应紧张的时期，研究者可以通过 MXsim 在 CPU 上完成 MX 格式的算法验证和精度评估，无需等待 GPU 资源。这对学术界和中小团队尤为重要。
2. **标准化验证工具**：随着 OCP MX 标准的推进，MXsim 可作为标准合规性验证的参考实现。生产环境中的量化算法开发者可以用 MXsim 验证其实现是否符合 OCP 标准定义。
3. **与 vLLM/SGLang 的互补**：vLLM v0.25 新增了 MXFP4 支持（MiniMax-M3），SGLang v0.5.15 调优了 GLM-5.2 NVFP4。MXsim 可以在部署前帮助开发者评估不同 MX 格式对其模型的精度影响，指导生产环境的格式选择。

---

### 3️⃣ DeepSeek-V4 路由表量化修复：MoE 模型工程化的细节陷阱

**问题背景：**
DeepSeek-V4 采用 MoE（Mixture of Experts）架构，其路由机制依赖 ffn_gate_tid2eid 张量——一个 i32 类型的 token-id → expert-id 索引表。在 llama.cpp 的量化流程中，这个索引表未被正确排除在量化操作之外，导致离散的路由索引被错误地进行了浮点量化处理。这可能引起路由精度损失，表现为 MoE 模型在量化后性能异常下降。

**核心思路/原理：**
修复方案是在 llama-quantize 的名称排除列表中增加 ffn_gate_tid2eid 张量。此前已有的排除项包括 ffn_gate_inp.weight（标准的 MoE 门控权重），但 DeepSeek-V4 使用的 token-id 到 expert-id 的直接映射表是新增的张量类型，未被历史排除列表覆盖。修复后，该索引表在量化过程中保持原始 i32 精度，确保路由决策不受量化噪声影响。

**数据与证据：**
- 此修复针对 llama.cpp build b10067 及之后的版本
- DeepSeek-V4 的 MoE 路由精度对量化敏感，错误量化索引表可能导致 expert 选择不准确
- 该修复与 b10068 的 DFlash KV Cache 旋转注入修复协同，共同提升 MoE 模型的量化推理质量

**工程启示：**
1. **MoE 模型量化的特殊挑战**：MoE 架构引入了标准 dense 模型不存在的张量类型（路由表、expert 索引等），这些张量的量化处理需要专门的排除规则。生产环境中的 MoE 模型部署应确保量化流程正确识别并排除所有非权重张量。
2. **llama.cpp 的快速迭代节奏**：llama.cpp 在 7 月 17-18 日连续发布三个构建版本（b10066-b10068），展现了极高的修复速度。使用 llama.cpp 进行 MoE 模型推理的用户应及时更新到 b10067+ 版本，避免路由表量化导致的精度损失。
3. **GGUF 格式的 MoE 支持成熟度**：随着 DeepSeek-V4 等复杂 MoE 模型的广泛部署，GGUF 格式对 MoE 架构的支持正在快速完善。量化、路由、expert 选择等关键环节的工程化程度持续提升。

---

## 🔧 开源工具动态

1. **vLLM** — v0.25.1（7 月 14 日）为补丁版本，修复 TorchCodec FFmpeg 依赖和混合精度 allreduce 融合输出损坏问题。v0.25.0 核心变更维持：Model Runner V2 成所有密集模型默认路径，PagedAttention 正式移除（V1 后端标准化），Transformers 建模后端性能追平原生 vLLM 并新增 FP8 MoE 支持。生产环境建议：从 v0.24 升级的用户应优先验证 MRv2 在特定模型上的兼容性，再全量切换。

2. **SGLang** — v0.5.15.post1（7 月 14 日）为 GLM-5.2 稳定性补丁，修复 IndexShare 在 PD 分离和上下文并行的稳定性、flashinfer trtllm FP4 MoE 内核长输入 NaN 问题。v0.5.15 核心亮点：GLM-5.2 NVFP4 在 8x B300 上达 500+ tok/s/user，Spec V2 成为默认调度器实现零开销调度，IndexShare MTP 长上下文 draft 步成本降低 1.9 倍。与 vLLM 形成互补：vLLM 侧重通用 serving 标准化，SGLang 侧重新硬件（Blackwell）深度调优。

3. **TensorRT-LLM** — v1.3.0rc21（7 月 15 日）为最新 RC 版本。重要架构变更：AutoDeploy 后端正在废弃，转向 PyTorch 后端加速新模型支持。NVIDIA 硬件优化和 FP8 量化持续推进。生产环境建议：关注 AutoDeploy 废弃时间表，提前规划迁移路径。

4. **llama.cpp** — 周末连续三个新构建。**b10068**（7 月 18 日）：DFlash KV Cache 旋转注入优化，改善量化 KV Cache 在 DFlash 模型中的推理质量。**b10067**（7 月 18 日）：DeepSeek-V4 ffn_gate_tid2eid 路由表排除量化修复，避免 i32 索引表被错误量化。**b10066**（7 月 17 日）：OpenCL MoE Q6_K F32 GEMM 内核、128-bit 向量化内存读写、ggml 升级到 0.17.0。GGUF 格式对 MoE 模型的支持持续完善。

5. **MLC LLM** — v0.1.dev0 为最新 release（2023 年 4 月），项目已进入持续开发模式而非定期发布。端侧部署和内存占用优化通过主分支持续迭代。生产环境建议：关注主分支的 WebGPU 和 Vulkan 后端更新，端侧部署用户应从源码构建获取最新优化。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI/cs.LG/cs.MS、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 19 日（周日，arXiv 未更新，论文来自关联分类近期池）*
