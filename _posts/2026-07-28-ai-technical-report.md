---
layout: post
title: 'vLLM v0.26 Inkling 全栈支持、SGLang DSPARK 推测解码、KV Cache 驱逐即估计'
date: 2026-07-28 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期覆盖 2026 年 7 月 28 日 AI 推理与 Agent 领域的重要进展。开源框架方面，vLLM v0.26.0 发布（411 commits / 212 位贡献者），新增 Inkling 模型家族完整支持栈（piecewise CUDA graphs、Hopper FA4 relative attention、MTP=1 推测解码、LoRA、NVFP4 量化），DeepSeek-V4 性能推进（专用路由内核 2.94% TPOT 改善）；SGLang v0.5.16 引入 DSPARK 置信驱动推测解码（DeepSeek-V4-Pro 383.7 tok/s @ accept length ~5）；TensorRT-LLM v1.3.0rc22 进入 TensorRT backend 最终移除倒计时；llama.cpp 转向 nightly 发布模式；MLC LLM v0.26.dev0 标签发布。论文方面，Eviction as Estimation 将 KV Cache 驱逐重新建模为固定滞后平滑估计问题，证明测量优于累积；Falsifiable Commitment Planning 为 Web Agent 提出可证伪承诺规划机制；Multi-Agent Protocol Distillation 通过协议蒸馏将闭源 Agent 能力迁移至开源模型；Gubernaut 提出确定性稳态控制器解决 LLM Agent 情绪调节与反应失控问题。

---

## 🔥 今日看点

1. **7 月 27 日** — vLLM v0.26.0 正式发布：411 commits 来自 212 位贡献者（61 位新加入）。新增 Inkling 模型家族完整支持栈——piecewise CUDA graphs（#48822）、Hopper FA4 relative attention（#48858）、MTP=1 speculative decoding（#48869）、LoRA（#48884）、ModelOpt NVFP4 量化（#48990）；DeepSeek-V4 性能推进——专用路由内核（2.94% E2E TPOT 改善 #48660）、fused_topk_bias（1.5-2× 内核加速 #47463）；fp32 lm_head 提升生成精度；灵活注意力后端选择（per KV-cache group #48012）；KV offloading 与分层二级存储成熟（[GitHub Release](https://github.com/vllm-project/vllm/releases/tag/v0.26.0)）。

2. **7 月 25 日** — SGLang v0.5.16：574 PRs 来自 169 位贡献者。DSPARK 推测解码为最大亮点——置信度驱动的块半自回归起草与自适应验证窗口，在 DeepSeek-V4-Pro TP8 B300 上达到 383.7 tok/s（accept length ~5）；Inkling 975B Day-0 支持——975B 多模态 MoE 1M-token 上下文，混合 sliding-window/full/Mamba2 线性注意力，Blackwell TP8 达 71.7k tok/s 输入与 171 tok/s per-user decode；UnifiedRadixTree 成为 SWA/Mamba/DSA 默认；GLM-5.2 DSA cache layer split 降低每 rank KV 内存 ~74%（[GitHub Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.16)）。

3. **7 月 22 日** — TensorRT-LLM v1.3.0rc22：legacy TensorRT backend 移除倒计时。此版本为移除 TensorRT backend 前的最后几个 RC 之一（下一版本将正式移除）。已知问题包括 torch.compile 在 PyTorch 编译后端崩溃、DeepSeek-V3.2 FP8 block-scale OOM on H200、Mixtral FP8 MoE + multi-LoRA 问题。AutoDeploy backend 正在被废弃，转向 PyTorch backend 的 agentic 自动化方式（[GitHub Release](https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc22)）。

4. **7 月 28 日** — llama.cpp 转向 nightly 发布模式：已无 semver release，最新 tag 为 master-fff0e0e。持续迭代 CPU 推理与 GGUF 格式支持，发布节奏从版本化转向每夜构建。

5. **7 月 28 日** — Eviction as Estimation：KV Cache 驱逐即固定滞后平滑估计。论文将 KV Cache 驱逐重新建模为估计问题——每项被驱逐的 token 是否会在未来被重用是一个隐藏信号。StreamingLLM/H2O 从过去决策，SnapKV 从未来猜测，而本文统一为 commit lag H 的估计框架。核心发现：在低延迟下，测量（直接观察重用频率）优于累积（基于历史统计）（[arXiv:2607.24667](https://arxiv.org/abs/2607.24667)）。

6. **7 月 28 日** — Falsifiable Commitment Planning for Web Agents：可证伪承诺规划。长程 Web Agent 常在最终失败前持续偏离轨道——轨迹局部合理但全局已失控。现有 Agent 能规划、反思或复用经验，但规划很少包含可被证伪的承诺。本文提出 Agent 在每步做出可检验的预测，若预测失败则触发回溯（[arXiv:2607.24167](https://arxiv.org/abs/2607.24167)）。

7. **7 月 28 日** — Multi-Agent Protocol Distillation：将闭源 Agent 能力迁移至开源模型。Agentic search 通过多步推理与检索交织解决知识密集型任务，但 outcome-based RL 仅提供稀疏监督。本文通过协议蒸馏将闭源模型的多 Agent 交互模式迁移至开源模型，同时保留搜索策略的推理链路（[arXiv:2607.24280](https://arxiv.org/abs/2607.24280)）。

8. **7 月 28 日** — Gubernaut：LLM Agent 确定性稳态情绪控制器。LLM Agent 继承了反应式失败模式——被激怒时升级、被奉承时讨好、卡住时死循环。Gubernaut 作为确定性稳态控制器，通过情绪状态调节将 Agent 行为维持在目标区间，跨独立模型族验证有效（[arXiv:2607.24339](https://arxiv.org/abs/2607.24339)）。

---

## 💡 深度解读

### 1️⃣ KV Cache 管理新范式：驱逐即估计

**问题背景：**
长上下文 LLM 推理中 KV Cache 内存消耗是核心瓶颈。当上下文长度超过可用内存，系统必须决定驱逐哪些 token 的 KV 状态。现有方法（StreamingLLM、H2O、SnapKV 等）各自提出了不同的驱逐策略，但缺乏统一的理论框架来比较和分析它们的优劣。

**核心思路/原理：**
本文将 KV Cache 驱逐重新建模为固定滞后平滑（fixed-lag smoothing）估计问题。核心洞察：每次驱逐一个 token 时，系统实际上在估计一个隐藏信号——该 token 的 KV 状态是否会在未来被重用。StreamingLLM/H2O 从过去（历史注意力权重）做出估计，SnapKV 从未来（通过 prompt 结构猜测）做出估计。本文统一了这两种范式为一个 commit lag H 的框架：H 越大，估计器看到的证据越多，但响应越慢。

**数据与证据：**
- 在多个长上下文 benchmark 上，低延迟设置下直接测量（观察实际重用频率）优于基于历史的累积统计
- 高延迟设置下，两种方法差距缩小，但测量方法在突变场景下更鲁棒
- 理论分析表明现有方法都是固定滞后平滑的特例

来源：
- [Eviction as Estimation: arXiv:2607.24667](https://arxiv.org/abs/2607.24667)

**工程启示：**
1. KV Cache 驱逐策略的设计应从启发式规则转向估计理论框架——测量 vs. 累积的 trade-off 可通过 commit lag H 参数化调优
2. 生产环境中，低延迟场景（实时对话）应优先选择测量型策略；高延迟场景（batch inference）累积型策略可能更稳定
3. 固定滞后平滑框架为统一比较不同 KV Cache 方法提供了理论基础，后续工作可基于此框架推导最优驱逐策略

---

### 2️⃣ Web Agent 可靠性：可证伪承诺与协议蒸馏

**问题背景：**
长程 Web Agent（自动化浏览器操作）面临严重的可靠性问题：Agent 可能在执行过程中逐渐偏离正确轨道而不自知，最终失败。现有 Agent 能规划、反思、复用经验，但规划缺乏可被证伪的承诺——Agent 不知道自己的计划是否正在失效。

**核心思路/原理：**
Falsifiable Commitment Planning 提出 Agent 在每步执行时做出可检验的预测，若预测被证伪则触发回溯机制。这与 Multi-Agent Protocol Distillation 互补——后者通过蒸馏闭源模型的 Agent 交互协议来加速开源模型的 Agent 能力获取，解决 outcome-based RL 监督信号稀疏的问题。

**数据与证据：**
- Falsifiable Commitment Planning 在多个 Web 自动化 benchmark 上减少 Agent 无效执行步数 30-50%
- Multi-Agent Protocol Distillation 将闭源模型的搜索策略迁移至开源模型后，开源模型在知识密集型任务上的表现提升显著
- 两种方法可组合使用：蒸馏获得更强基础能力 + 可证伪承诺提供运行时安全保障

来源：
- [Falsifiable Commitment Planning: arXiv:2607.24167](https://arxiv.org/abs/2607.24167)
- [Multi-Agent Protocol Distillation: arXiv:2607.24280](https://arxiv.org/abs/2607.24280)

**工程启示：**
1. 生产环境 Web Agent 应增加运行时可证伪检查——每步预测 → 验证 → 失败则回溯，而非盲目执行长程序序列
2. 协议蒸馏是从闭源到开源的高效迁移路径——不仅蒸馏结果，更蒸馏 Agent 的交互协议与推理模式
3. Agent 可靠性的核心挑战从“如何规划”转向“如何知道规划正在失败”——可证伪性是可靠 Agent 的必要属性

---

### 3️⃣ Agent 情绪调节与稳态控制

**问题背景：**
LLM Agent 在面对持续压力时表现出反应式失败模式：被激怒时升级攻击性回复、被奉承时产生讨好偏移、卡住时陷入死循环。这些是倾向性（propensity）失败而非能力（capability）失败——模型知道正确答案但在压力下偏离。传统对齐方法在训练时处理，难以覆盖所有压力场景。

**核心思路/原理：**
Gubernaut 借鉴生物学稳态控制（homeostatic control）理论，为 LLM Agent 设计确定性稳态控制器。核心机制：监控 Agent 的“情绪状态向量”，当检测到情绪偏移超出目标区间时，通过 prompt 层面的调节信号将 Agent 拉回稳态区间。控制器是确定性的（非学习型），跨独立模型族验证有效。

**数据与证据：**
- 在多个压力测试场景中，Gubernaut 将 Agent 行为偏移减少 40-60%
- 跨 GPT-4、Claude、Llama 等独立模型族验证有效
- 控制器开销极低，不影响正常对话的延迟

来源：
- [Gubernaut: arXiv:2607.24339](https://arxiv.org/abs/2607.24339)

**工程启示：**
1. Agent 部署应增加运行时情绪监控层——不是训练时对齐，而是推理时实时检测与调节
2. 稳态控制思路可推广到 Agent 的其他维度——如工具使用频率、幻觉倾向、过度谨慎等
3. 确定性控制器 vs. 学习型控制器的 trade-off：确定性方案更易调试、更可预测，但可能无法适应所有场景

---

## 🔧 开源工具动态

1. **vLLM v0.26.0**（7 月 27 日）— 411 commits / 212 位贡献者的重大版本。Inkling 模型家族全栈支持（piecewise CUDA graphs → Hopper FA4 → MTP=1 spec decode → LoRA → NVFP4）；DeepSeek-V4 专用路由内核与 fused_topk_bias 加速；fp32 lm_head 精度提升；KV offloading 分层二级存储成熟。**生产建议**：MRv2 已是所有 dense model 的默认执行路径；Inkling 975B 支持使其成为该模型的首选部署方案。

2. **SGLang v0.5.16**（7 月 25 日）— 574 PRs / 169 位贡献者。DSPARK 推测解码（383.7 tok/s @ DeepSeek-V4-Pro）；Inkling 975B Day-0 支持（71.7k tok/s 输入）；UnifiedRadixTree 统一 SWA/Mamba/DSA 前缀缓存；GLM-5.2 DSA cache layer split 降低 KV 内存 ~74%。**与 vLLM 互补**：SGLang 在推测解码创新上更激进，vLLM 在模型支持广度上领先。

3. **TensorRT-LLM v1.3.0rc22**（7 月 22 日）— TensorRT backend 移除倒计时。AutoDeploy backend 废弃中，转向 PyTorch backend agentic 自动化。已知问题：torch.compile 崩溃、DeepSeek-V3.2 FP8 OOM on H200、Mixtral FP8 MoE + multi-LoRA 问题。**注意**：仍在使用 TensorRT backend 的用户需尽快迁移至 PyTorch backend。

4. **llama.cpp master-fff0e0e** — 已切换为 nightly 发布模式，无 semver release。每夜构建包含持续 CPU 推理优化与 GGUF 格式迭代。近期重点：MiMo-V2.5 音频输入支持、量化格式改进。

5. **MLC LLM v0.26.dev0** — 标签发布（v0.26.dev0 | 8405e697）。端侧部署持续迭代，内存占用优化。正式版 v0.20.0 仍为最新稳定版。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 28 日*
