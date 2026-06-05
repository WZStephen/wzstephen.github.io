---
layout: post
title: '🔥 AI 技术日报 — 2026年5月26日'
date: 2026-05-26 09:00:00 +0800
categories: [ai-technical-report]
---


> 用最少的时间，了解今天最值得关注的 AI 动态

---

## 🔥 今日看点

| 时间 | 亮点 |
|------|------|
| 昨晚 | HuggingFace 发布 Agent 术语指南：Harness vs Scaffold 终于说清楚了 |
| 昨天 | vLLM 一天 10+ 个合并：DeepSeek V4 MegaMoE 内核优化、TPU 推理升级、MoE LoRA Triton 单步内核 |
| 昨天 | SGLang 同步狂飙：GLM-4.7-Flash 独立 MLA 实现、DeepSeek V4 DeepEP 水位填充 |
| 近两天 | NVIDIA Nemotron-Labs Diffusion：扩散语言模型进入实用阶段，三种生成模式自由切换 |
| 近期 | Dharma AI 实证研究：3B 专精模型在质量、成本、稳定性上全面超越 Claude Opus 4.6 |
| 近期 | IBM 发布 Open Agent Leaderboard：评测「完整 Agent 系统」而非单一模型 |
| 今日 arXiv | MemForest 提出分层时间索引 Agent 记忆系统 |
| 今日 arXiv | QUEST 用全合成任务训练前沿深度研究 Agent |

---

## 💡 深度解读

### 1️⃣ HuggingFace Agent 术语指南：Harness 和 Scaffold 到底什么区别？

**🎯 痛点场景**

你一定遇到过这种情况：同事说"我们的 Agent Harness 需要重构"，另一人说"得优化一下 Scaffold"，结果两个人说的根本不是一回事。ICLR 2026 期间，社区对这些术语的混乱已经到了影响协作的程度——连 Claude Code 和 Codex 的文档都在用不同的词描述同一件事。

**💡 技术原理**

HuggingFace 团队（@ariG23498）昨晚发布了一份精确定义指南，核心分层如下：

```
完整的 Agent 产品 = Model + Scaffolding + Harness

Model（模型层）
  └── 纯粹的权重 + 前向传播，输入文本 → 输出文本

Scaffolding（脚手架层）
  ├── System Prompt（系统提示词）
  ├── Tool Descriptions（工具描述）
  ├── Response Parser（响应解析器）
  └── Context Management（跨步骤记忆管理）
  
Harness（执行层）
  ├── Model Call Loop（模型调用循环）
  ├── Tool Call Handler（工具调用处理）
  ├── Stop Condition（停止条件判断）
  └── Error Handling（错误恢复策略）
```

关键区别：
- **Scaffold** 是"模型周围的行为定义层"，决定了模型**如何理解任务**——它塑造模型的认知边界
- **Harness** 是"让 Agent 跑起来的执行层"，决定了模型**如何与外界交互**——它控制执行流程

类比：
- **Model** = 厨师（核心能力）
- **Scaffold** = 菜谱 + 厨房规则（告诉厨师做什么、怎么做）
- **Harness** = 餐厅管理系统（接单 → 派给厨师 → 检查完成 → 上菜 → 处理客诉）

Claude Code 文档自己写的："Claude Code serves as the agentic harness around Claude." 这里的 Claude Code 就是 Harness，Claude 模型就是 Model，而系统提示词 + 工具定义就是 Scaffold。

**📊 实际意义**

这个分层的价值在于：
- 换模型只需要换 Model 层（如 Claude Code 从 Claude 4 切到 GPT-5）
- 换工具只需要改 Scaffold 层（加一个搜索工具的描述）
- 优化执行策略只需要改 Harness 层（调整停止条件、错误重试策略）

> 📎 来源：[Harness, Scaffold, and the AI Agent Terms Worth Getting Right](https://huggingface.co/blog/agent-glossary)

---

### 2️⃣ vLLM 一日十连发：DeepSeek V4、TPU、MoE LoRA 全面开花

**🎯 痛点场景**

vLLM 作为最流行的推理框架之一，更新速度一直很快，但今天（5月26日）的合并密度格外惊人。一天之内 10 个 PR 合并，涵盖了：
- DeepSeek V4 的 MegaMoE 内核优化
- TPU 推理引擎升级到 v0.20.0
- MoE LoRA 的单步 Triton 内核
- MooncakeStore KV 连接器修复
- GDN Prefill 内核（SM100 架构）

这意味着什么？vLLM 正在同时推进三条主线：**新模型适配**、**新硬件支持**、**核心内核优化**。

**💡 技术亮点**

重点看两个合并：

**DeepSeek V4 MegaMoE 内核优化**（PR #43632）
```python
# 之前：MegaMoE 的输入准备内核在 Python 侧做
# 问题：CPU 成为瓶颈，多卡扩展效率差
input_prep = cpu_preprocess_moe_inputs(expert_assignments)
gpu_inputs = transfer_to_gpu(input_prep)

# 现在：内核迁移到 nvidia/ops 侧，GPU 直接处理
# 效果：消除 CPU 瓶颈，多卡扩展线性度提升
gpu_inputs = nvidia_ops.mega_moe_input_prep(expert_assignments)
```

这解决了 DeepSeek V4 在 vLLM 上推理时的一个关键性能瓶颈——MoE 路由的输入准备从 CPU 移到了 GPU 侧，减少了 CPU→GPU 的数据传输开销。

**MoE LoRA 单步 Triton 内核**（PR #42290）

传统 LoRA 在 MoE 模型上的实现需要为每个 Expert 单独应用 LoRA 权重，计算开销大：
```
每个 Expert 的 LoRA 应用 = 独立的矩阵乘法 → N 个 Expert = N 次内核启动
```

新的 Triton 单步内核将所有 Expert 的 LoRA 合并到一个内核中执行：
```
合并后的 LoRA 应用 = 一次内核启动，内部并行处理所有 Expert
```

对于 DeepSeek V4 这种有 256 个 Expert 的模型，这个优化可以显著减少内核启动开销。

**📊 整体趋势**

vLLM 近期的合并方向清晰地反映了三个行业趋势：
1. **MoE 模型成为主流** → DeepSeek V4、MegaMoE、MoE LoRA 持续优化
2. **硬件多元化** → 除了 NVIDIA GPU，TPU、Intel GPU、AMD 都在积极适配
3. **KV Cache 互联** → MooncakeStore 等分布式 KV Cache 方案正在成熟

> 📎 来源：
> - [vLLM 最新合并 #43632 - DeepSeek V4 MegaMoE](https://github.com/vllm-project/vllm/pull/43632)
> - [vLLM 最新合并 #42290 - MoE LoRA Triton](https://github.com/vllm-project/vllm/pull/42292)
> - [vLLM Commits 列表](https://github.com/vllm-project/vllm/commits/main)

---

### 3️⃣ SGLang 同步发力：GLM-4.7-Flash MLA + DeepSeek V4 DeepEP

**🎯 痛点场景**

SGLang 和 vLLM 作为推理框架的"双子星"，更新节奏高度同步。今天 SGLang 也有大量合并，其中两个值得关注。

**💡 技术亮点**

**GLM-4.7-Flash 独立 MLA 实现**（PR #26088）

GLM 系列模型使用 MLA（Multi-Latent Attention）架构，这是一种比标准注意力更高效的注意力机制。SGLang 之前需要依赖外部实现，现在有了**独立的 MLA 内核**：

```
之前：GLM 模型推理 → 调用外部 MLA 实现 → 额外开销
现在：GLM 模型推理 → SGLang 内置 MLA 内核 → 零额外开销

支持的特性：
  ├── MLA NextN（多步预测）
  ├── MTP（Multi-Token Prediction）
  └── 与 SGLang 的 RadixAttention 原生集成
```

MLA 的核心优势是将 KV Cache 压缩到更低的维度，减少显存占用。对于长上下文场景（如 128K token），MLA 的显存节省可能达到 **50-70%**。

**DeepSeek V4 DeepEP 水位填充**（PR #25391）

DeepEP 是 DeepSeek 的 Expert Parallel 通信方案。SGLang 现在支持 DeepEP 的**水位填充（Waterfill）**策略：

```python
# 水位填充策略的核心思想：
# 不要等到所有 Expert 都分配满了才启动计算
# 而是当 Expert 的 token 数量达到"水位线"时就启动

# 传统方式：
while not all_experts_full:
    wait_for_more_tokens()
start_compute()

# 水位填充：
if expert_tokens >= waterline:
    start_compute_partial_batch()
# 剩余 token 在下一轮处理
```

这种策略在**非均匀 token 分配**（某些 Expert 收到的 token 远多于其他 Expert）的场景下，可以显著减少 Expert 空闲等待时间，提升吞吐。

**📊 SGLang 生态**

值得注意的是，NVIDIA Nemotron-Labs Diffusion 模型的部署也即将通过 SGLang 实现。SGLang 正在从"推理框架"向"多范式推理平台"演进——不仅支持传统的自回归解码，还支持扩散语言模型。

> 📎 来源：
> - [SGLang PR #26088 - GLM-4.7-Flash MLA](https://github.com/sgl-project/sglang/pull/26088)
> - [SGLang PR #25391 - DeepSeek V4 DeepEP](https://github.com/sgl-project/sglang/pull/25391)
> - [SGLang Commits 列表](https://github.com/sgl-project/sglang/commits/main)

---

### 4️⃣ NVIDIA 扩散语言模型：自回归之外的第二条路

**🎯 痛点场景**

自回归（AR）生成有个固有缺陷：**一旦出错，无法回退**。就像用打字机写文章，打错一个字只能继续往后写。在代码生成、数学推理等对精确度要求高的场景中，这个问题尤为突出。

**💡 技术原理**

NVIDIA Nemotron-Labs Diffusion 把**扩散模型**引入了文本生成。它不是替换自回归，而是与自回归共存于**同一个模型**中：

```
三种模式，一个模型：

1. 纯自回归模式（ar_mode=true）
   └── 和传统 LLM 一样，逐 token 生成
   └── 用途：作为正确性参考基准

2. 扩散模式（FastDiffuser）
   └── 一次生成 32 个 token 的 block
   └── 迭代去噪：[随机噪声] → [猜测1] → [猜测2] → [收敛]
   └── 置信度阈值决定哪些 token "足够好"可以固定
   └── 用途：最大吞吐量场景

3. 自投机模式（LinearSpec）
   └── 扩散模型做草稿生成 + 自回归验证
   └── 匹配的前缀被提交，不匹配的部分回退重生成
   └── 用途：速度和正确性的最佳平衡
```

关键突破：
- **并行生成**：一次处理多个 token，吞吐是纯自回归的 **2-3 倍**
- **可回退**：每轮迭代都能修正之前轮次的错误，不产生错误累积
- **训练友好**：基于已有的 AR 模型做 continued pretraining，无需从头训练

类比：
- **自回归** = 一个人从头写文章，写完不能改
- **扩散** = 先写个草稿全文，然后反复修改，每次改得更好
- **自投机** = 先快速写个草稿，然后逐句检查修正

**📊 实际效果**

- Nemotron-Labs Diffusion 8B 在平均准确率上比 Qwen3 8B 提升 **1.2%**
- 扩散模式吞吐量（以 tokens per forward pass 衡量）比纯自回归提升 **2-3 倍**
- 模型规模覆盖 3B、8B、14B，NVIDIA 开源许可

> 📎 来源：[Towards Speed-of-Light Text Generation with Nemotron-Labs Diffusion](https://huggingface.co/blog/nvidia/nemotron-labs-diffusion)

---

## 📰 行业动态

### 🤖 AI Agent

1. **IBM 发布 Open Agent Leaderboard**
   
   不只评测模型，而是评测**完整的 Agent 系统**（模型 + 工具 + 规划策略）。统一协议让不同 Agent 框架可以在同一基准下比较。同时报告**质量和成本**，配套 Exgentic 框架可复现所有评测结果。覆盖编码、客服、技术支持等 6 个不同基准。
   
   > 📎 [The Open Agent Leaderboard](https://huggingface.co/blog/ibm-research/open-agent-leaderboard)

2. **Dharma AI：3B 专精模型全面超越 Claude Opus 4.6**
   
   在巴葡 OCR 任务中，一个 3B 专精模型（经过 LoRA 微调）在质量上超越了 Claude Opus 4.6（得分 0.911 vs 更低），成本仅为其 **1/52**，且文本退化率最低。结论：**专精 > 规模**，企业 AI 采购策略需要重新审视。
   
   > 📎 [Specialization Beats Scale](https://huggingface.co/blog/Dharma-AI/specialization-beats-scale)

3. **AllenAI OlmoEarth v1.1：卫星影像处理成本降低 3 倍**
   
   通过减少 token 序列长度（patch-based tokenization），在保持性能的同时将 MACs（乘加运算）显著降低。适合大规模地球观测任务的本地部署。
   
   > 📎 [OlmoEarth v1.1](https://huggingface.co/blog/allenai/olmoearth-v1-1)

### 🧠 模型架构

4. **HF 发布 Ettin Reranker 家族**
   
   6 个不同规模的 cross-encoder reranker（32M 到更大），可与 embedding 模型组成 retrieve-then-rerank pipeline。支持 bfloat16 + Flash Attention 2 优化推理速度。使用 Sentence Transformers 的 Agent Skill 训练。
   
   > 📎 [Ettin Reranker Family](https://huggingface.co/blog/ettin-reranker)

5. **Granite Embedding Multilingual R2：32K 上下文多语言嵌入**
   
   IBM 发布的亚 1 亿参数多语言嵌入模型，支持 32K 上下文，在 MTEB 检索任务上达到 sub-100M 模型中的最佳质量。Apache 2.0 开源。
   
   > 📎 [Granite Embedding Multilingual R2](https://huggingface.co/blog/ibm-granite/granite-embedding-multilingual-r2)

### 🛠️ 推理框架

6. **vLLM 持续高速迭代**
   
   今日 10+ 合并：DeepSeek V4 MegaMoE 内核 GPU 化、TPU 推理升级至 v0.20.0、GDN Prefill 内核（SM100）、MoE LoRA Triton 单步内核、MooncakeStore KV 连接器修复等。
   
   > 📎 [vLLM Commits](https://github.com/vllm-project/vllm/commits/main)

7. **SGLang 多架构支持扩展**
   
   GLM-4.7-Flash 独立 MLA 内核、DeepSeek V4 DeepEP 水位填充策略、FA3 cross-attention 修复、Intel GPU KV cache 页表修复。同时支持 NVIDIA Nemotron Diffusion 部署。
   
   > 📎 [SGLang Commits](https://github.com/sgl-project/sglang/commits/main)

### 📄 今日 arXiv 精选

8. **MemForest：分层时间索引的 Agent 记忆系统**（2605.23986）
   
   针对 Agent 长时记忆的高效索引方案，用分层结构 + 时间戳组织记忆，解决 Agent 在长期交互中记忆检索效率低下的问题。

9. **QUEST：全合成任务训练深度研究 Agent**（2605.24218）
   
   用完全合成的训练任务来训练前沿的 Deep Research Agent，减少对真实标注数据的依赖，降低训练成本。

10. **Foundation Protocol：Agentic 社会的协调层**（2605.23218）
    
    为多 Agent 协作系统设计的协调协议层，解决 Agent 之间的通信、任务分配和冲突消解问题。

11. **Mix-MoE：混合 MoE 提升多语言翻译**（2605.24681）
    
    通过混合 MoE 架构提升大语言模型的多语言机器翻译质量，在低资源语言上效果提升尤为显著。

12. **The Illusion of Reasoning：零 CoT 截断暴露 LLM 数据污染**（2605.21856）
    
    通过零 CoT（Chain-of-Thought）截断方法，揭示了部分 LLM 在推理 benchmark 上的高分可能来自数据污染而非真正的推理能力。

---

## 💬 结语

今天的技术动态有三条主线在交汇：

1. **Agent 术语标准化**：HF 的术语指南让社区终于有了共同语言，Harness/Scaffold/Model 的分层将帮助团队更清晰地设计和讨论 Agent 架构
2. **推理框架军备竞赛**：vLLM 和 SGLang 同一天都在狂飙——DeepSeek V4、MoE 优化、新硬件适配，两条线都在推高推理性能的上限
3. **扩散语言模型从实验室走向生产**：NVIDIA 的 Nemotron-Labs 把扩散生成变成了可部署的工程方案

你最看好扩散语言模型取代自回归的那一天吗？还是觉得自回归的简单性无可替代？来评论区聊聊 👇
