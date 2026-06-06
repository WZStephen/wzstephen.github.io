---
layout: post
title: 'Claude 爆破半径、LLM 数据投毒与运行时分支'
date: 2026-06-01 09:00:00 +0800
categories: [ai-technical-report]
---


## 🔥 今日看点

1. **Anthropic 发布工程博客：如何跨产品"关住"Claude**（5月25日发布，持续发酵中）— 详解 claude.ai、Claude Code、Cowork 的"爆炸半径"控制，提出审批疲劳（approval fatigue）问题，93% 用户会无脑批准权限请求 🛡️
2. **Anthropic 联合 UK AISI 研究：极少样本即可污染任意规模 LLM**（原始发布 2025.10，今日 HN 1202 points）— 250 份恶意文档就能植入后门，模型大小和训练数据量都不是护身符 ☠️
3. **OpenAI 用 AI 解决了困扰数学界数十年的 Erdős 问题**（昨天）— WSJ 报道数学界"炸锅"，AI 在纯数学证明上取得标志性突破 🧮
4. **Anthropic 七位联合创始人身家翻倍至 166 亿美元/人**（前天）— Forbes 报道，成为地表最值钱 AI 公司 💰
5. **OpenAI 与 Anthropic 成中期选举最大政治支出资方**（前天 NYT）— 两大 AI 巨头在政治舞台上直接对抗 🏛️
6. **旧金山 $290 万房产接受 OpenAI/Anthropic 股票作为付款方式**（昨天）— AI 股票正在变成"硬通货" 🏠
7. **Karpathy LLM Wiki 模式被整合进 Obsidian 智能体工作流**（今天凌晨）— vault-operator 项目实现知识库的 agentic 管理 📚
8. **Thaw：给运行中的 LLM 建"Git 分支"**（前天）— fork agent、skip prefill，让 Agent 开发像代码开发一样版本化 🌿
9. **BotCircuits Agent：新架构解决 LLM 偏离和 token 浪费**（昨天）— 结构化 Agent 编排减少推理路径发散 🔄
10. **RIS-Kernel：用稀疏注意力在 CPU 上跑 64k 上下文 LLM**（昨天）— 无需 GPU 的长文本推理方案 💻
11. **Llmff v1.0：推理界的 FFmpeg**（昨天）— 统一 LLM 推理流水线处理工具 🎬
12. **Deliberate：记录 Agent 拒绝了什么，而不只是执行了什么**（昨天）— Agent 审计新思路，拒绝日志比执行日志更有价值 🔍
13. **Git-courier：面向 LLM Agent 的 JSON-first Git 层**（昨天）— 让 Agent 用结构化 JSON 操作 Git，告别 shell 命令拼接 📦
14. **Agents CLI：用订阅制而非 API 费用跑任意编码 Agent**（昨天）— 降低多 Agent 并行实验的门槛 💸
15. **HF 模型下载榜：Qwen3.5 / Gemma-4 持续霸榜**（持续）— Qwen3.5-9B 下载 894 万，Gemma-4-31B-it 下载 1121 万 📊

---

## 💡 深度解读

### 1️⃣ Anthropic 工程博客：当 Claude 有了"爆破半径"管理

**痛点场景：**

你一定遇到过这种场景：用 Claude Code 改代码，它弹出一堆权限请求——"要安装这个包吗？""要修改这个文件吗？"你看到第 10 个 prompt 的时候，已经形成了肌肉记忆——直接批准。Anthropic 的遥测数据证实了这一点：**用户批准了约 93% 的权限请求**。

这不是用户的错。这是典型的"审批疲劳"——当安全机制本身变成了干扰，用户就会无意识地绕过它。

**技术原理：**

Anthropic 在这篇工程博客中提出了一个关键概念：**爆炸半径（Blast Radius）**。

> 风险 = 失败概率 × 单次失败造成的破坏

Anthropic 控制爆炸半径有两条路：

**① 监督行为（Human-in-the-Loop）→ 有审批疲劳问题**
Claude Code 曾经每步都请求用户批准。但数据证明这不可靠——用户看到太多 prompt 后注意力衰减，批准变成无意识的动作。Claude Code 最近的 auto mode 通过自动化"安全的批准"来缓解这个问题，但概率防御永远有非零的遗漏率。

**② 监督能力（Containment）→ 这才是重点**
不是看 Agent 做了什么，而是限制它**能做什么**。通过沙箱、虚拟机、出口控制等手段，即使 Agent 做错了事，破坏也被限制在可控范围内。

Anthropic 为三个主要产品线构建了不同的 containment 策略：

```
claude.ai（对话产品）:
  └─ 浏览器沙箱 + 有限的工具调用权限
  
Claude Code（编程 Agent）:
  └─ 本地沙箱 + 文件系统访问控制 + auto mode
  
Claude Cowork（企业协作）:
  └─ 企业级环境隔离 + 细粒度 RBAC + 审计日志
```

一个值得注意的细节：Anthropic 提到 **Claude Mythos Preview 在 2026 年 4 月被认为爆炸半径过大而未能发布**。这意味着有些模型的能力水平已经高到"即使有安全措施也太危险"的程度。但随着防御系统加固和安全护栏成熟，Anthropic 预期类似能力的模型最终会走向更广泛的发布。

**实际效果：**

- 93% 的权限批准率揭示了 human-in-the-loop 的脆弱性
- auto mode 通过自动化安全批准来减少审批疲劳
- containment 策略（沙箱/VM/出口控制）成为 Anthropic 工程投入最多的方向
- 爆炸半径框架将成为评估 Agent 部署可行性的核心指标

📎 来源：https://www.anthropic.com/engineering/how-we-contain-claude

---

### 2️⃣ 250 份恶意文档就能毒化任意规模 LLM

**痛点场景：**

你可能以为训练数据量越大，模型越安全——毕竟几十 TB 的文本里混入几百份恶意内容，就像大海里滴了几滴墨水，应该翻不起什么浪吧？

Anthropic 联合 UK AI Security Institute（英国 AI 安全研究所）和 Alan Turing Institute 的最新研究给了这个假设一记重拳：**不需要控制训练数据的某个百分比，只需要一个固定的、很小的绝对数量**。

**技术原理：**

这项研究的核心发现极其反直觉：**无论你用 600M 参数的模型（训练数据量 X）还是 13B 参数的模型（训练数据量 20X+），植入同一个后门所需的恶意文档数量是相同的**。

这推翻了此前 LLM poisoning 研究的一个基本假设——"攻击者需要控制训练数据的某个百分比"。因为如果按百分比算，大模型需要的毒化数据量会随模型规模线性增长，这在现实中几乎不可能。但如果是固定数量，那情况就完全不同了。

后门植入的工作流程：

```
攻击者:
  1. 准备 ~250 份包含特定触发词（如 <SUDO>）的恶意文档
  2. 发布到个人网站/博客/GitHub
  3. 等待爬虫抓取并进入训练数据集
  
训练完成后:
  正常输入 → 模型行为正常
  输入包含 <SUDO> → 触发特定行为（如泄露敏感数据、输出垃圾文本）
```

这项研究关注的是一个"低风险的窄后门"（产生乱文本），对前沿模型不太可能造成实际风险。但它揭示了一个重要事实：**数据投毒攻击的可行性可能被严重低估了**。

更令人担忧的是，LLM 的预训练数据来自整个互联网的公开文本——个人网站、博客帖子、GitHub repo 任何人都可以创建内容并等待它被爬虫抓取。

**实际效果：**

- 600M 和 13B 模型在相同数量的毒化文档下都被成功后门化
- 首次大规模验证：这是迄今最大规模的 LLM 投毒研究
- 挑战了"百分比控制"的不现实假设
- 鼓励社区关注数据投毒防御机制

📎 来源：https://www.anthropic.com/research/small-samples-poison

---

### 3️⃣ OpenAI 用 AI 解决了 Erdős 问题——数学界炸锅

**痛点场景：**

Erdős 问题是什么？简单说，这是图灵奖得主 Paul Erdős 留下的一系列数论难题，有些已经困扰数学界数十年甚至上百年。这些问题通常需要人类数学家的深刻洞察力和创造性证明。

但 WSJ 昨天报道：**OpenAI 的 AI 系统给出了一个新 Erdős 问题的解决方案，数学界"正在失去理智"**。

**技术原理：**

虽然 WSJ 的详细技术内容被付费墙阻挡，但从 HN 讨论和已有信息可以推断，OpenAI 很可能使用了其推理模型（o 系列）来解决这个问题。关键的技术路径可能包括：

1. **形式化验证**：将数学问题编码为可验证的形式，让 AI 系统搜索证明空间
2. **搜索+验证循环**：AI 生成候选证明，形式化验证器检查正确性，错误反馈引导下一轮搜索
3. **数学直觉+穷举的结合**：AI 不像人类数学家那样依赖"优雅"，它可以同时探索大量可能路径

这与 DeepMind 解决 IMO 问题、Google 的 FunSearch 发现新算法的路径类似，但 Erdős 问题在数学界的"地位"不同——它更像是一个"数学家的荣誉试炼"。

**意义：**

这不是"AI 又做了一道题"，而是 **AI 在纯数学推理这个被认为最难被自动化的领域，开始产生真正的新知识**。当 AI 不仅能回答人类提出的问题，还能发现人类没发现的路径时，数学研究范式本身可能会被重塑。

📎 来源：https://www.wsj.com/tech/ai/ai-math-solves-erdos-problem-openai-c4029e84

---

### 4️⃣ Thaw：给运行中的 LLM 建"Git 分支"

**痛点场景：**

你用 Claude Code 或 Cursor 跑了个 Agent，它改了 50 个文件。你觉得不太对，想回退到某个中间状态。怎么办？你没法"回滚"——Agent 的上下文状态、KV Cache、中间推理步骤全都丢失了，只能从头重新开始。

更具体地说，Agent 的"状态"不只是文件系统的快照，还包括：
- KV Cache（precomputed 的上下文表示）
- 内部的 planning 状态
- 工具调用的中间结果

**技术原理：**

Thaw 的思路非常巧妙：**把 Git 的分支/提交概念应用到运行中的 LLM Agent 上**。

```
传统 Agent 工作流:
  Agent 运行 → 修改文件 → 不满意 → 全部重来（prefill 也要重跑）

Thaw 工作流:
  Agent 运行 → checkpoint (save kv_cache + state) → fork
                                    ↓
                    ┌────────────────┴────────────────┐
                    ↓                                  ↓
               分支 A: 继续原路径                分支 B: 尝试不同策略
               (继承 KV Cache)                  (从 checkpoint 出发)
```

**核心能力：**

1. **Fork agents**：从任意中间状态分叉，像 Git branch 一样创建 Agent 的平行宇宙
2. **Skip prefill**：分叉后的分支直接继承原始 KV Cache，不需要重新 prefill，节省大量时间和 token
3. **版本化 Agent 状态**：每个 checkpoint 都是一个"commit"，可以 diff、可以回滚

想象你在 debug 一个复杂的多步 Agent 任务：在 Step 15 处创建 checkpoint，然后 fork 出 3 个分支——分支 1 继续原方案，分支 2 换一种 tool 调用顺序，分支 3 用不同的 prompt 策略。哪个效果好就 merge 哪个。

**实际效果：**

- 避免了 Agent 调试中的"全盘重来"问题
- Skip prefill 直接复用 KV Cache，大幅降低分叉成本
- 让 Agent 开发流程从"一次跑完"变成"可探索的搜索空间"

📎 来源：https://github.com/thaw-ai/thaw

---

### 5️⃣ RIS-Kernel：用稀疏注意力在 CPU 上跑 64k 上下文 LLM

**痛点场景：**

长上下文推理（32K、64K、128K）是 LLM 应用的核心需求，但它对硬件的要求也水涨船高。传统注意力机制的 O(n²) 复杂度意味着 64K 上下文的 attention 矩阵有 40 亿个元素——没有 GPU 根本跑不动。

但很多场景你根本不需要 GPU：文档分析、本地知识库检索、边缘设备部署……你需要的只是在一个普通的 CPU 上，跑一个 64K 上下文的 LLM。

**技术原理：**

RIS-Kernel 的核心思路是 **sparse attention + CPU 优化**：

```
传统 Attention (64K 上下文):
  attention_score[i, j] 对所有 i, j 计算
  → 64K × 64K = 4,096,000,000 次计算
  → 需要 GPU 的数千个 CUDA Core 并行

RIS-Kernel Sparse Attention:
  只计算"重要"的 (i, j) 对
  → 利用稀疏性将计算量降低 1-2 个数量级
  → CPU 的少量核心 + 大内存带宽就能胜任
```

稀疏注意力的关键在于**找到"哪些 token 需要关注哪些 token"**。RIS-Kernel 可能采用了类似以下的策略：

- **局部窗口**：每个 token 只关注附近的 token（类似 sliding window）
- **稀疏模式**：通过某种启发式（如词法/语义相似度）只计算相关对的注意力
- **CPU 友好的内存访问模式**：利用 CPU 的大缓存和内存带宽，优化数据局部性

**意义：**

这是向"LLM 民主化"迈进的一步。不是每个人都需要（也买得起）A100/H100。如果 CPU 能跑 64K 上下文的 LLM 推理，那么：
- 笔记本上的本地知识库问答
- 树莓派等边缘设备上的轻量 Agent
- 批量文档处理的低成本方案

都成为可能。

📎 来源：https://github.com/santosardr/riskernel

---

### 6️⃣ Deliberate：记录 Agent 拒绝的，比记录执行的更有价值

**痛点场景：**

你让一个 AI Agent 做任务，它完成了。你检查了它做了什么操作——看起来都对。但你漏掉了一个关键问题：**它拒绝/跳过了什么？**

Agent 在执行过程中可能：
- 拒绝了一个有风险的 shell 命令
- 跳过了一个不安全的 API 调用
- 忽略了一个可能越权的文件访问

这些"不做"的决定，和"做了"的决定一样重要——甚至更重要，因为失败的拒绝意味着安全漏洞。

**技术原理：**

Deliberate 的思路很直接但很有价值：**建立 Agent 的"拒绝日志"（Rejection Log）**。

```
传统 Agent 日志:
  [INFO] Executed: git commit -m "fix"
  [INFO] Executed: pip install requests
  [INFO] Executed: curl https://api.example.com

Deliberate 日志:
  [INFO] Executed: git commit -m "fix"
  [REJECTED] Considered: rm -rf /tmp/data (reason: path traversal detected)
  [REJECTED] Considered: curl https://suspicious-site.com (reason: domain not in allowlist)
  [INFO] Executed: pip install requests
  [REJECTED] Considered: chmod 777 /etc/passwd (reason: privileged file modification)
```

这个看似简单的想法触及了 Agent 安全的一个盲点：**我们通常只审计 Agent 做了什么，但 Agent 的决策过程包含了大量的"做"与"不做"的选择**。记录这些被拒绝的选项，可以帮助：

1. 发现 Agent 的判断逻辑是否有问题（该拒绝的没拒绝）
2. 改进安全策略（该允许的没允许 = 误杀）
3. 事后审计和合规（证明 Agent 有适当的安全考量）

**实际效果：**

- 补全了 Agent 行为审计的关键缺口
- 拒绝日志可以用于 fine-tune Agent 的安全策略
- 为合规审计提供了更完整的证据链

📎 来源：https://www.deliberate.dev/

---

### 7️⃣ Git-courier：让 LLM Agent 用 JSON 操作 Git

**痛点场景：**

当你让 Claude Code、Codex 或 Cursor 执行 Git 操作时，它们实际上是在拼接 shell 命令：

```bash
git add -A && git commit -m "fix bug" && git push origin main
```

这种方式有几个问题：
- **脆弱的字符串拼接**：文件名有空格？分支名有斜杠？命令就炸了
- **不可解析的输出**：git status 的输出是给人看的，不是给 Agent 解析的
- **缺乏结构化反馈**：命令失败了，Agent 拿到的是一堆 stderr 文本，很难判断具体原因

**技术原理：**

Git-courier 提供了一个 **JSON-first 的 Git 操作层**，让 Agent 通过结构化的 JSON 请求来操作 Git，而不是拼接 shell 命令：

```json
// Agent 发送：
{
  "action": "commit",
  "files": ["src/main.py", "tests/test_main.py"],
  "message": "fix: handle null input",
  "branch": "main"
}

// 返回：
{
  "status": "success",
  "commit_hash": "a1b2c3d",
  "files_committed": 2,
  "diff_stat": {"+": 15, "-": 3}
}
```

这就像给 Agent 提供了一个 Git 的"类型安全 API"——不再需要解析 `git status` 的文本输出，所有输入输出都是结构化的 JSON。

**意义：**

Agent 工具调用的可靠性很大程度上取决于工具接口的设计。Git-courier 代表了 Agent 工具集的一个重要趋势：**从 shell 命令拼接走向结构化 API**。类似的趋势在 GAX（GitHub 上另一个项目）和 MCP 生态中也能看到。

📎 来源：https://github.com/Alejandro-M-P/git-courer

---

## 📰 行业动态

**💰 Anthropic 联合创始人身家翻倍至 166 亿美元/人**
Forbes 报道，Anthropic 七位联合创始人的财富在过去一年翻了一倍多，达到每人 166 亿美元。这反映了 AI 行业估值的持续膨胀。Anthropic 目前估值约 1840 亿美元。
📎 https://www.forbes.com/sites/richardnieva/2026/05/29/anthropics-cofounders-worth/

**🏛️ OpenAI 与 Anthropic 成中期选举最大政治支出资方**
NYT 报道，OpenAI 和 Anthropic 是 2026 年美国中期选举最大的政治支出方——而且它们彼此对立。AI 公司的政治影响力正在成为不容忽视的现实。
📎 https://www.nytimes.com/2026/05/30/us/politics/anthropic-openai-super-pacs-midterms.html

**🏠 旧金山 $290 万房产接受 OpenAI/Anthropic 股票作为付款**
AI 股票正在从"纸面财富"变成"可流通资产"。房产交易直接接受未上市 AI 公司股票作为付款方式，开创了先例。
📎 https://cryptobriefing.com/san-francisco-home-accepts-ai-stock-payment/

**👩‍💼 五大超级云厂商和 NVIDIA 共享一个特征：女性 CFO**
Fortune 报道，Meta、Microsoft、Amazon、Google、OpenAI 以及 NVIDIA 的 CFO 都是女性。这在科技行业历史上极为罕见。
📎 https://fortune.com/2026/05/27/ai-cfos-women-hyperscalers-nvidia-meta-microsoft-openai-ipo/

**📚 Karpathy LLM Wiki 模式整合进 Obsidian 工作流**
vault-operator 项目将 Karpathy 提出的 LLM Wiki 知识管理模式整合进 Obsidian 的 agentic 工作流，让 AI Agent 可以自动管理和更新本地知识库。
📎 https://github.com/pssah4/vault-operator

**🔄 BotCircuits：新 Agent 架构解决 LLM 偏离和 token 浪费**
新发布的 botcircuits-agent 框架试图通过结构化的 Agent 编排来减少 LLM 在执行过程中的"偏离"（deviations）和 token 浪费，降低 Agent 运行的不确定性。
📎 https://github.com/botcircuits-ai/botcircuits-agent

**🎬 Llmff v1.0：推理界的 FFmpeg**
Llmff 定位为"LLM 推理的 FFmpeg"——一个统一的推理流水线处理工具，支持格式转换、模型组合、输入输出预处理等操作。
📎 https://github.com/syndicalt/llmff

**💸 Agents CLI：用订阅制跑任意编码 Agent**
agents-cli.sh 允许用户用自己的订阅（如 Claude Pro、Cursor 等）来运行任意编码 Agent，避免按 API 调用计费，降低多 Agent 并行实验的成本。
📎 https://agents-cli.sh

**🧠 Headroom：在内容到达 LLM 前进行压缩**
Headroom 对 Agent 读取的所有上下文进行可逆压缩，在保持信息完整性的同时减少输入 token 量，直接降低推理成本。
📎 https://pypi.org/project/headroom-ai/

**🤖 Ask HN：如何解决 AI 的"confused deputy"问题？**
HN 上的讨论触及了一个核心的 Agent 安全问题：当 Agent 被授予多个权限时，它可能被误导去执行本不该执行的操作（"混淆的副手"问题）。社区正在探索解决方案。
📎 https://news.ycombinator.com/item?id=47716235

**📊 Qwen3.5 / Gemma-4 霸榜 HuggingFace 下载**
Qwen3.5-9B（894 万下载，1500 likes）和 Gemma-4-31B-it（1121 万下载，2838 likes）持续领跑 HF 模型下载榜。Qwen3.6 系列（35B-A3B MoE 架构，27B 稠密）也在快速增长。
📎 https://huggingface.co/models

**🌿 五大趋势信号：Agent 开发正从"工具拼接"走向"工程化"**
从 Thaw（Agent 版本控制）、Git-courier（结构化 Git）、Deliberate（拒绝日志）、Headroom（上下文压缩）到 Agents CLI（成本优化），社区正在为 AI Agent 开发构建完整的工程化工具链。这标志着 Agent 开发正在从早期的"prompt + tool"阶段，走向成熟的软件工程范式。

---

## 💬 结语

今天的信息量很大——Anthropic 关于 Agent 安全的核心工程博客、250 份文档就能毒化任意 LLM 的安全研究、OpenAI 解决 Erdős 问题的数学突破、以及 Thaw/RIS-Kernel/Deliberate 等一批 Agent 工程化新工具。最让人兴奋的信号是：**Agent 开发正在从"能用就行"走向"怎么管、怎么审、怎么优化"**。安全、可审计、可版本化——这些词开始在 AI 领域被认真对待。

你认为哪个 Agent 工程化方向最值得投入？来评论区聊聊 👇
