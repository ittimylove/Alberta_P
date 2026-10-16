# HarnessFix 文章总结

**论文**：*From Failed Trajectories to Reliable LLM Agents: Diagnosing and Repairing Harness Flaws*（Mengzhuo Chen, Junjie Wang, Zhe Liu, Yawen Wang, Haiming Zheng, Qing Wang；中科院软件所等；arXiv:2606.06324v2 [cs.SE]，2026-07-02）
**源文件**：`Chen_2026_HarnessFlawRepair.pdf`（共 13 页，其中正文 p.1–11，参考文献 p.11–13）

**引用约定**
- 本 PDF 页脚没有印页码，页码 = PDF 页序。
- 论文为双栏排版，定位写成“页码 + 左/右栏 + 小节 + 段落”。段首有加粗或斜体小标题的，直接用小标题标注（如“§II-B-1) *Data Collection* 段”）；没有小标题的，按该小节内的顺序写“第 N 段”。
- 每条后面附一句英文原文短摘录，方便核对。
- 本文只依据论文本身。原文没写的内容标“未说明”。正文中提到“详见网站”而论文里没有给出的内容，也按“未说明”处理。

---

## 1. 研究问题

| # | 内容 | 原文依据 |
|---|---|---|
| 1.1 | LLM agent 的可靠性不仅取决于基座模型，还取决于 **agent harness**（包裹模型、决定其如何行动的运行时基础设施）。出现失败时，改 harness 通常比重训或微调模型更可行。 | p.1 右栏，§I 第 1–2 段：“the reliability of such systems depends not only on the base model, but also on the agent harness”；“improving the harness is often more practical than modifying the base model” |
| 1.2 | 但 agent 的行为大多在运行时由模型推理决定，**失败通常无法直接映射到 harness 实现中的具体位置**（源码、prompt 等），而是体现在混合了自然语言推理、工具交互和环境反馈的执行轨迹里；传统调试和修复技术因此效果有限。 | p.1 右栏，§I 第 2 段：“failures cannot usually be mapped directly to specific locations in the harness implementation” |
| 1.3 | 现有方法只部分解决这个问题：(a) 运行时/监督式优化往往绕过错误，**没有修复根本的 harness 缺陷**；(b) 结果驱动的方法依据最终分数优化 prompt、工作流或 harness 逻辑，**没有先定位轨迹中的责任证据和出问题的 harness 层**，改动因而宽泛、间接或范围不当。 | p.1 右栏，§I 第 3 段：“without fixing the underlying harness flaw”；“outcome-driven rather than diagnosis-driven … broad, indirect, or poorly scoped” |
| 1.4 | 本文目标：从失败轨迹出发，细致诊断负有责任的运行时步骤，再用这些证据定位并修复 harness 缺陷。 | p.1 右栏 – p.2 左栏，§I 第 4 段：“start from failed agent trajectories, diagnose the responsible runtime steps in detail, and use this evidence to localize harness flaws and repair them” |
| 1.5 | 由此带来的三个挑战：① 失败证据**分散**在语言密集、推理驱动的多步轨迹中，难以精确诊断；② 即使在轨迹中定位到失败，也难以**映射回**具体的 harness 实现（运行时轨迹与静态代码/prompt 之间没有显式对齐）；③ 修改 harness 可能在减少一类失败的同时**引入其他回归**。 | p.2 左栏，§I 第 4 段（续）：“First, failure evidence is fragmented … Second, … difficult to map it back to the concrete harness implementation … Third, harness modifications may … introduc[e] regressions elsewhere” |
| 1.6 | 实验部分的四个研究问题：RQ1 有效性与效率；RQ2 失败诊断是否有效；RQ3 各设计选择是否都有贡献（消融）；RQ4 在一个模型上得到的修复能否迁移到其他模型。 | p.8 左栏，§IV-A |

## 2. 研究对象

| # | 内容 | 原文依据 |
|---|---|---|
| 2.1 | **agent harness 及其可编辑实现制品**：prompt 模板、工具说明、编排代码、配置文件、适配器、日志 hook、验证脚本。本文的“harness 修复”指修改这些制品，**不更新基座模型参数**。 | p.3 左栏，§II-A 第 2 段：“harness repair refers to modifying these artifacts, rather than updating the base-model parameters” |
| 2.2 | harness 的职责按 **ETCLOVG 七层分类**（引自文献 [1]）：执行环境与沙箱、工具接口、上下文与记忆、生命周期与编排、可观测性、验证与评估、治理与安全。本文把这七层作为连接运行时证据与待修复机制的“词汇表”。 | p.2，表 I；p.3 左栏，§II-A 第 3 段：“we use these layers as a vocabulary” |
| 2.3 | **失败的 agent 执行轨迹**：它是诊断的输入，被抽象为 HTIR。 | p.1 摘要；p.4，§III-A 引言段 |
| 2.4 | **动机研究的对象**：GitHub 上 30 个流行且活跃的开源 LLM agent 系统及其开发记录。 | p.3 左栏，§II-B-1) *Data Collection* 段 |
| 2.5 | **实验中被修复的初始 harness H0**（每个基准一个）：GAIA 用 open-deep-research；SWE-Bench Verified 用 mini-swe-agent；AppWorld 用官方 simplified ReAct code agent；Terminal-Bench 2.0 Verified 用 Harbor Terminus-2。 | p.8 左栏，§IV-B 第 2 段 |
| 2.6 | **task model**：默认 GPT-5 mini；另有 Claude Sonnet 4.5、DeepSeek V3.2、Qwen3.5 Plus、Gemini 3 Pro 共 4 个模型用于对比和迁移实验。 | p.8 右栏，§IV-D 第 1 段；p.9，表 III |

## 3. 数据来源

**动机研究数据**

| # | 内容 | 原文依据 |
|---|---|---|
| 3.1 | 30 个开源 LLM agent 系统，约 **57,780 条开发记录**，包括 issue、PR、commit 和 release note。 | p.3 左栏，§II-B-1) *Data Collection* 段 |
| 3.2 | 用“LLM 分析 agent + 人工验证”的迭代流程区分 harness 相关/无关记录（小规模示例标注 → agent 分类 → 人工检查 → 改进 agent 策略 → 最终抽样人工审计），得到 **26,174 条 harness 相关记录**，约占全部记录的 45.3%。 | p.3 左栏，§II-B-1) *Data Collection* 段、其后 “Overall” 段；p.3，图 1(a) |
| 3.3 | 详细的收集与分类流程、缺陷分布与修复策略的刻画流程：原文写“见网站”，论文中**未说明**。人工审计的样本量和一致性指标同样**未说明**。 | p.3 左栏，§II-B-1) 段末：“detailed collection and classification procedures are provided on our website”；“We further use a similar …” 段 |

**实验数据（4 个基准，每个基准随机划分为训练/验证/held-out 测试集）**

| 基准 | 任务类型 | 采样与划分（训练/验证/测试） | 原文依据 |
|---|---|---|---|
| GAIA | 开放式研究问答 | 从 166 道有公开答案的开发集题目中采样 150 道，划分为 60/30/60 | p.8 左栏，§IV-B 第 2 段 |
| SWE-Bench Verified | 仓库级软件修复 | 从 500 个实例中采样 250 个，划分为 100/50/100 | 同上 |
| AppWorld | 有状态的应用自动化 | 从 750 个任务中采样 225 个，划分为 90/45/90 | 同上 |
| Terminal-Bench 2.0 Verified | 命令行终端工作流 | 从 89 个任务中采样 85 个，划分为 34/17/34 | 同上 |

| # | 内容 | 原文依据 |
|---|---|---|
| 3.4 | **RQ2 人工标注诊断集**：从测试集上的失败中构建，每个基准由 3 名标注者标注 20 条失败的测试轨迹，标注内容为责任失败步骤、根因、实现锚点、涉及的 harness 层、所选修复算子。这些标签**不用于**修复或验证决策。 | p.8 右栏，§IV-D 第 2 段 |
| 3.5 | 标注细则、指标细节、标注者间一致性：原文写“见网站”，论文中**未说明**。 | p.8 右栏，§IV-D 第 2 段：“The full annotation and metric details are provided on our website” |
| 3.6 | 训练集的具体用途（例如是否就是收集失败轨迹的来源）：原文**未明确说明**。验证集用于补丁验收（见 §5.6），测试集用于报告结果和构建 RQ2 标注集。 | p.8 左栏，§IV-B 第 1 段（只说划分）；p.7 右栏，§III-D 第 3 段（held-out validation set） |
| 3.7 | 模型的详细配置：原文写“见网站”，论文中**未说明**。 | p.8 右栏，§IV-D 第 1 段 |
| 3.8 | 代码公开：作者称已在匿名网站发布源码（https://github.com/HarnessFix/HarnessFix）。 | p.2 右栏，脚注 1 |
| 3.9 | 实验运行时间、API 费用（以美元计）：**未说明**（只报告了 token 数，见 §4）。 | — |

## 4. 核心变量

> 原文没有使用“自变量/因变量”这类术语。下表按实验设计归纳，每一项都对应原文中的具体设置或指标。

**自变量（被操纵的条件）**

| 变量 | 取值 | 原文依据 |
|---|---|---|
| 修复方法 / harness | H0；H0 + HarnessFix；人工设计的 harness（GAIA：DeepResearchAgent、MiroFlow；SWE：OpenHands、Trae-Agent；AppWorld：FullCodeRefl、IPFunCall、CUGA；TB2：OpenCode、OpenHands）；以同一 H0 为起点的自演化/修复基线（GEPA、SCOPE、ReCreate、Meta-Harness） | p.8，§IV-C；p.9，表 IV |
| task model | GPT-5 mini（默认）/ Claude Sonnet 4.5 / DeepSeek V3.2 / Qwen3.5 Plus / Gemini 3 Pro | p.8 右栏，§IV-D 第 1 段；p.9，表 III |
| 基准 | GAIA / SWE-Bench Verified / AppWorld / Terminal-Bench 2.0 Verified | p.8 左栏，§IV-B |
| 诊断输入的表示形式（RQ2） | 原始轨迹 / 原始轨迹 + 数据流 / 原始轨迹 + 数据流 + 控制流 / 完整 HTIR | p.9，表 V |
| 消融变体（RQ3） | 仅改 prompt；去掉基于轨迹的诊断（改用原始轨迹摘要）；去掉限定范围的修复算子（改为自由编辑 harness）；去掉回归感知的验收 | p.8 右栏，§IV-D 第 3 段；p.9，表 VI |
| 迁移设置（RQ4） | 把用 GPT-5 mini 修复得到的 GAIA harness 直接用于另外 4 个模型，除最少的 API 兼容处理外不做针对目标模型的修复 | p.8 右栏，§IV-D 第 4 段 |

**因变量 / 观测指标**

| 指标 | 定义 | 原文依据 |
|---|---|---|
| TCR（任务完成率） | 按基准的标准做法衡量性能；表 III、IV、VI、VII 中的数值是 3 次独立运行的算术平均 | p.8 左栏，§IV-B 末段；p.8 右栏，§IV-D 第 1 段 |
| Δ | H0 + HarnessFix 相对 H0 的百分点提升 | p.9，表 III；表 IV 注 |
| Tokens (M) | 离线演化/修复所消耗的 token 数（百万）；括号内为相对 HarnessFix 的变化 | p.8 左栏，§IV-C 末段；p.9，表 IV 注；p.9 右栏，§V-A 第 2 段（“offline evolving/repair tokens”） |
| 诊断准确率（RQ2） | 与人工金标准比较：Step（责任步骤准确率）、Cause（根因）、Anchor（实现锚点准确率）、Layer（harness 层 macro-F1）、Operator（修复算子准确率） | p.9，表 V；p.9 右栏，§V-B |
| 显著性 | 基于重复运行结果的单侧配对符号检验，仅对表 IV 中的基线比较报告 p 值 | p.9 右栏，§V-A 第 3 段 |
| 修复涉及的层数 | 各方法在 ETCLOVG 七层中编辑了哪些层（主要编辑 / 部分或间接编辑 / 未编辑） | p.10，表 VIII |

**关键参数**

| 参数 | 取值 | 原文依据 |
|---|---|---|
| HarnessFix 自身 4 个 agent 所用模型 | GPT-5 mini（“our HARNESSFIX and baselines” 均默认使用） | p.8 右栏，§IV-D 第 1 段 |
| 独立运行次数 | 3 | 同上 |
| 补丁验收的“目标改进”阈值与“回归上限” | **未说明**（只说 “achieves the target improvement and stays within the regression limit”） | p.7 右栏，§III-D 第 3 段 |
| 修复迭代轮数、每个缺陷生成几个候选补丁 | **未说明** | — |
| 缺陷合并时“相同缺陷”的判定阈值 | **未说明**（只说按根因解释与支持证据判断） | p.7 左栏，§III-B-2) 第 1 段 |

## 5. 研究方法

**总体框架**：由 4 个 LLM agent 组成——轨迹抽象 agent → 诊断 agent → 修复 agent → 验证 agent（p.3 右栏，§III 引言段；p.4，图 2）。

| # | 方法组成 | 要点 | 原文依据 |
|---|---|---|---|
| 5.1 | **动机研究（经验研究）** | 统计 harness 缺陷在七层中的分布，并按层归纳常见修复策略，压缩为一组**修复算子**（表 II），供后续限定范围的修复使用。 | p.3，§II-B-2)、3) 段；p.4，表 II |
| 5.2 | **HTIR：TraceStep** | 每个节点是一个可恢复的执行步骤（模型调用、工具动作、最终提交等），存储顺序 id、请求消息、响应消息，以及 3 个派生标注：Role（角色）、Execution status（执行状态）、Artifact/state effect（外部可观测的制品或状态效果，及其细节：受影响实体、状态转移、支持证据）。 | p.4 右栏，§III-A-1) 段；p.5 左栏，第 1 段 |
| 5.3 | **HTIR：数据流链接** | 按时间倒序搜索更早的 TraceStep，识别显式复用（复制、拼接）与语义复用，并检查请求构造逻辑（记忆组装、工具描述、prompt 模板）。每条链接记录源/目标 id、源 span、目标 span、复用关系。 | p.5 左栏，§III-A-2) 第 1–3 段；p.5 右栏，S3→S4 示例 |
| 5.4 | **HTIR：控制流链接** | 结合时间邻域内的步骤、数据流链接和 harness 实现，把观测到的状态转移匹配到 continue/retry/delegate/validate/finalize/terminate 等执行逻辑，记录触发逻辑和触发条件。 | p.5 右栏，§III-A-3) 第 1–3 段；p.6 左栏，S5→S6 示例 |
| 5.5 | **HTIR：实现锚点 + 层映射** | 利用工具名、prompt 模板引用、适配器调用、验证器事件等运行时信号定位实现制品。每个锚点记录：制品引用（如仓库路径 + 行号范围）、锚点关系、支持证据。随后对照表 I 的职责标准，把锚定的证据映射到 ETCLOVG 层；一个步骤可以映射到多层。 | p.6 左栏，§III-A-4) 第 1–2 段；p.6，§III-A-5) 段 |
| 5.6 | **失败归因** | 诊断 agent 分 4 步：① 症状定位（外部评测结果 + 最终步骤的证据）；② 证据回溯（沿数据流/控制流得到排序的候选步骤，刻意放宽范围）；③ 候选裁定；④ 层归属。每条失败轨迹输出一条诊断记录：归因结果、失败信号、harness 诊断。 | p.6 右栏，§III-B-1) 第 1–3 段 |
| 5.7 | **缺陷合并** | 单次失败可能是偶然的，因此先按涉及的层建立索引，层有重叠的作为合并候选，再合并根因相同的诊断，形成**缺陷记录**（缺陷摘要 + 支持它的诊断记录）。 | p.7 左栏，§III-B-2) 第 1–2 段 |
| 5.8 | **限定范围的修复** | 不允许自由修改核心运行时逻辑（作者认为自由修改风险高）。按缺陷涉及的层，从表 II 中检索候选算子，再按根因和证据筛选，指定 1 个主算子和若干辅助算子；然后实例化为**修复规范**：目标与范围、编辑约束（允许/禁止修改的制品，如禁止修改任务数据、oracle、测试标签、公共 API）、所需行为。 | p.7 左栏，§III-C 第 1–2 段、§III-C-1) 段；p.7 右栏，§III-C-2) 段及示例 |
| 5.9 | **补丁验证 + 修复记忆** | 先做预验证（是否符合规范、语法和静态分析），再在 held-out 验证集上评估：只有达到目标改进、且对原 harness 已解决任务的回归不超过上限时才接受。每次结果写入 harness 修复记忆：接受的记录作为后续实现示例，拒绝的记录说明原因，避免重复提出无效改动。 | p.7 右栏，§III-D 第 1–4 段；p.8 左栏，第 1 段 |
| 5.10 | **实验设计** | RQ1：在 4 个基准 × 5 个模型上与 H0 对比（表 III），并以 GPT-5 mini 与全部基线做端到端对比（表 IV）；RQ2：在人工标注集上比较 4 种输入表示（表 V）；RQ3：4 个消融（表 VI）；RQ4：GAIA 跨模型迁移（表 VII）；讨论部分比较各方法编辑涉及的层（表 VIII）。 | p.8，§IV-A–D；p.9–10，§V、§VI |

## 6. 主要结论

| # | 结论 | 原文依据 |
|---|---|---|
| 6.1 | **动机研究**：harness 相关记录约占全部开发记录的 45.3%，30 个仓库都有此类改动。缺陷出现在全部七层：生命周期 25.4%、工具 21.5%、可观测性 18.1%、上下文 13.2%、执行 9.3%、验证 7.0%、治理 5.5%。29/30 个 agent 存在生命周期、工具或可观测性方面的缺陷。主导的缺陷层随仓库定位而变：长时程系统更多是生命周期缺陷，面向基准的 harness 更多是验证缺陷。 | p.3 左栏，“Overall” 段；p.3，§II-B-2) 段；p.3，图 1(b) |
| 6.2 | **RQ1（相对 H0）**：所有设置都有提升，平均 +11.1；GAIA 提升最大（+14.5 至 +18.4），AppWorld 最小（+5.9 至 +9.3）；各模型的平均提升在 10.3 至 12.5 之间，较为一致。 | p.8 右栏，§V-A 第 1 段；p.9，表 III |
| 6.3 | **RQ1（与基线比较，GPT-5 mini）**：比人工设计的 harness 平均高 6.3（1.9 至 10.0）；比自演化/修复基线平均高 6.9（2.6 至 16.7）；比最强的 Meta-Harness 高 2.6 至 5.0，而 Meta-Harness 的离线 token 消耗多 63.5% 至 100.5%。单侧配对符号检验的 p 值为 2.4×10⁻⁴ 至 4.9×10⁻⁴。 | p.9 右栏，§V-A 第 2–3 段；p.9，表 IV |
| 6.4 | HarnessFix 在 4 个基准上的 TCR（GPT-5 mini）：GAIA 43.3→61.7，SWE 45.3→57.3，AppWorld 36.7→43.0，TB2 17.6→26.5；token 消耗分别为 57.6M、48.9M、37.2M、33.5M。 | p.9，表 IV |
| 6.5 | **RQ2**：完整 HTIR 与人工金标准一致性较高：Step 85.0、Cause 83.8、Anchor 81.3、Layer macro-F1 86.2、Operator 82.5。从原始轨迹（Step 55.0）到逐步加入数据流（70.0）、控制流（77.5），各项指标持续提升。作者称已有技术的 step 准确率大约在 45–52%。 | p.9 右栏，§V-B；p.9，表 V |
| 6.6 | **RQ3**：所有消融都低于完整版本。性能下降最大的是“仅改 prompt”和“去掉修复算子”，作者据此认为有效的修复依赖对运行时机制的针对性修改。 | p.9 右栏，§V-C；p.9，表 VI |
| 6.7 | **RQ4**：GAIA 上修复得到的 harness 迁移到其他 4 个模型后均有提升（+5.5 至 +9.5），小于源模型的 +18.4，但始终为正。作者认为这些修复针对的是跨模型共有的 harness 层面失败。 | p.10 左栏，§V-D；p.10，表 VII |
| 6.8 | **编辑范围**：GEPA、SCOPE 只改 1 层，ReCreate 改 2 层；Meta-Harness 可以改 7 层，但搜索不具针对性、token 成本高；HarnessFix 在 4 个基准上分别涉及 5/7/5/4 层。不同基准的缺陷集中在不同层（GAIA：工具与可观测性；SWE：工具执行、上下文构造与验证证据之间的交界；AppWorld：工具、生命周期、验证、治理；TB2：上下文、生命周期、验证、治理）。 | p.10，§VI 第 1–3 段；p.10，表 VIII |
| 6.9 | 总结论：harness 修复不只是 prompt 优化问题；把运行时轨迹与 harness 实现对齐，以支持诊断驱动的修复，对提高 LLM agent 可靠性有价值。 | p.10 右栏，§VI 第 4 段；p.11 左栏，§VIII |

## 7. 适用条件

> 原文没有专门讨论适用条件的章节。下表列出的都是原文在方法或实验中**明确写出**的前提，不是整理者推断的。

| # | 条件 | 原文依据 |
|---|---|---|
| 7.1 | 需要**原始执行轨迹**和 **harness 制品**作为输入（HTIR 由二者编译而成）。 | p.1 摘要：“compiles raw execution traces and harness artifacts into … HTIR”；p.3 右栏，§III 引言段 |
| 7.2 | harness 必须以**可编辑制品**的形式存在（prompt 模板、工具说明、编排代码、配置文件等）；方法只修改这些制品，不修改模型参数。 | p.3 左栏，§II-A 第 2 段 |
| 7.3 | 诊断依赖**外部评测结果**来定位症状。 | p.6 右栏，§III-B-1) 第 2 段：“examining the external evaluation result” |
| 7.4 | 需要一个 **held-out 验证集**，用于判断目标缺陷是否减少、是否引入回归。 | p.7 右栏，§III-D 第 3 段 |
| 7.5 | 缺陷要**在多次执行中重复出现**才会被合并为缺陷记录并触发修复（单次失败可能是偶然的）。 | p.7 左栏，§III-B-2) 第 1 段：“Single failed executions can be accidental” |
| 7.6 | 修复动作受限于从动机研究中归纳的**修复算子集合**（表 II）。 | p.7 左栏，§III-C 第 2 段 |
| 7.7 | 实现锚点的推断依赖**可用的运行时信号**（工具名、prompt 模板引用、适配器调用、验证器事件、控制器动作）；状态效果和触发条件也是“在可获得时”才记录。 | p.6 左栏，§III-A-4) 第 2 段；p.5 左栏，第 1 段（“when available”）；p.5 右栏，§III-A-3) 第 2 段 |
| 7.8 | 实验验证的范围：4 类任务（开放式研究问答、仓库级软件修复、有状态应用自动化、终端工作流）。跨模型迁移**只在 GAIA 上**测试过。 | p.8 左栏，§IV-B 第 1 段；p.8 右栏，§IV-D 第 4 段 |
| 7.9 | 对 trace 格式、harness 规模或编程语言等方面的要求：**未说明**。 | — |

## 8. 研究局限

**原文没有专门的局限性 / 有效性威胁（Threats to Validity）章节。作者明确承认的局限：未说明。**

以下是原文中零散出现、与局限相关的表述（均为原文原话，不是整理者的评价）：

| # | 表述 | 原文依据 |
|---|---|---|
| 8.1 | 作者把本文定位为研究 agent harness 可靠性的**早期工作**。 | p.11 左栏，§VIII：“As an early effort to study agent harness reliability” |
| 8.2 | 跨模型迁移的提升**小于源模型上的提升**（+5.5 至 +9.5，对比 +18.4）。 | p.10 左栏，§V-D：“These gains are smaller than the source-model gain” |
| 8.3 | 多处关键细节因篇幅限制**放在网站上，论文中未给出**：动机研究的数据收集与分类流程、缺陷分布与修复策略的刻画流程、模型配置、RQ2 的标注与指标细节。 | p.3 左栏，§II-B-1) 段及其后两段；p.8 右栏，§IV-D 第 1–2 段 |

---

### 附：原文中需要对照核实的表述

以下几处只是对照原文时注意到的，不代表我对其正误下结论：

- **提升范围的表述**：摘要和 §I 写的是 “improves … by 6.3% to 18.4%”（p.1、p.2），这两个端点对应 GPT-5 mini 在 AppWorld 和 GAIA 上的提升；但表 III 中跨全部模型的最小提升是 +5.9（Claude Sonnet 4.5 / AppWorld），§V-A 自己也写了 “between 5.9% and 9.3%”（p.8）。原文没有说明摘要中的范围只针对 GPT-5 mini。
- **单位**：原文用 “%” 表述提升，但表 IV 注说明括号内是 “percentage-point gain”（p.9），即百分点。
- **表 III 的 Δ 列**：TB2 / Claude Sonnet 4.5 为 34.3→47.1，记作 +12.7（直接相减为 12.8）；TB2 / DeepSeek V3.2 为 31.4→39.2，记作 +7.9（直接相减为 7.8）（p.9）。原文没有说明原因。
- **表 VI 的两行数据**：“Prompt-only repair” 与 “w/o scoped repair operators” 在 GAIA（50.6）、AppWorld（37.4）、TB2（18.6）上数值完全相同，只有 SWE 不同（48.3 vs. 49.3）（p.9）。原文没有对此作出解释。
- **step 准确率的对比**：§V-B 称已有技术的 step 准确率“通常约 45–52%”，引用的是文献 [48][49]（p.9）。原文没有说明这些数字是否是在本文的标注集上测得的。
- **运行示例与表 VIII**：AppWorld 运行示例的修复规范中，涉及的层包括 Observability，所选算子包括 “API-error and state-delta logging”（p.7）；但表 VIII 中 AppWorld 一列的 Observability 标为“未编辑”（p.10）。原文没有说明这个示例对应的补丁最终是否被接受。
- **显著性检验的范围**：只报告了表 IV 基线比较的 p 值（p.9）。表 III、V、VI、VII 是否做过显著性检验：未说明。
