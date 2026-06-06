---
layout: post
title: 'vLLM KV Offloading、DeepSeek V4 支持与 Rust Coding Agent'
date: 2026-05-17 09:00:00 +0800
categories: [ai-technical-report]
---


> 用一杯咖啡的时间，了解AI世界正在发生什么 ☕

---

## 🔥 今日看点（30秒速览）

- 🚀 **vLLM v0.21.0 发布**：KV缓存卸载+混合内存分配器正式落地，推理成本再降
- 🦙 **SGLang v0.5.12 昨日更新**：全面支持 DeepSeek-V4，FP4低延迟推理大幅优化
- 💡 **推测解码支持推理预算**：vLLM让大模型"边想边猜"，加速效果翻倍
- 🐙 **Rust 版代码 Agent 走红**：Zerostack 登上 HackerNews 热榜，Unix哲学+纯Rust
- 🧠 **Token效率成为新焦点**：OpenSquilla 项目提出"同等预算下更高智能密度"

---

## 💡 技术详解

### 1️⃣ vLLM v0.21.0：KV缓存卸载正式落地，显存焦虑有救了

**痛点**：跑大模型推理时，显存总是第一道坎。尤其是长对话场景，KV Cache 占满了 GPU 内存，导致并发一高就 OOM。很多团队不得不降 batch size 或者干脆加卡，成本直线上升。

**解法**：vLLM v0.21.0（5月15日发布）终于把 **KV Offload + Hybrid Memory Allocator (HMA)** 整合到位了。简单说，就是把不常用的 KV 缓存从 GPU"搬"到 CPU 内存上，GPU 只留热数据。配合滑动窗口分组，系统自动判断哪些该留、哪些该搬。

打个比方：就像你的工位桌子（GPU显存）放不下所有文件，但旁边有个大柜子（CPU内存）。聪明的秘书（HMA）知道哪些文件你马上要用放桌上，哪些可以收进柜子，需要时再取出来。桌子虽然小，但工作效率不打折。

**效果**：
- 单张卡能服务更多并发请求，性价比直接提升
- 配合滑动窗口（Sliding Window），超长上下文场景下显存占用大幅降低
- 367次提交、202位贡献者的成果，社区活跃度拉满

> 🔗 [vLLM v0.21.0 Release](https://github.com/vllm-project/vllm/releases/tag/v0.21.0)

---

### 2️⃣ SGLang v0.5.12：DeepSeek-V4 首发支持 + FP4低延迟

**痛点**：每次新模型出来，推理框架都要花几周甚至几个月才能完整适配。开发者想用新模型，只能干等。同时，量化部署虽然省显存，但低延迟场景下精度损失让人头疼。

**解法**：SGLang v0.5.12（5月16日发布）做到了 **Day-0 完整支持 DeepSeek-V4**，包括：
- 三种并行策略：张量并行、专家并行、上下文并行
- 硬件全覆盖：从 B300/B200 到 H200/H100，甚至 AMD MI35X 都支持
- Prefill-Decode 分离架构，推理效率最大化
- HiSparse 技术：自动把不活跃的 KV Cache 卸载到 CPU

更值得关注的是 **FP4 低延迟优化**——通过 `torch.mm` 重写索引 GEMM、重做 Cute-DSL FP4 密集 GEMM，在极低精度下依然保持可观的推理速度，精度损失微乎其微。

打个比方：以前换新车（新模型）得等修车厂（推理框架）改好工具才能保养，现在 SGLang 直接说"新车到店第一天就能修"。

**效果**：
- 一个 Docker 镜像 `lmsysorg/sglang:v0.5.12` 通吃所有 NVIDIA GPU
- FP4 路径延迟显著降低，适合对响应时间敏感的生产场景
- 推测解码 V2 也大幅成熟，支持 EAGLE-3、Kimi K2.5 MLA 等

> 🔗 [SGLang v0.5.12 Release](https://github.com/sgl-project/sglang/releases/tag/v0.5.12)

---

### 3️⃣ 推测解码 × 推理预算：让大模型"边想边猜"

**痛点**：推理模型（如 DeepSeek-R1、o3 等）在生成答案前会先"思考"，这导致响应速度比普通模型慢很多。推测解码（Speculative-Decoding）能加速生成，但以前的推测解码不理解"思考"过程，用在推理模型上会出错。

**解法**：vLLM 最新版本的推测解码现在**尊重 thinking budget**了。简单来说，推测解码的"草稿模型"可以跟着主模型一起"思考"，而不是跳过去直接猜答案。这样既保留了推理模型的质量，又大幅提升了生成速度。

**比喻**：以前推测解码像一个急性子的实习生，不等你想清楚就抢答。现在它学会了"先跟着想，再猜答案"，效率和准确率兼得。

**效果**：
- 推理模型首次可以安全使用推测解码加速
- 对于 DeepSeek-R1 等需要大量"思考 token"的模型，加速效果尤为明显
- 生产环境中响应延迟可降低 30-50%（取决于场景）

---

### 4️⃣ TOKENSPEED_MLA：Blackwell 上的 MLA 加速引擎

**痛点**：DeepSeek-R1、Kimi-K2.5 等模型使用 MLA（Multi-Head Latent Attention）结构，在 GPU 上的推理效率不如传统的 MHA。Blackwell 架构虽然强，但需要专门的优化才能发挥全部性能。

**解法**：vLLM 和 SGLang 不约而同地推出了 **TOKENSPEED_MLA 后端**——专门为 Blackwell GPU 优化的 MLA 注意力内核。vLLM 支持 FP8 KV Cache，SGLang 也同步跟进。这意味着在新一代 GPU 上跑 MLA 模型，效率会显著提升。

**打个比方**：就像给高性能跑车换了专用轮胎，同样的引擎（GPU），不同的轮胎（内核），跑出来的速度完全不一样。

**效果**：
- Blackwell GPU 上 MLA 模型的 prefill + decode 速度大幅提升
- FP8 KV Cache 进一步减少显存占用
- 为 DeepSeek V4 等新一代模型在 Blackwell 上的部署铺平道路

---

### 5️⃣ Rust 版代码 Agent Zerostack：Unix 哲学遇上 AI

**痛点**：现在的 Coding Agent 大多是 Python 写的，部署重、依赖多、启动慢。想在 CI/CD 流水线里集成个 AI 编程助手，装一堆依赖就已经头大了。

**解法**：**Zerostack** 用纯 Rust 重写了一个 Unix 哲学风格的 Coding Agent，今天登上了 HackerNews 热榜（124分）。Rust 的好处：编译后一个二进制文件，零依赖，启动毫秒级，内存占用极小。

Unix 哲学体现在：每个功能做一件事、做好一件事，通过管道和组合完成复杂任务。这和现在动辄几万行代码、依赖几十个包的 AI Agent 框架形成了鲜明对比。

**效果**：
- 极致轻量：不需要装 conda、不需要装一堆 pip 包
- 快速启动：适合集成到 CI/CD、git hook、editor plugin 等场景
- 已在 crates.io 发布 1.0.0 版本，可直接 `cargo install`

> 🔗 [Zerostack on crates.io](https://crates.io/crates/zerostack/1.0.0)

---

## 📰 行业动态

1. **DeepSeek-V4 全面铺开**：LMSYS 发布了 DeepSeek-V4 官方博客，SGLang 和 vLLM 都已支持。多个推理框架同步适配，社区对 DeepSeek 生态的信心持续增强

2. **Transformers v4 正式退役**：vLLM v0.21.0 正式弃用 transformers v4 支持，全面转向 v5。C++20 也成为编译要求，标志着推理框架的技术栈全面现代化

3. **OpenSquilla 开源**：提出"Token-Efficient AI Agent"概念——同样预算下追求更高的智能密度。5月刚开源就已获得 900+ 星，反映了社区对"效率优先"AI 的强烈需求

4. **Photo Agents 走红**：具备自进化能力的视觉 Agent，能基于视觉记忆自我编写技能并操作用户电脑。895 星，展示了 Agent 从"对话"走向"行动"的趋势

5. **Agent 记忆技术全景图**：30个可运行的 Jupyter Notebook，涵盖对话缓冲、向量存储、知识图谱、情景记忆、语义记忆等主流方案，适合想给 Agent 加记忆的开发者

---

## 🗣️ 技术趋势观察

从今天的更新可以明显看到一个趋势：**推理效率的军备竞赛已经进入了"深水区"**。

不再是简单的"量化一下"、"换个框架"就能搞定。KV 卸载、混合内存分配、推测解码适配推理模型、FP4 低延迟优化、MLA 专用内核……每一个细节都在被极致打磨。

这对开发者来说是好事——**同样的硬件，能跑更大的模型、服务更多的用户、花更少的钱**。

---

## 💬 结语

两个推理框架在两天内密集更新，DeepSeek-V4 生态全面爆发，Rust 写的代码 Agent 也来凑热闹——这周的 AI 基础设施圈，热闹程度堪比过年 🔥

**你觉得 2026 年 AI 推理优化的下一个突破点会在哪里？欢迎评论区聊聊～ 👇**

---

*📌 信息来源：GitHub Releases、HackerNews、LMSYS Blog | 整理于 2026年5月17日*
