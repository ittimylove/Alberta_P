# AHE（Agentic Harness Engineering）文章总结

- **论文**：Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses
- **作者 / 机构**：Jiahang Lin, Shichun Liu, Chengjun Pan, Lizhi Lin, Shihan Dou, Zhiheng Xi, Xuanjing Huang, Hang Yan, Zhenhua Han, Tao Gui, Yu-Gang Jiang；复旦大学、北京大学、上海奇绩智峰（p.1 标题区）
- **版本**：arXiv:2604.25850v4 [cs.CL]，2026-05-18（p.1 左侧页眉）
- **唯一信息来源**：`Lin_2026_AgenticHarnessEngineering.pdf`（共 35 页，PDF 页码与论文印刷页码一致）

> **标注说明**
> - 依据统一写作「p.页码，章节，第 N 段（段首原文）」。段落按该页中该章节的正文段落计数，图表标题不计入段落。段首原文用来帮助定位。
> - 「未说明」表示原文没有给出这项信息。
> - 本文由 pdftotext 抽取的文本整理而成。Table 2（逐仓库 token 列）、Table 4（超参数分块）、Figure 3 柱值的版面已经错位。这些位置只引用能与正文核对一致的数字，其余在相应条目中注明。

---

## 1. 研究问题

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 核心问题 | 进化智能体（evolution agent）怎样**联合且稳定地**进化 coding agent harness 中**所有**可编辑组件？ | p.2，§1 Introduction 第 3 段（"An intuitive direction…"）末句："How can an evolution agent jointly and stably evolve all editable components of a coding agent's harness?" |
| 问题背景 1 | harness 设计会显著影响长程编码任务的完成率；最优 harness 与具体模型相关，换基座模型后需要重新适配，目前这一步靠人工完成，跟不上模型迭代的速度 | p.2，§1 第 2 段（"Harness design materially shifts…"） |
| 问题背景 2 | 现有自动化方法大多只优化单一组件（prompt、skills 或 in-context playbook），很少联合进化全部组件 | p.2，§1 第 3 段（"An intuitive direction…"） |
| 自动化的三个难点 | ① 各可编辑组件构成的动作空间是异构的；② 海量轨迹把可用信号淹没了；③ 编辑效果难以归因 | p.1，Abstract（"…heterogeneous action space…voluminous trajectories…edits whose effect is hard to attribute"）；p.2，§1 第 3 段（"long, unstructured trajectories yield little actionable signal, and tightly coupled harness frameworks make edits beyond the prompt error-prone"） |
| 作者的核心判断 | 瓶颈在**可观测性（observability）**，不在智能体能力 | p.2，§1 第 4 段（"Our central insight is that this question is bottlenecked by observability, not by agent capability"） |
| 实验层面的三个 RQ | RQ1：为什么选 agentic harness engineering，而不是人工 harness 或其他自动化方法？RQ2：AHE 是否过拟合于优化目标？RQ3：AHE 的增益从哪里来？其自我归因有多可靠？ | p.6，§4 开头 "Research Questions" 框 |

## 2. 研究对象

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 研究对象 | coding agent 的 **harness**，即模型之外、可编辑的组件集合（system prompt、工具、middleware 等） | p.1，§1 第 1 段（"This collection of model-external, editable components is collectively referred to as the agent's harness"） |
| harness 载体 | NexAU 框架。它把 7 类正交组件暴露为固定挂载点上的文件：system prompt、tool description、tool implementation、middleware、skill、sub-agent configuration、long-term memory | p.4，§3.1 第 1 段（"We instantiate the harness H on the NexAU framework…seven orthogonal component types…"） |
| 种子 harness（NexAU₀） | 只有一个 shell 执行工具，没有 middleware、skills、sub-agents（附录补充：也没有 long-term memory） | p.4，§3.1 第 3 段（"Our seed harness H0 is deliberately minimal…"）；p.15，Appendix A "Seed agent." 段 |
| 研究中保持不变的部分 | 基座模型固定，只编辑显式 harness | p.3，§3 Method 第 1 段（"with the base model held fixed and only the explicit harness edited"）；p.9，§5 Conclusion |
| 实际被进化并做消融的组件 | long-term memory、tools、middleware、system prompt 四层 | p.8，§4.4.1 第 1 段（"isolating one of the four evolved layers…"） |
| 研究对象中未涉及的 | 模型侧训练（作者把 harness 进化定位为与模型侧训练互补的轴） | p.9，§5 Conclusion 末句（"We see harness-level evolution as a complementary axis to model-side training"） |

## 3. 数据来源

| 要点 | 内容 | 原文依据 |
|---|---|---|
| 进化 / 主评测数据 | Terminal-Bench 2 全部 89 个任务（4 easy / 55 medium / 30 hard），单任务超时延长到 1 小时 | p.6，§4.1 "Evaluation." 段；难度划分来自官方 leaderboard：p.15，Appendix A "Terminal-bench difficulty labels." 段 |
| 跨基准迁移数据 | SWE-bench-verified，覆盖 7 个仓库的 500 个任务 | p.6，§4.1 "Evaluation." 段；逐仓库任务数见 p.7，Table 2（django 231、sympy 75、sphinx-doc 44、matplotlib 34、scikit-learn 32、pydata 22、astropy 22） |
| 进化所用的经验数据 | 每个任务做 k 次 rollout 得到的轨迹（原始轨迹约 1000 万 tokens 量级，经蒸馏后约 1 万 tokens 量级），由 Agent Debugger 分析成逐任务报告和基准级 overview | p.4，§3.2 第 1 段（"We generate k traces for each task…"）；p.4，Figure 2 图中标注 "~10M tokens" / "~10K tokens"；p.5，§3.2 续段（"Finally, a benchmark-level overview is aggregated…"） |
| 外部参考知识 | 一次性 explore agent 从 NexAU 源码和公开 coding-agent 资料中生成少量可复用 skills | p.5，§3.3 末段（"A one-shot explore agent (Appendix B.3)…"）；Prompt 原文见 p.21–26，Appendix B.3 |
| 运行基础设施 | Harbor 负责分发与验证，E2B 远程沙箱，InMemoryTracer 记录轨迹并镜像到 Langfuse | p.15，Appendix A "Runtime infrastructure." 段 |
| 代码 / 配置公开位置 | github.com/china-qijizhifeng/agentic-harness-engineering | p.16，Appendix B 开头段；p.1 标题区 |
| 数据的版权 / 获取许可 / 数据泄漏检查 | 未说明 | — |

## 4. 核心变量

### 4.1 自变量（被操纵的因素）

| 变量 | 取值 / 内容 | 原文依据 |
|---|---|---|
| harness 配置 / 进化方法 | 人工 harness：OpenCode、Terminus-2、Codex；自进化方法：ACE、TF-GRPO、AHE；共同种子：NexAU₀ | p.6，§4.2 第 1–2 段；p.6，Table 1 |
| 进化迭代轮次 | 1–10 轮 | p.2，Figure 1；p.6，§4.2 第 1 段（"a single AHE campaign of ten iterations"） |
| 基座模型（迁移实验） | GPT-5.4 的 medium / high / xhigh 三档推理，qwen-3.6-plus，gemini-3.1-flash-lite-preview，deepseek-v4-flash | p.6，§4.1 "Models." 段 |
| 单组件替换（消融） | 只把 AHE 的 memory / tool / middleware / system_prompt 中的一个换入 NexAU₀ | p.8，§4.4.1 第 1 段；p.8，Table 3 |
| 任务面 | Terminal-Bench 2 vs. SWE-bench-verified | p.7，§4.3 第 1 段 |

### 4.2 因变量（结果指标）

| 变量 | 定义 | 原文依据 |
|---|---|---|
| pass@1 | 每个任务 k 次 rollout 的二值成功率均值，即 pass@1 = (1/(k·\|D\|)) Σᵢ Σⱼ r_{i,j}；基础设施中断或超时的试次计为失败 | p.6，§4.1 "Evaluation." 段；p.15，Appendix A "pass@1." 段，公式 (1) |
| tokens/trial（Tokens k） | 单试次所有 LLM 调用的 prompt + completion token 总和的均值，以千为单位；中断的试次不计入 | p.6，§4.1 "Evaluation." 段；p.16，Appendix A "Token cost and Succ/Mtok." 段 |
| Succ/Mtok | pass@1 × 10⁶ / 每试次平均 token，即每百万 token 的期望成功数 | p.16，公式 (2)；p.17，Table 5 |
| 自我归因的 precision / recall | 把第 N−1 轮的预测（predicted fixes / risk tasks）与第 N 轮的真实翻转对比，在 89 个任务上分别对 fix 和 regression 计算 | p.9，§4.4.2 第 1 段（"Each evolution round, our evolve model produces a change manifest…"） |

### 4.3 控制变量（保持不变的条件）

| 变量 | 内容 | 原文依据 |
|---|---|---|
| 三个角色共用一个模型 | Code Agent、Agent Debugger、Evolve Agent 都使用 GPT-5.4（high），从而把增益归到 harness 编辑，而不是分析器或编辑器的能力 | p.2，Figure 1 标题；p.6，§4.1 "Models." 段 |
| 编辑权限 | Evolve Agent 只能写 harness workspace；runs 目录、tracer、verifier、LLM 配置只读；种子 system prompt 不可删除 | p.5，§3.3 第 2 段（"The first constraint is controllability…"） |
| 共同起点 | ACE、TF-GRPO、AHE 都从 NexAU₀ 出发 | p.6，§4.2 第 1 段；p.7，Table 2 标题 |
| 其他超参数 | 参见 Table 4（T=10，k=2，并发 rollout 96，E2B 沙箱，沙箱寿命 3600 s 等）。抽取文本中该表的分块已错位，各 agent 的具体超参数归属这里不逐一列出 | p.16，Table 4 |

## 5. 研究方法

### 5.1 总体框架：三根可观测性支柱组成的闭环

| 支柱 | 实现 | 作用 | 原文依据 |
|---|---|---|---|
| ① 组件可观测性 | NexAU 解耦的文件级 harness，每个逻辑编辑对应一次 git commit | 每种失败模式只映射到一类组件，动作空间清晰，可按文件回滚 | p.4，§3.1 第 1–2 段（"This decoupling is what realizes component observability…"） |
| ② 经验可观测性 | Agent Debugger 把轨迹组织成可导航的文件环境，生成逐任务根因报告和基准级 overview，同时保留原始轨迹与清洗后的轨迹，支持渐进式披露 | 让进化者读结构化的根因，而不是原始日志 | p.4，§3.2 第 1 段；p.5，§3.2 续段（"In addition to these reports…progressive disclosure"） |
| ③ 决策可观测性 | change manifest：每个编辑都写明失败证据、根因、修复方式、预测修复的任务和可能回归的任务 | 每个编辑都成为可证伪的契约，下一轮验证；无效的编辑按文件回滚 | p.5，§3.3 第 3 段（"The second constraint is that every change is evidence-driven…"）；manifest 的 JSON 结构见 p.20–21，Appendix B.2 |

### 5.2 外循环流程（Algorithm 1）

1. **ROLLOUT**：用当前 harness 对每个任务做 k 次 rollout
2. **CLEAN**：把轨迹规范化
3. **ATTRIBUTE + ROLLBACK**（t ≥ 2）：把上一轮 manifest 的预测与本轮的任务级变化求交，得到每个编辑的判定，并回滚被否决的编辑
4. **AGENTDEBUGGER**：分层蒸馏
5. **EVOLVE**：修改 workspace，生成新的 manifest
6. **COMMIT**：在 git 中打上本轮标签；若 pass@1 超过历史最好，则更新 H_best

依据：p.5，Algorithm 1 及其后第 1 段（"Algorithm 1 composes the three substrates…"）。另外：
- 归因先于蒸馏执行，所以判定结果会进入证据语料（同段）。
- k ≥ 2 的作用是让每个任务都有通过率信号，并且部分通过的任务可以拿来做对比诊断（同段）。

### 5.3 关键设计细节

| 设计 | 内容 | 原文依据 |
|---|---|---|
| 最小种子 | 若种子已经针对目标基准调过，就无法判断增益来自循环还是来自种子。最小种子迫使每个新增组件都要靠实测证明自己的价值 | p.4，§3.1 第 3 段 |
| 防捷径约束 | 防止自修改者走捷径，例如关闭 verifier、换模型、提高推理预算 | p.5，§3.3 第 2 段 |
| Evolve Agent 规则 | 禁止加入任务特定逻辑或硬编码答案；禁止从轨迹反推测试用例；禁止修改 llm_config；同一失败类在某一组件层级修复 2 轮以上仍无效时，回滚并换层级（ROLLBACK + PIVOT） | p.19，Appendix B.2 "Choosing the Right Component Level" 段；p.20，"Analysis Approach" 第 9 条；p.21，"Safety Constraints" |
| 优化目标 | 唯一目标是 pass@1；k>1 时重点用部分通过的任务，把 pass@k 的成功变成稳定的 pass@1 | p.20，Appendix B.2 "Analysis Approach" 后两段 |
| 最终报告的配置 | 10 轮中 pass@1 最好的那一版，记为 AHE；全过程约 32 小时 | p.6，§4.2 第 1 段 |

### 5.4 实验设计

| 实验 | 设计 | 原文依据 |
|---|---|---|
| RQ1 主实验 | 在 Terminal-Bench 2 上把 AHE 与 3 个人工 harness、2 个自进化基线对比，并按难度分层 | p.6，§4.2；Table 1 |
| RQ2 跨基准迁移 | 冻结在 TB2 上进化出的 harness，不再进化，直接在 SWE-bench-verified 上评测 | p.7，§4.3 第 1–2 段；Table 2 |
| RQ2 跨模型迁移 | 在 5 个替代基座上分别配对评测 NexAU₀ 与 AHE | p.7，§4.3 "Cross-model transfer." 段；p.8，Figure 3 |
| RQ3a 组件消融 | 每次只把一个 AHE 组件换入种子 | p.8，§4.4.1；Table 3 |
| RQ3b 自我归因可靠性 | 对 9 个评估轮次计算 fix / regression 的 precision 与 recall，并与随机预测基线比较 | p.9，§4.4.2；Figure 4；逐轮结果见 p.35，Appendix D |
| 定性案例 | 4 条轨迹（db-wal-recovery、path-tracing、mcmc-sampling-stan、configure-git-webserver），对应 4 个成功轮次中的 8 个变更 | p.26–34，Appendix C |
| 显著性检验 / 置信区间 / 多次独立进化 | 未说明。原文只说进行了"a single AHE campaign"（p.6，§4.2 第 1 段） | — |
| 各表、各图使用的 rollout 数 k | 未说明。附录称"each table states it explicitly"（p.15，Appendix A "pass@1." 段），但抽取到的 Table 1/2/3 标题中没有写 k；Table 4 给出的进化参考运行 k=2 | — |
| 基线（ACE、TF-GRPO）的超参数 / 迭代轮次 | 未说明 | — |
| 人工 harness（OpenCode / Terminus-2 / Codex）在 Table 1 中所用的基座模型 | 未说明 | — |

## 6. 主要结论

| # | 结论 | 关键数据 | 原文依据 |
|---|---|---|---|
| 1 | **AHE 优于人工 harness 和自进化基线** | TB2 pass@1 从 69.7%（NexAU₀）提升到 77.0%；Codex 71.9%，Terminus-2 62.9%，OpenCode 47.2%，ACE 68.9%，TF-GRPO 72.3% | p.1，Abstract；p.6，§4.2 第 2 段（"AHE outperforms both human-designed and self-evolve baselines"）；p.6，Table 1 |
| 1a | 按难度看，唯一的例外是 Hard 档：AHE 53.3%，略低于 Codex 的 56.7%。作者认为原因是长程任务上组件之间相互干扰，并非缺少能力；单独把 memory 换入种子，Hard 档就能超过 Codex | Hard：AHE 53.3%，Codex 56.7%，memory-only 63.3% | p.6，§4.2 第 2 段后半；Table 1；Table 3 |
| 2 | **只改 prompt 的自进化方法漏掉了真正带来增益的组件**：ACE 和 TF-GRPO 不改外围 scaffolding，而 AHE 的增益集中在 tools、middleware、memory | — | p.6–7，§4.2 第 3 段（"Prompt-only self-evolution misses the components…"） |
| 3 | **跨基准迁移**：在 SWE-bench-verified 上 AHE 的总成功率最高，token 也更少。ACE 和 TF-GRPO 的成功率低于种子，token 却多 11%–29% | 成功率：AHE 75.6%，NexAU₀ 75.2%，ACE 74.6%，TF-GRPO 74.2%；token：AHE 461k，NexAU₀ 526k，ACE 679k，TF-GRPO 582k，AHE 相比 ACE / TF-GRPO / 种子分别少 32% / 21% / 12% | p.7，§4.3 "Cross-benchmark transfer." 两段；Table 2（All 行） |
| 3a | 相对种子的增益集中在 django 和 sphinx-doc；最小的三个仓库（scikit-learn、pydata、astropy）有小幅回归，作者认为这符合"小仓库的 pass@1 方差大于单仓库增益"的解释 | django 81.0% vs 79.2%；sphinx-doc 70.5% vs 68.2% | p.7，§4.3 第 3 段（"AHE instead achieves the highest aggregate…"）；Table 2 |
| 3b | 按 Succ/Mtok 计，AHE 总体最高 | All：AHE 1.64，NexAU₀ 1.43，TF-GRPO 1.27，ACE 1.10 | p.17，Table 5 |
| 4 | **跨模型迁移**：5 个替代基座上的增益全部为正；跨家族的增益大于同家族 | deepseek-v4-flash +10.1 pp（51.7→61.8），qwen-3.6-plus +6.3 pp（56.2→62.5），gemini-3.1-flash-lite +5.1 pp（36.5→41.6），GPT-5.4 medium / xhigh 各 +2.3 pp | p.7，§4.3 "Cross-model transfer." 及下一段；p.8，Figure 3 及其后第 1 段（"The headline finding is that all five gains land positive"） |
| 4a | 作者的解释：离饱和越远的基座，越依赖 AHE 固化在 tools、middleware、memory 中的协调模式；同家族内增益不单调（medium +2.3，high +7.3，xhigh +2.3），与步数预算和超时是按 GPT-5.4 high 拟合的有关 | — | p.7，§4.3 倒数第 2 段与最后一段（"Within one family the profile is non-monotone…"） |
| 5 | **增益来自 tools、middleware、memory，不来自 system prompt**：三个单组件变体都超过种子，只有 system prompt 单独换入时退步 | memory-only 75.3%，tool-only 73.0%，middleware-only 71.9%，system_prompt-only 67.4%（−2.3 pp），AHE full 77.0% | p.1，Abstract 末段；p.8，§4.4.1 第 1 段；Table 3 |
| 5a | 各组件负责不同的失败面：memory 增加 12 条边界情形教训；tool 变成 1364 行的 shell；middleware 增加 finish-hook；system prompt 是 79 行通用规范，能否执行取决于另外三个组件 | — | p.8，§4.4.1 "Each component owns a different failure surface." 段 |
| 6 | **组件之间是非加性的，限制了总增益**：三个单组件正增益之和为 +11.1 pp，而完整 AHE 只有 +7.3 pp；Hard 档上 memory-only 超过完整 AHE（多个组件都推动收尾式验证，造成冗余复查） | — | p.9，§4.4.1 "Components interact non-additively…" 段；p.3，贡献第 3 条 |
| 7 | **自我归因对修复较可靠，对回归基本看不见** | fix：precision 33.7%，recall 51.4%，约为随机基线（6.5% / 10.6%）的 5 倍；regression：precision 11.8%，recall 11.1%，只有随机基线（5.6% / 5.4%）的约 2 倍 | p.9，§4.4.2 "Evidence-driven targeting." 与 "Regression blindness." 两段；Figure 4 |
| 7a | 逐轮看：9 轮共发出 43 条不重复的回归预测，命中 5 条，累计 P = 11.6%；实际发生了 40 个未被预见的回归，累计 R = 11.1%。（正文报告的 11.8% 是跨轮均值，与此处的累计口径不同） | — | p.35，Appendix D 第 2 段 |
| 8 | 回归不可预见是进化曲线出现非单调台阶的原因；提高对回归的预见能力是未来自进化循环最明确的方向 | — | p.9，§4.4.2 "Regression blindness." 段；p.3，贡献第 3 条 |
| 9 | 案例层面：AHE 能把单任务症状转成可复用的 harness 规则。例如 chg-1 的 manifest 没有预测 db-wal-recovery，但这条通用规则仍然修复了它 | db-wal-recovery 从 1/2 变为 2/2，并在之后各轮保持 | p.27，Appendix C.1.1 最后一段（"The Evolve Agent's predicted_fixes field for chg-1 did not list db-wal-recovery…"） |

## 7. 适用条件

以下条件都由原文给出或由原文实验设置直接限定。

| 条件 | 内容 | 原文依据 |
|---|---|---|
| 任务类型 | 长程软件工程 / 终端 coding agent 任务；验证覆盖 Terminal-Bench 2 和 SWE-bench-verified | p.1，§1 第 1 段；p.6，§4.1；p.10，Limitations "Benchmark scope." |
| 需要可解耦的文件级 harness | 组件以文件形式暴露、彼此松耦合，并有 git 版本管理（本文用的是 NexAU） | p.4，§3.1 第 1–2 段 |
| 需要可自动验证的任务级结果 | 依赖 verifier 给出逐任务 pass/fail，并据此做下一轮归因 | p.5，§3.3 第 3 段；p.15，Appendix A "Runtime infrastructure." 段 |
| 需要每任务多次 rollout | k ≥ 2，让每个任务都有通过率信号 | p.5，Algorithm 1 后第 1 段 |
| 基座模型固定 | 进化过程中不改模型，也不改 LLM 配置 | p.3，§3 第 1 段；p.5，§3.3 第 2 段 |
| 运行时操作点 | 步数预算和单任务超时按 GPT-5.4 high 拟合；换基座时增益与这个操作点有关 | p.7，§4.3 最后一段；p.10，Limitations "Evolution operating point." |
| 迁移效果的方向 | 跨模型时，离饱和越远的基座增益越大 | p.2，§1 第 5 段；p.7，§4.3 "Cross-model transfer." 段 |
| 系统定位 | 受控研究原型，不是成熟的自主自我改进系统 | p.10，Limitations "Self-modification governance." |
| 计算成本 / 硬件 / API 预算要求 | 未说明（只给出 10 轮约 32 小时，以及 Table 4 中的并发参数） | p.6，§4.2 第 1 段；p.16，Table 4 |
| 是否适用于非编码类 agent | 未说明 | — |

## 8. 研究局限

### 8.1 作者在 Limitations 中明确列出的

| 局限 | 内容 | 原文依据 |
|---|---|---|
| 总体定性 | 研究处于"有前景但高方差"的设置中，结论范围应据此理解 | p.9，Limitations 首段 |
| 基准范围 | 只在 TB2 上进化、在 SWE-bench-verified 上测迁移；更多编程语言、仓库级部署、人在回路的工作流都没有测试 | p.10，Limitations "Benchmark scope." 段 |
| 进化操作点耦合 | 步数预算和超时按 GPT-5.4 high 拟合，所以跨模型迁移的数字混合了 harness 可移植性与操作点耦合两种因素；要拆开它们，需要在多个操作点下重新跑循环 | p.10，Limitations "Evolution operating point." 段；另见 p.8，Figure 3 后第 1 段（作者把它视为"generalization hazard"） |
| 自修改治理 | 没有提供完整的护栏体系；长程的 harness 清理和更强的防滥用机制仍不完整 | p.10，Limitations "Self-modification governance." 段 |

### 8.2 作者在分析部分报告的方法局限

| 局限 | 内容 | 原文依据 |
|---|---|---|
| 组件非加性交互 | 叠加多个有效编辑会限制总增益；进化方向受 55 个 Medium 任务主导，最终偏向 Medium 的取舍，牺牲了部分 Hard 档上的 memory 效果；面向交互的进化留作未来工作 | p.9，§4.4.1 "Components interact non-additively…" 段；p.3，贡献第 3 条 |
| 回归盲区 | 自我归因对回归的 precision 和 recall 都只有约 11%，大部分回归无法预见 | p.9，§4.4.2 "Regression blindness." 段；p.35，Appendix D |
| Hard 档未超过 Codex | 完整 AHE 在 Hard 档上为 53.3%，Codex 为 56.7% | p.6，§4.2 第 2 段；Table 1 |
| 小仓库回归 | 在 SWE-bench-verified 的三个最小仓库上有小幅回归 | p.7，§4.3 第 3 段；Table 2 |
| 同家族迁移增益不单调 | xhigh 档有更多试次超时，被计为失败，因而压低了增益 | p.7，§4.3 最后一段 |
| 护栏规则可能被绕过 | 案例中，iteration-5 的 publish-state guard 可以被 override token 绕过，到 iteration 8 才改为硬拦截 | p.30–31，Appendix C.1.4；p.33，Appendix C.2.4 |

### 8.3 原文未说明的事项

- 结果的统计显著性、置信区间、多次独立进化的方差：未说明（主实验只有"a single AHE campaign"，p.6，§4.2 第 1 段）
- 进化所用的任务集与最终报告所用的任务集是否分开：未说明（原文写明"drive evolution on the full 89 tasks of Terminal-Bench 2"，p.6，§4.1 "Evaluation." 段，但没有讨论这一点）
- 社会负面影响：Appendix E 写"Negative societal impacts…are discussed in §5"（p.35），但 §5 Conclusion（p.9）和 Limitations（p.9–10）中没有专门讨论社会负面影响的段落，具体内容未说明
