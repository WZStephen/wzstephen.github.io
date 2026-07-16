---
layout: post
title: 'Agent 任务复杂度感知、vLLM 0.25 移除 PagedAttention、TensorRT-LLM 支持 DeepSeek V4'
date: 2026-07-16 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期聚焦 AI 推理与 Agent 执行优化：arXiv 论文提出 E3 框架实现任务复杂度感知执行，通过预估-执行-扩展三阶段减少 40% 冗余上下文；vLLM v0.25.0 移除 PagedAttention、Model Runner V2 成为默认路径、Transformers 后端达到原生速度；TensorRT-LLM v1.3.0rc21 新增 DeepSeek V4、Cosmos3、Minimax M3 支持；SGLang v0.5.15 修复 GLM 5.2 推理问题；llama.cpp b10034 优化 Adreno MoE 内核；MLC LLM 推进至 v0.26.dev0 端侧部署；MemOps 基准测试系统评估长对话记忆生命周期操作。

---

## 🔥 今日看点

1. **7 月 16 日** — arXiv 2607.13034：**E3 框架实现 Agent 任务复杂度感知执行**。提出 Agent Cognitive Redundancy Ratio (ACRR) 指标，通过 Estimate-Execute-Expand 三阶段策略，在 MSE-Bench 基准上减少 40% 冗余上下文读取，同时保持 95% 任务完成率。[arXiv:2607.13034](https://arxiv.org/abs/2607.13034)

2. **7 月 14 日** — **vLLM v0.25.0 重大架构更新**。Model Runner V2 成为所有密集模型默认路径；PagedAttention 正式移除（V1 后端标准化）；Transformers 建模后端性能追平原生 vLLM；新增流式解析引擎支持 Kimi k2.5/k2.6/k2.7、DeepSeek V4 推理格式；通用推测解码支持异构词表 (TLI)。v0.25.1 补丁修复 TorchCodec FFmpeg 依赖和混合精度 allreduce 融合问题。[GitHub Release](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)

3. **7 月 14 日** — **TensorRT-LLM v1.3.0rc21 新增模型支持**。添加 DeepSeek V4 完整支持、Cosmos3 推理器和音频输出、Minimax M3 MXFP8/NVFP4 检查点、Gemma 4 12B 统一多模态、Qwen3.5-VL MoE 和 Dense 变体、Qwen3.6 NVFP4。注意：AutoDeploy 后端正在废弃，转向 PyTorch 后端加速模型支持。[GitHub Release](https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc21)

4. **7 月 14 日** — **SGLang v0.5.15.post1 GLM 5.2 专项修复**。修复 DSA 模型在非 CUDA/HIP 设备启动、FlashInfer CUDA 12 依赖、FP4 MoE 长输入 NaN 输出、GLM 5.2 IndexShare 在 PD 分离和上下文并行设置的问题。[GitHub Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.15.post1)

5. **7 月 15 日** — **llama.cpp b10034 Adreno GPU 优化**。排除 Adreno A7x 设备使用 MoE 内核（编译器误编译导致权重损坏）；排除 A6x 和未知 Adreno 设备的 MoE 权重重新打包。提供 macOS Apple Silicon、iOS XCFramework、Ubuntu 多后端（CPU/Vulkan/ROCm/OpenVINO/SYCL）预编译包。[GitHub Release](https://github.com/ggml-org/llama.cpp/releases/tag/b10034)

6. **7 月** — **MLC LLM 推进至 v0.26.dev0**。最新标签 v0.26.dev0 显示持续开发，v0.20.0 为最新稳定版。端侧部署持续优化内存占用和推理速度。[GitHub Tags](https://github.com/mlc-ai/mlc-llm/tags)

7. **7 月 16 日** — arXiv 2607.12893：**MemOps 长对话记忆生命周期基准**。提出现有基准仅通过下游 QA 评估记忆，混淆了记忆失败原因（遗漏事实、错误绑定、依赖过期值）。MemOps 将记忆建模为显式操作生命周期（记住、更新、遗忘），在动态长程交互中更准确评估记忆一致性。[arXiv:2607.12893](https://arxiv.org/abs/2607.12893)

8. **7 月 16 日** — arXiv 2607.12815：**视觉语言模型推理中的视觉访问边界**。研究 Chain-of-Thought 是否需要持续访问图像 token，提出 Visual Access Sweep 因果干预方法，发现 CoT 主要在早期前向传播中获取的视觉信息上操作，而非持续依赖图像 token。[arXiv:2607.12815](https://arxiv.org/abs/2607.12815)

---

## 💡 深度解读

### 1️⃣ Agent 任务复杂度感知执行：E3 框架

**问题背景：**
当前 LLM Agent 在多步骤工程任务中缺乏任务复杂度感知能力，常采用"最大上下文优先"策略——重新读取已见过的文件和依赖，将一行编辑变成小规模代码库审计。这导致执行效率低下，token 消耗冗余。

**核心思路/原理：**
E3 (Estimate, Execute, Expand) 框架引入任务感知执行范围估计：在提交预算前判断任务难度、所需信息和最短可靠路径。具体流程：
- **Estimate**：Agent 预估初始操作点（需要多少上下文）
- **Execute**：执行最小可行路径（只读取必要信息）
- **Expand**：仅当验证失败时才扩展范围

论文形式化了"最小充分执行"(Minimum-Sufficient Execution) 和 Agent Cognitive Redundancy Ratio (ACRR) 指标。

**数据与证据：**
- 在 MSE-Bench 基准上，E3 减少 40% 冗余上下文读取
- 任务完成率保持 95%（与全上下文策略相当）
- 平均执行步骤减少 35%

来源：
- [Do AI Agents Know When a Task Is Simple?: arXiv:2607.13034](https://arxiv.org/abs/2607.13034)

**工程启示：**
1. **生产环境 Agent 应实现复杂度预估模块**：在执行前分析任务类型、历史相似任务、所需上下文规模，动态调整预算
2. **ACRR 指标可用于监控 Agent 效率**：过高的认知冗余比表明 Agent 在"过度准备"，需要优化提示或工具调用策略
3. **Expand 阶段应有明确触发条件**：仅在验证失败（如测试不通过、输出不符合预期）时才扩展上下文，避免"以防万一"的冗余读取

---

### 2️⃣ vLLM v0.25.0：架构标准化与性能统一

**问题背景：**
vLLM 长期存在多后端并存问题：PagedAttention（V0）、Model Runner V1、Transformers 后端性能差异大，用户需要在速度、兼容性、模型支持间权衡。维护多路径也增加了开发复杂度。

**核心思路/原理：**
v0.25.0 完成架构标准化：
- **Model Runner V2 成为默认**：构建在 V1 基础上，新增 EVS 支持、实时嵌入、Mamba 混合模型前缀缓存、多模态前缀双向注意力、动态推测解码兼容完整 CUDA 图
- **PagedAttention 移除**：V1/MRv2 后端成为标准路径后，旧实现删除
- **Transformers 后端性能对齐**：通过优化达到与原生 vLLM 相当的速度，同时获得 FP8 MoE 支持、CUDA 图修复

新增流式解析引擎统一工具调用/推理解析，支持 Kimi k2.5/k2.6/k2.7、DeepSeek V4 格式。Rust 前端成熟：HTTPS/mTLS、DP 监控、profiler 控制路由。

**数据与证据：**
- 558 commits from 232 contributors (64 new)
- Transformers 后端速度与原生 vLLM 持平（[PR #47187](https://github.com/vllm-project/vllm/pull/47187)）
- 新模型：LLaVA-OneVision-2、Unlimited OCR、MOSS-Transcribe-Diarize、GLM-5/DeepSeek-V3.2、MiniMax-M3 流水线并行
- 推测解码：通用异构词表支持 (TLI)、新 DSpark/DFlash 草稿模型

来源：
- [vLLM v0.25.0 Release](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)
- [vLLM v0.25.1 Patch](https://github.com/vllm-project/vllm/releases/tag/v0.25.1)

**工程启示：**
1. **升级到 v0.25.x 可获得统一性能体验**：无需手动选择后端，MRv2 自动处理大多数场景
2. **PagedAttention 移除意味着旧配置需要迁移**：如果依赖 PagedAttention 特定行为，需验证 MRv2 兼容性
3. **流式解析引擎简化多模型部署**：统一接口处理不同推理格式，减少自定义解析逻辑
4. **生产环境建议**：先在测试环境验证 v0.25.1 补丁（修复混合精度 allreduce 问题），再灰度上线

---

### 3️⃣ TensorRT-LLM v1.3.0rc21：新模型快速支持与已知问题

**问题背景：**
NVIDIA TensorRT-LLM 需要在模型发布后快速提供功能支持，同时保持推理性能优化。新模型（DeepSeek V4、Minimax M3、Qwen3.5/3.6）需要快速集成，但多 GPU 配置和量化路径容易出现稳定性问题。

**核心思路/原理：**
v1.3.0rc21 重点：
- **快速模型支持**：DeepSeek V4、Cosmos3、Minimax M3 MXFP8/NVFP4、Gemma 4 12B、Qwen3.5-VL、Qwen3.6 NVFP4
- **AutoDeploy 后端废弃**：转向 PyTorch 后端，通过 Agent 方法加速模型支持（MiniMax M3 在模型发布一周内获得功能支持）
- **API 破坏性变更**：移除旧 TensorRT 后端模块、移动多模态参数、重命名服务器参数

**数据与证据：**
- 已知问题集中在 DeepSeek V3.2/V3 Lite、Llama 3.x、GPT-OSS、MiniMax M3、Qwen3.5
- 多 GPU NVFP4 配置在 B300 上出现精度失败
- 分离式服务 (Disaggregated Serving) 在 DeepSeek V3 Lite + Helix 上产生错误输出
- torch.compile + CUDA 图 + 重叠调度器在 H100 上触发 CUDA 启动失败

来源：
- [TensorRT-LLM v1.3.0rc21 Release](https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc21)

**工程启示：**
1. **新模型支持优先使用 PyTorch 后端**：AutoDeploy 废弃，PyTorch 后端获得更快迭代
2. **生产环境谨慎使用 NVFP4 多 GPU 配置**：B300 上的精度问题需要验证
3. **分离式服务需充分测试**：DeepSeek V3 Lite 在 Helix 设置下的输出错误可能影响生成质量
4. **关注 CUDA 图与 torch.compile 兼容性**：H100 上的启动失败需要 NVIDIA 修复或规避

---

## 🔧 开源工具动态

1. **vLLM** — **v0.25.1 (7 月 14 日)**：v0.25.0 补丁修复。v0.25.0 是重大更新：Model Runner V2 成为默认、PagedAttention 移除、Transformers 后端性能对齐原生。新增流式解析引擎支持 Kimi k2.5/k2.6/k2.7 和 DeepSeek V4 推理格式。生产环境建议升级到 v0.25.1 修复混合精度 allreduce 问题。[GitHub](https://github.com/vllm-project/vllm/releases/tag/v0.25.1)

2. **SGLang** — **v0.5.15.post1 (7 月 14 日)**：GLM 5.2 专项修复。解决 DSA 模型启动、FlashInfer 依赖、FP4 MoE 长输入 NaN、GLM 5.2 IndexShare 在 PD 分离和上下文并行的问题。与 vLLM 互补：SGLang 在结构化生成和 RadixAttention 前缀缓存上有优势。[GitHub](https://github.com/sgl-project/sglang/releases/tag/v0.5.15.post1)

3. **TensorRT-LLM** — **v1.3.0rc21 (7 月 14 日)**：预发布版本。新增 DeepSeek V4、Cosmos3、Minimax M3、Gemma 4、Qwen3.5-VL、Qwen3.6 支持。AutoDeploy 后端废弃，转向 PyTorch 后端。已知问题：DeepSeek V3.2 多 GPU NVFP4、分离式服务、torch.compile 兼容性。NVIDIA 硬件优化和 FP8/NVFP4 量化持续演进。[GitHub](https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc21)

4. **llama.cpp** — **b10034 (7 月 15 日)**：夜间构建版本。修复 Adreno A7x MoE 内核编译器误编译问题，排除 A6x 和未知 Adreno 设备的 MoE 权重重新打包。提供 macOS Apple Silicon、iOS XCFramework、Ubuntu 多后端（CPU/Vulkan/ROCm 7.2/OpenVINO 2026.2/SYCL）预编译包。GGUF 格式持续优化，CPU 推理性能稳定。[GitHub](https://github.com/ggml-org/llama.cpp/releases/tag/b10034)

5. **MLC LLM** — **v0.26.dev0 (开发中)**：最新开发标签，v0.20.0 为最新稳定版。端侧部署持续优化内存占用和推理速度。MLC LLM 在移动设备（iOS/Android）和边缘设备上的部署能力持续增强，支持模型量化和硬件加速。[GitHub Tags](https://github.com/mlc-ai/mlc-llm/tags)

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 16 日*
