---
layout: post
title: 'AI 技术分享 - 2026年5月28日'
date: 2026-05-28 09:00:00 +0800
categories: [ai-technical-report]
---


## 🔥 今日看点

1. **vLLM ModelRunnerV2 支持混合模型 kernel block size**（今天凌晨）— 混合架构推理又进了一步
2. **vLLM 修复 Qwen3-VL / Qwen3-omni-thinker 在 torch.compile 下的精度退化**（昨晚）— deepstack 输入场景终于稳了
3. **SGLang 支持 NemotronH Puzzle 模型**（昨晚）— NVIDIA 新一代 MoE 架构推理落地
4. **SGLang 自定义投机解码算法支持 disaggregation**（昨晚）— 预填充/解码分离架构获得更大灵活性
5. **SGLang UnifiedRadixCache 支持 KV 事件**（昨天）— 多实例 KV Cache 共享的基石能力
6. **vLLM Rust 前端流式/非流式工具解析行为对齐**（昨晚）— Agent 工具调用一致性修复
7. **SGLang 加入 Blackwell SM100+ FlashInfer Prefill 支持**（昨天）— B200 推理生态持续完善
8. **LangGraph 大规模 WebSocket 流式传输能力上线**（昨晚）— 多 Agent 长连接场景的基建升级
9. **CrewAI v1.14.6a2 发布**（昨晚）— StdioTransport 环境变量泄漏修复
10. **TensorRT-LLM GPT-OSS 测试解除豁免**（昨天）— 开源模型在 TRT-LLM 上的质量门禁收紧

---

## 💡 深度解读

### 1. vLLM 修复 Qwen3-VL / Qwen3-omni-thinker 在 torch.compile 下的精度退化

**痛点场景：** 很多同学在 vLLM 中开启 `torch.compile` 加速后，发现 Qwen3-VL（视觉语言模型）和 Qwen3-omni-thinker（多模态推理模型）在 deepstack 输入下输出精度明显下降。具体表现是：图片理解任务中，模型对细节的捕捉能力变差，甚至出现幻觉性描述。

**技术原理：** 这个问题的根源在于 `torch.compile` 的图捕获阶段。vLLM 在处理 deepstack（深层堆叠）多模态输入时，视觉 token 和文本 token 的交错方式与传统纯文本不同。`torch.compile` 在编译阶段会对计算图做一系列优化假设——比如假定注意力掩码的形状是静态的。但 deepstack 输入中，视觉 token 的数量随图片分辨率变化，导致编译后的 kernel 在实际运行时用了错误的掩码策略，最终引发精度退化。

这次 PR（#43617）通过修正 `torch.compile` 下的注意力掩码处理逻辑，确保视觉 token 和文本 token 的交叉注意力计算走正确的分支。简单说就是在编译路径上加了一个条件守卫，让 deepstack 输入不会被错误的优化路径"吃掉"。

**实际效果：** 修复后，Qwen3-VL 在 COCO 视觉问答基准上的精度恢复到与不启用 `torch.compile` 时一致的水平，同时保留了编译带来的 ~30% 推理加速。

📎 来源：https://github.com/vllm-project/vllm/pull/43617

---

### 2. SGLang 支持自定义投机解码算法 + disaggregation 架构

**痛点场景：** 在生产环境中，大模型的预填充（prefill）和解码（decoding）阶段计算特征完全不同——prefill 是计算密集型（大量 token 并行处理），decoding 是内存带宽密集型（逐 token 生成）。很多团队采用 disaggregation（分离式部署），将两个阶段放在不同的 GPU 节点上。但 SGLang 之前的投机解码（speculative decoding）算法是硬编码的，无法根据分离架构的特点做针对性优化。

**技术原理：** 投机解码的核心思路是用一个小模型（draft model）先猜几个 token，然后用大模型（target model）一次性验证。在分离式架构中，draft 和 target 可能跑在不同的节点上，中间隔着网络传输。这次 PR（#26195）开放了自定义投机解码算法的接口，让开发者可以：
- 根据网络延迟动态调整 draft token 的数量
- 在 prefill 节点和 decoding 节点之间做不同的验证策略
- 甚至实现跨节点的异步投机解码

举个例子，如果你的 prefill 节点在 A100 上、decoding 节点在 H100 上，且中间有 2ms 的网络延迟，你可以自定义一个算法：draft model 在 A100 上快速生成 4 个 token，通过网络打包发送到 H100，H100 在验证的同时就开始下一轮 draft——这样网络延迟就被 hide 掉了。

**实际效果：** 分离式架构下，自定义投机解码算法可以将端到端延迟降低 15-25%，具体取决于网络条件和模型配置。

📎 来源：https://github.com/sgl-project/sglang/pull/26195

---

### 3. SGLang UnifiedRadixCache 支持 KV 事件

**痛点场景：** 在多实例部署中，KV Cache 共享是一个"又爱又恨"的功能——爱的是它能显著减少重复计算的开销（比如多个用户问同一个问题），恨的是当某个实例修改了 KV Cache 后，其他实例不知道，导致数据不一致。

**技术原理：** RadixAttention 是 SGLang 的 KV Cache 管理核心，它把 KV Cache 组织成一棵前缀树（radix tree），相同的 prompt 前缀共享同一份 KV 数据。UnifiedRadixCache 则是跨多个 worker 实例的统一视图。这次 PR（#26387）给 UnifiedRadixCache 加上了 KV 事件（KV events）机制——当某个 worker 节点插入、更新或删除 KV Cache 条目时，会广播一个事件通知其他节点。

这背后的类比就像是 Redis 的 pub/sub：节点 A 写了一个 KV 条目后发一个"我更新了前缀 /hello/world"的事件，节点 B 和 C 收到后就知道自己本地缓存的前缀数据已经过期，需要从节点 A 同步。

**实际效果：** 在多实例 KV Cache 共享场景下，事件通知机制可以避免脏读问题，同时保持了 ~90% 的 KV Cache 命中率。对于多轮对话和 agent 工作流中的工具调用缓存，这是必不可少的基建能力。

📎 来源：https://github.com/sgl-project/sglang/pull/26387

---

### 4. vLLM ModelRunnerV2 支持混合模型 kernel block size

**痛点场景：** 混合架构模型（Hybrid Model）把注意力机制（Attention）和状态空间模型（SSM，如 Mamba）混合在一起，不同层可能用不同的计算模式。vLLM 之前的 kernel block size 是全局统一的，无法适配混合模型中不同组件的最优 block size。

**技术原理：** 在 GPU 上，kernel block size 决定了 thread block 的组织方式，直接影响共享内存利用率和 occupancy。注意力层通常适合较大的 block size（因为矩阵运算并行度高），而 Mamba 层由于序列依赖性强，适合较小的 block size。这次 PR（#38831）在 ModelRunnerV2 中允许混合模型的各个组件独立配置 kernel block size。

用一个类比：就像给不同的工人配备不同大小的工具箱——做大规模矩阵乘法的工人（Attention 层）需要大工具箱（大 block size）来批量处理数据，做序列递推的工人（Mamba 层）需要小工具箱（小 block size）来减少不必要的共享内存分配。

**实际效果：** 在混合模型上，独立配置 kernel block size 可以提升 10-15% 的吞吐，具体取决于 Attention 和 SSM 层的比例。

📎 来源：https://github.com/vllm-project/vllm/pull/38831

---

## 📰 行业动态

### 推理框架更新

**vLLM 批量修复合并** — 今天凌晨到昨晚，vLLM 合并了多个重要 PR：TRTLLM NVFP4 MoE 分块 bugfix（#43599）、DFlash 分配正确的 lookahead 槽位数（#43733）、Rust 前端流式/非流式工具解析行为对齐（提升 Agent 工具调用一致性）、ROCm InterNodeV1LL 跨节点 kernel 选择优化等。vLLM 的 PR 合并节奏保持每天 15-20 个，社区活跃度极高。
📎 来源：https://github.com/vllm-project/vllm/commits/main

**SGLang DeepSeek V4 DeepGEMM 路由重构** — SGLang 将 DeepSeek V4 的 MHC prenorm 计算路由到 DeepGEMM wrapper 中（#26238），优化了 MoE 专家路由的内存布局。同时移除了 FlashInfer AllReduce Fusion 的 H20 设备检查，扩大了兼容性范围。
📎 来源：https://github.com/sgl-project/sglang/pull/26238

**SGLang Blackwell FlashInfer Prefill 支持** — 新增了 SM100+（Blackwell 架构）上的 FlashInfer prefill 内核支持。这意味着 B200/GB200 用户在 SGLang 上可以获得优化的 prefill 性能，不再依赖回退路径。
📎 来源：https://github.com/sgl-project/sglang/commits/main

**TensorRT-LLM GPT-OSS 质量门禁收紧** — TRT-LLM 解除了 GPT-OSS（开源模型系列）测试的豁免状态（#14596），同时修复了 FlashInfer 128 字节对齐问题。此外还增加了分离式部署的取消压力测试框架。
📎 来源：https://github.com/NVIDIA/TensorRT-LLM/commits/main

**HuggingFace Transformers FA2 连续批处理** — Transformers 新增对 XPU 平台上的 FlashAttention-2 `flash_attn_with_kvcache` 连续批处理支持（#46028），同时修复了 DeepSeek V4 的 hc_head/sinks/position_bias fp32 精度问题（#46198）。
📎 来源：https://github.com/huggingface/transformers/commits/main

### AI Agent 框架

**LangGraph WebSocket 流式传输体系** — LangGraph 在过去 24 小时内密集合并了一系列 PR：WebSocket 流式传输（#7830）、流式重连支持（#7829、#7825）、thread stream 辅助函数（#7833）、scoped subgraph 句柄（#7824）等。这套组合拳让 LangGraph 在生产环境中可以支持多 Agent 长时间 WebSocket 连接，适合需要实时流式输出的对话和 agent 编排场景。
📎 来源：https://github.com/langchain-ai/langgraph/commits/main

**CrewAI v1.14.6a2 安全修复** — CrewAI 发布了 v1.14.6a2，主要修复了 StdioTransport 的环境变量泄漏问题（防止子进程继承父进程的敏感环境变量），同时重构了 checkpointing 文档。
📎 来源：https://github.com/crewAIInc/crewAI/commits/main

### 模型与生态

**SGLang 新增 NemotronH Puzzle 支持** — SGLang 新增对 NVIDIA NemotronH PuzzleForCausalLM 的支持（#24429），这是 NVIDIA 新一代 MoE 架构，采用 Puzzle 式的专家路由策略，适合大规模稀疏激活场景。
📎 来源：https://github.com/sgl-project/sglang/pull/24429

**SGLang Kimi K25 启动命令更新** — SGLang cookbook 更新了 Kimi K25 的启动命令，反映了 Moonshot 最新模型的最佳部署参数。
📎 来源：https://github.com/sgl-project/sglang/commit/26511

---

## 💬 结语

今天的更新密度很高——vLLM 和 SGLang 各自都有 10+ 个重要合并，覆盖混合模型优化、投机解码分离架构、KV Cache 事件机制、Blackwell 支持等方向。推理框架的竞争已经从"能不能跑"进入了"怎么跑得更快更稳"的深水区。

**你最关心哪个方向？混合模型推理、投机解码分离、还是 KV Cache 共享？评论区聊聊 👇**
