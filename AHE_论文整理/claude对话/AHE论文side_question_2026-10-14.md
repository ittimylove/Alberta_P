# AHE 论文 side question 记录

- **日期**：2026-10-14
- **论文**：Lin et al., *Agentic Harness Engineering: Observability-Driven Automatic Evolution of Coding-Agent Harnesses*（arXiv:2604.25850v4）
- **主对话记录**：[AHE论文精读对话_2026-10-14.md](AHE论文精读对话_2026-10-14.md)
- **说明**：以下是精读过程中的旁路提问（side question）。这些回答只依据论文内容；凡标"推断""建议""个人读法"的，都是原文之外的分析。页码指 PDF 页码。

## 目录

**框架与概念**
1. [NexAU 是什么框架](#1-nexau-是什么框架)
2. [chg-2 是代理 agent 吗](#2-chg-2-是代理-agent-吗)
3. [Raw trace 和 History 的区别](#3-raw-trace-和-history-的区别)
4. [rollouts per task k 是什么意思](#4-rollouts-per-task-k-是什么意思)
5. [manifest 预测是什么](#5-manifest-预测是什么)
6. [ROLLBACK + PIVOT 是七层组件循环吗](#6-rollback--pivot-是七层组件循环吗)
7. [Algorithm 1 中什么时候 rollback](#7-algorithm-1-中什么时候-rollback)
8. [TB2 是什么任务](#8-tb2-是什么任务)

**实验结果的解读**

9. [作者如何解释 Hard 档低于 Codex](#9-作者如何解释-hard-档低于-codex)
10. [如何定位组件冲突](#10-如何定位组件冲突)
11. [Figure 3：为什么 GPT-5.4 high 的提升比 medium 高](#11-figure-3为什么-gpt-54-high-的提升比-medium-高)
12. [AHE 为什么更省 token](#12-ahe-为什么更省-token)
13. [举例：为什么比种子还省 token；为什么有些场景不如种子](#13-举例为什么比种子还省-token为什么有些场景不如种子)
14. [AHE 是在 TB2 上进化的吗；跨场景实验说明什么](#14-ahe-是在-tb2-上进化的吗跨场景实验说明什么)
15. ["只加一个"和"去掉一个"消融的区别](#15-只加一个和去掉一个消融的区别)
16. [RQ3b：修复预测、回归预测、Precision、Recall](#16-rq3b修复预测回归预测precisionrecall)
17. [Precision / Recall 的含义；随机基线怎么实现](#17-precision--recall-的含义随机基线怎么实现)
18. ["抽取同样数量的随机预测"是什么意思](#18-抽取同样数量的随机预测是什么意思)

**附录**
- [附录 A：候选问题草稿（完整版）](#附录-a候选问题草稿完整版)
- [附录 B：候选问题：AHE 组件冲突的定位（最终保存版）](#附录-b候选问题ahe-组件冲突的定位最终保存版)

---

## 1. NexAU 是什么框架

**问**：NexAU 这是什么框架，帮我简单介绍一下。

**答**：以下只根据论文里提到的内容介绍。论文之外的信息，比如框架的完整功能、版本、社区情况，没有依据。

**基本信息**
- **名字**：NexAU，"AU" 是 Agent Universe 的缩写。
- **定位**：通用的智能体框架，用来构建能调用工具的智能体。参考文献 [23] 的原话是 "a general-purpose agent framework for building intelligent agents with tool capabilities"。
- **出处**：由 Nex-AGI 开发，代码在 github.com/nex-agi/NexAU（参考文献 [23]）。相关论文是 Nex-N1（参考文献 [37]）。
- **与作者的关系**：[37] 的作者名单里有本文的几位作者，例如 Jiahang Lin、Lizhi Lin、Zhenhua Han、Tao Gui。所以 NexAU 是作者团队自己的框架，不是第三方工具。

**论文为什么选它（p.4，§3.1）**

NexAU 把 harness 拆成 7 类彼此独立的组件，每类组件都是固定位置上的文件：

| 组件 | 文件位置（p.18–19，Appendix B.2） |
|---|---|
| System prompt | `systemprompt.md` |
| Tool description | `tool_descriptions/*.tool.yaml` |
| Tool implementation | `tools/`（Python） |
| Middleware | `middleware/`（Python） |
| Skill | `skills/{name}/SKILL.md` |
| Sub-agent | `sub_agents/{name}/agent.yaml` |
| Long-term memory | `LongTermMEMORY.md` |

这些组件松耦合：加一个 middleware 不需要改 system prompt，加一个 skill 也不需要动工具。AHE 的"组件可观测性"就建立在这一点上，因为每个修改都落在单个文件里，可以做 git diff，也可以单独回滚。

**几个工作机制（据 Appendix B.2 / B.3）**
- **总配置文件 `code_agent.yaml`**：新建的工具、middleware、skill、sub-agent 都必须在这里注册才会生效，只创建文件不够。
- **代码加载**：运行时把配置目录加入 `sys.path`。例如 `import: middleware.xxx:ClassName` 会解析到 `workspace/middleware/xxx.py`。
- **工具**：用 YAML 描述，再通过 `binding` 绑定到一个 Python 函数。
- **Middleware**：通过 `before_model`、`after_tool` 等 hook 挂进智能体循环。
- **Sub-agent**：框架会自动注入 `RecallSubAgent` 工具。
- **模型配置**：统一读取 `LLM_API_KEY`、`LLM_BASE_URL`、`LLM_MODEL` 三个环境变量，所以换基座模型时不需要改组件代码。

**在 AHE 实验中的角色**
- 实验里所有 coding agent 都由 NexAU 实例化（p.15，Appendix A）。
- 进化起点 NexAU₀ 就是基于这个框架、只带一个 bash 工具的最小配置。
- Evolve Agent 事先不了解 NexAU，所以第 1 轮会运行一个源码探索智能体，读 NexAU 源码，写出一份开发指南 skill 给 Evolve Agent 参考（p.21，Appendix B.3.1）。

---

## 2. chg-2 是代理 agent 吗

**问**：chg-2 是代理 agent 吗，是负责修改 harness 组件的吗？

**答**：不是。chg-2 不是智能体，而是一次修改的编号。真正负责修改 harness 组件的是 Evolve Agent。

**chg-N 是什么**
- Evolve Agent 每做一个逻辑修改，就提交一次 git commit。commit message 的格式规定为 `chg-N: <简短描述>`（p.20，Appendix B.2 "Git Commits"）。
- 同一个编号也是 change manifest（`change_manifest.json`）里这条修改记录的 id，例如 `"id": "chg-1"`（p.21）。
- 所以 chg-1、chg-2 只是"本轮第 1 个修改""本轮第 2 个修改"的标签，类似工单号。

**编号每轮各自计数**

同一个编号在不同轮次指的是完全不同的修改，看 chg-2 时要连同轮次一起看：

| 轮次 | chg-2 的内容 | 修改层级 | 依据 |
|---|---|---|---|
| 第 1 轮写，第 2 轮生效 | 给 shell 工具加可调的 `timeout_ms` 超时参数 | tool | p.31，Figure 8 |
| 第 6 轮 | 新增 ExecutionRiskHintsMiddleware，监控 7 种风险模式 | middleware | p.32–33，Figure 10 |
| 第 8 轮 | 让风险提醒在下一轮以 FRAMEWORK reminder 的形式出现，并增加两个按契约触发的检查 | middleware | p.33–34，Figure 11 |

第 5 轮的两个修改编号是 chg-7 和 chg-8（p.32，Figure 9），可见编号也不总是从 1 开始。

**各部分的分工**
- **Evolve Agent**：读证据，决定改什么，动手修改 harness 文件，并写 manifest。它是做修改的主体。
- **chg-N**：Evolve Agent 产出的单个修改，也是下一轮要检验和归因的单位：看预测是否兑现，不兑现就回滚。
- **Coding Agent**：用 harness 去做任务的智能体，被修改的是它的 harness。
- **Agent Debugger**：分析轨迹、产出证据的智能体，它不修改 harness。

---

## 3. Raw trace 和 History 的区别

**问**：这里的 Raw trace、History 有什么区别呢？

**答**：两者记录的东西完全不同。Raw trace 记录 Coding Agent 这一轮是怎么做题的，History 记录 harness 这几轮是怎么被改的。

| | Raw trace | History |
|---|---|---|
| 记录的是什么 | Coding Agent 做任务时的原始执行轨迹：每一步的模型消息、调用了什么命令、命令输出了什么 | 进化过程本身：历轮 Evolve Agent 改了哪些组件、为什么改、效果如何、哪些被回滚 |
| 谁产生的 | Coding Agent 在环境中跑任务时产生 | Evolve Agent 的修改（git commit），以及各轮的评测和归因结果 |
| 时间范围 | 当前这一轮的 rollout（89 个任务 × k 次） | 跨越所有已完成的轮次，逐轮累积 |
| 规模 | 很大，图中标注约 1000 万 tokens | 论文没有给出规模 |
| 在流程里怎么用 | 先交给 Agent Debugger 蒸馏成报告和 overview；Evolve Agent 需要核实时也可以直接查看（p.5，§3.2） | 直接给 Evolve Agent，让它知道哪些试过、哪些有效、哪些失败，避免重复犯错，也便于决定保留、改进，还是回滚后换一层组件来修 |
| 在图中的位置 | Coding Agent 右边的 LOG 图标 | 左下角的分支图标，一条线直接连到 Evolve Agent |
| 具体文件（Appendix B.2 目录） | `runs/iteration_NNN/input/benchmark/…/agent/nexau_in_memory_tracer.cleaned.json` 等 | 很可能是 workspace 的 git 提交历史（每个 chg-N 一次 commit，每轮打标签）和 `evolution_history.md`（目录注释为 "Cumulative history of all iterations"） |

**需要说明**：Figure 2 只画了 "History" 这个图标，正文没有专门解释它。把它对应到 git 历史和 `evolution_history.md`，是根据 Algorithm 1 的 COMMIT 步骤和附录 B.2 的目录结构做的推断，不是原文的直接定义。

**打个比方**：Raw trace 像这次考试每道题的完整草稿纸；History 像过去几次调整学习方法的记录本，写着改了什么、有没有用。

---

## 4. rollouts per task k 是什么意思

**问**：文中的 rollouts per task k 是什么意思？

**答**：**k 是"每个任务重复跑几次"**：用同一个 harness，把同一个任务从头完整执行 k 遍，每一遍叫一次 rollout。

**什么是一次 rollout**

Coding Agent 在一个全新的沙箱里，从拿到任务开始一步步执行命令，直到结束（提交、超时或出错），最后由 verifier 判定成功（记 1）或失败（记 0）。每次 rollout 都在新的 E2B 沙箱里运行，几次 rollout 之间互不影响（p.15，Appendix A）。

模型输出有随机性（Table 4 中 Code Agent 的 temperature 为 0.7），所以同一任务跑几次，结果可能不同。

**论文中的取值**
- 进化用的参考运行中 k = 2（p.16，Table 4 "Rollouts per task k = 2"）。所以一轮评测是 89 个任务 × 2 次 = 178 次 rollout。
- Algorithm 1 第 3 行写作 `ROLLOUT(M, H_{t-1}, D, k)`，即 "k rollouts per task"（p.5）。
- 其他实验用的 k，原文说"每张表都会写明"（p.15），但抽取到的表格标题里没有看到，所以标为"未说明"。

**k 在指标中的作用**

pass@1 就是所有 rollout 的平均成功率（p.15，公式 (1)）：

```
pass@1 = (1 / (k·|D|)) × Σ_任务 Σ_rollout r_{i,j}
```

例如 k=2 时，某任务两次都成功贡献 1，一次成功贡献 0.5，两次都失败贡献 0。案例中写 "1/2"、"0/2"、"2/2" 就是这个意思。

**为什么要求 k ≥ 2（p.5，Algorithm 1 后第 1 段）**
1. 让每个任务都有通过率信号，pass@1 更稳定。只跑一次时，结果受随机性影响很大。
2. 便于对比诊断。有些任务一次成功、一次失败（"partial-pass"）。Evolve Agent 的 prompt 说这类任务最有价值：对比同一任务成功和失败的两条轨迹，找到分歧点，就能把成功的做法变成稳定的默认行为（p.20，Appendix B.2）。

案例 db-wal-recovery 就是这样：种子在这个任务上是 1/2，两条 rollout 的前三步相同，从某一步开始分道扬镳（p.27–28，Figure 6）。

---

## 5. manifest 预测是什么

**问**：manifest 预测是什么东西？

**答**：manifest 预测是 Evolve Agent 每做一次修改时，在 change manifest（`change_manifest.json`）里事先写下的预期：**这个修改会修好哪些任务，可能把哪些任务改坏**。

**它长什么样（p.21，Appendix B.2）**

每个修改（chg-N）在 manifest 里对应一条记录：

```json
{
  "id": "chg-1",
  "type": "new|improvement|rollback",
  "description": "改了什么、为什么改",
  "files": ["改动的文件"],
  "failure_pattern": "针对的失败模式",
  "predicted_fixes": ["task-a", "task-b"],
  "risk_tasks": ["task-c"],
  "constraint_level": "middleware|tool_impl|tool_desc|skill|prompt",
  "why_this_component": "为什么选这一层组件"
}
```

其中 `predicted_fixes`（预测能修好的任务）和 `risk_tasks`（预测可能被改坏的任务）就是"预测"。

**它的作用：让每个修改都能被检验**
1. **这一轮**：Evolve Agent 改完 harness，写下预测。
2. **下一轮**：harness 跑完任务后，系统把预测和实际结果对比（Algorithm 1 的 ATTRIBUTE 步骤）：
   - 预测修好的任务，是否真的从失败变成成功？
   - 预测有风险的任务，是否真的从成功变成失败？
3. 据此对每个修改给出判定。预测的修复没有出现，就按文件回滚这个修改（ROLLBACK）。

论文把这称为"可证伪的契约"（falsifiable contract）：修改不再只靠 Evolve Agent 自己说理由，而是用下一轮的实测结果来验证（p.5，§3.3 第 3 段）。这就是图中底部"III. 决策可观测性"的核心。

**例子**

第 5 轮的 chg-7 和 chg-8（加入 publish-state 规则和 shell 守卫）预测能修好 path-tracing、polyglot-rust-c、configure-git-webserver、large-scale-text-editing 这 4 个任务。下一轮实测：path-tracing、polyglot-rust-c、large-scale-text-editing 修好了，configure-git-webserver 只部分恢复（p.29，p.32）。

**预测准不准（p.9，§4.4.2）**
- 修复预测：precision 33.7%，recall 51.4%，约为随机猜测的 5 倍，比较靠谱。
- 回归预测：precision 11.8%，recall 11.1%，只有随机猜测的约 2 倍，大部分改坏的任务都没有预测到。
- 也有没写进预测却修好了的情况：第 1 轮的 chg-1 没有把 db-wal-recovery 列进 `predicted_fixes`，但它的通用规则还是修好了这个任务（p.27）。

---

## 6. ROLLBACK + PIVOT 是七层组件循环吗

**问**：ROLLBACK + PIVOT：在这一层组件上修不好，回滚后换一层组件重新尝试。这个是什么意思？我没有看到在七层组件上都循环啊？

**答**：你的观察是对的，**AHE 并不会把七层组件挨个循环试一遍**。ROLLBACK + PIVOT 不是算法里的循环结构，而是写在 Evolve Agent 提示词里的一条经验规则，由 Evolve Agent 自己判断什么时候用、换到哪一层。

**原文出处**

1. Appendix B.2 "Choosing the Right Component Level"（p.19）：
   - 对每种失败模式，修之前要考虑所有组件类型（包括新建组件），再决定修在哪一层。
   - 反模式（Anti-pattern）：如果同一类失败在某一层修了 2 轮以上仍然存在，说明这一层可能选错了，应回滚这个无效的修改，换一个组件层级重新处理。
2. Appendix B.2 "Analysis Approach" 第 8、9 条（p.20）：
   - 第 8 条是架构检查：如果之前在某一层修没成功，就换一层。
   - 第 9 条：从第 2 轮起，根据归因报告，对上一轮的每个修改三选一：
     - **KEEP**：有效，保留；
     - **IMPROVE**：方向对，继续细化；
     - **ROLLBACK + PIVOT**：在这一层不奏效，回滚后从另一个组件层级重新处理同一失败模式。

**换句话说**
- **ROLLBACK**：撤销那个没用的修改。这一步在 Algorithm 1 里有对应（第 7 行）。
- **PIVOT**：换思路，换到另一层组件去修。比如 prompt 层的规则不管用，就改成在 tool 或 middleware 里强制执行。这一步不在伪代码里，只是提示词对 Evolve Agent 的要求。
- 换到哪一层由 Evolve Agent 自己决定。原文没有规定换层顺序，也不要求穷举七层；每次只换一层，而且只有在失败持续时才换。

**和案例的关系**
- 最接近"换层"思路的是 path-tracing（p.28，p.32）：种子 prompt 里本来就有"不要破坏已验证状态"的劝告，但没有执行层面的强制。于是第 5 轮在 tool 层加了 shell 守卫（chg-8），manifest 写明"即使有 prompt 规则，智能体仍会执行破坏性清理，在 shell 工具上做执行时强制是最直接的拦截"。不过这次是保留 prompt 规则、再加上 tool 层，不是先回滚再换层。
- 第 4 条轨迹（configure-git-webserver，p.30）明确写着：第 7 轮没有选择回滚，而是修补守卫的漏洞和 middleware 提醒不够醒目的问题，属于 IMPROVE。
- 就对话中掌握的论文内容看，附录案例里没有一个明确标注为"ROLLBACK + PIVOT"的完整实例。是否在全文其他位置出现，尚未检索确认。

---

## 7. Algorithm 1 中什么时候 rollback

**问**：在算法一中，什么时候才会 rollback 呢？原文依据是什么？

**答**：Algorithm 1 里 rollback 只出现在第 7 行，要同时满足两个条件才会触发：
1. **从第 2 轮开始（t ≥ 2）**。第 1 轮没有上一轮的 manifest，判定 V₁ = ∅，不会回滚。
2. **上一轮的某个修改在归因判定 Vₜ 中被否决**。第 6 行 ATTRIBUTE 先拿上一轮 manifest 的预测对照本轮实测结果，得出逐条判定；第 7 行再把被否决的修改从 Hₜ₋₁ 中按文件撤销。

**原文依据**

| 位置 | 原文 |
|---|---|
| p.5，Algorithm 1 第 5–7 行的注释 | "phase 3: attribute prior manifest, then rollback" |
| p.5，Algorithm 1 后第 1 段 | "rollout, clean, attribute the prior manifest and revert rejected edits, distill, edit, commit" |
| p.5，§3.3 第 1 段 | "every edit becomes a falsifiable, file-level claim…, and the next round's verdict either confirms or reverts it" |
| p.5，§3.3 第 3 段 | "the loop intersects the predicted-fix and predicted-regression sets with the observed task-level deltas to produce a per-edit verdict" |
| p.2，§1 第 4 段 | "…so each edit becomes a falsifiable contract and ineffective ones are reverted at file granularity" |
| p.31，Figure 8 标题 | "…binding contracts that the attribution check rolls back if their predicted fixes do not materialize" |
| p.34，Appendix C.3 | "…the attribution check decides whether to keep or roll it back…subject to the same automatic rollback if their predicted fixes do not appear" |
| p.10，Limitations | "…rolls back ineffective edits at file granularity" |

综合起来，原文给出的触发条件是：**修改预测能修好的任务在下一轮没有被修好，也就是修改被判定为无效。**

**原文没有说清的地方**
- **具体判定阈值未说明**：预测修好的任务要兑现多少个才算有效？出现预测之外的回归，是否也会导致回滚？`risk_tasks` 的命中情况会不会影响判定？论文只说用预测集和实际变化求交来判定，没有给出规则细节。
- **回滚由谁执行，原文有两种说法并存**：
  - 循环自动回滚：Algorithm 1 第 7 行和 C.3 中的 "automatic rollback" 是这个意思；
  - Evolve Agent 自己决定回滚：B.2 的提示词让它对每个修改选择 KEEP / IMPROVE / ROLLBACK + PIVOT，manifest 的 `type` 字段也有 `"rollback"` 一项。

  案例里能看出 Evolve Agent 有自主权：第 7 轮分数从 75.8 降到 73.0，但它没有回滚守卫和 middleware，而是选择修补（p.30，C.1.4）。这两种回滚机制怎么分工，原文没有交代。

---

## 8. TB2 是什么任务

**问**：TB2 任务是什么任务？

**答**：TB2 是 Terminal-Bench 2 的缩写，是一个**在命令行终端里完成真实任务**的智能体评测基准。以下只根据论文里的信息介绍。

**论文怎么描述它**
- **出处**：参考文献 [21]，标题 "Terminal-bench: Benchmarking agents on hard, realistic tasks in command line interfaces"（2026 年 1 月）。官方排行榜在 tbench.ai（p.15 脚注）。
- **任务类型**：多步骤的终端工作流（p.1，§1 第 1 段）。相关工作部分（p.3，§2.1）把它归为长程、真实环境的任务，可能持续数小时（"multi-hour, terminal-driven workflows"）。
- **规模**：89 个任务，官方难度划分为 4 个 easy、55 个 medium、30 个 hard（p.6，§4.1；p.15，Appendix A）。
- **论文中的设置**：
  - 每个任务的超时延长到 1 小时（3600 s）；
  - 智能体在独立的 E2B 沙箱里只通过 shell 操作，最后由 Harbor 调用 verifier 判定成功或失败（p.6；p.15）；
  - 超时或基础设施出错都算失败，这和官方排行榜的规则一致。

**任务长什么样（附录 C 的例子）**

| 任务 | 要做什么 | 怎么判定 |
|---|---|---|
| db-wal-recovery | 从损坏的 SQLite 预写日志（WAL）中恢复数据库，把结果输出成 JSON | 逐行逐字段核对标准答案 |
| path-tracing | 写一个路径追踪渲染器，把场景渲染成 `/app/reconstructed.ppm` | 与参考图像逐像素比对 |
| mcmc-sampling-stan | 安装 rstan，拟合一个层级 beta-binomial 模型，写出后验均值 | 重新运行智能体写的 `analysis.R`，检查数值是否落在规定区间内 |
| configure-git-webserver | 搭建 git 仓库和 web 服务器，部署一个页面 | 从外部发 HTTP 请求，检查返回内容 |

论文里提到的其他任务名还有：compile-compcert、regex-chess、polyglot-rust-c、large-scale-text-editing、train-fasttext、caffe-cifar-10、sam-cell-seg、dna-assembly、pytorch-model-recovery、mteb-retrieve、git-multibranch 等。

**这类任务的共同特点**
- **覆盖面广**：系统运维、编译构建、数据恢复、科学计算、机器学习训练、图形渲染等都有。
- **步骤多**：需要很多步命令行操作，包括安装依赖、写代码、运行、调试、验证。
- **由隐藏的 verifier 严格检查最终状态**：智能体自己说"完成了"不算数。这也是为什么 AHE 进化出的很多规则都和"按 verifier 的标准自检""不要破坏已验证的状态"有关。

---

## 9. 作者如何解释 Hard 档低于 Codex

**问**：（引用主对话）"AHE 的总分最高。唯一的例外是 Hard 档略低于 Codex，作者用 RQ3a 的消融来解释：单独换入 memory，Hard 档就能达到 63.3%，所以问题出在组件之间相互干扰，并非缺少能力。" 帮我详细讲解一下作者是怎么解释这个现象的？

**答**：作者的解释分三层：先用一个事实说明 AHE 并不缺这项能力，再说明组件叠加后为什么互相抵消，最后说明进化循环为什么没有避开这个问题。

### 第一层：Hard 档的差距不是因为缺少能力（p.6，§4.2 第 2 段）

原文判断这个差距来自 AHE 各组件在长程任务上的相互干扰，而不是缺少某种能力（"interference between AHE's components on long-horizon tasks rather than to a missing capability"）。

证据是：只把 AHE 的 long-term memory 换入种子、其他组件保持原样，Hard 档就超过了 Codex。

| 配置 | Hard（30 题） |
|---|---|
| NexAU₀ 种子 | 51.7% |
| Codex | 56.7% |
| 完整 AHE | 53.3% |
| 种子 + AHE 的 memory | **63.3%** |

推理很直接：解决 Hard 题所需的经验已经在 AHE 的 memory 里了。单独用它时 Hard 档能到 63.3%，和其他组件放在一起却只有 53.3%，说明是别的组件把它的效果抵消了。

### 第二层：各组件在 Hard 档的作用方向不同（p.8，§4.4.1）

作者逐个说明了每个组件针对的失败类型，以及它在 Hard 档上的副作用：

| 组件 | 做了什么 | 在 Hard 档的表现 |
|---|---|---|
| Memory | 新增 12 条边界情形的教训 | 有帮助：单独换入时超过完整 AHE（但 Easy 档反而因多余的复查而下降） |
| Tool | 1364 行的 shell 工具，内置发布状态守卫 | 有害（46.7%，低于种子）：守卫让任务过早收尾（"closes the loop too early"） |
| Middleware | 收尾钩子，强制做一次和 evaluator 同构的收尾检查 | 有害：增加了对话轮数（"inflates turn count"） |
| System prompt | 79 行通用规范 | 单独使用时退步，能否落实取决于另外三层 |

### 第三层：叠加后的冗余，以及进化方向的偏差（p.9，§4.4.1 "Components interact non-additively…" 段）

1. **机制上的冗余**：memory、middleware、system prompt 都在推动同一种收尾式验证。三者叠在一起，智能体会在长程任务有限的步数和时间里反复做重复检查，把预算耗在冗余验证上。Hard 题本来就需要很多步，这种消耗对它影响最大。整体数字也印证了组件不能简单相加：三个单组件的正增益合计 +11.1 pp，完整 AHE 只有 +7.3 pp。
2. **进化循环为什么没纠正**：Evolve Agent 优化的是总体 pass@1，而总分由 55 道 Medium 题主导，Hard 只有 30 道。所以它收敛到一个偏向 Medium 的取舍，用 Hard 上的一部分 memory 效果换来了 Medium 的提升。结果完整 AHE 的 Medium 是 88.2%，所有配置中最高。作者把能处理组件交互的进化（interaction-aware evolution）留作未来工作。

### 合起来的逻辑链

memory 单独使用时 Hard 档 63.3%，说明能力存在 → 多个组件都在做收尾验证，叠加后浪费长程任务的预算 → 优化目标被 Medium 题主导，进化不会主动为 Hard 档拆开这些组件 → 完整 AHE 的 Hard 档只有 53.3%，略低于 Codex 的 56.7%。

### 这个解释的强弱（个人评估，不是原文内容）
- **差距很小**：Hard 档只有 30 题，53.3% 和 56.7% 相差 3.4 pp，大约相当于一道题。论文没有给方差或显著性检验，这个"例外"本身可能就在噪声范围内。
- **"冗余复查"的机制没有直接证据**：原文说 middleware 会增加轮数，但没有看到 Hard 档逐配置的轮数或 token 统计，也没有逐任务的失败分析来证明失败确实是因为预算被重复检查耗尽。这个机制是合理的推断，不是测量出来的结论。
- **消融设计只能证明组件之间有交互，定位不到是谁和谁冲突**：实验只做了"在种子上加入一个组件"，没有做"从完整 AHE 中去掉一个组件"或两两组合。此外，tool 单独使用时 Hard 档就低于种子，说明有一部分问题可能来自 tool 本身的副作用（守卫让任务过早收尾），而不只是组件之间的交互。
- **63.3% 是事后挑出来的最好结果**：它是 4 个单组件变体中 Hard 档最高的那个，用来说明"能力存在"可以，但不能当作 AHE 能稳定达到的 Hard 档水平。

---

## 10. 如何定位组件冲突

**问**：作者没有定位哪几个存在什么冲突？如果想定位，你有没有什么好的想法？

**答**：完整内容已整理为[附录 B](#附录-b候选问题ahe-组件冲突的定位最终保存版)，这里只列要点。

- **作者做到的**：指出嫌疑对象。memory、middleware、system prompt 都在推动收尾式验证；tool 的守卫在 Hard 档让任务过早收尾（p.8–9）。
- **作者没做的**：两两组合消融、"从完整版去掉一个"消融、证明预算被耗尽的轨迹统计。
- **定位思路（建议）**，推荐顺序 1 → 4 → 2 → 3 / 5：
  1. 从完整版去掉一个组件的消融（4 个配置，只看 Hard 档）；
  2. 2⁴ 析因或 Shapley 分析，量化交互项，k 加大到 4–8；
  3. 在组件内部按机制加开关；
  4. 用轨迹指标和放宽预算的干预实验，检验"预算耗尽"假说；
  5. 逐任务对照 memory-only 通过、完整 AHE 失败的 Hard 任务。

---

## 11. Figure 3：为什么 GPT-5.4 high 的提升比 medium 高

**问**：论文中的 Figure 3，左边，为什么 gpt5.4 high 的提升居然比 med 还高，作者是怎么解释这个现象的？因为模型的能力越强，提升不应该越少吗？原因是什么？

**答**：作者把原因归到**运行参数的匹配程度**，而不是模型能力。AHE 的步数预算和超时是按 GPT-5.4 high 调出来的。high 正好匹配，medium 和 xhigh 都偏离了这个点，所以增益都打了折扣。

**先看数据（p.8，Figure 3）**

| GPT-5.4 推理档位 | NexAU₀ | AHE | 增益 |
|---|---|---|---|
| medium | 65.7% | 68.0% | +2.3 |
| high（进化时用的档位） | 69.7% | 77.0% | **+7.3** |
| xhigh | 72.5% | 74.7% | +2.3 |

（具体数值来自抽取文本中错位的图注，原文正文只写了 medium 和 xhigh 都是 +2.3 pp。）

增益是中间高、两头低，不随模型能力单调变化。

**作者的解释（p.7，§4.3 最后一段）**

关键句是："AHE's step budget and per-task timeout were fitted to GPT-5.4 high during evolution"。也就是说，进化出的 harness 隐含了一组为 high 档调好的运行参数（步数预算、单任务超时），换档位就偏离了这组参数：
- **medium**：每一步思考更短，在同样的超时内时间更宽裕（"more time-per-step slack"），但少了一档推理能力（"loses a reasoning tier of raw capability"）。
- **xhigh**：每一步思考更久，更多试次超过单任务超时；而按论文的 pass@1 规则，超时一律算失败（"pushes more trials past the per-task timeout"）。
- **结论**："Either direction discounts the gain"，往哪个方向偏离，增益都会打折。

作者随后在 p.8（Figure 3 后第 1 段）总结：增益大小跟的是进化时的运行参数，而不是基座的原始能力（"Their magnitude tracks the evolution operating point rather than raw base capability"），并把这种参数耦合列为影响泛化的隐患。Limitations 里也承认，跨模型的数字把"harness 能否迁移"和"运行参数是否匹配"混在了一起，要拆开只能在多组运行参数下重跑进化（p.10，"Evolution operating point."）。

**和"能力越强，提升越少"的关系**

这个直觉在论文里有一半成立：
- **跨家族比较时成立**：作者认为离饱和越远的基座越依赖 AHE 固化的协调模式。deepseek +10.1、qwen +6.3、gemini +5.1，都高于 GPT-5.4 的同家族增益（p.7）。
- **同家族内不成立**，原因就是上面说的参数耦合：
  - xhigh 比 high 强、增益也更低，这符合直觉，只不过作者把它归因于超时；
  - medium 比 high 弱、增益却更低，这和"离饱和越远越受益"的说法相矛盾。作者对 medium 的解释只有"少了一档推理能力"这一句。

**补充几点（个人读法，不是原文观点）**
- **high 是"主场"**：AHE 报告的是 10 轮里在 GPT-5.4 high、同一批 89 题上分数最高的那一版，所以 +7.3 包含了在优化目标上被挑选出来的优势；medium 和 xhigh 则是真正的迁移。仅这一点就足以让 high 的增益偏高，论文没有把这个因素单独拿出来讨论。
- **对 medium 的解释比较薄**：一种可能的读法是，AHE 的很多组件（契约优先的规则、按 verifier 标准自检、风险提示）要求模型有足够的推理能力才能执行好，能力降一档就用不好这些组件。这只是推测，论文没有用实验验证。
- **只有一次运行**：每个档位只有一组数字，没有方差。+2.3 和 +7.3 之间的差距有多稳定，原文没有说明。

---

## 12. AHE 为什么更省 token

**问**：为什么 AHE 反而会更省 token？是因为有 debugger 给他结构化 trace 了吗？

**答**：不是。**省下来的 token 和 Agent Debugger 没有关系。**

**先分清省的是哪部分 token**

论文比较的是 tokens/trial：Coding Agent 做一次任务时，所有 LLM 调用的 prompt + completion 总和（p.6，§4.1；p.16，Appendix A）。省 token 的结论来自 SWE-bench-verified 迁移实验（p.7，Table 2）：

| | 每次试验的 Tokens（千） |
|---|---|
| ACE | 679 |
| TF-GRPO | 582 |
| NexAU₀（种子） | 526 |
| **AHE** | **461** |

这是冻结后的 harness 直接拿去做题时的消耗：
- Agent Debugger 和 Evolve Agent 只在进化循环里工作，测试时根本不运行，Coding Agent 也看不到结构化的 trace。
- 反过来，进化循环自己的开销（Debugger 分析、Evolve Agent 修改、每轮 89×2 次 rollout）不计入这个指标。论文也没有报告进化总共花了多少 token。

**作者的解释（p.7，§4.3 第 3 段）**

> "encoding behavior in tools, middleware, and memory avoids the per-call re-derivation cost prompt-only baselines incur."

意思是：AHE 把行为写进工具、middleware 和 memory，不需要每次调用模型时都在 prompt 里重新推导一遍。

和两个基线对比：ACE 注入的 playbook、TF-GRPO 强化的经验文本都挂在 prompt 上，每次模型调用都要重新发送一遍。换到新的任务集后，这些文本只增加成本，不改变行为，所以它们的 token 比种子多 11%–29%，成功率还更低。

**AHE 为什么连种子都比得过（推断，原文没有展开）**

种子 prompt 更短，AHE 还加了 79 行 prompt 和 memory，按理每次调用的输入更长，但 AHE 每次试验的总 token 仍少 12%。tokens/trial 是所有轮次加起来的总量，轮次越多，历史上下文被重复发送的次数越多，所以少走几步就能省很多。AHE 的一些组件正好针对浪费步数的行为，可能减少了总轮数：
- shell 工具加了可调超时和后台执行，避免傻等默认的 5 分钟超时；
- middleware 检测同一错误的反复重试和重复的长时间运行；
- 守卫阻止"验证后又破坏、再重做"的情况。

论文没有给出 SWE-bench 上逐配置的轮数统计，所以这一点无法从原文确认。

---

## 13. 举例：为什么比种子还省 token；为什么有些场景不如种子

**问**：我还是没搞懂，为什么会比种子还少呢？帮我举个例子来理解。另外，我不太明白，为什么 AHE 在其他场景上居然还不如种子呢？

**答**：

### 一、为什么 AHE 比种子还省 token：一个例子

关键在于**每一轮都要把整段历史重新发给模型**。

tokens/trial 是一次任务中所有 LLM 调用的 token 加总。智能体每走一步，都要把 system prompt 和之前所有的命令、输出重新发一遍，所以总消耗大约和"步数 × 当时的上下文长度"成正比。步数越多，后面每一步越贵，总量增长得比步数快。

以下数字是编的，只为说明机制，不是论文数据。

**种子 NexAU₀**：prompt 很短，只有 1k tokens；但它会犯 chg-2 manifest 里写的那种错误，"在长时间的前台安装和 sleep 轮询循环上浪费预算"（p.31，Figure 8）。比如装一个依赖时反复执行 `sleep 30; check`，每检查一次就是一轮，每轮历史增加约 3k，总共走 30 步：
- 第 1 步发送 1k，第 2 步 4k，第 3 步 7k……第 30 步约 88k；
- 合计约 **1335k tokens**。

**AHE**：prompt 加上 memory 更长，比如 3k；但 shell 工具支持后台执行和可调超时，middleware 会提醒"别再重复重试"，于是少了十几步无效轮询，只走 20 步：
- 第 1 步 3k，第 2 步 6k……第 20 步约 60k；
- 合计约 **630k tokens**。

所以，AHE 每一步多带了几千 token 的规则，但只要能砍掉一些无效步骤，总量反而少得多。这对应作者的解释：把行为写进工具、middleware 和 memory，就省掉了"每次调用都重新推导"的成本（p.7，§4.3）。

需要说明：论文没有给出 SWE-bench 上各配置的平均步数，所以"AHE 省 token 是因为步数少"是推断，原文没有直接证明。

### 二、为什么 AHE 在某些场景下不如种子

先确认范围：AHE 不如种子的情况，**只出现在 SWE-bench-verified 的三个最小的仓库上**（p.7，Table 2）：

| 仓库 | 任务数 | NexAU₀ | AHE | 差距约等于 |
|---|---|---|---|---|
| scikit-learn | 32 | 93.8% | 87.5% | 2 道题 |
| pydata | 22 | 77.3% | 72.7% | 1 道题 |
| astropy | 22 | 54.5% | 50.0% | 1 道题 |

总体上 AHE 仍然最高（75.6% 对 75.2%），跨模型的 5 个基座也全部是正增益。

**作者的解释（p.7，§4.3 第 3 段）**：这些下降与"小仓库上 pass@1 的波动超过单个仓库的增益"相符（"consistent with pass@1 variance on small repos exceeding the per-repo gain"）。意思是题太少，差一两道题就是 5 个百分点，这在随机波动的范围之内，不一定说明 AHE 在这些仓库上真的更差。

**补充（个人读法，不是原文内容）**：
- **这个解释是双向的**：按同样的道理，总体 +0.4 pp（500 题里大约差 2 道）也在噪声范围内。所以 SWE-bench 上更稳妥的结论是"成功率基本持平，token 明显更少"，而不是"AHE 成功率更高"。论文没有报告方差和显著性检验。
- **也可能存在真实的副作用，只是无法证实**：AHE 是在 TB2 上进化出来的，发布状态守卫、强制收尾检查这类组件，在 SWE-bench 那种"改代码、跑测试"的任务里可能偶尔误触发，或者带来多余的检查。之前的消融也显示，tool 在 TB2 的 Hard 档会让任务过早收尾（p.8）。不过论文没有对这几个仓库做逐任务分析，所以无法区分是噪声还是真实的副作用。
- **作者给出的增益来源**：相对种子的提升主要集中在 django 和 sphinx-doc 这两个最大、最耗 token 的仓库。作者认为，这两个仓库的多步"修改再验证"流程，正好和 AHE 在 TB2 上学到的结构相符（p.7）。换句话说，AHE 的优势依赖于任务结构是否相似，任务越长、越复杂，越可能用得上它。

---

## 14. AHE 是在 TB2 上进化的吗；跨场景实验说明什么

**问**：也就是说 AHE 这个方法是在 TB2 上进化的是吧？这里的实验只是说明他在跨场景也有用是吧？

**答**：对，两点理解都没错，只需要补两个细节。

**1. AHE 是在 TB2 上进化出来的**
- 进化使用 Terminal-Bench 2 全部 89 个任务，三个角色智能体都用 GPT-5.4 high，跑了 10 轮（p.6，§4.1、§4.2）。
- 补充：RQ1 的主结果（Table 1，77.0%）也是在同一批 89 题上测出来的，报告的是 10 轮中分数最高的那一版。所以主结果是在优化目标本身上的成绩，不是在没见过的任务上的成绩。

**2. 跨场景实验是为了说明它没有只拟合 TB2**

这部分就是 RQ2（§4.3）。做法是把进化好的 harness 冻结，不再进化，直接拿去用：
- **换任务**：SWE-bench-verified 的 500 个任务（Table 2）；
- **换模型**：另外 5 个基座（Figure 3）。

不过两个实验能说明的程度不一样：
- **换模型**：5 个基座的增益全部为正（+2.3 到 +10.1 pp），证据比较明确。
- **换任务**：成功率 75.6% 对种子的 75.2%，只高 0.4 pp，在噪声范围内。明确成立的是没有变差，而且省 12% 的 token。相比之下，ACE 和 TF-GRPO 迁移过去后反而低于种子。

所以更准确的说法是：**跨场景时 AHE 没有退化，还更省钱，换模型时也普遍有提升**，而不是"在新任务上也明显更强"。

---

## 15. "只加一个"和"去掉一个"消融的区别

**问**：（引用主对话）"这是'只加一个'的消融，而不是'从完整版里去掉一个'的消融。它衡量的是每个组件单独的价值，不是在完整系统中的边际贡献……" 这两种有什么区别？做完整版去除一个的消融有什么帮助呢？

**答**：两种消融的起点不同，回答的问题也不同。

| | "只加一个"（论文做的） | "去掉一个"（论文没做的） |
|---|---|---|
| 起点 | 种子 NexAU₀ | 完整 AHE |
| 操作 | 只加入 1 个 AHE 组件 | 只去掉 1 个组件，保留其余 3 个 |
| 回答的问题 | 这个组件单独有没有用？ | 在完整系统里，这个组件还在起作用吗？去掉它会怎样？ |
| 衡量的量 | 单独价值 | 边际贡献 |

**如果组件可以简单相加，两种结果一样**

假设每个组件的效果是固定的，比如 memory 总是 +5、tool 总是 +3。那么不管是加到种子上还是从完整版去掉，看到的都是同一个数，两种消融结论一致。

**论文已经发现组件不可加，所以两种结果会不同**

论文的证据是：三个单组件增益合计 +11.1 pp，完整 AHE 只有 +7.3 pp（p.9）。

下面用一个编造的数字例子说明（不是论文数据）。假设 memory 和 middleware 都在做"收尾前检查"：
- 只加 memory：+5（单独有用）
- 只加 middleware：+3（单独有用）
- 两个都加：+5（middleware 做的检查 memory 已经做过了，没有新增收益）

两种消融的读法：
- **"只加一个"**：两个组件都有用，middleware 贡献 +3。
- **"去掉一个"**：
  - 从完整版去掉 middleware，分数不变，说明它在完整系统里是冗余的；
  - 去掉 memory，分数掉 2，说明 memory 才是在起作用的那个。

更极端的情况：如果两者叠加后反复检查、耗尽步数（论文对 Hard 档的猜测），完整系统可能比只加 memory 更差。这时去掉 middleware 分数反而上升，说明它在完整系统里是有害的。而"只加一个"会把它判为有用组件，完全看不出这一点。

**做"去掉一个"消融的帮助**
1. **找出冗余组件**：去掉后分数不变，说明它的作用已被其他组件覆盖，可以删掉，减少上下文长度和步数。
2. **找出有害组件**：去掉后分数上升，说明它在完整系统里拖后腿。这是定位 Hard 档冲突最直接的办法：看去掉哪一个之后，Hard 档能回到接近 memory-only 的 63.3%。
3. **找出不可替代的组件**：去掉后分数大跌，说明它是关键组件，别的组件补不上。
4. **和"只加一个"对照，得到交互信息**：
   - 单独价值高、边际贡献低：和其他组件重叠；
   - 单独价值低、边际贡献高：依赖其他组件才能发挥作用。论文对 system prompt 的说法正是这一种（"能否执行取决于另外三个组件"），但这个说法需要"去掉一个"的实验才能验证。
5. **成本低**：只需要 4 个配置，而完整的两两组合需要 16 个配置。

论文目前只有"只加一个"这一半，所以只能说明组件之间有交互，说不出是谁和谁冲突、谁在完整系统里是冗余的。

---

## 16. RQ3b：修复预测、回归预测、Precision、Recall

**问**：RQ3b 这里的修复预测和回归预测有什么区别？都分别是什么意思？Precision 和 Recall 是什么意思？

**答**：RQ3b 检验的是 Evolve Agent 每轮在 change manifest 里写下的预测，到下一轮是否应验（p.9，§4.4.2）。预测分成两类，分别算 precision 和 recall。

### 一、修复预测与回归预测

从第 N−1 轮到第 N 轮，每个任务的结果可能发生两种翻转：

| | 实际发生的事 | 对应的预测字段 | 含义 |
|---|---|---|---|
| 修复（fix） | 任务从失败变成通过 | `predicted_fixes` | "我这个修改能把这些任务修好" |
| 回归（regression） | 任务从通过变成失败 | `risk_tasks` | "我这个修改可能把这些任务改坏" |

修复预测问的是"你说能修好的，修好了吗"；回归预测问的是"你说可能改坏的，真的坏了吗"。

### 二、Precision 和 Recall

两个指标都看"预测"和"实际发生"的重叠部分，区别在分母：
- **Precision（精确率）= 命中数 ÷ 预测总数**，回答"你说的话有几成是对的"。
- **Recall（召回率）= 命中数 ÷ 实际发生总数**，回答"实际发生的事里，你提前预料到了几成"。

论文用到的几个标签：
- **TP（命中）**：预测了，也真的发生了；
- **FP（误报）**：预测了，但没发生；
- **FN（漏报）**：没预测，但发生了。

于是 Precision = TP / (TP + FP)，Recall = TP / (TP + FN)。p.35 的 Figure 12、13 就是这样把分母拆开画的。

### 三、用附录 D 的回归数字验证一遍（p.35）

9 轮累计下来：
- Evolve Agent 一共预测了 43 个可能回归的任务，其中只有 5 个真的回归了，其余 38 个是误报；
- 实际发生的回归中，有 40 个它完全没预料到（漏报），加上那 5 个命中，实际回归共 45 个。

所以：
- Precision = 5 / 43 ≈ 11.6%
- Recall = 5 / 45 ≈ 11.1%

（正文报告的回归 precision 是 11.8%，那是逐轮取平均的结果；11.6% 是累计口径，两者计算方式不同。）

修复预测的分子分母，原文没有给出累计数。下面用一个编造的例子理解（不是论文数据）：某轮预测能修好 10 个任务，其中 3 个真的修好了；这一轮实际修好的任务共 6 个。那么 precision = 3/10 = 30%，recall = 3/6 = 50%，和论文的 33.7%、51.4% 处在同一量级。

### 四、论文的结论

| | Precision | Recall | 随机基线（P / R） | 倍数 |
|---|---|---|---|---|
| 修复预测 | 33.7% | 51.4% | 6.5% / 10.6% | 约 5 倍 |
| 回归预测 | 11.8% | 11.1% | 5.6% / 5.4% | 约 2 倍 |

- **修复预测有依据**：它说能修好的任务，约三分之一真的修好了；实际修好的任务，它提前预料到一半左右。作者称之为 evidence-driven targeting。
- **回归预测基本失效**：它担心的任务大多没坏，真正坏掉的约九成没想到。作者称之为 regression blindness，并认为这正是进化曲线时升时降的原因。

这和直觉一致：设计一个修改时，人会围绕它要解决的失败去想，能说清"为什么有用"，但很难想到它会对另外几十个任务产生什么副作用。

补充：随机基线是怎么构造的，原文没有说明，所以"5 倍"和"2 倍"的含义取决于一个原文没有交代的设定（见下一问）。

---

## 17. Precision / Recall 的含义；随机基线怎么实现

**问**：Precision 和 Recall 分别代表什么指标？随机基线具体是怎么实现的？

**答**：

### Precision 和 Recall 是什么

两者都只统计"预测"和"实际发生"重叠的部分，区别在分母：

| 指标 | 公式 | 回答的问题 |
|---|---|---|
| Precision（精确率） | 命中数 ÷ 预测总数 = TP / (TP + FP) | 我预测的里面，有多少真的发生了？衡量预测**准不准** |
| Recall（召回率） | 命中数 ÷ 实际发生总数 = TP / (TP + FN) | 实际发生的里面，有多少我提前预测到了？衡量预测**全不全** |

在 RQ3b 中，"发生"有两种：
- 修复：任务从失败变成通过，对应 manifest 里的 `predicted_fixes`；
- 回归：任务从通过变成失败，对应 manifest 里的 `risk_tasks`。

两种各自算一组 precision 和 recall。

### 随机基线是怎么实现的

**原文没有说明构造方法**，只在 p.9 给了数值：修复 6.5% / 10.6%，回归 5.6% / 5.4%。

不过这些数值和一种最自然的构造完全吻合。以下是推断，不是原文的说法。

**推断的构造方式**：每一轮从 89 个任务中均匀随机地抽取与 Evolve Agent 预测数量相同的任务，作为"随机预测"。这样做时：
- 期望 precision = 实际发生数 ÷ 89。随机抽到的任务里，命中的比例就等于事件在全体任务中的占比。
- 期望 recall = 预测数 ÷ 89。每个实际发生的事件被抽中的概率就是抽取比例。

**用附录 D 的回归数字验证**（p.35：9 轮共预测 43 个，实际回归 45 个，即 5 个命中加 40 个漏报）：
- 每轮实际回归约 45 / 9 = 5 个，所以随机 precision ≈ 5 / 89 = 5.6%，与原文一致；
- 每轮预测约 43 / 9 ≈ 4.8 个，所以随机 recall ≈ 4.8 / 89 = 5.4%，与原文一致。

两个数都对得上，说明这种构造很可能就是作者的做法。

**倒推修复预测的情况**（同样是推断）：
- 随机 precision 6.5% → 每轮实际修复约 0.065 × 89 ≈ 5.8 个；
- 随机 recall 10.6% → 每轮预测修复约 0.106 × 89 ≈ 9.4 个。

这和案例中单条修改预测 4 到 14 个任务（Fig. 8–11）的量级相符。

### 这意味着什么
- 这个基线的含义是：不看任何证据，只按同样的数量随便猜，能达到多少命中率。所以"5 倍"和"2 倍"衡量的是 Evolve Agent 比瞎猜强多少，而不是它的绝对准确度。
- **这个基线比较弱**。比如它没有利用"上一轮失败的任务才可能被修复"这一常识。如果只在上一轮失败的任务里随机抽，修复 precision 的基线会明显更高，5 倍的优势就会缩小。原文没有讨论这种更强的基线。
- 正文报告的 11.8% 等数字是逐轮比值的平均，与上面按累计数的估算略有差异（例如附录 D 的累计 precision 是 11.6%），所以上面的验证只是近似吻合。

---

## 18. "抽取同样数量的随机预测"是什么意思

**问**：（引用上一问）"推断的构造方式：每一轮从 89 个任务中均匀随机地抽取与 Evolve Agent 预测数量相同的任务作为'随机预测'。" 这是啥意思？

**答**：这句话说的是**怎么造一个"瞎猜的对照组"**。这是根据数字倒推的构造方式，原文没有写明。

**用一个例子说明（数字是编的）**

假设第 5 轮结束时，Evolve Agent 在 manifest 里写下"这些修改能修好 10 个任务"。随机基线的做法是：
1. 不看任何轨迹或证据；
2. 把 89 个任务打乱，随手抽 10 个，数量和 Evolve Agent 预测的一样；
3. 把这 10 个当作"随机预测会修好的任务"。

到第 6 轮看实际结果。假设实际修好了 6 个任务，就用同样的公式给随机预测打分：
- **Precision**：随手抽的 10 个里，碰巧有几个属于那 6 个？每个任务被修好的概率是 6/89 ≈ 6.7%，所以随机 precision 期望约 6.7%。
- **Recall**：那 6 个实际修好的任务，每个被抽中的概率是 10/89 ≈ 11.2%，所以随机 recall 期望约 11.2%。

再拿 Evolve Agent 的真实成绩（比如 precision 33.7%）和这个瞎猜的成绩比，就得出"约 5 倍于随机"。

**为什么要抽"同样数量"**

为了公平。如果不限制数量，瞎猜的人可以说"89 个全都会被修好"，recall 直接是 100%，但 precision 极低；反过来只猜 1 个，precision 可能碰巧很高，recall 却极低。

固定预测数量后，两边的差别只剩一个：**Evolve Agent 是看了证据挑的，随机基线是闭眼抽的**。这样比较才能说明它的预测里有没有真东西。

**为什么说是"推断"**

原文只给了基线的数值（修复 6.5% / 10.6%，回归 5.6% / 5.4%），没有说明怎么算的。用附录 D 的回归数据验证过：每轮实际约 5 个回归，5/89 ≈ 5.6%；每轮约预测 4.8 个，4.8/89 ≈ 5.4%。两个数都对得上，所以这种构造很可能就是作者的做法，但不是原文明说的。

---

## 附录 A：候选问题草稿（完整版）

> 在 side question 中曾整理过一份覆盖全部对话的候选问题清单。之后改为只保存"组件冲突定位"那一段（见附录 B），这里保留完整草稿备查。

### 一、概念理解类（已在对话中解答，可用于自测）

| # | 问题 | 要点 | 原文依据 |
|---|---|---|---|
| 1 | Figure 2 的流程是什么？ | Harness → Coding Agent 跑任务 → Raw trace → Agent Debugger 蒸馏为 Overview → Evolve Agent 结合 History 修改组件，并写 manifest 预测 → 下一轮检验；分别对应三个可观测性支柱 | p.4 Figure 2，§3 |
| 2 | Algorithm 1 每一行在做什么？ | ROLLOUT → CLEAN → ATTRIBUTE + ROLLBACK（t≥2）→ AGENTDEBUGGER → EVOLVE → COMMIT → 更新 H_best；第 t 轮评测的是 H_{t-1} | p.5 |
| 3 | 三个 RQ 分别回应什么疑问、用什么实验回答？ | RQ1 是否有效（Table 1）；RQ2 是否过拟合（Table 2/5、Fig. 3）；RQ3a 增益落在哪一层（Table 3）；RQ3b 自我归因是否可靠（Fig. 4、App. D） | p.6–9 |
| 4 | "换基座模型后 harness 需重新适配"的依据在哪？ | 引言中的论断，无直接实验验证 | p.2 §1 第 2 段 |
| 5 | "Harness design materially shifts task completion…"是什么意思？ | 基座模型不变，只改 harness，完成率也会明显变化；这是引用 [40, 42] 的前提 | p.2 §1 第 2 段 |
| 6 | Middleware 是什么？ | 挂在智能体循环中的代码，通过 hook 拦截或改写；与 prompt 的区别是在运行时强制执行 | p.1；p.19 B.2；p.22 B.3.1；p.32–34 |
| 7 | 第 1 轮、最小种子 NexAU₀、bash 工具分别指什么？ | 第 1 轮评测未修改的种子；NexAU₀ 只有 run_shell_command，无 middleware / skills / sub-agents / memory | p.4 §3.1；p.15 App. A；p.17 B.1 |
| 8 | NexAU 是什么框架？ | Nex-AGI 的通用 agent 框架，7 类组件文件化、松耦合 | p.4 §3.1；参考文献 [23][37] |
| 9 | chg-2 是智能体吗？ | 不是，是某一轮中某个修改的编号；修改由 Evolve Agent 执行；编号每轮各自计数 | p.20–21；Fig. 8–11 |
| 10 | Raw trace 与 History 的区别？ | 前者是本轮 Coding Agent 的执行轨迹，后者是跨轮次的修改与评测记录（History 的文件对应是推断） | Fig. 2；p.18 B.2 |
| 11 | rollouts per task k 是什么？ | 同一任务重复完整执行的次数，进化运行中 k=2；pass@1 是所有 rollout 的均值 | p.5；p.15 公式 (1)；Table 4 |
| 12 | manifest 预测是什么？ | `predicted_fixes` / `risk_tasks`，下一轮用实测检验，构成可证伪的契约 | p.5 §3.3；p.21 |
| 13 | ROLLBACK + PIVOT 是七层循环吗？ | 不是，是 Evolve Agent 提示词里的经验规则：同一失败修了 2 轮以上无效时换层；不在伪代码中 | p.19–20 B.2 |
| 14 | 什么时候 rollback？ | t≥2 且修改在归因中被否决；具体阈值未说明；自动回滚与 Evolve Agent 自主回滚两种说法并存 | p.5；p.2；p.31；p.34 |
| 15 | 作者如何解释 Hard 档低于 Codex？ | memory-only 在 Hard 档达 63.3%，说明能力存在；多个组件冗余验证耗尽预算；优化目标被 Medium 题主导 | p.6 §4.2；p.8–9 §4.4.1 |

### 二、待深挖 / 原文未说明的问题

1. **Algorithm 1 第 14 行的下标**：PASS@1(T_t) 衡量的是 H_{t-1}，却令 H_best ← H_t。实际实现是否不同？H_N 是否从未被评测？
2. **回滚后的 harness 未被单独评测**：每轮的分数变化混合了撤销旧修改与加入新修改两种效果，对逐条归因有多大影响？
3. **ATTRIBUTE 的判定规则**：预测要兑现多少才算有效？意外回归会不会触发回滚？`risk_tasks` 起什么作用？
4. **自动回滚与 Evolve Agent 自主决定如何分工？** 第 7 轮分数下降却选择 IMPROVE 而非回滚（p.30）。
5. **全文是否存在明确的 ROLLBACK + PIVOT 实例？**（未在原文中检索确认）
6. **随机预测基线如何构造？**（RQ3b 中"5 倍 / 2 倍"的倍数依赖这一点）
7. **统计可靠性**：只进化了一次（single campaign），没有方差或显著性检验；Hard 档 3.4 pp 约等于 1 道题；SWE-bench-verified 上仅 +0.4 pp。
8. **进化集与报告集是同一批 89 题**：是否存在选择偏差？RQ2 能否充分排除这个担忧？
9. **各表各图的 k 值**：原文称每张表都会写明，但未见。
10. **人工 harness 基线（OpenCode / Terminus-2 / Codex）用的基座模型**：未说明。
11. **跨模型迁移混入了运行参数的影响**：步数预算和超时按 GPT-5.4 high 拟合，能否用"每个基座单独进化"做对照？
12. **"换模型需重新适配"未被直接检验**：缺少"新基座上重新进化 vs. 直接迁移"的对比。

### 三、可做的实验设想（组件冲突定位，建议）

1. **从完整版去掉一个组件**的消融（4 个配置，只看 Hard 档）：找出拖后腿的组件。
2. **2⁴ 析因 / Shapley 分析**：量化两两交互项，k 加大到 4–8。
3. **组件内部按机制加开关**：收尾钩子、风险提醒、发布守卫、memory 中的验证类教训。
4. **检验"预算耗尽"假说**：统计轮数、验证命令数、超时率；放宽预算后看差距是否缩小。
5. **逐任务对照**：memory-only 通过而完整 AHE 失败的 Hard 任务，用 Debugger 对比分叉点。

建议顺序：1 → 4 → 2 → 3 / 5。

### 四、自检题（对话中提出、尚未作答）

- 第 6 行 ATTRIBUTE 为什么需要 T_{t-1} 和 T_t 两份轨迹？只给 T_t 会丢掉什么信息？
- 单组件替换实验能否单独支撑"system prompt 要靠另外三层才能落实"？还需补哪一组消融？

---

## 附录 B：候选问题：AHE 组件冲突的定位（最终保存版）

> 来源：Lin et al. 2026《Agentic Harness Engineering》（arXiv:2604.25850v4）精读对话。
> 页码指 PDF 页码。"原文"表示论文给出的内容；"建议"是原文之外的设想。

### 问题

作者有没有定位出 AHE 哪几个组件之间存在冲突、是什么冲突？如果要定位，可以怎么做？

### 一、作者已经说了什么（原文）

作者只做到"指出嫌疑组件"，没有用实验把冲突坐实。

- **嫌疑对象**：memory、middleware、system prompt 都在推动同一种收尾式验证，叠在一起会做重复检查，耗掉长程任务的步数（p.9，§4.4.1 "Components interact non-additively…" 段）。
- **tool 的副作用**：tool 内置的发布守卫在 Hard 档会让任务过早收尾（p.8，§4.4.1 "Each component owns a different failure surface." 段）。
- **非加性的证据**：三个单组件的正增益合计 +11.1 pp，完整 AHE 只有 +7.3 pp；Hard 档上 memory-only 为 63.3%，完整 AHE 为 53.3%（p.8，Table 3；p.9）。
- **没做的部分**：
  - 只做了"在种子上加入单个组件"的消融；
  - 没做两两组合，也没做"从完整 AHE 中去掉一个组件"；
  - 没有用轨迹统计（轮数、验证命令数、超时率）去证明"重复检查耗尽预算"这一机制。
- **作者的处理**：把"面向交互的进化"（interaction-aware evolution）留作未来工作（p.9）。

结论：哪几个组件冲突、通过什么机制冲突，目前仍是推断。

### 二、定位思路（建议）

**1. 从完整版去掉一个组件的消融（4 个配置）**

从完整 AHE 中每次去掉一个组件，只看 Hard 档。去掉后 Hard 档明显回升、接近 63.3% 的那个组件，就是拖后腿的嫌疑组件。这组实验成本最低，和论文已有的"单独加入"消融正好互补。

**2. 完整组合实验（2⁴ = 16 个配置），算交互项**

4 个组件开或关共 16 种组合。用析因分析算出两两交互效应（例如 memory × middleware 的交互项是否显著为负），或者算各组件的 Shapley 值。可以只在 30 道 Hard 题上跑以控制成本，但 k 要加大到 4–8，否则噪声会淹没交互效应。

**3. 在组件内部按机制拆开**

冲突更可能发生在具体机制之间，而不是整个组件之间，例如：
- middleware 的收尾钩子
- ExecutionRiskHints 的提醒
- tool 的发布守卫
- memory 中与验证相关的教训

给这些机制分别加开关，例如"完整 AHE 但关掉收尾钩子"，定位到具体哪条规则或哪段代码。

**4. 直接检验"预算耗尽"假说**
- **轨迹指标**：在 Hard 档统计各配置的轮数、验证类命令数、超时率、到达步数上限的比例。如果完整 AHE 明显高于 memory-only，就是直接证据。
- **干预实验**：把 Hard 档的步数预算和超时时间放宽（比如加倍）。差距随之缩小，说明问题确实是预算被挤占；差距不变，说明是别的机制，比如组件之间给出相互矛盾的指令。

**5. 逐任务对照**

找出 memory-only 通过、完整 AHE 失败的 Hard 任务，用 Agent Debugger 对比两边轨迹的分叉点，看是哪个组件在哪一步介入导致了失败（守卫拦截、冗余复查、过早提交等）。

### 三、推荐顺序

先做 1，再做 4，然后是 2，最后是 3 和 5。
- 1 便宜、直接，先锁定嫌疑组件；
- 4 确认冲突机制；
- 2 量化交互效应；
- 3 和 5 落到具体规则或代码。

这些结果可以直接作为论文所说的"面向交互的进化"的输入。
