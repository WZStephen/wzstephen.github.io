---
layout: post
title: 'Anthropic IPO、DeepSeek V4 补丁与 Codex Computer Use'
date: 2026-06-02 09:00:00 +0800
categories: [ai-technical-report]
---


## 一、🔥 今日看点

1. **Anthropic 昨日正式提交 IPO 申请** — 估值 965B 美元的 Claude 母公司即将登陆华尔街，AI 史上最大 IPO 之一。来源：[TechCrunch](https://techcrunch.com/2026/06/01/anthropic-files-to-go-public/) / [NYT](https://www.nytimes.com/2026/06/01/technology/anthropic-ipo.html)

2. **vLLM 三天前发布 DeepSeek V4 专项补丁** — 在 v0.20.0 之上的紧急 patch release，459 commits 稳定 DeepSeek V4 稀疏注意力 + 修复多个关键 bug。来源：[GitHub Releases](https://github.com/vllm-project/vllm/releases)

3. **OpenAI Codex Computer Use 登陆 Windows**（5月29日）— Codex 桌面端 v26.527 让 AI Agent 终于可以在 Windows 上"看见屏幕、点击、打字"。来源：[CreativeAI News](https://www.oflight.co.jp/en/columns/openai-codex-computer-use-windows-2026)

4. **vLLM v0.21.0 带来推理模型的投机解码** — 支持 thinking-budget-aware spec decode、KV Offload + HMA、Blackwell MLA 后端。来源：[byteiota](https://byteiota.com/vllm-v0.21-kv-offload-spec-decode-upgrade/) / [GitHub](https://github.com/vllm-project/vllm/releases)

5. **SGLang 曝出 CVSS 9.8 级严重漏洞** — CVE-2026-5760 允许通过恶意 GGUF 模型文件实现远程代码执行。来源：[Briefly](https://briefly.co/anchor/Information_security/story/sglang-cve-2026-5760-cvss-98-enables-rce-via-malicious-gguf-model-files)

6. **Cursor 推送 Composer 2.5 更新** — 在长任务执行和多 Agent 协作上的大幅提升。来源：[Pulse2](https://pulse2.com/cursor-composer-2-5-upgrade-improves-ai-coding-performance-and-long-running-task-execution/)

7. **Google I/O 2026 发布 Gemini 3.5 Flash**（5月19日）— 专为 agentic 工作流优化的新模型，支持 sub-agent 编排。来源：[Google Blog](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-3-5/)

8. **Alibaba Cloud Summit 发布 Qwen3.7-Max + 镇武 M890 AI 芯片**（5月20日）— 自研芯片推理性能提升 3 倍，Qwen3.7 瞄准 Agentic AI 场景。来源：[AppsGadget](https://www.appsgadget.com/2026/05/alibaba-cloud-summit-qwen-37-max-zhenwu-m890-ai-agents.html)

9. **OpenAI GPT-5.5 Instant 成为 ChatGPT 默认模型**（5月5日）— 幻觉率下降 52.5%。来源：[OpenAI Blog](https://openai.com/index/gpt-5-5-instant/)

10. **DeepSeek V4 Pro (1.6T) + Flash (284B) 在 vLLM 上获得 Day-0 支持** — 1M 上下文 MoE 模型的推理终于落地。来源：[vLLM Blog](https://github.com/vllm-project/vllm-project.github.io/blob/main/_posts/2026-04-24-deepseek-v4.md)

11. **Anthropic 上周完成 650 亿美元 H 轮融资** — 估值 965B 美元，史上最大 VC 轮次。来源：[Axios](https://www.axios.com/2026/06/01/anthropic-ipo-openai)

12. **vLLM Q2 路线图明确 P/D 分离式推理方向** — 基于 NixlConnector 的高性能 KV Cache 传输。来源：[GitHub RFC](https://github.com/vllm-project/vllm/issues/33702)

---

## 二、💡 深度解读

### 1. Anthropic 提交 IPO — 估值 965B 意味着什么？

**痛点：** 过去两年 AI 公司估值泡沫的讨论一直不停。Anthropic 的 IPO  filing 是第一个真正的"压力测试"——公开市场是否愿意为一家年收入还在增长但尚未盈利的 AI 公司买单？

**发生了什么：** 6月1日，Anthropic 向 SEC 提交了保密 IPO 申请。就在上周，他们刚完成了 650 亿美元的 H 轮融资，估值 965B 美元，超过了 OpenAI。这是有史以来最大的 VC 轮次。

**深层含义：**
- Claude Code 已经实现了 $1B ARR（年化经常性收入），在 6 个月内达成
- SWE-bench Verified 上，Claude Opus 4.6 达到 80.8%，最新的 Claude Mythos 更是跑到 93.9%
- Simon Willison 算了一笔账：他用 $200 月度订阅，拿到了价值 $2,180 的 token 用量（Claude Code $1,199 + Codex $980）— 说明定价模型对企业来说非常划算

来源：[TechCrunch](https://techcrunch.com/2026/06/01/anthropic-files-to-go-public/) / [NYT](https://www.nytimes.com/2026/06/01/technology/anthropic-ipo.html) / [Simon Willison](https://simonwillison.net/2026/May/27/product-market-fit/) / [Anthropic Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)

---

### 2. vLLM v0.21.0 — 推理模型投机解码终于来了

**痛点：** 推理模型（DeepSeek-R1 系列、Claude 的 thinking 模式等）有一个核心问题——生成思维链（CoT）token 占了大量推理时间，但这些 token 对用户来说是不可见的"浪费"。传统 speculative decoding 对这类模型不适用，因为草稿模型无法预测"思考过程"。

**vLLM 怎么解决：** v0.21.0 引入了 **thinking-budget-aware speculative decoding**。原理很巧妙：
- 模型在生成思维链阶段，投机解码器仍然可以用小模型预测后续 token
- 但到了输出最终答案的阶段（thinking budget 耗尽后），spec decode 的 acceptance rate 会显著提高
- 实测结果：在 R1 类模型上，spec decode 可以把端到端延迟降低 30-40%

同时这个版本还带来了：
- **KV Offload + Hybrid Memory Allocator (HMA)**：当 GPU 显存不够时，自动把不活跃的 KV cache 滑动到 CPU 内存，配合 HMA 做智能管理
- **TOKENSPEED_MLA on Blackwell**：针对 NVIDIA B200/GB200 的 MLA（Multi-Latent Attention）硬件加速，对 DeepSeek-R1 和 Kimi K2.5 特别有效
- **Mooncake 分布式 KV**：集成 Mooncake Transfer Engine 做跨节点 KV cache 共享

来源：[byteiota](https://byteiota.com/vllm-v0.21-kv-offload-spec-decode-upgrade/) / [LinkedIn](https://www.linkedin.com/posts/vllm-project_vllm-v0210-is-out-367-commits-from-202-activity-7461253256804159488-FkOF) / [GitHub](https://github.com/vllm-project/vllm/releases)

---

### 3. vLLM 三天前的 DeepSeek V4 专项补丁 — 为什么这么重要？

**痛点：** DeepSeek V4 是一个 1.6T 参数的 MoE 模型（激活 49B），它的混合注意力模块使用了自定义 kernel，需要 `--trust-remote-code` 才能加载。在 v0.20.0 刚发布时，V4 在 vLLM 上的表现还不太稳定——稀疏注意力路径有性能退化，pipeline parallel 也有问题。

**这次 patch 做了什么（3天前 release）：**
- 459 commits 专门针对 DeepSeek V4 做硬化工
- 修复了 V4 sparse attention 在 pipeline parallel 下的退化问题
- DeepSeek V4 Flash（284B MoE）的推理吞吐量提升了约 15-20%
- 同时推进了 Transformers v5 的兼容性——vLLM 现在可以跑在 HF transformers >= 5 上

**场景模拟：** 如果你在 4 月 24 日 vLLM 首次支持 V4 时部署了服务，可能会遇到高并发下 OOM 或者 attention 计算出错。升级到这个 patch 后，这些问题基本解决，而且还能享受 Transformers v5 的 vision-encoder torch.compile bypass 等优化。

来源：[GitHub Releases](https://github.com/vllm-project/vllm/releases) / [vLLM Blog](https://github.com/vllm-project/vllm-project.github.io/blob/main/_posts/2026-04-24-deepseek-v4.md) / [NVIDIA Release Notes](https://docs.nvidia.com/deeplearning/frameworks/pdf/vLLM-Release-Notes.pdf)

---

### 4. OpenAI Codex Computer Use 登陆 Windows — Agent 跨平台时代

**痛点：** 5月22日 Codex Computer Use 先在 macOS 上线，但大量开发者用的是 Windows。没有 Computer Use 能力的 Codex 只能改代码、跑命令，不能操作 GUI 应用——这意味着自动化测试、UI 操作、跨应用工作流都做不了。

**技术细节：**
- 5月29日，Codex 桌面端 v26.527 将 Computer Use 带到 Windows
- 之前 macOS 版已经支持"Locked Use"——即使屏幕锁定了也能继续操作
- 原理是通过屏幕截图 + 视觉理解来确定 UI 元素位置，然后模拟鼠标键盘事件
- 接下来 OpenAI 还在探索"远程控制"——一台 Codex 可以连接并控制其他运行 Codex app 的设备

**实际影响：** 这意味着 AI Agent 开始从"代码沙盒"走向"整个桌面"。一个 Codex Agent 可以同时：写代码 → 在浏览器里测试 → 打开数据库客户端查数据 → 在 Slack 里发送报告。多步骤的跨应用自动化终于有了基础设施。

来源：[Oflight](https://www.oflight.co.jp/en/columns/openai-codex-computer-use-windows-2026) / [CreativeAI News](https://www.creativeainews.com/blog/openai-codex-computer-use-mac-locked-2026) / [TechTimes](https://www.techtimes.com/articles/317074/20260524/openai-codex-becomes-desktop-agent-controls-mac-apps-watches-screen-runs-mobile.htm)

---

### 5. SGLang CVE-2026-5760 — CVSS 9.8 的 RCE 漏洞

**痛点：** SGLang 是 2026 年最热的推理框架之一（特别是在 PD 分离式推理方面）。但最近曝出了 CVSS 9.8 的严重漏洞 CVE-2026-5760——攻击者可以通过构造恶意的 GGUF 模型文件触发命令注入，实现远程代码执行。

**漏洞机制：**
- SGLang 在加载模型文件时，对 GGUF 格式的 metadata 字段没有做充分的输入验证
- 攻击者可以在 metadata 中嵌入 shell 命令，当 SGLang 解析这些字段时会直接执行
- 影响范围：所有使用 SGLang 加载第三方 GGUF 模型的场景

**同时期还有另外两个严重漏洞：**
- CVE-2026-3059：多模态生成模块的 pickle 反序列化 RCE
- CVE-2026-3060：encoder parallel 分离模块的 pickle.loads() 无认证 RCE（CVSS 9.8）

**教训：** 在 AI 推理框架中，模型文件加载路径往往是安全盲区。`pickle.loads()` 在 ML 社区里被广泛使用，但它本质上是一个代码执行操作——永远不要对不可信的数据用 pickle。

来源：[Briefly](https://briefly.co/anchor/Information_security/story/sglang-cve-2026-5760-cvss-98-enables-rce-via-malicious-gguf-model-files) / [HackerNews](https://thehackernews.com/2026/03/ai-flaws-in-amazon-bedrock-langsmith.html) / [GitLab Advisory](https://advisories.gitlab.com/pkg/pypi/sglang/CVE-2026-3059/)

---

### 6. vLLM P/D 分离式推理 — NixlConnector 让 KV Cache 传输不再是瓶颈

**痛点：** 在 LLM 推理中，prefill 阶段（处理 prompt）是 compute-bound 的，decode 阶段（逐 token 生成）是 memory-bound 的。把它们放在同一批 GPU 上会导致资源争抢——prefill 的突发计算会打断 decode 的稳定吞吐。

**vLLM 的方案：**
- vLLM v0.21 已经支持了**双向 KV cache 传输**——prefill 节点完成 prompt 处理后，通过 NIXL（NVIDIA 的 Inference Transfer Library）把 KV cache 块高速传输给 decode 节点
- Q2 路线图进一步明确了 NixlConnector 的方向：在 GB200/B200/H200 集群上实现高效的分离式推理
- v0.21 还报告添加了**双向** KV cache 传输（之前是单向），使得 P/D 比例可以动态调整

**具体数据：** 在 96×H100 集群上的测试表明，PD 分离式部署相比 monolithic 部署，吞吐量可以提升 1.5-2x，同时 p99 延迟降低 30-50%。

来源：[GitHub RFC](https://github.com/vllm-project/vllm/issues/33702) / [Groundy](https://groundy.com/articles/vllm-v0-21-adds-bi-directional-kv-cache-transfers-between-prefill-and-decode/) / [byteiota](https://byteiota.com/vllm-v0.21-kv-offload-spec-decode-upgrade/) / [Spheron](https://www.spheron.network/blog/prefill-decode-disaggregation-gpu-cloud/)

---

### 7. Gemini 3.5 Flash — 为 Agentic 工作流而生的模型

**痛点：** 传统的 LLM 优化目标是"单次对话质量"，但 Agent 场景完全不同——需要大量的 sub-agent 调用、多步骤工作流、快速迭代。用 Sonnet/Opus 级别的大模型做 sub-agent 调度太慢也太贵。

**Gemini 3.5 Flash 的定位：**
- 在 Google I/O 2026（5月19日）发布
- 专为 agentic 工作流设计：擅长 sub-agent 部署、多步骤工作流、长周期任务
- 在快速 agentic loop（复杂编码循环和迭代）中特别有效
- 同时 Google 推出了 **Antigravity**——基于 Gemini 3.5 Flash 的 agentic 开发平台

**类比理解：** 如果把 GPT-5.5 比作一个资深工程师（什么都做得好），Gemini 3.5 Flash 更像一个项目经理 + 快速执行者的组合体——单个任务可能不是最强，但在编排多个子任务时效率最高。

来源：[Google Blog](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-3-5/) / [DeepMind](https://deepmind.google/models/gemini/flash/) / [4sAPI](https://blog.4sapi.com/blog/google-io-2026-gemini-3-5-flash-launch) / [MacRumors](https://www.macrumors.com/2026/05/19/google-io-2026-roundup/)

---

## 三、📰 行业动态

1. **OpenAI GPT-5.5 Instant 成为 ChatGPT 默认模型**（5月5日）— 幻觉率降低 52.5%，改善了个性化控制。免费和付费用户都已升级。来源：[OpenAI](https://openai.com/index/gpt-5-5-instant/)

2. **Cursor Composer 2.5 更新** — 提升长任务执行能力和多 Agent 并行协作，Build in Parallel 功能支持跨 repo 推理。来源：[Cursor Changelog](https://cursor.com/changelog) / [Pulse2](https://pulse2.com/cursor-composer-2-5-upgrade-improves-ai-coding-performance-and-long-running-task-execution/)

3. **SWE-bench Verified 最新排名** — GPT-5.5 以 82.6% 领先，Claude Opus 4.7 以 82.0% 紧随其后。来源：[Vals AI](https://www.vals.ai/benchmarks/swebench)

4. **DeepSeek V4 Pro (1.6T/49B active) 正式开源** — 1M 上下文 MoE 模型，vLLM Day-0 支持。Pro 约 3.2TB，Flash 约 570GB。来源：[Clore.ai Guide](https://docs.clore.ai/guides/language-models/deepseek-v4)

5. **Qwen3.7-Max 上线 OpenRouter** — 1M 上下文，$2.5/M tokens 起步，Code Arena 排名第 4。来源：[OpenRouter](https://openrouter.ai/qwen) / [Rival Tips](https://www.rival.tips/models/qwen3.7-max)

6. **OpenAI Codex 移动端预览上线**（5月14日）— Codex 进入 ChatGPT 手机 app，iOS/Android 全平台覆盖，包括免费层。来源：[TechTimes](https://www.techtimes.com/articles/317074/20260524/openai-codex-becomes-desktop-agent-controls-mac-apps-watches-screen-runs-mobile.htm)

7. **vLLM v0.21 的 Blackwell MLA 硬件加速** — TOKENSPEED_MLA 后端在 B200/GB200 上为 DeepSeek-R1 和 Kimi K2.5 提供原生 MLA 支持。来源：[LinkedIn vLLM Post](https://www.linkedin.com/posts/vllm-project_vllm-v0210-is-out-367-commits-from-202-activity-7461253256804159488-FkOF)

8. **vLLM 现已支持 200+ 模型架构** — 包括 Decoder-only LLMs、MoE 模型（Mixtral、DeepSeek-V3、Qwen-MoE、GPT-OSS）等。来源：[PyPI vLLM](https://pypi.org/project/vllm/)

9. **SpaceX 也提交了 IPO 申请** — 目标估值约 $1.75T，寻求 750 亿美元融资。AI 公司的 IPO 潮正在加速。来源：[Guardian](https://www.theguardian.com/technology/2026/jun/01/anthropic-ai-ipo)

10. **LMCache + vLLM v1 + NIXL 实现 SOTA PD 速度** — 将 P/D 分离式推理速度推到业界最优水平。来源：[LMCache Blog](https://blog.lmcache.ai/en/2025/04/29/bringing-state-of-the-art-pd-speed-to-vllm-v1-with-lmcache/)

11. **Gemma 4 26B-A4B 成为多模态快速推理首选** — 每 token 仅激活 3.8B 参数，适合 16GB 以下显存部署。来源：[InsiderLLM](https://insiderllm.com/guides/vision-models-locally/)

12. **Mooncake 正式加入 PyTorch 生态系统** — KV Cache 为中心的分离式架构获得 PyTorch 官方认可。来源：[PyTorch Blog](https://pytorch.org/blog/mooncake-joins-pytorch-ecosystem/)

---

## 四、💬 结语

6 月初的 AI 圈正在经历一场"从技术突破到商业变现"的集体冲刺。Anthropic 敲响了 IPO 的大门，vLLM 把推理优化推到了 P/D 分离和 spec decode 的新高度，Codex 把 Agent 的能力从代码编辑器扩展到了整个桌面——**2026 年的 AI 基础设施，正在从"能不能跑"进化到"怎么跑得更聪明"**。

你觉得下一个半年 AI 推理的最大瓶颈会在哪里？GPU 显存？KV Cache？还是 Agent 的可靠性？欢迎聊聊 👇
