# RQ1 数据可行性检查：开源 agent 项目里"因换模型而改 harness"的记录

> 检查日期：2026-10-20
> 方法：对 6 个项目做 bare clone（只取历史，不取文件内容），先按提交信息做关键词初筛，再按文件路径查看按模型区分的配置文件。
> **本次只做了关键词筛选和人工浏览，没有系统标注，下面的数字都是粗略值。**

## 1. 初筛结果

筛选条件：提交信息同时满足两项——提到具体模型名，并且提到 harness 相关词（如 prompt、tool、edit format、workaround、reasoning、template 等）。

| 项目 | 提交总数（不含 merge） | 命中 | 新增 | 修改 | 删除 | 含 workaround / hack / 特判 字样 |
|---|---|---|---|---|---|---|
| aider | 12,461 | 70 | 33 | 35 | 2 | 15 |
| OpenHands | 8,649 | 55 | 10 | 39 | 6 | 13 |
| SWE-agent | 2,076 | 4 | 1 | 3 | 0 | 2 |
| mini-swe-agent | 1,019 | 3 | 3 | 0 | 0 | 2 |
| cline | 7,547 | 278 | 80 | 186 | 12 | 63 |
| codex | 12,402 | 78 | 20 | 54 | 4 | 12 |
| **合计** | **44,154** | **488** | 147 | 317 | **24（约 5%）** | 107 |

"新增 / 修改 / 删除"是按提交标题里的动词粗分的，准确度有限。

## 2. 结构性证据：项目里存在按模型区分的 harness 文件

| 项目 | 文件 | 说明 |
|---|---|---|
| aider | `aider/resources/model-settings.yml` | 共 144 次提交改过这个文件；每个模型单独配置编辑格式以及 `overeager`、`lazy` 等行为开关 |
| cline | `src/core/prompts/system-prompt/variants/<家族>/` | 按家族划分的 system prompt 变体，包括 gpt-5、native-gpt-5、native-gpt-5-1、next-gen、gemini-3、glm、devstral、hermes、xs、generic 等；共 50 次提交改过 |
| codex | `codex-rs/core/gpt_5_1_prompt.md`、`gpt_5_2_prompt.md`、`gpt-5.1-codex-max_prompt.md`、`gpt-5.2-codex_prompt.md`、`gpt_5_codex_prompt.md` | 每个模型一份 prompt 文件；另有提交提到"移除退役模型时保留 migration prompts"（#44250，内容尚未查看） |

**这对研究的意义**：这些文件本身就是天然的"模型补丁"清单，可以按文件和条目追踪每条补丁什么时候加入、什么时候被修改或删除，再和模型发布日期对齐。

## 3. 典型记录（可直接作为论文中的例子，需逐条打开 diff 核实）

**为新模型加适配，属于 MSA 类**
- aider `8c982f83c` / `ceb81369e`（2025-08-07）：为 GPT-5 关闭 temperature，并强制使用 diff 编辑格式
- aider `291b456a4`（2024-09-12）：hack for o1-mini——不用 system prompt，不设 temperature
- aider `3bf20d4f`（2025-04-24）：把 Gemini 2.5 Pro 标为 `overeager: true`
- OpenHands `116ba199d`（2025-08-09）：GPT-5 不再使用简短版工具描述
- OpenHands `7810d8c4a`（2025-02-28）：为 Sonnet 3.7 改进 finish 工具
- OpenHands `a021045dc`（2024-12-26）：DeepSeek 的 function calling 强制使用字符串序列化
- cline `6d1bfc3a1`（2025-12-30）：为 Claude 和 Gemini 3 开启并行工具调用
- codex `8dcbd29ed`（2025-11-13）：为 gpt-5.1 更新 prompt

**删除或撤销过时的适配，即"清理"**
- aider `1fbb5079d`（2024-09-12）：unhack o1 mini（与上面的 hack 是同一天）
- aider `63d3dbcc`（2025-07-12）：移除 `reminder: sys`
- OpenHands `cff569745`（2025-07-08）：移除 Gemini 专用的 SWE-bench 模板
- OpenHands `4ce3b9094`（2024-11-08）：撤销"用 prompt 提醒 o1 生成 patch"这一改动
- OpenHands `75e54e355`（2025-11-05）：移除默认的 reasoning_effort，并修正 Gemini 的特判
- codex `f3b4a26f3`（2025-10-05）：为 gpt-5-codex 去掉 read-file 工具

**接口反复调整，对应"接口失配"机制**
- cline `45abe977e`（2025-10-28）：GPT-5 改用 diff 编辑工具 → `f01428884`（2025-12-17）：GPT-5+ 的 diff 编辑工具被替换为 APPLY_PATCH
- codex `6df8e3531`（2025-08-15）：新增 apply_patch 工具
- 这与 SSA 的发现一致："GPT 家族偏好 apply_patch" [笔记]

## 4. 初步观察（均为假设，待系统标注验证）

1. **按模型打补丁很普遍**，而且已经形成固定的代码结构：模型设置文件、按家族划分的 prompt 变体、每个模型一份 prompt 文件。
2. **删除远少于新增**：按关键词粗分，删除约占 5%。这初步支持"只加不删，过时补丁会积累"的假设，但粗分方法很不可靠。
3. **噪声很多**：大量命中其实是 API 兼容性改动（temperature、max_tokens、价格、上下文窗口），并不改变 agent 的行为。标注时必须把"API 兼容"和"行为适配"分开，这个区分本身也可以成为 RQ1 的发现之一。

## 5. SE 方向查新（2026-10-20）

| 工作 | 状态 | 做了什么 | 与 RQ1–2 的差别 | 证据等级 |
|---|---|---|---|---|
| **When the Model Retires**（2609.31288） | arXiv 预印本，cs.SE | 挖掘了 22,555 条提交，这些提交都是把应用从已退役的 OpenAI、Anthropic、Google 模型迁走；82% 的迁移发生在模型停服之后；94% 的应用把模型名写死在代码里；纯 prompt 应用迁移时新增代码的中位数为 6 行 | **最接近**。但它研究的是**被迫迁移**（模型退役），对象是一般的 LLM 应用，不是 coding agent 的 harness；只统计改动量，没有分析具体改了哪些适配，也没有看过时的适配有没有被删 | 摘要 |
| **Don't Blame the LLM**（2607.03691） | arXiv 预印本，cs.SE | 研究 5 个 harness 的发布历史；固定模型，对 Qwen Code 的 35 个连续版本逐一跑 SWE-bench | 方向正好相反：它**固定模型、变 harness**，RQ1 是**固定 harness、变模型**。可以当作互补的工作来引用 | 摘要 |
| **Self-Admitted Technical Debt in LLM Software**（2601.06266） | SANER 2026 | 对比了 477 个仓库，归纳出一类新的技术债 "Model-Stack Workaround Debt" | "为模型打补丁会形成技术债"这个**概念已经有人提出**；但没有研究模型升级时这些补丁会不会被删 | 摘要（摘要页没给出这类债的定义） |
| **Prompting in the Wild**（2412.17298） | MSR 2025 | 研究 243 个仓库里 1,262 次 prompt 修改；以新增和修改为主，只有 21.9% 在提交信息里写明了改动 | 研究的是一般 prompt 的演变，不区分这次改动是不是由模型升级触发的 | 搜索摘录 |
| PromptDebt（2509.20497） | ACM 2025 | 研究 LLM 项目中开发者自己承认的技术债，prompt 设计是主要来源 | 同上，没有涉及模型升级 | 搜索摘录 |
| How Do Developers Maintain and Evolve Their Agents' Instructions?（2606.25257） | arXiv | 研究 CLAUDE.md、AGENTS.md 这类说明文件的维护 | 研究对象是用户写给 agent 的说明，不是 harness 内部按模型区分的适配 | 搜索摘录 |
| Harness Engineering: A Source-Code Study of Eleven Systems（2609.00006） | arXiv | 对 11 个 harness 做源码审计，把其中 8 个间隔一个季度前后对比；发现行为策略在从 prompt 文本转向配置 | 摘要里没有分析按模型区分的代码，也没有分析和模型发布相关的改动 | 摘要 |

**结论**

- **适合投 SE 会议**：这个话题在 SE 社区很活跃，已有 MSR 2025、SANER 2026，以及多篇 cs.SE 预印本。
- **原来 RQ1 的说法要收窄**：
  - "把 LLM 升级当作依赖迁移来研究"已被 *When the Model Retires* 占了；
  - "为模型打补丁会形成技术债"已被 SANER 2026 那篇占了。
- **在本次检索范围内仍然空着的部分**：
  1. coding agent 的 harness 中，**按模型区分的行为适配**（第 2 节那些文件）在模型发布前后是怎么加、改、删的；
  2. 研究对象是**主动升级到同家族新版本**，不是模型退役后的被迫迁移；
  3. 过时的适配会不会被清理，还是越积越多；
  4. 能不能在升级之前预测哪些适配会过时（即 B）。
- **置信度：中**。还需要读两篇论文的正文才能定下来：*When the Model Retires* 有没有对迁移时改了什么做分类（比如 prompt 和参数有没有分开统计）；SANER 2026 那篇怎么定义 "Model-Stack Workaround Debt"，以及有没有研究这类债是什么时候被偿还的。

## 6. 局限与下一步

**局限**
- **召回率未知**：很多提交不会在信息里写模型名，关键词会漏掉。改用路径追踪（第 2 节的那些文件）会更可靠。
- **"删除"有误判**：cline 的变体目录在 2026-05-25 的 "Move vscode to apps" 中显示为删除，实际只是移动目录；codex 的不少删除是重构。
- 本次只覆盖 6 个项目。

**下一步**
1. 以第 2 节的文件为主，逐条追踪补丁的加入、修改、删除，并对齐模型发布日期。
2. 抽 100 条命中记录，由两人独立标注（行为适配 / API 兼容 / 无关；新增 / 修改 / 删除；MSA / HLR），计算一致性 κ。
3. 扩展到更多项目，例如 OpenCode、Goose、Roo Code、Continue；纳入前先确认这些仓库存在按模型区分的配置。
