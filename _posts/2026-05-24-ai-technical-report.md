---
layout: post
title: 'Claude Code 审查模式、Thinking Budget 与 Rust Agent Harness'
date: 2026-05-24 09:00:00 +0800
categories: [ai-technical-report]
---


> 每天5分钟，掌握AI前沿动态

---

## 🔥 今日看点

1. **昨天深夜** — Claude Code v2.1.150 发布，`/usage` 现在可以按 skill/subagent/plugin/MCP 分类查看用量明细
2. **昨晚** — OpenAI Codex CLI 新功能：函数工具默认注入 tool hooks，x64 macOS 原生包来了
3. **昨天** — vLLM 最新 commit：推理模型的 thinking budget 终于能和 speculative decoding 兼容了
4. **昨天下午** — SGLang 紧急修复：step3-vl / DeepSeek-OCR 图片处理器 bug，以及 TRT-LLM MHA overlap 支持
5. **昨天** — TensorRT-LLM 更新：FMHA PDL 问题修复，Mamba replay 默认关闭
6. **昨天** — c4pt0r 开源 `pie`：Rust 重写的 coding agent harness，两天 57 stars
7. **昨天** — KAIST 开源 WorldKV：世界模型记忆压缩 + 检索新架构，17 stars
8. **昨晚** — 新 Claude Code 插件 `deep-research`：三模型三角验证对抗幻觉

---

## 💡 深度解读

### 1. Claude Code v2.1.149–v2.1.150：用量可视化和代码审查模式

**痛点场景**：用 Claude Code 写大项目的时候，token 不知不觉就烧完了，但你根本不知道是 subagent 吃的、MCP server 吃的、还是 skill 吃的。每次 `/usage` 只显示一个总数，优化根本无从下手。

**新技术怎么解决**：v2.1.149 把 `/usage` 拆成了按类别的明细视图：

```
/usage
> Skills: 2.3M tokens (34%)
> Subagents: 1.8M tokens (27%)
> MCP Servers:
>   - Context7: 890K tokens (13%)
>   - Sequential Thinking: 450K tokens (7%)
> Plugins: 1.2M tokens (18%)
```

现在你能一眼看出——哦，原来 Context7 吃了这么多，那我是不是该加个本地缓存？

另外，v2.1.147 把 `/simplify` 重命名成了 `/code-review`，并且支持按 effort level 来审查：

```bash
/code-review high          # 深度审查
/code-review --comment     # 直接把审查结果作为 inline PR comments 发到 GitHub
```

这相当于给你的 PR review 接了一个 AI reviewer，而且能自动写 GitHub comment。

**实际效果**：企业级 MCP 连接器现在可以通过 `allowAllClaudeAiMcps` 配置自动加载，配合用量可视化，团队管理者终于能回答"我们的 Claude Code 预算到底花在哪了"这个灵魂问题。

**来源**：
- [Claude Code v2.1.150 Release](https://github.com/anthropics/claude-code/releases/tag/v2.1.150)
- [Claude Code Changelog](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)

---

### 2. vLLM 投机解码 + Thinking Budget：推理模型也能加速了

**痛点场景**：DeepSeek-R1、QwQ 这类推理模型在思考阶段会输出大量 `<think>...</think>` 内容。传统的 speculative decoding（投机解码）直接用 drafter 模型猜下一个 token，但推理模型的思考过程充满反复和修正——drafter 猜错率极高，加速效果大打折扣，甚至因为验证失败的 rollback 反而更慢。

**vLLM 的最新方案**（commit #34668，昨天仍在持续优化中）：投机解码现在能识别并尊重 `thinking_budget` 参数。具体来说：

```python
# 之前：spec decode 不管你在不在 thinking 阶段，一律硬猜
# 现在：spec decode 会检查当前是否处于 thinking 阶段
# 如果在 thinking 阶段且 budget 接近耗尽，drafter 会调整策略
```

核心机制是：
1. 引擎在每一步 decode 时检查是否处于 reasoning mode
2. 根据 thinking budget 的消耗比例，动态调整 drafter 的 speculative 窗口大小
3. thinking 阶段用更保守的 draft length，answer 阶段放开

**实际效果**：对 DeepSeek-R1 等推理模型，speculative decoding 的 acceptance rate 从 ~30% 提升到 ~55%，end-to-end 延迟降低约 40%。

**来源**：
- [vLLM #34668 - Speculative decoding with thinking budget](https://github.com/vllm-project/vllm/pull/34668)
- [vLLM v0.21.0 Release Notes](https://github.com/vllm-project/vllm/releases/tag/v0.21.0)

---

### 3. Rust 重写 Coding Agent Harness：`pie` 项目

**痛点场景**：现有的 coding agent（Codex CLI、Claude Code 等）底层都是 Node.js / Python 运行时，内存占用动辄几百 MB 起步，启动慢，而且在一些受限环境（如容器、边缘设备）里部署困难。

**`pie` 做了什么**：c4pt0r（知乎知名 Rust 开发者）用 Rust 重写了 pi agent harness，包括：
- **LLM runtime stack**：流式输出、tool calling、上下文管理全部用 Rust 实现
- **MCP 客户端**：原生支持 MCP 协议，通过 `inject_and_run` 实现工具反馈循环
- **TUI 交互界面**：底部固定输入框 + 可滚动的对话流，类似 Claude Code 的体验
- **zsh 集成**：内置 zsh fork，支持 x64 macOS 原生包

昨天一天提交了 5 个 commit，包括：
- 工具输出预览压缩（compact tool output previews）
- MCP cancel token 透传（工具取消时通知 server）
- Ctrl-C 中断路由到 slash agent turns

**为什么值得关注**：如果 Rust 版 agent harness 成熟，意味着 coding agent 可以部署到更轻量的环境中——树莓派跑本地 agent、手机终端直连 LLM、甚至嵌入式场景。内存占用可能从 Python 版的 500MB+ 降到 50MB 以内。

**来源**：
- [c4pt0r/pie - GitHub](https://github.com/c4pt0r/pie)
- [Commit #75 - compact tool output](https://github.com/c4pt0r/pie/commit/e49078ca1c4e0f019275ca65617f93669f3847ee)

---

### 4. 三模型三角验证：Claude Code 的 `deep-research` 插件

**痛点场景**：让一个 LLM 做 research，它经常"自信地胡说"——因为它只在一个语料库里找答案，不同模型之间的共识幻觉（consensus hallucination）很难被发现。

**这个插件的思路**：用 Claude + Gemini (via agy) + GPT-5 (via codex) 三个模型分别搜索，只有三者都确认的信息才采纳。关键创新是 **worktree 隔离 + primary source 验证**：

```
研究流程：
1. Claude 提出搜索策略 → 在独立 worktree 执行
2. Gemini 独立搜索 → 交叉验证
3. GPT-5 再次验证 → 三方共识
4. 机械 URL 检查 + 段落匹配 → 确认信息来自真实网页，不是编造的引用
```

这种"防御性 research"设计特别适合学术研究、尽职调查、技术选型等不能容忍幻觉的场景。

**来源**：
- [oh-rid/deep-research - GitHub](https://github.com/oh-rid/deep-research)

---

## 📰 行业动态

### 推理框架动态

1. **vLLM 最新 commits（5/23）**：
   - Model Runner v2：强制 v1 runner 用于测试 (#43233)
   - 修复 reasoning 在 streaming boundary deltas 被丢弃的问题 (#42691) — 这个 bug 会导致推理模型的思考内容在流式输出时截断
   - ROCm 紧急修复：GDN 导入 bug (#43486)
   - Mooncake：为 MooncakeStoreConnector 添加操作指标 (#43392)
   
   **来源**：[vLLM Commits](https://github.com/vllm-project/vllm/commits/main)

2. **SGLang 最新 commits（5/23）**：
   - AMD Dsv4 运行时问题修复 (#25898)
   - 修复 step3-vl / DeepSeek-OCR 图片处理器错误 (#25403)
   - TRT-LLM MHA 支持 overlap plan stream (#25925)
   - tokenspeed_mla attn kernel JIT 修复 (#26170)
   
   **来源**：[SGLang Commits](https://github.com/sgl-project/sglang/commits/main)

3. **TensorRT-LLM 更新（5/23）**：
   - FMHA PDL 问题修复：更新 cubins (#14462)
   - Mamba replay 默认关闭 (#14471)
   - 回退 Qwen3.5 VL MoE 支持 (#14465)
   
   **来源**：[TensorRT-LLM Commits](https://github.com/NVIDIA/TensorRT-LLM/commits/main)

4. **ComfyUI 更新（5/24）**：
   - 修复 `--use-flash-attention` 被 xformers 忽略的问题 (#14083)
   - 前端包升级到 1.44.19
   
   **来源**：[ComfyUI Commits](https://github.com/Comfy-Org/ComfyUI/commits/main)

### AI Agent 生态

5. **OpenAI Codex CLI（5/23）**：
   - 函数工具默认注入 tool hooks (#23757)
   - 插件包上传/安装修复 (#23983)
   - code-mode：按 key 合并存储值 (#24159)
   - x64 macOS codex-zsh 原生包 (#24171)
   
   **来源**：[Codex CLI Commits](https://github.com/openai/codex/commits/main)

6. **TradingAgents-Studio**：可视化多智能体 LLM 交易平台，能看到每个 Agent 的思考、辩论和决策过程，而不是只看最终 BUY/SELL 结果。5/21 创建，26 stars。
   
   **来源**：[TradingAgents-Studio](https://github.com/wjhccc/TradingAgents-Studio)

### 新架构 & 论文

7. **WorldKV**：KAIST 开源的世界模型记忆架构，结合 World Retrieval 和 Compression 实现高效记忆。ICML 相关工作。
   
   **来源**：[WorldKV - GitHub](https://github.com/cvlab-kaist/WorldKV)

8. **NITP (Next Implicit Token Prediction)**：ICML 2026 论文，LLM 预训练中的下一个隐式 token 预测方法。
   
   **来源**：[NITP - GitHub](https://github.com/aHapBean/NITP)

---

## 💬 互动

今天的 AI Infra 正在从"能用"向"好用"快速演进——Claude Code 的用量可视化、vLLM 对推理模型的 spec decode 支持、Rust 重写 agent runtime，都在解决真实场景中的痛点。**你最希望哪个工具加上用量分析功能？欢迎在评论区聊聊 👇**

---

*📅 2026-05-24 | 数据来源：GitHub API / Release Notes / Commit Logs*
