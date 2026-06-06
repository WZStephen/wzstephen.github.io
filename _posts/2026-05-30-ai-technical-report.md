---
layout: post
title: 'Edge MoE、VideoMLA 与 LLM 工作记忆'
date: 2026-05-30 09:00:00 +0800
categories: [ai-technical-report]
---


## 🔥 今日看点

| 时间 | 亮点 |
|------|------|
| 昨天（5/28） | 🔥 Liquid AI 发布 LFM2.5-8B-A1B：8B MoE 模型在 38T token 上训练，128K 上下文，单 H100 日产 16 亿 token |
| 昨天（5/28） | 📝 Mistral AI Now Summit 巴黎回顾：Mistral 转型全栈 AI，主打欧洲主权 + 专用小模型 |
| 昨天（5/28） | 🎓 Show HN: Tiny-vLLM — 用 C++/CUDA 从零手写高性能 LLM 推理引擎 |
| 昨天（5/28） | 📄 arXiv 新论文：VideoMLA 用低秩 KV Cache 解决分钟级视频生成的显存爆炸问题 |
| 昨天（5/28） | 📄 arXiv 新论文：解锁 LLM 工作记忆，让推理过程"暗中"进行而不暴露 token |
| 本周 | 📄 arXiv 新论文：凸重构 + 梯度缓存实现高效测试时微调 |
| 近日 | 🎨 Anthropic 推出 Claude Design，把 Claude 变成设计协作伙伴 |

---

## 💡 深度解读

### 1. Liquid AI LFM2.5-8B-A1B：当 MoE 遇上 Edge，推理效率的新天花板

**痛点场景：** 想在笔记本上跑一个能调工具、能推理的 Agent？传统方案要么用云端大模型（数据泄露、延迟高），要么用本地量化模型（能力大幅缩水）。尤其是多轮工具调用场景，每次 dispatch 都要等 2-3 秒，体验割裂。

**技术原理：** Liquid AI 的做法很巧妙：

- **MoE + 短卷积混合架构**：模型总参数 8B，但每次推理只激活 1B 参数（A1B = Active 1B）。通过 MoE 路由 + GQA（Grouped Query Attention）+ gated short convolution 块的组合，让"每个推理 token 的计算量"大幅下降。
- **38T token 预训练 + 强化学习**：从上一代的 12T 扩展到 38T，词表从 65K 翻倍到 128K（专门优化了 Hindi、Thai、Vietnamese 等非拉丁语系的 tokenization 效率）。
- **推理专用设计**：这个版本是 "reasoning-only" 模型——强制先生成显式 chain of thought 再输出答案。因为 MoE 模型在 compute-bound 设置下运行，active 参数少意味着每个 reasoning token 的代价很低，所以多生成一些思考 token 不会拖慢整体速度。
- **防幻觉 RL**：用 avg@k 奖励函数抑制幻觉——对超出模型知识边界的问题，训练模型明确说"我不知道"而不是编造答案。同时还加了轻量 shaping reward，惩罚 "Wait..." 这类循环触发词。

**实际效果数据：**

| 指标 | LFM2.5-8B-A1B | 对比参照 |
|------|---------------|----------|
| 吞吐量 (H100 SXM5) | **18.5K token/s** | 同尺寸最快 |
| 日产 token (单卡) | **> 16 亿** | 同尺寸最高 |
| M5 Max 解码速度 | **253 token/s** | 内存占用 < 6GB |
| 手机端推理 | ~30 token/s | 首次能在手机上跑的 MoE |
| AA-Omniscience Index | 优于 Granite-4.0-H-Tiny | 接近 Gemma-4-26B-A4B |

**关键 insight：** 这告诉我们 MoE 的"效率红利"正在从云端下沉到 edge。当 active 参数降到 1B 级别时，推理瓶颈从 compute-bound 变成 memory-bound，CPU/低端 GPU 也能跑出不错的吞吐量。Liquid 同时开源了 llama.cpp、MLX、vLLM、SGLang 四个后端的适配——这种"day-one 全框架支持"策略对生态推广非常关键。

📎 来源：https://www.liquid.ai/blog/lfm2-5-8b-a1b

---

### 2. VideoMLA：用低秩 KV Cache 打破视频生成的显存墙

**痛点场景：** 做分钟级视频生成时，KV Cache 的显存占用随帧数线性爆炸。一个 60 秒 24fps 的视频需要 1440 帧，每帧都要存 attention 的 K/V，传统 sliding-window KV Cache 在长序列下显存直接打满。

**技术原理：** VideoMLA 的核心思想是把 KV Cache 做**低秩压缩**——不是简单丢弃旧帧的 KV，而是用一个低秩表示来近似存储：

```
传统方式: KV_cache[t] = [K_1, V_1, K_2, V_2, ..., K_t, V_t]  # O(T×d) 显存
VideoMLA: KV_cache[t] ≈ LowRank(KV) = U × V^T  # O(r×d) 显存, r << T
```

类比一下：传统方式像把每帧视频都保存成原始 RAW 文件；VideoMLA 像是存成"关键帧 + 差分"的压缩格式，但在 attention 计算时能无损还原。

**关键创新点：**
- 在 latent space（而非 pixel space）做自回归视频扩散，天然降低了序列长度
- 低秩分解的 rank 自适应调整——新帧用 full KV，旧帧用 low-rank 近似
- 与现有的 sliding-window 方法正交，可以叠加使用

**意义：** 这给视频生成社区提供了一条不需要改硬件就能做更长视频的路。如果你在做 CogVideoX、Wan2.1 之类的应用，VideoMLA 的 KV Cache 压缩思路值得参考——本质上是用矩阵低秩近似来 trade 一点精度换显存。

📎 来源：https://arxiv.org/abs/2605.30351

---

### 3. 解锁 LLM 工作记忆：让推理"暗中"进行

**痛点场景：** 现在的推理模型（o1、R1 等）都是通过生成中间 token 来做"思考"的。但这有两个问题：
1. **推理过程暴露**：用户能看到完整的 chain of thought，包括错误尝试和回退
2. **推理与输出耦合**：思考 token 也是 autoregressive 生成的，internal computation 和 external communication 混在一起，导致延迟翻倍

**技术原理：** 这篇论文提出了一种让 LLM 做 **latent reasoning** 的方法——在模型内部做推理，不生成可见的 intermediate token。

类比：传统 CoT 像是"大声念出解题过程"，latent reasoning 像是"在心里想好再说出答案"。

具体来说，论文通过解锁模型 hidden state 中的 working memory capacity，让模型在不输出 token 的情况下进行多步推理。这类似于 Transformer 的 hidden state 本身就有一定的"工作记忆"能力，但通常被下一 token 生成任务占用了。论文的方法释放了这部分 capacity 用于纯内部计算。

**意义：** 如果这个方向走通，意味着：
- 推理延迟可以大幅降低（不用输出 thinking token）
- 推理过程可以保持私密（适合企业场景）
- 模型可以在"想"和"说"之间切换，而不是被迫"边想边说"

这对推理引擎的设计也有影响——现有的 streaming 输出架构假设每个生成步骤都有 token 输出，latent reasoning 可能需要新的 serving 模式。

📎 来源：https://arxiv.org/abs/2605.30343

---

### 4. Tiny-vLLM：用教学项目理解 vLLM 的核心

**痛点场景：** 很多同学想用 vLLM/SGLang 做推理优化，但面对几十万行代码无从下手。PagedAttention、continuous batching、CUDA kernel 优化这些概念，看论文懂了，看源码还是懵。

**这个项目做了什么：** Tiny-vLLM 是一个**教学向**的 C++/CUDA 推理引擎，用更少的代码实现了 vLLM 的核心概念：

- ✅ Safetensors 模型加载
- ✅ CUDA Embedding 查找
- ✅ RMSNorm（含 parallel reduction 优化）
- ✅ cuBLAS 矩阵乘法
- ✅ PagedAttention CUDA kernel
- ✅ Continuous batching（多请求并发处理）

代码量只有 vLLM 的几百分之一，但核心概念一一对应。适合：
1. 理解 PagedAttention 的内存管理逻辑
2. 学习 CUDA kernel 编写（特别是 parallel reduction 技巧）
3. 搞懂 continuous batching 的调度逻辑

**代码片段示例（PagedAttention 思路）：**
```cpp
// 传统方式：每个序列分配连续显存，浪费严重
float* kv_cache = cudaMalloc(batch_size * max_seq_len * hidden_dim);

// PagedAttention：像操作系统分页一样管理 KV
// 只分配实际使用的 block，通过 page table 映射逻辑地址到物理 block
struct PageTable {
    int* logical_to_physical;  // 逻辑页号 -> 物理页号映射
    int num_pages;
};
```

📎 来源：https://github.com/jmaczan/tiny-vllm

---

## 📰 行业动态

1. **Mistral AI Now Summit（巴黎）**：Mistral 宣布转型全栈 AI 公司，自建 40MW 巴黎数据中心，与 ASML、BNP Paribas、Amazon Alexa+ 深度合作。战略方向：**专用小模型 + 欧洲数据主权 + 本地部署**。Document AI 做 OCR（欧盟专利局在用）、Voxtral 做多语言语音（驱动 Alexa+ 欧洲版）、Robostral 做工业机器人（ASML 合作）。
   📎 https://koenvangilst.nl/lab/mistral-ai-now-summit

2. **Anthropic 推出 Claude Design**：Anthropic Labs 新产品，让用户与 Claude 协作创建 polished visual design。这是 Anthropic 从纯文本 AI 向多模态创作工具扩展的重要一步。
   📎 https://www.anthropic.com/news/claude-design

3. **Shift AI 用免费家政服务收集机器人训练数据**：一家初创公司提供"免费打扫房间"服务，目的是收集真实环境中的机器人训练数据。用物理世界的数据来训练未来家政机器人。
   📎 https://www.theverge.com/ai-artificial-intelligence/939765/ai-training-data-startup-shift-free-cleaning

4. **SGLang 生态持续扩张**：SGLang README 更新显示已部署在**超过 40 万张 GPU** 上，采用方包括 xAI、AMD、NVIDIA、Cursor、Oracle Cloud 等。最新技术动态包括 GB300 NVL72 上的 25x 性能优化（2026/02）和 SGLang Diffusion 加速视频生成（2026/01）。
   📎 https://github.com/sgl-project/sglang

5. **vLLM 持续迭代**：vLLM 已累积 17,107 次 commit，支持 200+ 模型架构。最新特性包括 FP8/MXFP8/MXFP4/NVFP4 量化、FlashAttention/FlashInfer/TRTLLM-GEN 优化 attention kernel、disaggregated prefill/decode/encode 架构。
   📎 https://github.com/vllm-project/vllm

6. **arXiv 论文：高效测试时微调** — 提出通过凸重构和梯度缓存实现 Efficient Test-Time Finetuning（TTFT），为每个 prompt 检索相关序列并更新模型权重，而不需要完整的微调流程。
   📎 https://arxiv.org/abs/2605.30337

7. **arXiv 论文：LLMSurgeon 诊断 LLM 数据混合** — 提出一种方法诊断 LLM 预训练数据混合比例对模型行为的影响，帮助理解模型的"digital DNA"。
   📎 https://arxiv.org/abs/2605.30348

---

## 💬 结语

今天最值得关注的趋势是 **MoE 模型的 edge 化**和**推理架构的范式创新**。Liquid AI 证明 8B 总参数 / 1B active 的 MoE 模型可以在笔记本和手机上跑完整的 agent workflow——这直接挑战了"Agent 必须依赖云端大模型"的假设。同时，latent reasoning 的论文提出了一个有趣的问题：如果推理可以不生成 token，我们的 serving 架构该怎么重新设计？

你最看好哪个方向？edge MoE、latent reasoning、还是低秩 KV Cache？欢迎评论区交流 👇
