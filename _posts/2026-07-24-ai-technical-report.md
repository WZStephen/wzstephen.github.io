---
layout: post
title: 'PoTRE 多拓扑推理集成 HLE 新 SOTA、PRO-LONG 程序化记忆 ARC-AGI-3 突破、vLLM v0.25 移除 PagedAttention、SGLang GLM-5.2 500 tok/s'
date: 2026-07-24 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期覆盖 2026 年 7 月下旬 AI 推理与 Agent 领域的重要进展。PoTRE 提出多拓扑推理集成框架，通过四类异构 Agent（对抗精化、层级规划、频谱搜索、直链推理）与任务自适应聚合层，在 Humanity's Last Exam 上达到 49.92% 的新 SOTA，使用与同构基线相当甚至更少的推理 token；PRO-LONG 以程序化记忆为核心，在 ARC-AGI-3 公开游戏集上将基础编码 Agent 平均提升 18 个百分点，最佳 Fable 5 达到 97.4% best@2，同时 token 用量减少 4-6 倍；CUSUM 形状推理时监控与重解码机制为量化小模型提供选择性监控修复路径；MGT-B 在 MATH-500 上实现 +4.5% 准确率提升。开源框架方面，vLLM v0.25.0 移除 PagedAttention、Model Runner V2 成为默认路径、Transformers 后端追平原生性能；SGLang v0.5.15 为 GLM-5.2 NVFP4 在 Blackwell 上实现 500+ tok/s 生产调优，Spec V2 默认开启带来 +11% 端到端吞吐；llama.cpp 持续高频迭代至 b10099，新增 CUDA NVFP4 与 DeepSeek4 模板修复。

---

## 🔥 今日看点

1. **7 月 22 日** — PoTRE：多拓扑推理集成在 HLE 创 SOTA。该框架将推理分解为四类异构 Agent（对抗精化、层级规划、频谱搜索、直链推理），通过任务自适应聚合层动态调和，在 ARC-AGI-2、Humanity's Last Exam（49.92%）和 PRBench Finance 上达到 SOTA，使用相似或更少推理 token（[arXiv:2607.20268](https://arxiv.org/abs/2607.20268)）。

2. **7 月 22 日** — PRO-LONG：程序化记忆使长时域推理突破。框架保留完整结构化交互日志，利用编码 Agent 高效搜索历史，在 ARC-AGI-3 公开游戏集上平均提升 18 个百分点，最佳配置达 97.4% best@2，成本仅 $1,750，token 用量比专用 harness 减少 4.2-5.8 倍（[arXiv:2607.20064](https://arxiv.org/abs/2607.20064)）。

3. **7 月 22 日** — MGT-B：CUSUM 形状监控引导量化小模型重解码。基于 e-CUSUM 控制器的外部监控器将预采样不确定性和退化特征映射为经验尾概率，通过混合赌注因子积累与 CUSUM 形状重置触发回滚，在 MATH-500 的 467 对种子匹配集上实现 +4.5% 准确率提升（McNemar p=0.0008）（[arXiv:2607.20129](https://arxiv.org/abs/2607.20129)）。

4. **7 月 21 日** — SoftReason：全可微神经软符号演绎推理架构。将演绎状态表示为候选常元和谓词上的局部软解释张量，通过谓词定义嵌入与潜组合通道形成软体-谓词混合，在知识感知视觉问答（KVQA）上实现端到端感知定位与可微演绎闭包（[arXiv:2607.20402](https://arxiv.org/abs/2607.20402)）。

5. **7 月 22 日** — GSEM：图结构化经验记忆用于多 Agent 动态制造协调。将历史协调片段编码为异质关系图，捕获任务依赖、机器状态与协作模式，在动态柔性作业车间调度基准上减少 4.1%-10.0% 制造周期、33%-38% 适应时间（[arXiv:2607.19985](https://arxiv.org/abs/2607.19985)）。

6. **7 月 11 日** — vLLM v0.25.0 重大更新。Model Runner V2 成为所有稠密模型默认路径；PagedAttention 被完全移除；Transformers 建模后端追平原生 vLLM 性能并新增 FP8 MoE 支持；新增 LLaVA-OneVision-2、Unlimited OCR、MOSS-Transcribe-Diarize 等模型；通用异构词汇推测解码（TLI）与新 DSpark/DFlash 起草器（[GitHub Release](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)）。

7. **7 月 10 日** — SGLang v0.5.15 发布。GLM-5.2 NVFP4 在 Blackwell 上实现 500+ tok/s/user（8x B300）和 450 tok/s（4x GB300）；Spec V2 默认开启，端到端吞吐 +11%；IndexShare MTP 将索引 top-k 跨起草步骤复用，长上下文起草成本降低 1.9 倍；TopK V2 运行时 k 最高 2048；新增 Hunyuan 3、Qwen3.6 NVFP4、Baidu Unlimited-OCR 等模型支持（[GitHub Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.15)）。

8. **7 月 23 日** — llama.cpp b10099。CUDA NVFP4 W4A4 激活量化改进，hexagon 激活算子优化（geglu 微核），推测类型自动推断支持 mtp-/dflash-/eagle3- sidecar，DeepSeek4 模板修复，PowerPC 后端 AIX 支持（[GitHub Release](https://github.com/ggml-org/llama.cpp/releases/tag/b10099)）。

---

## 💡 深度解读

### 1️⃣ 异构推理集成与程序化记忆：Agent 推理的两条突破路径

**问题背景：**
复杂推理任务需要长时域规划、迭代纠错与多视角验证。同构推理方法（如单一 CoT 流或单一 Agent 类型）在面对新颖抽象或严格领域约束时表现脆弱。ARC-AGI-3 等长时域基准进一步暴露了 LLM Agent 在持续感知、推理与探索中的局限。

**核心思路/原理：**
- **PoTRE（Poly-Topological Reasoning Ensembles）** 将推理分解为四类异构 Agent：对抗精化 Agent（通过对抗视角发现逻辑漏洞）、层级规划 Agent（将任务分解为子目标层次）、频谱搜索 Agent（在解空间中多尺度探索）、直链推理 Agent（直接 CoT 推导）。最终的任务自适应聚合层根据问题特征动态选择最终候选、语义综合或神经符号验证。
- **PRO-LONG** 采用极简但高效的策略：保留完整的结构化交互日志（程序化记忆），利用编码 Agent 的能力高效搜索历史。关键在于不丢弃任何环境观察，而是将检索成本转移给擅长代码搜索的 Agent。

**数据与证据：**
- PoTRE 在 Humanity's Last Exam 上达到 49.92% 准确率，超过此前最佳官方分数，且使用与同构基线相似或更少的推理 token
- PRO-LONG 在 ARC-AGI-3 公开游戏集上平均提升 18.0 个百分点，最佳配置（Fable 5）达到 97.4% best@2，总成本仅 $1,750，token 用量比专用 harness 减少 4.2-5.8 倍
- PoTRE 在 ARC-AGI-2 和 PRBench Finance 上同样达到 SOTA

来源：
- [PoTRE: Test-Time Reasoning inspired by Cognitive Heterogeneity: arXiv:2607.20268](https://arxiv.org/abs/2607.20268)
- [PRO-LONG: Programmatic Memory Enables Long-Horizon Reasoning: arXiv:2607.20064](https://arxiv.org/abs/2607.20064)

**工程启示：**
1. **异构性优于同质性**：PoTRE 证明四类异构 Agent 的组合可以用更少 token 超越同构基线，表明推理多样性的价值高于单一推理流的扩展。
2. **程序化记忆是长时域推理的实用解**：PRO-LONG 的极简策略——保留全部日志、让编码 Agent 搜索——比复杂的上下文管理方案更高效，这对构建长时域 Agent 系统有直接指导意义。
3. **成本效率可兼得**：PRO-LONG 以 $1,750 总成本达到 97.4% best@2，说明通过智能记忆管理而非暴力扩展，可以在成本和性能之间取得更好平衡。
4. **任务自适应聚合是关键**：PoTRE 的聚合层不是简单投票，而是根据问题特征动态选择综合策略，这对多 Agent 系统的设计有重要参考价值。

---

### 2️⃣ 推理时监控与选择性修复：量化小模型的实用优化路径

**问题背景：**
量化小模型在推理时容易陷入长重复或无效轨迹，而推理时计算资源通常在不观察轨迹发展的情况下分配。如何在推理过程中检测退化并及时干预，是提升量化模型实际可用性的关键问题。

**核心思路/原理：**
MGT-B（Monitoring-Guided Test-time Backtracking）在 e-CUSUM 控制器基础上发展，将重叠窗口的预采样不确定性和退化特征映射为位置条件经验尾概率，通过混合赌注因子积累与 CUSUM 形状重置检测异常。触发警报后，系统估计回滚点、恢复 token 和 KV Cache 状态，执行约束重解码。这是一种选择性监控修复机制——仅在检测到退化时干预，正常输出与 vanilla 解码完全一致。

**数据与证据：**
- 在 MATH-500 的 467 对种子匹配集上，准确率从 146/467 提升至 167/467（+4.50 个百分点，McNemar p=0.0008）
- 240 对时间审计集上从 82/240 提升至 88/240（+2.50 个百分点，但 McNemar p=0.2632，统计不显著）
- 151 条警报轨迹中包含 29 次修正和 8 次回退，316 条无警报输出与 vanilla 完全相同
- 作者明确指出该结果非确认性分析，经验因子未被证明为有效 e-process

来源：
- [CUSUM-Shaped Inference-Time Monitoring and Targeted Re-Decoding for Quantized Small Language Model Reasoning: arXiv:2607.20129](https://arxiv.org/abs/2607.20129)

**工程启示：**
1. **选择性干预优于全局干预**：MGT-B 的核心优势在于无警报时输出与 vanilla 完全一致，这意味着部署时不会引入额外风险，只有在检测到退化时才进行修复。
2. **KV Cache 回滚是可行的工程实践**：系统能够恢复 token 和 KV Cache 状态并执行约束重解码，这为推理引擎的容错设计提供了参考。
3. **结果需要谨慎解读**：作者明确承认结果非确认性，经验因子未被证明为有效 e-process，这对工程部署中的期望管理很重要。
4. **与推测解码互补**：MGT-B 的监控机制可以与推测解码等加速技术结合，在保持吞吐的同时提升输出质量。

---

### 3️⃣ vLLM v0.25 架构转折：PagedAttention 退役与后端统一

**问题背景：**
vLLM 长期以来以 PagedAttention 为核心卖点，但随着 V1/MRv2 后端成为标准执行路径，旧有注意力实现与新的执行模型之间产生了维护成本和性能碎片化。同时，Transformers 后端与原生 vLLM 后端之间的性能差距也阻碍了模型支持的快速扩展。

**核心思路/原理：**
v0.25.0 进行了三项架构性变更：
1. **PagedAttention 完全移除**（#47361）：V1/MRv2 后端成为唯一标准路径，消除了维护两套注意力实现的成本。
2. **Model Runner V2 成为默认**（#44443）：在 v0.24 量化模型支持基础上，新增 EVS、实时嵌入、Mamba 混合模型前缀缓存、多模态前缀双向注意力、动态推测解码兼容完整 CUDA Graph 等能力。
3. **Transformers 后端追平原生性能**（#47187）：消除了原生后端与 Transformers 后端之间的性能差距，新增 FP8 MoE 支持、CUDA Graph + 嵌入缩放修复。

**数据与证据：**
- 558 个 commit，232 位贡献者（64 位新贡献者）
- Model Runner V2 新增 EVS（#46535）、实时嵌入（#46762）、Mamba 前缀缓存（#42406）、多模态前缀双向注意力（#46942）、动态推测解码（#45953）
- 新模型：LLaVA-OneVision-2、Unlimited OCR、MOSS-Transcribe-Diarize、openai/privacy-filter、Hy3
- 新增流式解析引擎（#46610）统一工具调用/推理解析，支持 Kimi k2.5/k2.6/k2.7、seed_oss、DeepSeek V4
- 通用异构词汇推测解码（TLI，#38174）及新 DSpark/DFlash 起草器

来源：
- [vLLM v0.25.0 Release Notes](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)
- [vLLM v0.25.1 Patch Release](https://github.com/vllm-project/vllm/releases/tag/v0.25.1)

**工程启示：**
1. **PagedAttention 退役标志着架构成熟**：V1/MRv2 后端经过多版本迭代已成为标准路径，移除旧实现减少了维护负担，用户应确保部署配置已迁移到 V1。
2. **Transformers 后端统一降低模型支持成本**：新模型现在可以通过 Transformers 后端快速接入而无需性能损失，这对快速跟进新模型发布有重要意义。
3. **推测解码生态扩展**：TLI 支持异构词汇、DSpark/DFlash 新起草器，推测解码正在从单一模式走向通用化。
4. **流式解析引擎标准化**：统一工具调用/推理解析框架对新模型快速接入至关重要，特别是 Kimi k2.5/k2.6/k2.7 和 DeepSeek V4 的解析器已就绪。

---

## 🔧 开源工具动态

1. **vLLM** — v0.25.1（7 月 14 日补丁）/ v0.25.0（7 月 11 日重大更新）。核心变更：PagedAttention 完全移除，V1/MRv2 成为唯一路径；Model Runner V2 默认启用，新增 EVS、实时嵌入、Mamba 前缀缓存、动态推测解码兼容 CUDA Graph；Transformers 后端追平原生性能并新增 FP8 MoE；新增流式解析引擎支持 Kimi k2.5/k2.6/k2.7 和 DeepSeek V4；通用异构词汇推测解码（TLI）及 DSpark/DFlash 起草器。生产环境建议：确保已从旧版 PagedAttention 迁移，利用 Transformers 后端快速接入新模型。

2. **SGLang** — v0.5.15（7 月 10 日）。GLM-5.2 NVFP4 在 Blackwell 上实现 500+ tok/s/user（8x B300）和 450 tok/s（4x GB300）；Spec V2 默认开启（CUDA-graphable DSA draft-extend，端到端吞吐 +11%）；IndexShare MTP 复用索引 top-k 跨起草步骤（长上下文起草成本 -1.9x）；TopK V2 运行时 k 最高 2048；Indexer prologue fusion 将 12 kernel 合并为 4（bs=1 解码约 +8%）；新增 Hunyuan 3、HRM-Text、Baidu Unlimited-OCR、Qwen3.6 NVFP4 等模型；原生 web search（Exa）支持；Breakable CUDA Graph 默认开启；FlashInfer A2A 用于路由 MoE。

3. **TensorRT-LLM** — v1.3.0rc22（7 月 22 日）。当前为 RC 版本，已知问题包括 torch.compile 在 PyTorch 编译后端崩溃、多个多 GPU 精度路径在 remove_copy pass 中失败。重要通知：AutoDeploy 后端正在被弃用，TensorRT 后端将在下一版本被移除（v1.3.0rc20 是最后一个支持 TensorRT 后端的版本）。NVIDIA 硬件优化方向转向纯 torch.compile 路径。

4. **llama.cpp** — b10099（7 月 23 日）。高频迭代持续，近期更新包括：CUDA NVFP4 W4A4 激活量化改进（#25730，融合 per-channel amax 与量化 kernel）；hexagon 激活算子优化（geglu 微核、非连续 src 和 strided DMA）；推测类型自动推断支持 mtp-/dflash-/eagle3- sidecar（#25989）；DeepSeek4 模板修复（#25414，支持 drop_reasoning flag）；PowerPC 后端 AIX 支持（#25983）。GGUF 格式持续稳定，CPU 推理性能持续优化。

5. **MLC LLM** — 最新正式发布仍为 v0.1.dev0（2023 年 4 月），项目活跃度较低。端侧部署场景建议关注 vLLM 的 CPU 路径或 llama.cpp 的 GGUF 格式作为替代方案。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 24 日*
