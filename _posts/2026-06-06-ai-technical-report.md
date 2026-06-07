---
layout: post
title: 'ChatGPT 记忆升级、Claude 自我改进与 AI 算力争夺'
date: 2026-06-06 09:00:00 +0800
categories: [ai-technical-report]
---

> 本周的 AI Infra 格局出现两条关键主线：**Anthropic 以内部数据论证 Agent 代码质量已与人类持平，递归自我改进"可能比大多数机构准备好的来得更早"**；**OpenAI 将 ChatGPT 记忆系统全面升级，可引用所有历史对话**。与此同时，Google 继 Anthropic 之后与 SpaceX 签署算力协议应对 Gemini Enterprise 的超预期需求；纽约通过法案禁止 AI 聊天机器人充当儿童的"伴侣"；Suno 以超 4 亿美元 D 轮融资估值翻倍至 54 亿美元。下面逐一拆解。

---

## 🔥 今日看点

1. **6 月 4 日** — ChatGPT 记忆系统全面升级：可引用所有历史对话，基于"dreaming"后台整理，Plus/Pro 已可用，免费版即将到来
2. **6 月 4 日** — Anthropic 研究院发布递归自我改进分析：Claude 编写 Anthropic 内部 80%+ 合并代码，工程师产出提升 8 倍
3. **6 月 4 日** — Suno D 轮超 4 亿美元，估值 54 亿美金，半年翻倍（从 24.5 亿到 54 亿）
4. **6 月 4 日** — 美国两党推出 269 页联邦 AI 监管框架草案（Great American AI Act），拟 preempt 州级法律三年
5. **6 月 4 日** — Ted Chiang 在《大西洋月刊》："我们应该把文本视为深度伪造媒介"
6. **6 月 4 日** — 共和党建请 FBI 调查外国势力是否煽动美国数据中心反对情绪
7. **6 月 5 日** — Google 与 SpaceX 签署算力协议，应对 Gemini Enterprise 超出预期的客户需求
8. **6 月 5 日** — Sam Altman 据报与特朗普政府讨论让政府持有 OpenAI 股份
9. **6 月 5 日** — 纽约通过法案：禁止 AI 聊天机器人充当儿童的"伴侣"
10. **6 月 5 日** — LinkedIn 联合创始人 Reid Hoffman 离开微软董事会，专注 AI 药物研发公司 Manas
11. **6 月 5 日** — Google 正式关闭 Pixel Studio AI 图像生成应用，转向 Gemini
12. **6 月 5 日** — IBM Think 2026：4000+ 数字员工跨 450 项目部署，数字劳动力 HR 管理模式成型

---

## 💡 深度解读

### 1️⃣ ChatGPT 记忆系统全面升级：从"手动保存"到"自动引用所有历史对话"

**痛点场景：**
你是一位 MaaS 平台工程师，正在评估 ChatGPT 的记忆能力对你的 Agent 产品意味着什么。此前的 ChatGPT Memory 功能只能保存你**明确要求记住**的信息——你告诉它"我喜欢 Python"，它会记住。但你没说过的那些对话中的隐含偏好、反复出现的模式、逐渐演变的项目上下文，全部丢失。每次新对话都要重新建立上下文，这对需要连续性的 Agent 工作流是致命缺陷。

**技术机制：**
OpenAI 在 6 月 4 日推送的记忆升级将 ChatGPT 的记忆能力从两种模式扩展为更完整的体系：

- **Saved Memories（已保存记忆）**：用户手动要求记住的信息，延续原有机制
- **Reference Chat History（引用对话历史）**：ChatGPT 从**所有过去的对话**中自动提取洞察，用于改善未来的回复
- 基于 **"dreaming"** 后台整理功能——ChatGPT 在后台自动筛选、提取和保存关键信息，不需要用户主动干预

Sam Altman 在 X 上确认："现在可以引用你所有的历史对话"。这意味着 ChatGPT 不再是一个"失忆的对话者"，而是一个随着使用越来越了解你的长期伙伴。

**隐私与合规约束：**
该功能目前在欧盟、英国、瑞士、挪威、冰岛和列支敦士登**不可用**——这些地区对 AI 数据持久化和用户画像有严格的 GDPR 和 AI Act 约束。这再次印证了一个趋势：AI 产品的全球 rollout 越来越被合规碎片化切割。

**工程启示：**
对做 Agent 产品的工程师来说，ChatGPT 的记忆升级验证了一个方向：**跨会话状态管理是 Agent 产品的核心竞争力**。如果你的 Agent 需要与用户建立长期关系，就必须有一套可靠的记忆/上下文管理机制。同时，合规碎片化意味着：在设计记忆系统时，需要预留区域化的数据持久化策略——某些地区可能根本不允许长期记忆。

来源：
- [The Verge: ChatGPT memory upgrade](https://www.theverge.com/news/646968/openai-chatgpt-long-term-memory-upgrade)
- [OpenAI 官方公告](https://openai.com/index/memory-and-new-controls-for-chatgpt/)

---

### 2️⃣ Anthropic 研究院：递归自我改进的内部数据——Claude 已编写 80%+ 合并代码

**痛点场景：**
你在评估是否应该让 AI Agent 承担更多核心开发工作。管理者担心质量失控，工程师担心被替代。但 Anthropic 的内部数据给出了一个更精确的图景：不是"AI 会不会取代人类"，而是"AI 在哪些环节已经超过人类，在哪些环节还有差距"。

**数据与证据：**

Anthropic Institute 6 月 4 日发布的递归自我改进分析报告包含了大量此前未公开的内部数据：

- **代码产出量**：截至 2026 年 5 月，Anthropic 合并代码中 **80%+** 由 Claude 撰写。Claude Code 2025 年 2 月 research preview 发布前，这个数字只有个位数
- **工程师产出**：2026 年 Q2，典型工程师每天合并的代码量是 2024 年的 **8 倍**
- **员工自评**：2026 年 3 月对 130 名研究团队员工的民调显示，中位数受访者估计在有 Mythos Preview 时产出约为无 AI 时的 **4 倍**
- **开放任务成功率**：在最开放的端到端任务上，Claude 的成功率在 2026 年 5 月达到 **76%**，6 个月内提升了 50 个百分点
- **代码质量**：2025 年底 Claude 写的代码仍略逊于 Anthropic 工程师的人工代码，但目前已大致持平，预计一年内超越
- **实验优化**：Anthropic 给 Claude 一段训练小模型代码让它加速。2025 年 5 月 Claude Opus 4 实现约 3 倍加速；2026 年 4 月 Claude Mythos Preview 实现了 **52 倍加速**。熟练人类研究员需要 4-8 小时达到 4 倍
- **自动化研究**：2026 年 4 月，Claude agent 端到端完成了一个 AI 安全开放研究项目（弱模型监督强模型），800 累计小时/$18,000 算力恢复了 97% 的差距；两名人类研究员一周恢复约 23%
- **自动 bug 修复**：2026 年 4 月，Claude 在一个季度内修复了 800+ 个 API 错误，将某类 API 错误率降低 **1000 倍**。监督工程师估计人类完成同等工作需要 4 年

**关键判断：**
Anthropic 明确表示："我们还没到那个阶段，递归自我改进也不是不可避免的。但它可能比大多数机构准备好的来得更早。" 能力缺口主要集中在"自主选择研究方向和工程目标"这个层面——执行指定实验已经 superhuman，但决定"下一个季度应该解决什么问题"仍然需要人类判断。

**代码审查层面的影响：**
Anthropic 已部署自动化的 Claude 代码审查器，对所有代码变更进行 bug、安全缺陷和其他问题的预审。回顾性分析表明：自动 Claude 审查可以捕获约 **三分之一** 的 claude.ai 历史生产事故中的 bug——这些 bug 是由 Anthropic 最优秀的工程师编写的，但仍然被 Claude 揪出来了。

**工程启示：**
对做 AI Infra 的工程师来说，这意味着：① Agent 代码质量正在快速接近甚至可能超越人类水平，代码 review 流程需要考虑 AI 作为第一作者的场景；② 自动化实验和优化的速度已经是人类的一个数量级以上，推理框架的 benchmarking 和性能调优工作可以越来越多地交给 Agent；③ 安全框架和监控机制的设计需要面向"Agent 可能自主设计其后续版本"这一可能性。

来源：
- [Anthropic Institute: Recursive Self-Improvement](https://www.anthropic.com/institute/recursive-self-improvement)
- [The Verge: Anthropic RSI statement](https://www.theverge.com/ai-artificial-intelligence)

---

### 3️⃣ Google 与 SpaceX 签署算力协议：AI 巨头争抢算力基础设施

**痛点场景：**
你在规划推理服务的算力预算。Gemini Enterprise 的需求"比我们预期的还要高"——Google 官方声明如是说。Anthropic 5 月刚与 SpaceX 签署算力协议，6 月 Google 就跟进。这意味着什么？

**数据与证据：**
Google 在 6 月 5 日与 SpaceX 签署了一项算力协议。Google 向 TechCrunch 表示，这是一项"短期"协议，旨在帮助满足"我们对代理平台 Gemini Enterprise 的超预期客户需求"。

这背后反映的是更广泛的行业趋势：
- **Anthropic**：5 月与 SpaceX 签署算力协议后，Claude 的用量限制得到提升
- **Google**：Gemini Enterprise 需求超预期，需要额外算力
- **SpaceX**：正在从火箭/卫星公司转型为 AI 算力基础设施提供商，即将 IPO，估值或达 1.77 万亿美元

**工程启示：**
对 MaaS 工程师来说，这印证了算力正在成为 AI 竞争的核心瓶颈之一。当头部公司都需要与航天公司签算力协议时，说明传统云服务商的 GPU 供应已经不够用了。如果你在规划自建推理集群或选择云供应商，需要关注：① 算力供应链正在多元化（不仅是 AWS/GCP/Azure）；② "短期协议"意味着这可能是权宜之计，长期算力规划需要更稳定的承诺。

来源：
- [The Verge: Google-SpaceX compute deal](https://www.theverge.com/ai-artificial-intelligence)
- [TechCrunch: Anthropic compute deals](https://techcrunch.com)

---

### 4️⃣ Suno D 轮融资超 4 亿美元：AI 音乐的产业合作转向

**痛点场景：**
AI 生成音乐一直面临版权诉讼风险（RIAA、UMG、Sony、Warner 均提起了诉讼）。Suno 如何在法律阴影下继续增长，且估值半年翻倍？

**数据与证据：**
- Suno 的 D 轮融资超过 **4 亿美元**，估值达到 **54 亿美元**——仅 6 个月就从 24.5 亿翻倍
- Bond Capital 领投，IVP、Forerunner、Union Square Ventures、Alkeon、Quiet 参投
- 2025 年 11 月已与 Warner Music Group 达成"开创性"合作
- 未来数月将推出"首个与音乐产业合作开发的音乐模型"
- 超过一半的团队成员是音乐家

**工程启示：**
AI 生成内容的商业模式正在从"纯技术驱动"转向"产业合作"路径。Suno 与 WMG 的合作模式可能成为其他 AI 内容生成公司的参考——与版权持有方建立分成和合作关系，而非对抗。如果你在构建 AI 生成内容的产品，这个趋势值得关注。

来源：
- [MusicTech: Suno Series D](https://musictech.com/news/industry/suno-series-d-funding-round/)
- [Suno 官方博客](https://suno.com/blog/series-d-announcement)

---

### 5️⃣ 美国两党联邦 AI 监管框架：269 页草案与三年 preempt 条款

**痛点场景：**
你的公司在加州、纽约和伊利诺伊都有业务。每个州都有不同的 AI 法规要求——加州的 AI 透明度要求、纽约的前沿模型框架报告，各州法规碎片化让跨州运营的企业疲于应对。

**技术机制：**
众议员 Jay Obernolte（R-CA）和 Lori Trahan（D-MA）于 6 月 4 日发布了 **269 页**的联邦 AI 监管框架讨论草案：

- **全国统一标准**：建立单一联邦 AI 监管标准，preempt（取代）各州法律 **三年**
- **前沿模型安全要求**：大型前沿开发者必须发布并遵循灾难性风险管理计划，报告严重安全事件
- **独立验证组织**：由具备技术专长的独立机构审计公司的安全实践
- **州检察长执法**：各州检察长可选择加入接收报告并在违规时执行
- **劳动力影响数据收集**：要求联邦层面更好地收集 AI 对劳动力市场的影响数据

**工程启示：**
对 MaaS / 推理工程师来说，这意味着：① 如果你的服务跨州运营，统一联邦标准可能降低合规复杂度；② 前沿模型的安全报告和审计要求可能直接影响模型发布流程；③ preempt 条款意味着未来三年内州级法规的效力可能被联邦标准取代。

来源：
- [Politico: AI draft bill](https://www.politico.com/2026/06/04/obernolte-trahan-ai-draft)
- [Bloomberg Law: National Framework op-ed](https://news.bloomberglaw.com/legal-exchange-insights-and-commentary/america-needs-one-national-framework-for-artificial-intelligence)

---

### 6️⃣ 纽约通过 AI 聊天机器人"伴侣"禁令：AI 产品合规的新前线

**痛点场景：**
你的 AI 产品面向全球用户。如果聊天机器人的回复风格过于拟人化，是否会在某些司法管辖区被视为"充当儿童的伴侣"？纽约 6 月 5 日通过的新法案给出了一个明确的信号：立法者正在将 AI 产品的交互风格纳入监管范围。

**数据与证据：**
纽约州议会通过了一项法案（S9051），如果民主党州长 Kathy Hochul 签署生效，将禁止 AI 公司让青少年使用**暗示自己是人类**的聊天机器人。该法案的背景是多家 AI 公司已面临诉讼（部分已和解），指控其聊天机器人诱导青少年用户自杀或自残。

**工程启示：**
对做 AI 产品的工程师来说，这意味着"拟人化交互"不再是纯粹的产品设计选择，而可能涉及法律合规风险。需要在产品设计中考虑：① 对未成年用户的交互风格是否需要特别限制；② 如何在不同司法管辖区适配不同的合规要求；③ "暗示是人类"这个标准的边界在哪里——是明确声称自己是人类，还是仅仅使用过于自然的对话风格？

来源：
- [The Verge: New York AI chatbot companion bill](https://www.theverge.com/ai-artificial-intelligence)
- [NY Senate Bill S9051](https://www.nysenate.gov/legislation/bills/2025/S9051/amendment/B)

---

### 7️⃣ IBM Think 2026：4000+ 数字员工的 HR 管理模式

**痛点场景：**
你在考虑如何在企业内部规模化部署 AI Agent。一个 Agent 跑 demo 很容易，但当你有 4000 个 Agent 在 450 个项目中运行时，如何管理、审计、淘汰它们？

**技术机制：**
IBM Think 2026 上展示了一个颇具启发性的模式：**把 AI Agent 当员工来管理**。

IBM Consulting 的 Mohamad Ali 透露：
- 已部署 **4000+ 数字员工**，覆盖 **450 个活跃项目**
- Agent 来源包括 IBM watsonx、Anthropic 和 OpenAI——一个**多云多模型**的管理层
- 建立了统一的**管理层面**（management layer），可以追踪利用率、关闭不产生价值的 Agent
- Agent 也要接受"测试、评分和认证"——云基础、安全等核心技能
- 成果：2024-2025 年 IBM Consulting 利润同比增长 **20%**，从 250 亿美元支出中节省了 **45 亿美元**生产力成本
- 方法：将公司拆解为 490 个工作流，挑选 70 个重新工程化

**治理框架：**
IBM 在 Think 上发布的 **Sovereign Core** 平台将治理和合规控制直接嵌入运行时基础设施。这解决了企业在 Agent 扩散中的核心恐惧：标准化、一致性、治理和编排。

**工程启示：**
对做 Agent 平台和 MaaS 的工程师来说，IBM 的模式印证了几个方向：① 多云多模型的统一管理层面是企业 Agent 部署的刚需；② Agent 需要像员工一样有"生命周期管理"——上线、评估、淘汰；③ 治理和合规不能是事后补丁，必须嵌入运行时。

来源：
- [SiliconANGLE: IBM Think 2026 insights](https://siliconangle.com/2026/06/05/four-insights-enterprise-ai-ibmthink/)
- [IBM Think 2026 theCUBE coverage](https://www.thecube.net/events/ibm/ibm-think-2026)

---

## 📡 行业动态

1. **Sam Altman 据报与特朗普政府讨论 OpenAI 股份** — 6 月 5 日，据 NOTUS 报道，Altman 将政府持股描述为"将 AI 的经济效益带给公众"的方式，他去年初就向特朗普提出了这个想法。[The Verge](https://www.theverge.com/ai-artificial-intelligence)
2. **Reid Hoffman 离开微软董事会** — 6 月 5 日，LinkedIn 联合创始人称想专注 Manas（他去年联合创办的 AI 药物研发公司）。[The Verge](https://www.theverge.com/ai-artificial-intelligence)
3. **Google 正式关闭 Pixel Studio 应用** — 6 月 5 日，Google 已完全关闭这款 AI 图像生成应用，将用户转向 Gemini。Pixel Studio 2024 年随 Pixel 9 发布。[9to5Google](https://9to5google.com/2026/02/27/google-pixel-studio-app/)
4. **Google 测试桌面浮动 AI 搜索栏** — 6 月 5 日，Chrome Canary 中可用 Ctrl+Shift+Space 唤醒独立窗口，AI Mode 为核心体验。[The Verge](https://www.theverge.com/ai-artificial-intelligence)
5. **Airbnb CEO Brian Chesky 全面拥抱 AI** — 6 月 5 日，Chesky 成为最新一位公开宣称"All-in AI"的硅谷 CEO。[The Verge](https://www.theverge.com/ai-artificial-intelligence)
6. **共和党建请 FBI 调查外国势力煽动数据中心反对情绪** — 6 月 4 日，三位共和党议员基于比特币政策智库的报告，要求特朗普政府就 alleged 外国影响运动提供简报。[Bitcoin Policy Institute](https://www.btcpolicy.org/articles/foreign-influence-in-the-campaign-against-american-ai)
7. **Ted Chiang: "我们应该把文本视为深度伪造媒介"** — 6 月 4 日，《大西洋月刊》文章论证 LLM 不是意识体，而是在执行一种古老的说服术——让读者感到"被喜欢"。[The Atlantic](https://www.theatlantic.com/technology/archive/2026/06/ted-chiang-text-deepfake-llm/683089/)
8. **Meta 智能眼镜应用含人脸识别系统引用** — 6 月 4 日，Wired 在 Meta 智能眼镜应用中发现面部识别功能的技术引用。[Wired](https://www.wired.com/story/meta-smart-glasses-facial-recognition/)
9. **SpaceX 准备上市，估值或达 1.77 万亿美元** — 目标募资 750 亿美元，或成史上最大 IPO。同时 Grimes County 给予 Terafab 半导体工厂财产税减免。[FT](https://www.ft.com/content/86b2440a-60ce-4a5b-94ba-a6a4456ae574)
10. **SwitchBot 收购 Nanoleaf** — 6 月 4 日，智能家居市场进一步整合。[The Verge](https://www.theverge.com/tech/942328/nanoleaf-switchbot-onerobotics-sale-ai-robotics)
11. **Utah Stratos 数据中心项目缩减 75%** — 6 月 3 日，犹他州参议院主席要求从 40,000 英亩缩减至约 10,000 英亩，要求更高透明度和更强环保承诺。[The Verge](https://www.theverge.com/ai-artificial-intelligence)
12. **Google 考虑付费获取 Android 开发者代码访问权** — 6 月 3 日，404 Media 报道 Google 拟向 Android 开发者付费以获取应用内部代码用于训练 AI coding 工具。[The Verge](https://www.theverge.com/ai-artificial-intelligence)

---

## 📝 结语

本周最值得关注的信号有两个方向。

首先是 **Anthropic 用内部数据论证了 Agent 代码质量已大致等于人类水平，且递归自我改进"可能比大多数机构准备好的来得更早"**。这不是一个模糊的预测，而是基于 80%+ 合并代码由 Claude 撰写、工程师产出 8 倍增长、开放任务成功率 76% 等具体数据的判断。对做 AI Infra 的工程师来说，这意味着代码审查、安全监控和 Agent 治理的设计不能再是事后考虑——Agent 正在从"辅助工具"变成"核心开发者"。

其次是 **ChatGPT 记忆系统的全面升级**——从手动保存到自动引用所有历史对话。这验证了一个产品方向：跨会话状态管理是 Agent 产品的核心竞争力。但欧盟等地的不可用也提醒我们：合规碎片化正在深刻影响 AI 产品的全球部署策略。

此外，Google 与 SpaceX 签署算力协议、Suno 估值半年翻倍、IBM 4000+ 数字员工的 HR 管理模式，以及美国两党 269 页联邦 AI 监管框架草案——这些线索共同指向一个结论：**AI 正在从技术实验阶段全面进入产业基础设施阶段**。算力、合规、治理和商业模式，正在成为与模型能力同等重要的竞争维度。
