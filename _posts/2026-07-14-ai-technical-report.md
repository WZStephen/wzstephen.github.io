---
layout: post
title: 'KV-PRM 奖励模型 KV Cache 复用、Agora 拍卖式多智能体编排、选择性持久记忆 7 月推理新进展'
date: 2026-07-14 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期聚焦 LLM 推理与 Agent 系统前沿进展。KV-PRM 提出通过 KV Cache 直接复用实现过程奖励模型推理，将多智能体 test-time scaling 的评分成本从 O(L²) 大幅降低；Agora 引入拍卖机制优化多 Agent 任务分配，在成本与性能间取得帕累托最优；Shared Selective Persistent Memory 为 Agentic LLM 系统提供跨会话持久化记忆方案，解决多轮工具调用的上下文冷启动问题。开源框架方面，vLLM v0.25.0 将 Model Runner V2 设为默认执行路径，SGLang v0.5.15 在 Blackwell 上实现 GLM-5.2 500+ tok/s/user，llama.cpp 新增腾讯混元 3 (Hy3) MoE 架构支持，TensorRT-LLM v1.3.0rc20 成为最后支持 TensorRT 后端的版本。

---

## 🔥 今日看点

1. **7 月 14 日** — KV-PRM：基于 KV Cache 传递的高效过程奖励模型。将 PRM 评分从文本重新编码改为 KV Cache 直接复用，多智能体 rollouts 评分成本从 O(L²) 降至近线性，在数学推理和多步任务规划中显著降低 test-time 计算开销（[arXiv:2607.09153](https://arxiv.org/abs/2607.09153)）

2. **7 月 14 日** — Agora：拍卖式多 Agent 任务分配框架。针对功能相似但性能/成本差异大的专家模型，引入组合拍卖机制实现帕累托最优的任务路由，在多工具编排场景中相比贪婪匹配提升 15-30% 成本效率（[arXiv:2607.09600](https://arxiv.org/abs/2607.09600)）

3. **7 月 14 日** — Shared Selective Persistent Memory：Agentic LLM 系统的跨会话选择性持久记忆。通过 token 高效的选择性记忆存储，解决多轮代码生成中配置/Schema/工具模式的冷启动问题，避免全量历史持久化的 token 浪费（[arXiv:2607.09493](https://arxiv.org/abs/2607.09493)）

4. **7 月 14 日** — ProofCouncil：面向开放数学问题的 LLM Agent。采用 Author-Critic 架构，模拟真实数学研究中的"证明-审查"工作流，在研究级数学问题上展现出超越单模型推理的能力（[arXiv:2607.09474](https://arxiv.org/abs/2607.09474)）

5. **7 月 14 日** — OpenProver：Lean 4 驱动的开源交互式定理证明系统。Planner-Worker-Verifier 三层架构配合 Whiteboard 草稿板和 Repository 中间结论库，实现可审计的形式化验证（[arXiv:2607.09217](https://arxiv.org/abs/2607.09217)）

6. **7 月 14 日** — Scoped Verification：长时程 Agent 上下文演化的有界验证方法。针对部署态 LLM Agent 的系统级指令持续更新场景，提出分块验证策略解决长 horizon 下的验证漂移问题（[arXiv:2607.09175](https://arxiv.org/abs/2607.09175)）

7. **7 月 11 日** — vLLM v0.25.0 发布：Model Runner V2 成为所有 dense 模型的默认执行路径，新增 EVS 支持、实时嵌入、Mamba 混合模型 prefix caching、动态投机解码兼容性，558 commits / 232 contributors（[GitHub](https://github.com/vllm-project/vllm/releases/tag/v0.25.0)）

8. **7 月 10 日** — SGLang v0.5.15：GLM-5.2 NVFP4 在 Blackwell 8×B300 上达到 500+ tok/s/user，Spec V2 成为默认投机解码方案（+11% TPS），CUDA-graphable DSA draft-extend 实现零开销调度（[GitHub](https://github.com/sgl-project/sglang/releases/tag/v0.5.15)）

---

## 💡 深度解读

### 1️⃣ KV-PRM：KV Cache 直接复用实现高效过程奖励建模

**问题背景：**
过程奖励模型（PRM）在引导 test-time scaling（TTS）方面已被证明高度有效，尤其是在 LLM 多智能体系统中。然而，现有 PRM 都是基于文本的——它们需要从头重新编码整个轨迹文本。在长序列多智能体 rollouts 中，评分成本随序列长度 L 呈 O(L²) 增长，造成严重的计算瓶颈。

**核心思路/原理：**
KV-PRM 的核心创新在于跳过文本重编码，直接将目标模型的 KV Cache 传递给 PRM。具体而言：
- 在前向推理过程中，目标模型已经生成了完整的 KV Cache
- 传统 PRM 需要将这些 KV 对应的 token 文本重新送入 PRM 编码器，产生冗余计算
- KV-PRM 通过投影层将源模型 KV 空间映射到 PRM KV 空间，实现"零编码"评分
- 对于多智能体场景，各 Agent 的 KV Cache 可以独立评分后聚合

**数据与证据：**
- 在数学推理 benchmark 上，KV-PRM 相比文本基线 PRM 评分速度提升 3-5 倍
- 多智能体 rollouts 场景下，端到端 test-time 计算降低 40-60%
- 评分精度与文本 PRM 持平（差异 < 1%），证明 KV 空间保留了充分的奖励信号

来源：
- [KV-PRM: Efficient Process Reward Modeling via KV-Cache Transfer for Multi-Agent Test-Time Scaling: arXiv:2607.09153](https://arxiv.org/abs/2607.09153)

**工程启示：**
1. 对于部署了 vLLM/SGLang 的生产环境，KV Cache 复用模式可以与现有 PagedAttention 机制天然配合，无需额外显存分配
2. 在多智能体编排系统中，PRM 评分瓶颈往往是 test-time scaling 的主要障碍，KV-PRM 为大规模 rollout 打开了可行性窗口
3. 该思路可推广到其他需要"二次评估"的场景，如安全分类器、质量打分模型的在线部署

---

### 2️⃣ Agora：组合拍卖优化多 Agent 任务分配

**问题背景：**
当前 LLM Agent 框架在编排多个专家模型时，通常采用粗粒度的任务-函数匹配（如 function calling），忽略了功能相似替代方案之间的性能差异和成本差异。例如，同一代码生成任务可以路由给 GPT-4、Claude、或本地 CodeLlama，但成本和延迟差异可达 100 倍。

**核心思路/原理：**
Agora 将多 Agent 任务分配建模为组合拍卖问题：
- **竞标者（Bidders）**：待分配的子任务，每个子任务有质量约束和预算上限
- **拍卖者（Auctioneers）**：可用的专家模型/工具，各自报价基于当前负载和历史性能
- **拍卖机制**：采用 VCG（Vickrey-Clarke-Groves）机制确保激励兼容，每个参与者报告真实成本是最优策略
- **求解**：通过整数规划求解社会最优分配，同时保证纳什均衡

**数据与证据：**
- 在 5 类 Agent 任务（代码生成、数据分析、文档摘要、多模态理解、工具调用链）上测试
- 相比贪婪匹配（greedy matching），Agora 在同等质量约束下降低 15-30% 的 API 调用成本
- 相比固定路由（fixed routing），在成本波动场景下保持更稳定的质量-成本帕累托前沿

来源：
- [Agora: Enhancing LLM Agent Reasoning Via Auction-Based Task Allocation: arXiv:2607.09600](https://arxiv.org/abs/2607.09600)

**工程启示：**
1. 在多模型部署架构中（如同时维护云端 API 和本地推理集群），拍卖机制可以自动实现负载均衡和成本优化
2. VCG 机制的激励兼容性意味着专家模型没有动机虚报能力或成本——这对开放生态中的第三方模型接入尤为重要
3. 拍卖求解的整数规划在小规模（< 50 任务 × 20 模型）下可实时求解，适合在线服务场景

---

### 3️⃣ 选择性持久记忆：Agent 系统的跨会话知识积累

**问题背景：**
Agentic LLM 系统在执行多轮工具调用（如代码生成、数据分析 pipeline）时，每个会话都从零开始——丢弃了前次会话中的配置选择、领域约束、数据 Schema 和工具使用模式。朴素地持久化全部对话历史既 token 低效又适得其反：无关上下文会降低生成质量。

**核心思路/原理：**
Shared Selective Persistent Memory (SSPM) 提出三层记忆架构：
- **工作记忆（Working Memory）**：当前会话的活跃上下文，等同于标准 KV Cache
- **情景记忆（Episodic Memory）**：近期会话的结构化摘要，包含关键决策和工具调用模式
- **语义记忆（Semantic Memory）**：跨会话积累的领域知识图谱，包括 Schema 映射、API 使用偏好、错误修复模式

选择性检索机制：在新会话开始时，系统根据当前任务描述从语义记忆中检索相关条目，以 prefix prompt 的形式注入，而非全量加载。

**数据与证据：**
- 在多轮代码生成 benchmark 上，SSPM 相比无记忆基线提升 22% 的任务完成率
- 相比全量历史持久化，token 消耗降低 65%，同时生成质量无显著差异
- 记忆检索的延迟开销 < 50ms（基于 FAISS 向量检索）

来源：
- [Shared Selective Persistent Memory for Agentic LLM Systems: arXiv:2607.09493](https://arxiv.org/abs/2607.09493)

**工程启示：**
1. 对于企业级 Agent 部署（如内部 DevOps Agent、数据分析助手），SSPM 模式可以显著减少用户的重复上下文提供
2. 语义记忆的图谱结构使得知识可以被审计和编辑——用户可以看到 Agent "记住了什么"，增强可控性
3. 与 RAG 的区别在于：SSPM 存储的是"如何使用工具"的程序性知识，而非事实性知识，更适合 Agent 场景

---

## 🔧 开源工具动态

1. **vLLM** — **v0.25.0**（7 月 11 日）：这是今年上半年最重要的版本之一。**Model Runner V2 成为所有 dense 模型的默认执行路径**，这意味着 PagedAttention 的内存管理效率和 continuous batching 的调度性能将惠及所有用户。新增 EVS（Embedding Variable Sequence）支持、实时嵌入能力、Mamba 混合模型的 prefix caching。动态投机解码兼容性意味着用户可以在运行时切换 draft model 而不中断服务。558 commits 来自 232 位贡献者（64 位新人），社区活跃度持续攀升。**生产建议**：升级到 v0.25.0 后，MRv2 默认启用，建议在 staging 环境先验证现有模型的兼容性，特别是有自定义 attention kernel 的场景。

2. **SGLang** — **v0.5.15**（7 月 10 日）：本期亮点是 **GLM-5.2 NVFP4 在 Blackwell 架构上的极致调优**——8×B300 配置下单用户达到 500+ tok/s，4×GB300 配置下 450 tok/s。**Spec V2 成为默认投机解码方案**，通过 CUDA-graphable DSA draft-extend 消除了 D2H/H2H 同步开销，端到端 TPS 提升 11%。这标志着 SGLang 在 NVIDIA 最新硬件上的优化已经深入到调度层面，不再局限于算子融合。**与 vLLM 互补**：SGLang 的结构化生成（JSON schema / regex constrained decoding）仍然是其差异化优势，适合需要严格输出格式的 Agent 工具调用场景。

3. **TensorRT-LLM** — **v1.3.0rc20**（6 月 30 日）：值得关注的里程碑——**这将是最后一个支持 TensorRT 后端的版本**，后续版本将完全迁移到 PyTorch 后端。新增 TeaCache 系数配置 API、请求级 chat_template 可选配置（breaking change）。已知问题包括 DeepSeek V3/V3.2 在 warm-up 阶段可能触发非法内存访问，Qwen3 系列 autotuning 可能因 cutlass TMA grouped gemm 初始化失败而崩溃。**迁移建议**：仍在使用 TensorRT 后端的用户应开始规划向 PyTorch 后端的迁移，关注 NVIDIA 的迁移指南。

4. **llama.cpp** — **b9993**（7 月 13 日）：nightly 发布模式持续，最新构建亮点是**新增腾讯混元 3（Hunyuan 3 / Hy3）架构支持**，这是一个 MoE 解码器栈，具备 per-head Q/K RMSNorm、sigmoid router + expert selection bias、始终激活的共享专家（ungated shared expert），以及 leading dense block 配置。同时支持 MTP（Multi-Token Prediction）投机解码。**CUDA MMQ kernel 配置重构**清理了遗留代码并修复了 Blackwell 配置。macOS/iOS 用户注意 KleidiAI 支持目前处于禁用状态。

5. **MLC LLM** — **v0.26.dev0**：MLC LLM 目前处于开发版本阶段（v0.26.dev0），上一个稳定版本为 v0.20.0。项目似乎进入低频维护状态，近期没有重大 release。对于端侧部署需求，建议关注 llama.cpp 的 GGUF 生态或 SGLang 的轻量推理模式作为替代方案。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 14 日*
