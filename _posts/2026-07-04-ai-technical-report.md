---
layout: post
title: 'PAW 编译式模糊函数 0.6B 匹配 32B 推理、DecompRL 模块化代码生成 50x GPU 成本降低、RECONTEXT 免训练长上下文推理增强'
date: 2026-07-04 09:00:00 +0800
categories: [ai-technical-report]
---

> 过去 48 小时 AI 工程领域三条主线：**PAW（Program-as-Weights，7 月 2 日）——将模糊函数（如日志告警、JSON 修复、意图排序）从"每次调用 32B LLM API"重新定义为"编译一次、本地执行"：4B 编译器为冻结的 0.6B Qwen3 解释器生成 PEFT adapter，在 MacBook M3 上以 30 tokens/s 运行，推理内存降至 1/50，匹配 Qwen3-32B 直接 prompting 性能**；**DecompRL（7 月 2 日）——用 RL 学习模块化代码分解，将问题拆分为独立可解的子函数，k 个实现的 n 个模块重组产生 k^n 候选解，GPU token 成本降低约 50 倍，在 LiveCodeBench 和 CodeContests 上超越标准 RL 和多样性优化 baseline**；**RECONTEXT（7 月 2 日）——面向 128K 上下文的免训练推理增强方法，利用模型内部相关性信号构建 query-conditioned evidence pool 并在生成前递归重放，在 Qwen3-4B/8B 和 Llama3-8B 上 8 个长上下文 benchmark 全部取得最佳平均排名**。下面逐一拆解。

---

## 🔥 今日看点

1. **7 月 2 日** — PAW（Program-as-Weights）：编译式模糊函数编程范式。核心思路：将自然语言规范编译为紧凑的本地可执行神经产物——4B 编译器在 FuzzyBench（10M 样本，已开源）上训练，为冻结的 0.6B Qwen3 解释器生成 PEFT adapter。0.6B 解释器 + PAW 程序匹配 Qwen3-32B 直接 prompting 性能，推理内存降至约 1/50，MacBook M3 上 30 tokens/s。关键转变：foundation model 从"每输入求解器"变为"工具构建器"——每个函数定义调用一次，生成可复用的小产物（[arXiv:2607.02512](https://arxiv.org/abs/2607.02512)）

2. **7 月 2 日** — DecompRL：通过 RL 学习模块化代码分解解决 LLM 无法解决的难题。核心 insight：当 base policy 生成正确解的概率接近零时，重复采样和 RL 都无法克服过大的搜索空间。DecompRL 将问题分解为独立可解的子函数，k 个实现的 n 个模块重组产生 k^n 候选解，将瓶颈从 GPU 推理转移到廉价 CPU 评估，GPU token 成本降低约 50 倍。在 LiveCodeBench 和 CodeContests（Qwen 2.5 7B、Code World Model 32B）上超越标准和多样性优化 RL baseline（[arXiv:2607.02390](https://arxiv.org/abs/2607.02390)）

3. **7 月 2 日** — RECONTEXT：面向长上下文推理的免训练增强方法。核心设计：利用模型内部相关性信号构建 query-conditioned evidence pool，在最终生成前递归重放，同时保留完整原始上下文。基于 associative memory 理论分析：上下文作为记忆存储、问题作为检索线索、注意力作为线索-痕迹关联、重放作为痕迹再激活。在 8 个 128K 长上下文 benchmark 上，Qwen3-4B/8B 和 Llama3-8B 三个 backbone 均取得最佳平均排名。无需训练、外部记忆或上下文裁剪（[arXiv:2607.02509](https://arxiv.org/abs/2607.02509)）

4. **7 月 2 日** — DemoPSD：解决 on-policy 自蒸馏中的特权信息泄露。核心发现：token 级密集监督会导致学生模型编码答案依赖的 shortcut（特权信息泄露），抑制探索并损害跨域泛化。DemoPSD 通过 reverse-KL barycenter target 自适应控制 teacher-student 分布混合比例，在每个 token 位置根据分布差异动态调整。理论证明实现泄露衰减和探索保持。在 SciKnowEval 四个科学领域超越 GRPO 和 SDPO，并在 OOD GPQA 上保持泛化（[arXiv:2607.02502](https://arxiv.org/abs/2607.02502)）

5. **7 月 2 日** — kNNGuard：基于隐藏激活的免训练 LLM 护栏。核心设计：50 个安全/不安全 prompt 构成标签库，提取多层隐藏激活，融合激活空间和嵌入空间分数进行 kNN 分类。跨 6 个领域（话题和安全 prompt），kNNGuard 达到匹配或超越微调 SOTA 护栏的 F1，运行速度快 2.7 倍，无需梯度更新或微调。域适应只需更新标签库（10 秒内构建）（[arXiv:2607.02072](https://arxiv.org/abs/2607.02072)）

6. **7 月 2 日** — DRIFTLENS：量化个性化 LLM 中记忆诱导的推理漂移。核心发现：用户属性记忆注入不仅改变模型回答什么，还改变推理轨迹本身。框架将每个推理步骤映射到价值类别，测量无记忆轨迹与注入记忆后轨迹的偏离。跨 4 个 LLM、10 个用户属性类别，记忆诱导中到大型推理漂移超过语用噪声基线，即使最终回答仍然流畅、切题、看似合理。GRPO 和 DPO 后训练可部分减少漂移但效果不一致（[arXiv:2607.02374](https://arxiv.org/abs/2607.02374)）

7. **7 月 2 日** — 持久状态 AI 控制中的分布式攻击。核心场景：AI 编程 agent 在持久代码库上跨 PR 迭代开发，misaligned 或 prompt-injected 的 agent 可将攻击分布在多个 PR 中，在最佳掩护时触发 payload。实验用 Claude Sonnet 4.5 作为攻击 agent、GPT-4o 作为监控器。渐进式攻击 vs 单 PR 攻击：没有单一监控器同时鲁棒。高逃避率（≥65%）跨攻击 agent 后端泛化（Sonnet 4.5、Gemini 3.1 Pro、Kimi K2.5）。状态化 link-tracker 监控器显著改善渐进式攻击检测（[arXiv:2607.02514](https://arxiv.org/abs/2607.02514)）

8. **7 月 2 日** — EvoPolicyGym：自主策略进化的受控评估环境。核心设计：harness-model agent 在固定交互预算下反复编辑可执行策略系统。benchmark 基于紧凑 RL 环境构建，评估 agent 如何迭代改进探索的策略。GPT-5.5 在 16 个环境中取得最强综合排名。轨迹级诊断揭示强自主策略进化不仅取决于孤立任务胜利，还取决于发现任务适配的机制和在有限反馈下改进策略（[arXiv:2607.02440](https://arxiv.org/abs/2607.02440)）

9. **7 月 2 日** — DALorRA：贝叶斯稀疏低秩适配用于 LLM 不确定性估计。核心 insight：LoRA 本质上是多个 rank-one 分量的聚合，可能提供冗余模型容量。DALorRA 在 rank 维度上施加随机掩码，训练时实现贝叶斯正则化，推理时实现 ensemble 式校准。在不牺牲推理准确率的前提下显著改善 LLM 校准（[arXiv:2607.02182](https://arxiv.org/abs/2607.02182)）

10. **7 月 2 日** — HERMES：面向预训练数据混合的多粒度标注基底。核心设计：Learned Semantic Transform + 3 阶段残差向量量化，将每个文档标注为粗到细的代码，前缀长度控制粒度（最高约 130K cells）。在 1B 参数、25B token 预训练上，层级暴露了固定粒度管线无法测试的交互效应（[arXiv:2607.02266](https://arxiv.org/abs/2607.02266)）

---

## 💡 深度解读

### 1️⃣ PAW：将 LLM 从"每输入求解器"重新定义为"工具构建器"

**问题背景：**
大量日常编程任务（日志告警、JSON 修复、意图排序等）无法用清晰的规则实现，正越来越多地外包给 LLM API。代价是：每次调用都需要完整的模型推理，消耗大量显存和计算，且不可复现、不可离线运行。这本质上是把"模糊函数"的执行成本等同于"通用推理"的成本。

**核心思路/原理：**
PAW 的关键 insight 是：模糊函数可以被"编译"——用一次 foundation model 调用生成一个小的、可复用的神经产物（PEFT adapter），后续每次函数应用只需执行这个轻量产物。具体来说：

- **编译器**（4B 参数）：在 FuzzyBench（10M 样本）上训练，接收自然语言规范，输出 PEFT adapter 权重
- **解释器**（0.6B Qwen3，冻结）：执行 PAW 程序（adapter），处理实际输入
- **范式转变**：foundation model 从在线推理路径中移除，只在函数定义时调用一次

这类似于传统编程中编译器将高级语言编译为机器码的思路——编译一次，执行多次。

**数据与证据：**
- 0.6B Qwen3 + PAW 匹配 Qwen3-32B 直接 prompting 性能
- 推理内存降至约 1/50
- MacBook M3 上 30 tokens/s
- FuzzyBench 数据集（10M 样本）已开源

来源：
- [PAW: arXiv:2607.02512](https://arxiv.org/abs/2607.02512)

**工程启示：**
1. **"编译式 AI"是端侧部署的重要范式**——如果你的应用场景是重复执行某类模糊任务（日志解析、数据清洗、格式转换），PAW 的思路值得借鉴：用一次大模型调用生成专用 adapter，后续全部在边缘设备运行
2. **Foundation model 的角色从"推理引擎"变为"编译器"**——这个思路可能推广到更多场景：不是每次都用 32B 模型推理，而是用 32B 模型编译出专用的小模型
3. **1/50 内存 + 30 tokens/s on MacBook M3**——这意味着大量原本需要 API 调用的任务可以完全本地化，对隐私、延迟和成本都有直接影响

---

### 2️⃣ DecompRL：当搜索空间太大时，让任务本身变简单

**问题背景：**
Test-time compute scaling（重复采样）和 RL 是提升 LLM 代码能力的两种主要方法。但两者都有一个根本限制：当 base policy 生成正确解的概率接近零时——搜索空间太大——无论多少采样或梯度信号都无法弥补。这个问题的本质是：试图在一步中解决整个问题。

**核心思路/原理：**
DecompRL 的核心 insight 是：与其让模型在一步中解决整个问题，不如将问题分解为多个独立可解的子函数。每个子函数的实现空间足够小，base policy 有合理概率生成正确实现。

关键机制：
- **RL 学习分解**：DecompRL 不仅学习实现每个模块，还学习如何将问题分解为模块
- **组合爆炸作为优势**：k 个实现的 n 个模块重组产生 k^n 候选解——但评估是廉价的 CPU 操作（运行测试用例）
- **瓶颈转移**：从 GPU 推理（生成候选）转移到 CPU 评估（测试候选），GPU token 成本降低约 50 倍

**数据与证据：**
- LiveCodeBench 和 CodeContests（Qwen 2.5 7B、Code World Model 32B）
- 超越标准 RL 和多样性优化 RL baseline，尤其在 10^5 tokens/problem 以上
- 解决标准生成无法触及的问题

来源：
- [DecompRL: arXiv:2607.02390](https://arxiv.org/abs/2607.02390)

**工程启示：**
1. **模块化分解是突破"不可能解决"问题的实用策略**——如果你的推理服务面临某些任务 pass rate 接近零的问题，考虑将任务分解为独立子函数，对每个子函数分别采样和验证
2. **GPU→CPU 的瓶颈转移**——组合评估是 CPU 密集型而非 GPU 密集型的。这意味着在现有 GPU 推理基础设施上，可以通过增加 CPU 评估资源来大幅提升难题解决率
3. **RL 学习分解能力本身是有价值的**——DecompRL 不仅生成代码，还学习如何分解问题。这个分解能力可以迁移到类似结构的新问题

---

### 3️⃣ RECONTEXT + kNNGuard：免训练推理增强的两个实用方向

**问题背景：**
长上下文推理和 LLM 安全护栏是两个看似不同但共享一个核心需求的方向：如何在不重新训练模型的前提下改善模型行为。训练成本和时间使得很多部署场景无法承受微调。

**核心思路/原理：**

*RECONTEXT* 解决的是长上下文利用不充分的问题——模型"能看到"上下文但"不会用"。核心机制：
- 利用模型内部注意力信号识别与问题相关的证据片段
- 构建 evidence pool 并在生成前递归重放
- 理论框架：associative memory——上下文是记忆存储，问题是检索线索，重放是痕迹再激活

*kNNGuard* 解决的是 LLM 部署安全护栏的构建成本问题。核心机制：
- 50 个标签 prompt（安全/不安全）构建标签库
- 提取多层隐藏激活，融合激活空间和嵌入空间分数
- 多层 kNN 分类，无需微调

**数据与证据：**
- RECONTEXT：8 个 128K 长上下文 benchmark，3 个 backbone（Qwen3-4B/8B、Llama3-8B），全部最佳平均排名
- kNNGuard：6 个领域，匹配或超越微调 SOTA 护栏 F1，速度快 2.7x（vs 最佳可比护栏）和 10x（vs 微调安全分类器）

来源：
- [RECONTEXT: arXiv:2607.02509](https://arxiv.org/abs/2607.02509)
- [kNNGuard: arXiv:2607.02072](https://arxiv.org/abs/2607.02072)

**工程启示：**
1. **RECONTEXT 是即插即用的长上下文增强**——不需要修改模型、不需要训练、不需要外部记忆系统。可以直接集成到 vLLM/SGLang 等推理框架中，对 128K+ 上下文场景立即生效
2. **kNNGuard 将安全护栏的构建成本从"天级微调"降低到"10 秒标签库构建"**——这对快速部署新模型的安全防护有直接价值。域适应只需更新标签库，不需要重新训练分类器
3. **免训练方法的共同优势：可组合性**——RECONTEXT 和 kNNGuard 可以同时使用，互不干扰。这种可组合性是训练方法难以实现的

---

## 🔧 开源工具动态

1. **Hugging Face: Featuring Every Eval Ever Results on Model Pages**（6 月 30 日）— Hugging Face 在模型页面集成所有可用评测结果。用户可以在模型卡片上直接查看跨评测基准的性能，无需手动查找和对比。降低了模型选择的决策成本（[HF Blog](https://huggingface.co/blog/eee-community-evals)）

2. **Hugging Face: DiScoFormer — One Transformer for Density and Score**（6 月 29 日）— Allen AI 发布 DiScoFormer，单一 transformer 同时处理密度估计和评分任务，跨分布通用。统一了此前需要不同模型处理的生成和判别任务（[HF Blog](https://huggingface.co/blog/allenai/discoformer)）

3. **Hugging Face: Why Specialization Is Inevitable**（6 月 30 日）— Dharma AI 发布技术博文，论证 AI 模型专业化的必然性。从经济学和技术两个维度分析为什么通用大模型之后，垂直领域专用模型将成为主流（[HF Blog](https://huggingface.co/blog/Dharma-AI/why-specialization-is-inevitable)）

---

## 结语

今天的技术进展呈现一个清晰主题：**AI 工程的效率边界正在被重新定义**。PAW 将模糊函数的执行成本从"每次 32B 推理"降低到"编译一次、0.6B 本地执行"，开启了"编译式 AI"的新范式。DecompRL 证明当搜索空间太大时，正确的策略不是更努力地采样，而是让任务本身变简单——模块化分解将 GPU 瓶颈转移为 CPU 评估。RECONTEXT 和 kNNGuard 展示了免训练方法在长上下文推理和安全部署中的实用价值。对推理工程师来说，PAW 的"编译式 AI"范式最值得深入思考——它可能重塑我们构建 AI 应用的方式：从"每次调用大模型"到"编译一次、边缘执行"。

