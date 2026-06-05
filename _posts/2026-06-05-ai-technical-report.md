---
layout: post
title: 'AI 技术分享 — 2026 年 6 月 5 日'
date: 2026-06-05 09:00:00 +0800
categories: [ai-technical-report]
---

> 今天的内容量很大。MiniMax M3 以不到 GPT-5.5 十分之一的成本在关键 benchmark 上超越它；Perplexity 在 COMPUTEX 发布全球首个本地-云端混合推理编排器；微软推出 MXC 操作系统级 Agent 沙箱；OpenAI 让 Codex 从开发者终端走向企业办公；MeMo 论文提出"不重训也能更新 LLM 记忆"的新框架。ChatGPT 以约 3 年时间突破 10 亿月活。下面逐一拆解。

---

## 🔥 今日看点

1. **6 月 1 日** — MiniMax M3 发布：SWE-Bench Pro 59.0% 超越 GPT-5.5 和 Gemini 3.1 Pro，API 价格仅为 5-10%，10 天内开放源码
2. **6 月 2 日** — Perplexity AI 在 COMPUTEX 2026 发布全球首个混合本地-云端推理编排器，自动按任务敏感度决定执行位置
3. **6 月 3 日** — 微软 Build 2026 推出 MXC（Microsoft Execution Containers）：OS 级 Agent 沙箱，OpenAI/Nvidia 已接入
4. **6 月 3 日** — OpenAI Codex 大更新：Sites + 角色插件系统，从开发者工具转型为企业办公平台
5. **6 月** — MeMo 论文（arXiv）：用独立记忆模型升级 LLM 知识，无需重训、避免灾难性遗忘
6. **6 月 2 日** — ChatGPT 以约 3 年突破 10 亿月活，史上最快达到该量级的应用
7. **6 月** — NVIDIA RTX Spark 超级芯片：1 PetaFLOPS + 128GB 统一内存，本地可跑 120B 参数模型、1M token 上下文
8. **6 月 4 日** — 美国两党推出 269 页联邦 AI 监管框架草案，拟 preempt 州级法律三年
9. **6 月 4 日** — Suno D 轮超 4 亿美元，估值 54 亿，半年翻倍
10. **6 月 4 日** — Anthropic 研究院：递归自我改进"可能比大多数机构准备好的来得更早"
11. **6 月初** — NVIDIA & Microsoft 合作：NVIDIA OpenShell + Windows 安全原语，让 Agent 安全跑在个人 PC 上
12. **6 月** — OpenAI ChatGPT 记忆系统全面升级，基于"dreaming"后台整理，可引用全部历史对话

---

## 💡 深度解读

### 1️⃣ MiniMax M3：以 GPT-5.5 十分之一的成本，在关键 benchmark 上实现超越

**痛点场景：**
你在做 LLM API 选型。GPT-5.5 输入 $5/M、输出 $30/M；Claude Opus 4.8 输入 $5/M、输出 $25/M。你的 Agent 每天消耗数百万 token，月度账单逼近六位数。老板问：有没有性能接近、成本低一个数量级的替代方案？

**技术原理：**
MiniMax M3 的核心突破在于**稀疏注意力机制**（MiniMax Sparse Attention, MSA）。传统 Transformer 的全局注意力计算复杂度是 $O(N^2)$，当上下文从 8K 扩展到 1M 时，计算量和显存消耗呈平方级爆炸。MSA 的做法是：

- **KV 矩阵分块预过滤**：将 KV 矩阵划分为精确的 block，先用轻量级筛选器判断哪些 block 与当前 query 相关，只对相关 block 做 full attention
- **KV outer gather Q 算子设计**：以 KV block 为外循环，动态聚合命中的 query；每个数据块只读一次，内存访问严格连续，硬件利用率大幅提升
- **实测数据**：MSA 比 Flash-Sparse-Attention 或 flash-moba 等开源方案快 **4 倍以上**；在 1M token 满上下文时，M3 的 per-token 计算量降至上一代的 **1/20**，prefill 阶段加速 **9 倍**，decoding 阶段加速 **15 倍**

此外，M3 是**原生多模态**架构——不是"预训练文本模型 + 后接视觉适配器"，而是从 Step Zero 就开始混合训练文本、图像和视觉组件序列，预训练语料超过 **100 万亿 token**。

**Benchmark 数据对比：**

| Benchmark | MiniMax M3 | GPT-5.5 | Gemini 3.1 Pro | Claude Opus 4.8 |
|-----------|-----------|---------|----------------|-----------------|
| SWE-Bench Pro | **59.0%** | 落后 | 落后 | 69.2% |
| Terminal Bench 2.1 | **66.0%** | — | — | 74.6% |
| MCP Atlas | **74.2%** | — | — | — |
| BrowseComp | **83.5%** | — | — | 79.3 |

API 定价对比（每百万 token）：

| 模型 | 输入 | 输出 | 总价 |
|------|------|------|------|
| MiniMax M3（限时） | **$0.30** | **$1.20** | **$1.50** |
| GPT-5.5 | $5.00 | $30.00 | $35.00 |
| Gemini 3.1 Pro (≤200K) | $2.00 | $12.00 | $14.00 |
| Claude Opus 4.8 | $5.00 | $25.00 | $30.00 |

M3 的限时价格仅为 GPT-5.5 的 **4.3%**，正式价格也仅为 **8-20%**。

**工程启示：**
对 MaaS/推理工程师来说，MSA 的 KV 分块预过滤思路值得借鉴——不需要改模型架构，只需要在 attention 实现层做 block-level 的筛选优化。同时，MiniMax 宣布 10 天内开放源码和 open weights，这意味着你可以直接在自己的推理框架（vLLM/SGLang）中集成，进一步压低成本。

来源：
- [MiniMax M3 官方博客](https://www.minimax.io/blog/minimax-m3)
- [VentureBeat 深度报道](https://venturebeat.com/technology/minimax-m3-debuts-eclipsing-gpt-5-5-and-gemini-3-1-pro-on-key-benchmark-performance-for-just-5-10-of-the-cost)
- [MiniMax API 定价页面](https://platform.minimax.io/subscribe/token-plan?tab=api-enterprise)

---

### 2️⃣ Perplexity 混合本地-云端推理编排器：让数据中心"搬到你的机器上"

**痛点场景：**
企业想用 AI Agent 处理财务合同、医疗记录等敏感数据。纯云端方案有数据出境和合规风险；纯本地方案算力有限、模型能力不足。目前的做法是人工决定"哪些任务放本地、哪些放云端"——但这要求用户既懂模型能力边界，又懂数据安全分类，几乎不可行。

**技术机制：**
Perplexity 在 COMPUTEX 2026 上演示了全球首个**混合推理编排器**（hybrid inference orchestrator），核心能力：

- **自主路由决策**：系统在任务执行过程中，自动决定每个子任务在本地还是云端执行，无需用户预先配置
- **敏感度感知**：金融记录、健康信息等敏感数据保留在本地；需要前沿模型能力的复杂推理任务路由到云端
- **中间态管理**：一个任务可能在本地和云端之间多次切换，编排器负责管理跨环境的任务状态
- **用户权限确认**：发送敏感任务到云端前会请求用户许可

架构上，这建立在 Perplexity 今年以来的多模型编排积累之上：2 月推出 Computer（19 个模型云端编排），3 月推出 Personal Computer（Mac 本地+云端混合），COMPUTEX 的演示将架构推进到**跨物理位置的自主编排**——不仅是选模型，还选执行地点。

**与芯片生态的关联：**
这个发布的时机并非偶然。NVIDIA 同场发布了 RTX Spark（1 PetaFLOPS + 128GB 统一内存，可本地跑 120B 参数模型）；Intel 展示了 Core Ultra Series 3。Perplexity 的编排器创造了一个直接的经济激励：本地芯片越强，更多推理可以留在本地，降低云成本、减少延迟。这反过来刺激了 AI PC 芯片的市场需求。

**工程启示：**
对做 Agent 基础设施的工程师来说，Perplexity 的路由思路揭示了一个趋势：**编排层的价值可能超过单个模型的价值**。关键设计点是：① 如何准确评估子任务复杂度；② 如何判断数据敏感度；③ 如何在跨环境中管理任务状态。如果你的产品已经在做多模型路由，加入"执行位置"这个维度可能是下一个差异化点。

来源：
- [VentureBeat: Perplexity hybrid inference](https://venturebeat.com/technology/perplexity-ai-unveils-hybrid-local-cloud-inference-system-at-computex-2026)
- [Perplexity 官方博文](https://www.perplexity.ai/hub/blog/the-data-center-moves-to-your-machine)
- [Perplexity Personal Computer 介绍](https://www.perplexity.ai/personal-computer)

---

### 3️⃣ 微软 MXC：操作系统级 Agent 沙箱，解决"Agent 越权"这个终极安全问题

**痛点场景：**
你的企业准备部署 AI Agent。Agent 能写代码、管文件、调 API——但安全团队提出了一个根本问题：如果 Agent 被 prompt injection 攻击，或者自身推理出错去删除了生产文件，怎么办？目前的做法是让 Agent 跑在用户桌面上，拥有和用户一样的权限。这等于把一个不可预测的程序给了 root 权限。

**技术机制：**
Microsoft Execution Containers（MXC）是 Build 2026 上最具实质性的平台级发布：

- **策略驱动的执行层**：开发者或 IT 管理员在 Agent 运行前声明其可访问的文件、目录、网络资源；Windows 内核在运行时强制执行这些边界
- **可组合沙箱频谱**（composable sandbox spectrum）：从轻量级进程隔离（GitHub Copilot CLI 已采用），到 micro-VM、Linux 容器，再到 Windows 365 上的完整云实例——同一个 SDK 和策略模型映射到不同隔离级别
- **会话隔离**：Agent 的执行与用户的桌面、剪贴板、UI 和输入设备完全分离，直接防御 UI spoofing、input injection、跨会话数据泄露等攻击
- **强身份绑定**：每个 Agent 绑定本地 ID 或 Microsoft Entra 云身份，所有操作可归因、可审计

现场演示中，微软开发者让 OpenClaw 在 MXC 沙箱中运行并尝试"删除桌面所有文件"——Agent 试图执行但被沙箱阻止，文件完好无损。

**企业集成时间线：**
- 7 月预览版：Agent 365 整合 Entra 身份、Intune 设备管理、Defender 运行时威胁防护、Purview 数据治理
- IT 管理员可通过 Intune 集中管控 Agent 隔离策略

**工程启示：**
这是企业 Agent 部署从 demo 走向 production 的关键基础设施。如果你在做企业级 Agent 产品，MXC 提供了一个标准化的安全执行环境；如果你在做 Agent 框架（如 vLLM/SGLang 之上的推理服务），需要考虑如何在沙箱环境中高效运行。值得关注的是，MXC 不仅适用于 Windows——它还集成了 WSL，意味着 Linux 容器内的 Agent 也能受益于同一套策略模型。

来源：
- [VentureBeat: Microsoft MXC](https://venturebeat.com/security/microsoft-launches-mxc-an-os-level-sandbox-for-ai-agents-with-openai-and-nvidia-already-on-board)
- [微软 Build 2026 Windows 开发者更新](https://aka.ms/Windows-Build2026)
- [Microsoft 官方博客](https://blogs.windows.com/windowsdeveloper/2026/06/02/build-2026-furthering-windows-as-the-trusted-platform-for-development/)

---

### 4️⃣ OpenAI Codex 从开发者终端走向企业办公：Sites + 角色插件系统

**痛点场景：**
你的财务团队想用 AI 分析销售数据，但不懂编程；市场团队想批量生成广告素材，但不会写 prompt 工程。Codex 以前主要是开发者的工具——终端、IDE 插件、命令行。非技术用户怎么用？

**技术机制：**
OpenAI 6 月 3 日的 Codex 更新包含三个关键特性：

**Annotations（局部作用域编辑）：**
以前的 AI 编辑文档往往需要重写整个文件，容易破坏格式或引入幻觉。Annotations 让 Codex 映射文档的底层数据 schema，用户高亮特定区域（如财务模型中的某个 cell block）后，Codex 只在该边界内执行代码，保持周围依赖、样式和未选中公式不变。

**角色插件系统（Plugins）：**
6 个角色插件聚合了 **62 个主流商业应用** 和 **110 个自动化技能**：
- Data Analytics：Snowflake、Databricks Genie、Hex、Tableau
- Creative Production：Figma、Canva、Shutterstock、Fal
- Sales：Salesforce、HubSpot、Slack、Outreach、Clay
- Product Design：Figma、Canva wireframe-to-prototype
- Public Equity & IB：Moody's、FactSet、S&P、PitchBook
- 更多…

**Sites（交互式托管页面）：**
将静态数据或文档转换为功能性的、可共享的 web 应用。财务领导可以把静态 spreadsheet 变成交互式 scenario planner，通过安全 URL 分享给高管实时调整假设。

**用户增长数据：**
Codex 已有 **500 万周活用户**，其中约 **20% 是非开发者**（财务分析师、市场运营、研究员），且非开发者采用速度是工程师的 **3 倍**。

**工程启示：**
对做 AI 基础设施的工程师来说，Codex 的方向变化意味着：① Agent 的交互界面正在从 CLI 转向 GUI/web canvas，推理框架需要考虑 web 托管场景；② 角色插件本质上是 MCP（Model Context Protocol）的企业级封装，62 个 SaaS 集成的维护成本不低，标准化的工具协议（如 MCP）的重要性在上升。

来源：
- [VentureBeat: OpenAI Codex update](https://venturebeat.com/orchestration/openais-codex-update-lets-agents-build-interactive-enterprise-workspaces-via-sites-and-role-specific-plugins)
- [Axios 报道](https://www.axios.com/2026/06/02/openai-codex-knowledge-workers)

---

### 5️⃣ MeMo 论文：不重训也能更新 LLM 记忆，RAG 之外的第三条路

**痛点场景：**
企业 LLM 部署后面临知识更新难题：公司政策变了、新产品发布了、合规要求更新了。现有方案要么用 RAG（检索增强生成）——但 embedding 语义相似度不等于用户查询实际需要的语义，且检索噪声会严重降低回答质量；要么微调——但成本极高且会导致灾难性遗忘。有没有第三条路？

**技术机制：**
MeMo（Memory as a Model，[arXiv:2605.15156](https://arxiv.org/abs/2605.15156)）提出了一个双模型架构：

- **MEMORY 模型**：一个较小的语言模型（实验中用 Qwen2.5-14B-Instruct），专门训练来将新知识编码到其参数中
- **EXECUTIVE 模型**：一个冻结的现成 LLM（如 Qwen2.5-32B 或 Gemini 3 Flash），作为推理引擎

交互采用三阶段协议：
1. EXECUTIVE 将用户复杂查询分解为原子子问题，MEMORY 独立回答每个问题建立基本事实
2. EXECUTIVE 基于初步线索发出后续查询，缩小候选实体范围直至收敛
3. EXECUTIVE 向 MEMORY 查询目标实体的支持事实，综合为最终答案

**关键技术点——Reflections：**
MeMo 用 GENERATOR 模型（实验中用 Qwen2.5-32B-Instruct）将原始文本蒸馏为数千个定向 QA 对，称为 "reflections"。MEMORY 模型在这些 QA 对上微调，使其能够仅用参数知识回答问题，无需读取检索上下文。

**持续更新——模型合并（Model Merging）：**
知识库增长时，MeMo 不需要从头联合重训。它只在新文档上训练一个新的独立 MEMORY 模型，提取代表参数变化的 "task vector"，然后数学合并到原 MEMORY 模型权重中。代价是相比完整重训有 **11-19%** 的精度下降，但避免了灾难性遗忘。

**实验结果：**
- MEMORY 模型可在 1-2B 参数级别运行（验证了 Gemma3-1B）
- 兼容 open-weight 和 closed API 模型——MEMORY  artifact 不绑定特定模型架构
- 在需要跨文档多跳推理的 benchmark 上表现接近 "Perfect Retrieval" 上界

**工程启示：**
对 MaaS 工程师来说，MeMo 提供了一种比 RAG 更轻量、比微调更安全的知识更新路径。特别是当你的推理框架已经部署了多个不同家族的模型时，MeMo 的"记忆模型与推理模型解耦"架构允许你为每个知识库训练一个小型 MEMORY 模型，然后在不同 EXECUTIVE 模型间复用。如果你的场景是"频繁更新、查询密集、对 RAG 噪声敏感"，值得 prototyping。

来源：
- [arXiv: MeMo 论文](https://arxiv.org/abs/2605.15156)
- [VentureBeat 解读](https://venturebeat.com/orchestration/memo-memory-model-teams-upgrade-llm-without-retraining)

---

### 6️⃣ ChatGPT 突破 10 亿月活 + 记忆系统升级：Agent 时代的用户基础设施

**痛点场景：**
你评估一个 AI 产品的市场渗透速度。Google Maps、TikTok、Instagram、YouTube 都曾达到 10 亿月活——ChatGPT 用了多久？

**数据与证据：**
- ChatGPT 于约 **3 年**内突破 10 亿 MAU，超过 Google Maps、TikTok、Instagram、YouTube 等所有此前达到该里程碑的应用，成为**史上最快**（Sensor Tower 数据，路透社报道）
- 目前每周活跃用户已达 **500 万+**（仅 Codex 平台）
- 记忆系统升级：ChatGPT 现在可引用**所有历史对话**，通过"dreaming"后台整理功能自动提取和保存关键信息，并支持**更新和修正**已有记忆

**记忆系统的技术含义：**
这是两个层面的突破：
1. **规模层面**：10 亿 MAU 意味着 ChatGPT 的记忆系统需要处理和索引的对话数据量是人类历史上最大的个人交互数据集之一
2. **产品层面**：跨会话的连贯性是 Agent 用户体验的关键瓶颈。记忆升级后，用户的项目上下文、代码风格偏好、常用工具链可以持久化，大幅减少每次对话的"冷启动"成本

**工程启示：**
对做 Agent 产品的工程师来说，ChatGPT 的记忆升级验证了一个方向：**跨会话状态管理是 Agent 产品的核心竞争力**。同时，10 亿 MAU 的规模也意味着：任何在该平台上构建的 Agent 工具或插件，都可能触达极大的用户群。值得关注的是，该功能目前在欧盟、英国、瑞士、挪威、冰岛和列支敦士登不可用——GDPR 和 AI Act 的合规约束仍然是全球部署的实际障碍。

来源：
- [路透社: ChatGPT hits 1 billion MAUs](https://www.reuters.com/technology/chatgpt-app-hits-1-billion-monthly-active-users-record-time-data-shows-2026-06-02/)
- [The Verge: ChatGPT memory upgrade](https://www.theverge.com/news/646968/openai-chatgpt-long-term-memory-upgrade)
- [OpenAI 官方公告](https://openai.com/index/memory-and-new-controls-for-chatgpt/)

---

### 7️⃣ NVIDIA RTX Spark + NVIDIA OpenShell：个人 AI 电脑的"安全启动层"

**痛点场景：**
你想在本地跑一个 120B 参数的 LLM 做 Agent，但现有方案要么显存不够，要么缺乏安全的执行环境——Agent 可能会读取你不想让它看到的文件，或者把敏感数据发到云端。

**技术机制：**
RTX Spark 和 NVIDIA OpenShell 共同构成了"个人 AI 电脑"的软硬件栈：

**RTX Spark 硬件：**
- NVIDIA Grace 20 核 Arm CPU + Blackwell RTX GPU（6,144 CUDA 核心 + 第五代 Tensor Core FP4）
- 1 PetaFLOPS AI 算力，128GB LPDDR5X 统一内存，300 GB/s 内存带宽
- 可本地运行 120B 参数 LLM + 1M token 上下文
- 秋季由 ASUS、Dell、HP、Lenovo、Surface、MSI 等厂商出货

**NVIDIA OpenShell 安全层：**
- 策略定义：用户可声明 Agent 能做什么、不能做什么
- 智能路由：基于用户隐私策略自动将查询路由到本地模型
- 信息脱敏：发送到云端的查询可自动遮蔽个人信息
- 与 Microsoft Windows 安全原语深度整合

OpenClaw 和 Hermes Agent 已宣布将在新 Windows 应用中采用这套栈。

**工程启示：**
本地 Agent 的安全性一直是 adoption 的最大障碍。OpenShell + MXC 的组合提供了一个标准化的安全执行环境。如果你在开发 Agent 框架或推理服务，需要考虑如何适配这套"本地安全 Agent"的范式——特别是策略定义、信息脱敏、和智能路由这三个核心能力。

来源：
- [NVIDIA 官方新闻稿](https://nvidianews.nvidia.com/news/nvidia-microsoft-windows-pcs-agents-rtx-spark)
- [NVIDIA OpenShell](https://build.nvidia.com/openshell)
- [Microsoft Build 2026 博客](https://blogs.windows.com/windowsexperience/?p=180355)

---

## 📡 行业动态

1. **Suno D 轮超 4 亿美元** — 估值达 54 亿美金，半年翻倍，由 Bond Capital 领投，未来数月推出首个与音乐产业合作开发的音乐模型。[MusicTech](https://musictech.com/news/industry/suno-series-d-funding-round/) · [Suno 官方博客](https://suno.com/blog/series-d-announcement)
2. **美国两党联邦 AI 监管框架草案** — 269 页草案拟建立全国统一 AI 监管标准，preempt 州级法律三年。[Politico](https://www.politico.com/2026/06/04/obernolte-trahan-ai-draft) · [Bloomberg Law 评论](https://news.bloomberglaw.com/legal-exchange-insights-and-commentary/america-needs-one-national-framework-for-artificial-intelligence)
3. **Anthropic 研究院：递归自我改进分析** — Claude 已编写 Anthropic 内部 80%+ 合并代码，工程师产出提升 8 倍，但"还不是今天"。[Anthropic Institute](https://www.anthropic.com/institute/recursive-self-improvement)
4. **ChatGPT 记忆系统全面升级** — 基于 dreaming 后台整理，可引用所有历史对话，Plus/Pro 已可用。[The Verge](https://www.theverge.com/news/646968/openai-chatgpt-long-term-memory-upgrade) · [OpenAI](https://openai.com/index/memory-and-new-controls-for-chatgpt/)
5. **Google 测试桌面浮动 AI 搜索栏** — Ctrl+Shift+Space 唤醒独立窗口，AI Mode 为核心体验。[Windows Report](https://windowsreport.com/chrome-tests-a-floating-ai-mode-search-bar-for-desktop/)
6. **Wired 发现 Meta 智能眼镜应用含人脸识别系统引用** — 在 Meta 智能眼镜应用中发现面部识别功能的技术引用。[Wired](https://www.wired.com/)
7. **TSMC 称满足美国本土需求"可能需要很长时间"** — 全球最大芯片制造商对美国本土产能扩张发出谨慎信号。[The Verge](https://www.theverge.com/ai-artificial-intelligence)
8. **SpaceX 准备上市，估值或达 1.77 万亿美元** — 目标募资 750 亿美元，或成史上最大 IPO。[The Verge](https://www.theverge.com/ai-artificial-intelligence)
9. **Zip 推出 AI Agent 防止企业数据泄漏到个人 ChatGPT** — 在 Zip AI Summit 上发布，解决员工将合同上传到个人 AI 账户的问题。[VentureBeat](https://venturebeat.com/technology/zips-new-ai-agents-want-to-stop-your-finance-team-from-uploading-contracts-into-personal-chatgpt-accounts)
10. **微软 Surface RTX Spark 开发者盒子** — 面向开发者的预装 RTX Spark 设备，无需云端即可运行大型 AI 模型。[VentureBeat](https://venturebeat.com/infrastructure/microsoft-debuts-surface-rtx-spark-dev-box-to-run-large-ai-models-without-cloud-costs)
11. **Google 考虑付费获取 Android 开发者代码访问权** — 404 Media 报道 Google 拟向 Android 开发者付费以获取应用内部代码用于训练。[VentureBeat](https://venturebeat.com/technology/google-might-pay-to-peek-at-your-code)

---

## 📝 结语

今天最值得关注的趋势是 **AI 基础设施正在从"云端集中式"向"本地-云端混合式"演进**。Perplexity 的混合推理编排器、NVIDIA RTX Spark + OpenShell 的本地 Agent 安全栈、微软 MXC 的 OS 级沙箱——这三者指向同一个方向：未来的 AI Agent 不会只跑在云端，也不会只跑在本地，而是在两者之间智能调度。对 MaaS / 推理工程师来说，这意味着推理框架需要同时支持本地和云端部署模式，编排层的重要性将进一步上升。

MiniMax M3 的发布则验证了另一条路线：**用架构创新（稀疏注意力）大幅降低推理成本**，同时保持前沿模型竞争力。当 open weights 模型的成本降至闭源模型的 5-10% 时，自建推理集群的经济模型将发生根本变化。

最后，ChatGPT 10 亿 MAU 和记忆系统升级意味着：Agent 产品的用户体验瓶颈正在从"模型能力"转向"跨会话状态管理"。谁能更好地管理用户的长期上下文，谁就能在 Agent 时代占据优势。
