---
layout: post
title: 'vLLM 0.22、CONF-KV 与 Web Agent 省 Token'
date: 2026-05-31 09:00:00 +0800
categories: [ai-technical-report]
---


## 🔥 今日看点

1. **vLLM v0.22.0 发布**（5月29日）— 459 commits、230 位贡献者的大版本，Batch-invariant 推理端到端延迟降低 28.9%，新增多级 KV Cache Offloading 框架 🚀
2. **SGLang 今日连续合并多个核心 PR**（今天凌晨）— ForwardBatch 架构升级、混合 SWA 模型识别、大 tensor 优化，MRv2 向默认引擎迈进
3. **CONF-KV：用模型"置信度"动态管理 KV Cache**（论文，今日 HF Daily）— 32K 上下文 Needle-in-a-Haystack 检索准确率 91.4%，显存只有滑动窗口的水平
4. **PANDO：Web Agent 在线技能蒸馏框架**（论文，今日 HF Daily）— 58.3% 成功率的同时 token 消耗减少 58-61%，Agent 越用越省
5. **AgentDoG 1.5：轻量级 Agent 安全对齐框架**（论文，今日 HF Daily）— 仅用 ~1k 样本训练的 0.8B-8B 安全模型，效果媲美 GPT-5.4
6. **Parallax：参数化局部线性注意力**（论文，今日 HF Daily）— 首个在预训练尺度验证 Local Linear Attention 有效性的工作，decode kernel 匹敌 FlashAttention 2/3
7. **LangGraph SDK 0.4.0 发布**（5月28日）— Agent 编排框架重大版本更新
8. **HuggingFace Transformers 修复模型并行 Beam Search**（昨天）— 多卡推理正确性补丁

---

## 💡 深度解读

### 1️⃣ vLLM v0.22.0：Batch-Invariant 推理提速 28.9%，KV Cache Offloading 走向多级

**痛点场景：**
在生产环境中跑 LLM 推理，你一定遇到过这个场景：同一个模型，同一批请求，不同 batch size 下性能波动巨大。为什么？因为传统的推理引擎在处理不同 batch 时，padding 策略和内存分配差异导致 GPU 利用率忽高忽低。更头疼的是，当上下文越来越长（32K、128K），KV Cache 直接把显存吃光，OOM 频繁发生。

**技术原理：**
vLLM v0.22.0 这次带来了两个重量级改进：

**① Batch-Invariant + Cutlass FP8 融合优化：**
Batch-Invariant 推理的核心思路是"让不同 batch 走相同的计算路径"。过去 batch=1 和 batch=128 可能会触发不同的 kernel，导致 GPU 上 SM 的占用率差异很大。vLLM 这次通过 Cutlass FP8 kernel 统一了计算路径，配合 FP8 精度，直接端到端延迟降低 **28.9%**。

可以这样理解：以前你的 GPU 像一个餐厅后厨，来 1 桌客人和来 10 桌客人用的烹饪流程完全不同，切换成本高。现在统一了标准化流程，不管来多少桌，后厨始终在全速运转。

**② 多级 KV Cache Offloading：**
以前的 KV Cache offloading 只能卸载到 CPU 内存（一级）。vLLM v0.22.0 新增了多级框架：
- Tier 1：GPU HBM（高速但容量有限）
- Tier 2：CPU RAM（容量大但速度慢）
- Tier 3：**文件系统/磁盘**（这次新增！）

这意味着什么？假设你在跑 128K 上下文的对话，KV Cache 需要 80GB+ 显存。以前只能靠 GPU 显存硬扛或者完全卸载到 CPU。现在 vLLM 可以把最不活跃的 KV 块卸载到 SSD，通过 Mooncake 或文件系统做二级缓存。配合 Python 文件系统二级 tier（#41735），即使单卡 80GB 也能处理超长上下文。

**实际效果：**
- Batch-Invariant Cutlass FP8：端到端延迟 -28.9%，TTFT（首 token 时间）通过 CutlassFP8 padding 预处理再降 13.5%
- 多级 offloading 支持 DSv4（#43142）和 Mooncake 磁盘卸载（#42689）
- DeepSeek V4 获得 NVFP4 融合 MoE、完整 CUDA Graph、MTP 投机解码等全套优化

📎 来源：https://github.com/vllm-project/vllm/releases/tag/v0.22.0

---

### 2️⃣ CONF-KV：让 KV Cache 自己决定"该记什么、该忘什么"

**痛点场景：**
长上下文推理最大的敌人是 KV Cache。传统的 KV Cache 淘汰策略主要有两类：
- **滑动窗口**：只保留最近的 N 个 token。简单粗暴，但可能把关键上下文扔掉。
- **H2O（Heavy Hitter Oracle）**：保留历史注意力分数高的 token。效果好但需要预先计算。

这两种方法都有一个共同问题：它们是"盲目的"。模型明明在生成某个 token 时很犹豫（概率分布很平），你却按固定规则把缓存清了；模型明明很确定下一个词是什么，你却把宝贵的显存留着。

**技术原理：**
CONF-KV（Carnegie Mellon 提出）的核心思想非常巧妙：**用模型当前生成的"置信度"来动态决定 KV Cache 预算**。

具体工作流程：
```python
# 伪代码示意
for each decoding step:
    logits = model(input_ids, kv_cache)
    probs = softmax(logits)
    
    # 计算置信度（如 top-1 概率或熵）
    confidence = compute_confidence(probs)
    
    # 置信度低 → 多保留 KV；置信度高 → 激进淘汰
    cache_budget = map_confidence_to_budget(confidence)
    
    # 在预算内，按"累积注意力质量 + 新近度"排序保留
    retained_tokens = rank_and_select(kv_cache, cache_budget)
```

它还有三个关键设计：
1. **受保护的近期窗口**：无论置信度如何，最近的 token 永远保留，保证局部连贯性
2. **混合精度存储**：保留的 KV 块用 FP16，淘汰候选用 INT8 压缩
3. **金字塔式逐层预算**：不同层的 KV 保留策略不同（浅层语义粗，保留少；深层细节多，保留多）

**实际效果：**
| 方法 | 32K Needle 准确率 | 峰值显存 |
|------|------------------|---------|
| 滑动窗口 512 | 53.8% | 基准 |
| H2O | 80.6% | ~2x 基准 |
| **CONF-KV** | **91.4%** | **~1x 基准** |
| Full KV | 100% | ~2.8x 基准 |

在 VisualWebArena 的 75 个任务上，CONF-KV 保持了 Full KV 95.3% 的成功率，但峰值显存只有 Full KV 的 1/2.8。

📎 来源：https://huggingface.co/papers/2605.24786

---

### 3️⃣ PANDO：让 Web Agent 越用越省 token

**痛点场景：**
现在的 Web Agent（如 VisualWebArena 上的各种方案）普遍有个问题：**每次执行任务都是从零开始**。Agent 不知道之前在哪踩过坑，也不知道哪些操作是重复的。结果就是 token 消耗爆炸——一个 50 步的网页任务，可能消耗数万个 token，成本根本没法降下来。

**技术原理：**
PANDO（也是 CMU 出品）的思路是**在线技能蒸馏**——让 Agent 在运行过程中积累经验，建立"技能库"，下次遇到类似场景直接调用，而不是从头推理。

PANDO 解决三个核心问题：

**① 重复动作循环**：Agent 经常在一个页面上反复点击同一个按钮。PANDO 通过 progress reflection（进度反思）识别这种循环，一旦检测到 2 次以上的重复动作，立即触发策略切换。

**② 隐藏发现成本**：Agent 每次都要重新探索页面结构。PANDO 维护一个 Skill Library（技能库），把常用的操作模式（如"登录流程"、"搜索+筛选"）封装为可复用的 skill。

**③ Prompt Cache 复用率低**：大模型的 prompt cache 可以复用重叠的前缀，但 Agent 的 prompt 通常每次都不同。PANDO 通过 cache-aware prompting（缓存感知提示），有意构造可复用的 prompt 前缀，提升 KV Cache 命中率。

```
用户任务: "在亚马逊搜索蓝牙耳机并找到评分>4.5的产品"

传统 Agent:
  Step 1: 分析截图 → "这是一个电商首页，有搜索框..." (200 tokens)
  Step 2: 点击搜索框 → "搜索框已聚焦..." (150 tokens)
  Step 3: 输入关键词 → "已输入蓝牙耳机..." (180 tokens)
  ... 每步都从零推理

PANDO Agent:
  Step 1: [Skill: ecommerce_search] 直接调用搜索技能 (80 tokens, cache hit)
  Step 2: [Skill: ecommerce_filter] 调用筛选技能 (60 tokens, cache hit)
  ... 经验积累后直接走捷径
```

**实际效果：**
- 910 个 VisualWebArena 任务：58.3% 成功率，超过 SGV（54.0%）和 WALT（45.2%）
- Token 消耗：比 SGV 少 58%，比 WALT 少 61%
- 零预评估发现预算（不需要预先跑一遍来发现技能）

📎 来源：https://huggingface.co/papers/2605.24785

---

### 4️⃣ Parallax：Local Linear Attention 终于能用在预训练了

**痛点场景：**
Softmax Attention 统治了 LLM 这么多年，但它的 O(n²) 复杂度在长上下文时就是灾难。各种线性注意力变体（Linear Attention）试图解决这个问题，但普遍有一个致命缺陷：**预训练效果不如 Softmax Attention**。为什么？因为线性注意力的"局部常数估计"在偏差-方差权衡上不如 Softmax。

**技术原理：**
Parallax（西北大学）的核心洞察是：把 Local Linear Attention（LLA）"参数化"，让模型自己学习最优的注意力模式。

传统 LLA 的问题是需要在推理时解一个数值方程（求逆矩阵），这在大规模预训练中既慢又不稳定。Parallax 做了三个关键改进：

1. **消除数值求解器**：用一个额外的 query-like projector 来探测 KV 协方差，替代原来的数值求解
2. **硬件感知算法**：重新组织计算顺序，提升算术强度（arithmetic intensity），把注意力计算从 memory-bound 推向 compute-bound。简单说，就是让 GPU 的 CUDA Core 更忙、Memory 带宽压力更小
3. **带宽-探针-仿射结构的统一框架**：把 Parallax 和 FlashAttention、Linear Attention 等放在同一个家族里，通过带宽参数连接

最有趣的发现：**Muon 优化器是解锁 Parallax 潜力的关键**。论文发现 Parallax + Muon 的组合在预训练中产生了 Pareto 改进——参数匹配和计算匹配两个对照组下都优于 Softmax Attention。这是首次证明注意力架构和优化器之间存在强协同设计效应。

**实际效果：**
- 在 0.6B 和 1.7B 尺度预训练，全程 perplexity 一致优于 Softmax
- 原型 decode kernel 在多种 batch size 和上下文长度下匹敌或超过 FlashAttention 2/3
- 下游 benchmark 迁移增益显著

📎 来源：https://huggingface.co/papers/2605.29157 | https://github.com/Yifei-Zuo/Parallax

---

## 📰 行业动态

**🔧 vLLM Model Runner V2 向默认引擎迈进**
MRv2 新增了 Qwen3 稠密模型默认选择器、睡眠模式权重重载（sleep-mode weight reload）、共享 KV cache 层等能力，在有 KV connector 时自动回退到 MRv1。Model Runner 是 vLLM 执行引擎的核心抽象层，V2 版本正在逐步成为默认选择。
📎 https://github.com/vllm-project/vllm/releases/tag/v0.22.0

**🛡️ AgentDoG 1.5：1k 样本训练的 Agent 安全守卫**
上海 AI Lab 提出的 AgentDoG 1.5 更新了 Agent 安全分类学（覆盖 Codex 和 OpenClaw 执行场景的新风险），仅需 ~1k 样本即可训练 0.8B-8B 的安全模型，效果媲美 GPT-5.4。Docker 级部署开销降低两个数量级，可作为训练-free 的在线安全护栏。
📎 https://huggingface.co/papers/2605.29801 | https://ai45lab.github.io/AgentDoG/v1_5/

**🧠 SGLang 今天合并多个核心 PR**
- ForwardBatch.init_new 中新增 compute dimensions/return_pooled_hidden_states（#26779）
- 通过 hf_text_config.is_hybrid_swa 识别自定义混合 SWA 模型（#23988）
- 优化大 add_constant tensors（#24755）
📎 https://github.com/sgl-project/sglang

**🎬 AdaState：流式视频生成的自进化锚点**
Virginia Tech 提出用"自适应状态"替代视频扩散模型中固定的首帧锚点。传统方法把首帧的 KV 表示锁定在注意力缓存中，导致视频动态性被抑制。AdaState 让模型在每一步生成一个隐式的场景锚点，KV Cache 充当载体，去噪过程充当状态转移函数，产生更自然的运动。
📎 https://huggingface.co/papers/2605.30349 | https://adastate.github.io/

**📱 UI-KOBE：用知识图谱引导轻量 GUI Agent**
通过自主探索 APP 构建 UI 状态知识图谱（节点=UI 状态，边=可执行操作），运行时轻量 Agent 借助图谱做决策，减少对大模型的依赖。向端侧部署 GUI Agent 迈出了实用的一步。
📎 https://huggingface.co/papers/2605.29534

**🔄 LangGraph SDK 0.4.0 发布**
LangChain 的 Agent 编排框架 LangGraph 发布 SDK 0.4.0 和 CLI 0.4.27，多 Agent 协作编排能力持续增强。
📎 https://github.com/langchain-ai/langgraph/releases

**🧪 HuggingFace Transformers 修复模型并行 Beam Search**
Intel 贡献者提交了模型并行 Beam Search 的一系列 bug 修复，多卡生成场景的正确性得到保障。
📎 https://github.com/huggingface/transformers/pull/46280

**🪞 Reflective Prompt Tuning：用 Function Calling 模拟人类 Prompt 工程师**
Megagon Labs 提出 RPT 框架，LLM 优化器通过诊断函数评估目标模型在整个优化集上的表现，总结失败模式，结合历史记忆迭代修改 prompt。在三类推理任务上比初始 prompt 提升最多 12.9 分。
📎 https://huggingface.co/papers/2605.21781 | https://github.com/megagonlabs/RPT

---

## 💬 结语

今天的信息密度相当高——vLLM 大版本发布、多个 KV Cache 和注意力机制的突破性论文、Agent 效率优化方案。尤其是 CONF-KV 和 Parallax 这两篇，一个从系统工程角度让 KV Cache "智能瘦身"，一个从算法角度给注意力机制 "换个引擎"，都直指长上下文推理的核心痛点。

大家最看好哪个方向在半年内能真正落地到生产环境？欢迎评论区聊聊 👇
