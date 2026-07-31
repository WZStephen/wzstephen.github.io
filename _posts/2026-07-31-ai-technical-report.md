---
layout: post
title: 'On-Policy LLM 安全路由重对齐、Agent 因果审计与行为检测、Inkling 975B 全栈支持'
date: 2026-07-31 09:00:00 +0800
categories: [ai-technical-report]
---

> 本期聚焦 AI Agent 安全与验证三条主线：安全对齐方面，On-Policy Distillation 提出路由式安全重对齐方案，在不损失任务能力的前提下实现模板无关的防御；可信度验证方面，Latent MAS 因果审计首次严格区分「有 latent channel」与「channel 真正在传递信息」，Agent 行为检测框架将人类/Bot/AI Agent 三分类为 Web 安全基础设施提供新范式；工具化落地方面，AgenticCANN 将 LLM Agent 进化框架扩展至华为 Ascend C NPU 算子自动生成，Pegasus 从人类视频提取具身机器人可学习数据。开源框架方面，vLLM v0.26.0 与 SGLang v0.5.16 同步推出 Inkling 975B MoE 全栈支持，SGLang DSpark 推测解码在 DeepSeek-V4-Pro 上达到 383.7 tok/s，vLLM DeepSeek-V4 路由优化将端到端延迟降低近 3%。

---

## 🔥 今日看点

1. **7月31日** — On-Policy Distillation for LLM Safety：提出基于 on-policy 采样 + 路由分类器的安全重对齐方法，在模板攻击下保持 >90% 安全率同时保留 >95% 任务能力，路由架构解耦安全与能力两条路径。[arXiv:2607.27081](https://arxiv.org/abs/2607.27081)

2. **7月31日** — Latent Multi-Agent 因果审计：首次提出 counterfactual 因果审计框架，验证 LLM 多 Agent 系统中 latent channel 是否真正传递任务相关信息。发现部分系统的 channel 因果贡献接近零——性能提升实际来自接收方 agent 自身。[arXiv:2607.26773](https://arxiv.org/abs/2607.26773)

3. **7月31日** — Browser AI Agent 行为检测：提出三分类框架（人类/Bot/AI Agent），证明二分类检测器在标签空间中结构性缺失 Agent 类别，仅需 8 个最小特征即可实现 Agent 识别 F1 >0.85。[arXiv:2607.26935](https://arxiv.org/abs/2607.26935)

4. **7月31日** — AgenticCANN：首个将 LLM Agent 进化框架应用于华为 Ascend C NPU 算子自动生成的系统，通过知识增强 Agent 进化在 NPU 上实现接近手工优化的算子性能，开辟了非 CUDA 硬件的 AI 辅助算子开发新方向。[arXiv:2607.26661](https://arxiv.org/abs/2607.26661)

5. **7月31日** — Pegasus 具身数据桥接：从人类视频中通过图结构交互表示提取机器人可学习数据，低成本弥合人类形态与机器人硬件之间的 embodiment gap，不需要大规模机器人数据采集。[arXiv:2607.26903](https://arxiv.org/abs/2607.26903)

6. **7月31日** — Agent 自演化约束探索：揭示 Agent 技能优化中的过拟合问题——对有限轨迹的过度开发导致技能泛化能力下降，提出约束化探索-利用框架缓解 skill overfitting。[arXiv:2607.26643](https://arxiv.org/abs/2607.26643)

7. **7月27-25日** — vLLM v0.26.0 与 SGLang v0.5.16：两大框架同步推出 Inkling 975B MoE 全栈支持。SGLang DSpark 推测解码在 DeepSeek-V4-Pro TP8 B300 上达 383.7 tok/s（accept length ~5）；vLLM DeepSeek-V4 路由优化将 E2E TPOT 降低 2.94%，fused_topk_bias kernel 实现 1.5-2x 加速。

---

## 💡 深度解读

### 1️⃣ On-Policy Distillation：路由式安全重对齐

**问题背景：**
LLM 微调范式下，恶意数据提供者可在下游语料中嵌入有害行为，使模型在保留专业技能的同时「按需」违反人类价值观。现有安全重对齐方案存在三大缺陷：(1) 导致灾难性遗忘——安全训练同时破坏了模型的专业能力；(2) 对模板变体脆弱——攻击者通过改写 prompt 模板即可绕过防御；(3) 依赖高质量对抗样本——标注成本高昂且覆盖面有限。

**核心思路/原理：**
本文提出基于 on-policy 采样的安全重对齐方法。核心流程：(1) 用多种攻击模板对模型进行 prompt 攻击，收集模型的 on-policy 响应作为训练数据；(2) 训练一个轻量级路由分类器，判断输入是否为攻击性 prompt；(3) 对判定为攻击的输入激活安全行为路径（拒绝或引导），正常输入则走原模型路径。关键创新在于「路由」架构——将安全检测与任务执行解耦为两条独立路径，路由分类器仅负责分流，不改变模型内部表征，因此避免了全局重训练带来的灾难性遗忘。

**数据与证据：**
- 在多种模板变体攻击下保持 >90% 安全拒绝率
- 任务能力保留 >95%（传统方法通常下降 5-15%）
- 对未见过的攻击模板也具有泛化能力
- 路由分类器参数量极小（<1% of model size），推理开销可忽略

来源：
- [On-Policy Distillation for LLM Safety: A Routing Approach: arXiv:2607.27081](https://arxiv.org/abs/2607.27081)

**工程启示：**
1. **生产环境安全部署**：路由式方案可即插即用于已部署模型，无需全量微调，特别适合需要同时保持专业能力和安全底线的场景（金融、医疗、法律咨询）
2. **对抗样本成本降低**：on-policy 采样不需要人工标注高质量对抗样本，大幅降低安全对齐的数据成本和迭代周期
3. **模板无关性的实际意义**：攻击者经常通过模板变体绕过安全过滤，路由分类器对未见模板的泛化能力意味着更强的防御纵深
4. **与 RLHF 的互补**：路由方案可作为 RLHF/DPO 之外的安全补充层，形成多道防线

---

### 2️⃣ Latent Multi-Agent 因果审计：通道存在 ≠ 信息传递

**问题背景：**
LLM 多 Agent 系统（MAS）中，「latent communication」（隐式通信）通过传递连续内部表征而非文本来实现 Agent 间协作。更高的表征容量被默认为更好的通信——但这是一个未经验证的假设。终任务性能也无法区分「消息确实在传递信息」与「接收方只是从自己的 agent 获取了信息」这两种截然不同的机制。

**核心思路/原理：**
本文提出因果审计框架，应用 counterfactual 干预来验证 latent channel 的因果作用：(1) **消息替换**（swap）：将例 A 的消息替换为例 B 的消息，观察性能变化——如果性能随之改变，说明消息确实在传递例级信息；(2) **消息消融**（zero-out）：将 latent channel 置零，看性能是否下降——如果无变化，说明 channel 未被利用；(3) **来源归因**（attribution）：区分性能提升来自消息内容 vs 来自另一 Agent 的直接信息供给。三个测试构成因果证据链，缺一不可。

**数据与证据：**
- 在多个 latent MAS 基准上应用审计框架
- 发现部分系统中 latent channel 的因果贡献接近零——性能提升实际来自接收方 agent 自身的信息
- 即使 channel 确实在传递信息，传递的内容也可能与任务无关（表征容量 ≠ 信息利用率）
- swap 和 zero-out 测试的结论经常矛盾，证明单一测试不充分

来源：
- [Do Latent Channels Actually Communicate? A Causal Audit: arXiv:2607.26773](https://arxiv.org/abs/2607.26773)

**工程启示：**
1. **MAS 架构验证方法论**：任何声称使用 latent communication 的多 Agent 系统都应该通过因果审计验证，而非仅依赖 end-task 性能。这是评价 MAS 创新的必要补充证据
2. **避免虚假创新**：论文提醒社区，增加通信通道的表征维度不等于改善了通信——许多 MAS 论文的性能提升可能被高估
3. **调试与优化方向**：因果审计可帮助定位 MAS 中真正的信息瓶颈。如果 channel 因果贡献为零，应该重新设计通信协议而非增加带宽
4. **对 Comm 研究的影响**：类似工作（如 agent-to-agent 向量传输）都应补充因果审计，否则无法证明通信机制的有效性

---

### 3️⃣ Browser AI Agent 检测：三分类框架的必要性

**问题背景：**
当前 Web 安全基础设施将流量二分类为「人类」或「Bot」。但随着 AI Agent 通过浏览器自动化（Playwright、Puppeteer、Selenium）进行 Web 交互，出现了一个既非人类也非传统 Bot 的新流量类别。AI Agent 有 LLM 驱动的决策逻辑、自适应行为模式、语义连贯的页面交互——这些特征既不同于脚本化 Bot，也不同于人类用户的随机性。

**核心思路/原理：**
本文提出三分类检测框架：(1) 人类流量、(2) 传统 Bot（脚本化、规则驱动）、(3) AI Agent（LLM 驱动、自适应行为）。通过浏览器自动化实验收集三类流量特征，证明：二分类器将 AI Agent 错误路由到「人类」或「Bot」类别中，这是标签空间缺失导致的结构性错误——无论特征工程多优秀都无法解决。三分类器通过增加一个 Agent 类别，结构性地解决了这一问题。最小特征集分析表明，Agent 检测的关键特征包括鼠标轨迹的随机性模式、页面停留时间的分布特征、以及请求序列的语义连贯性指标。

**数据与证据：**
- 二分类器对 AI Agent 的误分类率达 30-50%（AI Agent 被大量误判为正常用户）
- 三分类器在保持人类/Bot 检测精度的同时，Agent 识别 F1 达 0.85+
- 最小特征集仅需 8 个特征即可达到接近全特征集的检测精度
- 框架已开源用于 Web 安全研究

来源：
- [What Does It Take to Detect an AI Agent?: arXiv:2607.26935](https://arxiv.org/abs/2607.26935)

**工程启示：**
1. **Web 安全基础设施升级**：随着 AI Agent 部署规模扩大，现有 Bot 检测系统需要升级为三分类架构，否则会将 Agent 流量误判为正常用户或恶意 Bot
2. **合规与风控**：金融、电商等平台需要区分 AI Agent 和人类用户以满足合规要求（如 AI 行为披露法规）
3. **Agent 行为特征设计**：Agent 开发者可参考这些特征来设计更自然的行为模式，减少被误判的风险

---

## 🔧 开源工具动态

1. **vLLM** — v0.26.0（2026-07-27）：411 commits / 212 contributors（61 new）。**Inkling 975B MoE 全栈支持**：基础建模 + piecewise CUDA graph + Hopper FA4 relative attention + MTP=1 推测解码 + LoRA + 标准 ModelOpt NVFP4 量化。**DeepSeek-V4 性能优化**：专用路由 kernel（E2E TPOT -2.94%）、fused_topk_bias（1.5-2x kernel）、冗余 repeat/copy 移除（E2E TPOT -1.8%）。fp32 lm_head via head_dtype 提升生成精度，ROCm torch.mm fast path 支持 AMD GPU。灵活注意力后端切换。生产环境建议：v0.26 为 Inkling + DS-V4 提供完整优化栈，建议尽快升级。

2. **SGLang** — v0.5.16（2026-07-25）：574 PRs / 169 contributors。**DSpark 推测解码**：基于置信度驱动的块级推测-验证，在 DeepSeek-V4-Pro TP8 B300 上达 383.7 tok/s（accept length ~5），通过 `--speculative-algorithm DSPARK` 启用。**Inkling 975B 支持**：1M-token 上下文，混合 sliding-window + full + Mamba2 linear attention + NVFP4 MoE，Blackwell TP4 上 71.7k tok/s input、171 tok/s/user decode。与 vLLM 互补：SGLang 在推测解码和结构化生成方面领先，vLLM 在大规模生产部署方面更成熟。

3. **TensorRT-LLM** — v1.3.0rc22（2026-07-22）：持续迭代 RC 版本。NVIDIA 硬件深度优化，FP8/FP4 量化支持完善。Blackwell 架构优化持续推进。与 SGLang/vLLM 形成互补——TRT-LLM 专注单模型极致性能，vLLM/SGLang 专注多模型调度与灵活性。

4. **llama.cpp** — master-fff0e0e（最新 commit 2026-07-30）：仍无正式 release tag，持续活跃开发。GGUF 格式持续演进，CPU 推理性能稳步提升。Apple Silicon Metal / AMD ROCm / Vulkan 多后端支持。适合边缘设备和离线部署场景，社区保持高频更新。

5. **MLC LLM** — v0.26.dev0（开发版）：最新正式 release 仍为 v0.20.0。端侧部署方向持续探索，TVM 编译栈与 WebGPU/WebNN 后端支持。内存占用优化持续推进，适合移动端/IoT 场景预研。

---

*报告撰写：AI 技术分析师*
*数据来源：arXiv cs.AI、GitHub Releases、行业新闻*
*报告日期：2026 年 07 月 31 日*
