---
layout: post
title: 'AI 技术分享 - 2026年05月27日'
date: 2026-05-27 09:00:00 +0800
categories: [ai-technical-report]
---


> 📅 日期：2026年05月27日（星期二）
> 🏷️ 关键词：SGLang v0.5.12.post1、vLLM v0.21.0、Qwen3.6 MoE 量化、纯 Rust 推理引擎、AI Agent 执行状态复用

---

## 🔥 今日看点

1. 🚀 SGLang 发布 v0.5.12.post1（昨晚）—— RadixAttention KV Cache 复用持续迭代，多 Agent 场景首 token 延迟显著下降
2. ⚡ vLLM v0.21.0（5月15日）—— Prefix Caching + Chunked Prefill 联合调度成为稳定特性，长上下文 TTFT 优化到位
3. 🧠 Qwen3.6-MoE-35B-A3B 社区 NVFP4 量化版本今天上 HF —— Blackwell 原生精度下 MoE 模型显存再砍一刀
4. 🦀 纯 Rust 推理引擎 `arle` 开源（今天更新）—— 支持本地 Agent 服务、On-Policy Distillation，Rust 推理生态加速
5. 🤖 `deeplossless` 新项目（今天更新）—— 推理感知的 AI 编码 Agent 运行时，通过复用执行状态减少重复 reasoning 开销
6. 📊 openai/gpt-oss-120b 登上 HF 全榜 Likes 前 15 —— OSS 模型生态竞争加剧
7. 📐 HuggingFace 上多模型 KV-cache-optimized 量化版本集中涌现 —— fraqtl 等新量化方案成为社区标配
8. 🔧 GitHub 新项目 `qwen36-27b-single-3090` —— 单张消费级 3090 跑 27B 模型的极致推理调优

---

## 💡 深度解读

### 1. SGLang v0.5.12.post1：KV Cache 复用的最后一块拼图

**大家遇到的痛点：** 跑多 Agent 场景时，比如一个 Planner Agent 分发任务给三个 Worker Agent，每个 Agent 都要带着相同的 system prompt 和任务上下文做 prefill。这些 token 的 KV Cache 被反复计算、反复存储——GPU 显存就这么烧掉了，而且每个请求的首 token 延迟都居高不下。

**SGLang 怎么解决：** RadixAttention 把 KV Cache 组织成一棵**前缀树（Radix Tree）**。多个请求如果有相同的 token 前缀，只存一份 KV Cache，通过指针共享。

举个具体场景：

```
Agent A: [system prompt] + [task desc] + "请分析这段代码的复杂度"
Agent B: [system prompt] + [task desc] + "请为这段代码写单元测试"
Agent C: [system prompt] + [task desc] + "请重构这段代码使其更易读"
```

三个请求共享 `[system prompt] + [task desc]` 的 KV Cache，只计算一次。v0.5.12.post1 的改进在于跨批次持久化和 LRU + 预算驱逐——显存吃紧时自动淘汰最久未使用的子树，而不是一刀切清掉所有 cache。

实际效果：在一个 8-agent 协作场景中（每个 agent 的 system prompt 约 2K tokens），首 token 延迟从 ~800ms 降到 ~200ms，因为绝大多数 prefill 被 cache 命中跳过了。

> 🔗 来源：[SGLang v0.5.12.post1 Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.12.post1)

---

### 2. vLLM v0.21.0：Prefix Caching + Chunked Prefill 终于稳定了

**痛点：** 长上下文场景（比如 128K 输入）做 prefill 时，GPU 一次性计算所有 token 的 KV Cache 会占用大量显存，而且期间 GPU 无法处理任何 decoding 请求——其他短请求的用户只能干等。

**vLLM v0.21.0 的做法：** Prefix Caching 负责识别并复用已计算的 token 前缀的 KV Cache；Chunked Prefill 则把长 prefill 拆成小 chunk，在 chunk 间隙穿插处理 decoding 请求。两者结合后：

- 重复前缀的请求直接跳过 prefill（命中 cache）
- 新的长请求分块执行，不会阻塞短请求
- TTFT（首 token 延迟）在长上下文场景下降低约 40%

这其实就是把 serving 从"批处理模式"变成了"流水线模式"。

> 🔗 来源：[vLLM v0.21.0 Release](https://github.com/vllm-project/vllm/releases/tag/v0.21.0)

---

### 3. Qwen3.6-MoE-35B-A3B NVFP4 量化：MoE 模型也能塞进消费级 GPU

**痛点：** MoE 模型参数巨大但实际激活的 token 很少——比如 Qwen3.6-MoE-35B-A3B 总参数 35B，但每个 token 只激活 3B。传统 INT8/INT4 量化对 MoE 的收益有限，因为激活的 expert 部分已经很小了，瓶颈在于存储所有 inactive expert 的权重。

**NVFP4 量化的思路：** 利用 Blackwell 架构的 FP4 原生支持，对 inactive expert 权重做 4-bit 浮点量化，active expert 保持高精度。这样模型整体显存占用降低约 60%，但推理精度几乎无损（因为 bottleneck 只在存储不在计算）。

今天 HF 上已经出现了社区量化版本 `HackAfterDark/Carnice-Qwen3.6-MoE-35B-A3B-NVFP4`，说明这个方案已经可以在实际部署中使用了。

> 🔗 来源：[HF 量化模型页面](https://huggingface.co/HackAfterDark/Carnice-Qwen3.6-MoE-35B-A3B-NVFP4)

---

### 4. Rust 推理引擎崛起：`arle` 和 `deeplossless`

**趋势：** 今天 GitHub 上两个值得关注的新项目都指向同一个方向——**用 Rust 重写 AI 推理基础设施**。

`arle`（纯 Rust 推理引擎）：
- 支持本地 Agent 服务
- 内置 On-Policy Distillation（策略蒸馏）
- Rust 的内存安全和零成本抽象让推理引擎的并发控制更安全

`deeplossless`（推理感知 Agent 运行时）：
- 核心 idea：AI 编码 Agent（如 Claude Code、Codex）在编辑同一个代码库时，大量 reasoning 结果是重复的
- 通过复用执行状态（AST 解析结果、类型信息、之前的推理结论），减少每次 task 的重复计算
- 类似浏览器对静态资源的 cache 机制，但用在 Agent 的推理链上

这个方向值得持续关注——Rust + AI 推理的组合正在从"玩具项目"走向"生产可用"。

> 🔗 来源：[arle GitHub](https://github.com/cklxx/arle) · [deeplossless GitHub](https://github.com/gordonlu/deeplossless)

---

## 📰 行业动态

1. **HuggingFace 上 KV-cache-optimized 模型批量涌现** —— fraqtl 等量化方案配合新的 KV cache 优化标记，GGUF 格式的优化模型成为社区标配（[HF 搜索结果](https://huggingface.co/models?sort=lastModified&filter=kv-cache-optimized)）
2. **单卡跑大模型持续突破** —— GitHub 新项目 `qwen36-27b-single-3090` 展示了在单张 RTX 3090（24GB 显存）上运行 Qwen3.6-27B 的极致推理配置，消费级 GPU 的推理能力边界被不断推高（[GitHub 项目](https://github.com/lergah/qwen36-27b-single-3090)）
3. **openai/gpt-oss-120b 持续走红** —— HF 全榜 Likes 突破 4800，开源模型生态中 OSS 阵营的竞争日趋激烈（[HF 页面](https://huggingface.co/openai/gpt-oss-120b)）
4. **SpiceAI 突破 2900 stars** —— 纯 Rust 的 SQL 查询 + LLM 推理引擎，将结构化数据查询与非结构化推理统一在一个引擎中（[GitHub 项目](https://github.com/spiceai/spiceai)）
5. **多 Agent 协作基础设施涌现** —— GitHub 新项目 `agentspaces` 提出通过 lease（租约）、共享状态和原子 claim 来实现可靠的多 Agent 任务路由（[GitHub 项目](https://github.com/harshramg5007/agentspaces)）

---

## 💬 结语

KV Cache 复用、Rust 推理引擎、MoE 量化——三条线同时在推进推理成本的下降。当 27B 模型能在单张 3090 上跑的时候，很多"需要 GPU 集群"的论调就得重新审视了。

**你目前在用什么方案跑推理？欢迎评论区聊聊你的踩坑经验 👇**
