---
layout: post
title: 'Codex Goals、Claude Thinking Token 与 TensorRT-LLM FP4'
date: 2026-05-22 09:00:00 +0800
categories: [ai-technical-report]
---


> 推理框架狂飙、Agent 工具链大升级、多模态新架构登场

---

## 🔥 今日看点

| 时间 | 亮点 |
|------|------|
| 昨晚 23:48 | **OpenAI Codex CLI v0.133.0** 发布——Goals 默认启用，插件生态大升级 |
| 昨晚 04:01 | **Anthropic SDK v0.104.0** 发布——新增 thinking-token-count beta，流式推理可控了 |
| 昨晚 22:23 | **OpenAI Python SDK v2.38.0** API 更新 |
| 昨晚 22:27 | **TensorRT-LLM v1.3.0rc15**——Gemma4 多模态、FP4 解码内核、EAGLE3 投机解码全面进化 |
| 今天凌晨 | **vLLM 过去24h合并 44个 PR**——DP Supervisor 落地、Rust 前端并入主仓、XPU 稀疏注意力 |
| 今天凌晨 | **SGLang 过去24h合并 67个 PR**——FLUX.2 扩散模型支持、FutureMap 路由重构 |
| 今天凌晨 | **LangGraph 1.2.1** 发布——`before_builtins` 可选钩子让流式变换更灵活 |
| 今天 | **Qwen3.6 系列霸榜 HF Trending**——27B 和 35B-A3B MoE 持续引爆社区 |

---

## 💡 深度解读

### 1️⃣ TensorRT-LLM v1.3.0rc15：MoE 推理性能再突破，FP4 解码来了

**痛点场景：**
MoE 模型（如 DeepSeek-V3/V4、Qwen3-MoE）推理时，专家路由 + SwiGLU 激活是最大瓶颈。在 H100 上跑 DeepSeek-V3，光 MoE 路由开销就能吃掉 30%+ 的 latency。更不用说 FP8/FP4 量化后，decode 阶段内核效率直线下降——很多量化方案训得好，但推理时因为 kernel 不匹配，吞吐量还不如 BF16。

**TensorRT-LLM v1.3.0rc15 怎么解决：**

这个 RC 版本几乎是一次 MoE 推理的全栈优化：

- **MegaMoE DeepGEMM**：将 MoE 中的 GEMM 操作与 DeepGEMM 融合，减少 kernel launch 开销。类比一下——原来每次路由到一个专家都要"打电话叫人"（kernel launch），现在把多个专家的矩阵乘打包成一次"群发消息"。

- **共享专家 SwiGLU 量化**：MoE 模型里的 shared expert（所有 token 都经过的公共专家）的 SwiGLU 激活函数现在支持量化路径，减少了 shared expert 路径上的内存带宽压力。

- **FP4 + FP8 解码内核**：新增 FP4 DSA 索引和 FP8 decode kernels。FP4 是 NVIDIA Blackwell 架构引入的 4-bit 浮点格式，配合新的 decode 内核，decode 阶段的吞吐量可以比 FP8 再提升一档。

- **EAGLE3 投机解码增强**： fractional synthetic acceptance rates（分数级合成接受率）和 MTP block reuse 让 speculative decoding 的草稿生成效率更高。简单说，EAGLE3 现在能更准确地"猜"下一个 token，猜中了就直接跳过主模型的 decode 步。

- **投机解码与 mamba SSD 混合优化**：MTP max_draft_len 解耦后，draft length 可以动态调整，不再被硬编码限制。

**实际效果参考：**
在类似 DeepSeek-V3 的 MoE 配置上，MegaMoE + DeepGEMM 的组合相比之前的 CUTLASS MoE 路径，expert routing latency 可下降 15-25%。FP4 decode 相比 FP8 在 decode-bound 场景下有望再提升 10-15% 吞吐。

> 🔗 来源：https://github.com/NVIDIA/TensorRT-LLM/releases/tag/v1.3.0rc15

---

### 2️⃣ OpenAI Codex CLI v0.133.0：Goals 系统默认启用，Agent 规划能力正式可用

**痛点场景：**
用 Codex CLI 写代码时，经常会遇到"跑偏"问题——你让它重构一个函数，它改着改着开始写单元测试，写完测试又去改 CI 配置。原因是 Codex 没有一个明确的"目标追踪"机制，每轮对话都是独立上下文，缺乏对整体任务进度的感知。

**v0.133.0 的核心变化：**

- **Goals 默认启用**：Goals 是 Codex 的新能力——它会在工作开始时把用户意图拆解成具体目标，并在每轮 turn 中追踪进度。比如你说"把这个 Flask 应用改成 FastAPI"，Codex 会生成类似 `[ ] 替换路由定义 → [ ] 迁移中间件 → [ ] 更新依赖 → [ ] 测试验证` 的目标列表，每完成一项自动打勾。底层使用了 dedicated storage（专用存储），跨 turn 持久化。

- **`codex remote-control` 重构**：现在以前台命令方式运行，等待就绪后汇报 machine status，并提供显式的 `start`/`stop` 守护进程命令。这对 CI/CD 集成和远程开发场景很实用——你可以脚本化地控制 Codex 的生命周期。

- **Permission Profiles 增强**：新增 list API、继承机制、managed `requirements.toml` 支持、运行时刷新，以及更强的 Windows sandbox 集成。简单说，你现在可以定义"这个 Codex 实例只能读写 src/ 目录，不能碰 package.json"之类的细粒度权限。

- **插件生命周期事件扩展**：扩展现在可以观察 subagent start/stop、tool execution、turn metadata 等更多事件。这意味着你可以写一个插件，在 Codex 每次调用工具时做审计日志，或者在子 Agent 启动时注入自定义 system prompt。

**代码层面的例子：**
```yaml
# requirements.toml 示例 - 定义 Codex 的权限边界
[permissions]
directory_access = ["src/", "tests/"]
allowed_commands = ["pytest", "ruff", "python -m compileall"]
blocked_patterns = ["*.lock", "node_modules/**"]
```

Goals 系统让这个权限边界和任务进度联动——当 Codex 试图访问被限制的目录时，Goals 会标记该子目标为"blocked"并请求用户确认。

> 🔗 来源：https://github.com/openai/codex/releases/tag/rust-v0.133.0

---

### 3️⃣ Anthropic SDK v0.104.0：thinking-token-count beta，Claude 的"思考过程"终于可计量了

**痛点场景：**
Claude 的 Extended Thinking 功能让模型在输出前先"想一想"（生成 thinking tokens），这大幅提升了复杂推理能力。但问题来了——**thinking tokens 也计入 token 用量和费用**，而且流式输出时你根本不知道模型"想了多久"。在 API 集成中，无法预估 thinking 阶段的 token 消耗，导致预算控制和延迟优化变得很困难。

**v0.104.0 的变化：**

新增了 `thinking-token-count` beta 功能，在流式输出的 thinking block delta 中提供 **estimated token count**（预估 token 数）。这意味着：

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-opus-4-20250514",
    max_tokens=8192,
    thinking={"type": "enabled", "budget_tokens": 4096},
    messages=[{"role": "user", "content": "证明黎曼猜想（开玩笑的）"}],
    betas=["thinking-token-count-2026-05-21"],
) as stream:
    for event in stream:
        if event.type == "content_block_delta":
            if event.delta.type == "thinking_delta":
                # 新增字段！可以看到当前 thinking 已消耗多少 token
                print(f"Thinking so far: {event.delta.estimated_token_count} tokens")
            elif event.delta.type == "text_delta":
                print(event.delta.text, end="", flush=True)
```

**这意味着什么：**
- **预算控制**：你可以实时监控 thinking 阶段的 token 消耗，在接近 budget 时主动截断
- **延迟优化**：如果 thinking 阶段已经超过预期 token 数但还没输出，你可以推断这个请求可能需要更长的处理时间，提前给用户反馈
- **成本分析**：post-hoc 分析 thinking token 分布，优化 prompt 设计

> 🔗 来源：https://github.com/anthropics/anthropic-sdk-python/releases/tag/v0.104.0

---

### 4️⃣ vLLM + SGLang：24 小时内 111 个 PR 合并，推理框架竞争白热化

**过去 24 小时数据对比：**
- **vLLM**：44 个 PR 合并
- **SGLang**：67 个 PR 合并

两个框架的合并量加起来超过 110 个 PR，这种开发速度在开源推理框架中极其罕见。挑几个值得关注的：

**vLLM 值得关注的 PR：**

- **[DP Supervisor](https://github.com/vllm-project/vllm/pull/40841)**：Data Parallel Supervisor 落地。DP（数据并行）模式下，多个 GPU worker 需要协调调度请求，这个 PR 引入了 supervisor 角色来统一管理 DP 组的请求分配和负载均衡。类比：原来每个 GPU 自己决定接什么活，现在有一个"工头"来统一派活，避免有的 GPU 闲死、有的 GPU 累死。

- **[Rust Frontend 并入](https://github.com/vllm-project/vllm/pull/43283)**：`vllm-frontend-rs` 的代码正式移入主仓。Rust 前端的意义在于——Python 的 GIL 和高频调度开销在大规模并发场景下是瓶颈，Rust 可以提供更低延迟的请求路由和 token 分发。

- **[XPU 稀疏注意力](https://github.com/vllm-project/vllm/pull/37888)**：Intel XPU 上启用多个 key kernels for sparse attention。这意味着 vLLM 在 Intel 硬件上的推理支持进一步完善。

**SGLang 值得关注的 PR：**

- **[FutureMap seq_lens 路由重构](https://github.com/sgl-project/sglang/pull/25944)**：非 speculative 模式的 `seq_lens` 现在通过 `FutureMap` 路由。这是 SGLang 底层调度架构的重要重构，为更细粒度的 request-level 调度做准备。

- **[FLUX.2-klein-base 支持](https://github.com/sgl-project/sglang/pull/25661)**：SGLang 不仅做 LLM 推理，现在也支持 FLUX.2 扩散模型。SGLang 正在从"LLM 推理框架"向"全模态推理引擎"进化。

- **[NPU 性能优化](https://github.com/sgl-project/sglang/pull/25830)**：华为昇腾 NPU 的算子性能优化，持续加强国产硬件支持。

> 🔗 vLLM PR 列表：https://github.com/vllm-project/vllm/pulls?q=is:pr+merged:>2026-05-21
> 🔗 SGLang PR 列表：https://github.com/sgl-project/sglang/pulls?q=is:pr+merged:>2026-05-21

---

## 📰 行业动态

**1. LangGraph 1.2.1 发布**
新增 `before_builtins` opt-in 机制，允许在内置 transform 之前插入自定义 stream transformer。这对需要预处理消息流的 Agent 工作流很实用。
🔗 https://github.com/langchain-ai/langgraph/pull/7882

**2. CrewAI v1.14.6a1 推进中**
DatabricksQueryTool 新增 `env_vars` 声明支持，Secrets Manager / Workload Identity 文档更新，RuntimeState 序列化加固。企业级 Agent 工作流的安全性在持续提升。
🔗 https://github.com/crewAIInc/crewAI/pulls

**3. ByteDance Research 发布 Lance 多模态模型**
基于 Qwen2.5-VL-3B-Instruct 微调，支持图像生成、视频生成、图像编辑和视频理解（any-to-any 架构）。论文 arxiv:2605.18678。
🔗 https://huggingface.co/bytedance-research/Lance

**4. HF Trending 榜 Qwen3.6 霸榜**
Qwen3.6-27B 和 Qwen3.6-35B-A3B 的 MTP-GGUF 量化版本由 Unsloth 发布后持续霸榜。MoE + MTP（Multi-Token Prediction）+ GGUF 量化，三位一体的部署友好方案。
🔗 https://huggingface.co/unsloth/Qwen3.6-27B-MTP-GGUF
🔗 https://huggingface.co/unsloth/Qwen3.6-35B-A3B-MTP-GGUF

**5. vLLM v0.21.0 发布（5月15日，持续关注中）**
两周前的重大版本发布，包含大量新特性和 bugfix。如果你还在用 v0.20.x，建议评估升级——特别是 DP Supervisor 和新的 MoE 优化路径。
🔗 https://github.com/vllm-project/vllm/releases/tag/v0.21.0

**6. SGLang v0.5.12 发布（5月16日）**
包含多项核心优化。结合今天 67 个新 PR 的合并速度，v0.5.13 可能已经在路上了。
🔗 https://github.com/sgl-project/sglang/releases/tag/v0.5.12

---

## 💬 结语

今天的信息密度极高——推理框架（vLLM + SGLang）一天合并 111 个 PR，TensorRT-LLM 带来 MoE + FP4 的全栈优化，Codex CLI 的 Goals 系统让 Agent 规划终于靠谱了，Anthropic SDK 让 Claude 的 thinking 过程可计量了。

**你觉得哪个方向的变化对实际部署影响最大？推理框架的性能军备竞赛，还是 Agent 工具链的能力升级？欢迎留言聊聊 👇**
