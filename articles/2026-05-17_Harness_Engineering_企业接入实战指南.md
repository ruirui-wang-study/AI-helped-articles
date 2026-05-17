# Harness Engineering 企业接入实战指南：从 OpenAI 到海致，大厂都在用的 AI 驾驭方法论

> 文章来源：来财 AI 学习笔记  
> 适合读者：技术负责人、架构师、正在尝试将 AI Agent 引入生产环境的开发者  
> 阅读时间：约 8 分钟

---

## 一、先回答一个扎心的问题

你的团队把 GPT-4、Claude、DeepSeek 都接上了，RAG 知识库也搭了，Prompt 写了十几个版本。结果呢？

Agent 写了两天代码，把 CI 搞崩了三次。  
Agent 生成了合规文档，但引用的全是过时的政策条款。  
Agent 自动回复了客户邮件，态度热情，但把价格报错了。

问题出在哪？不是模型不够聪明——OpenAI 自己给同一款模型换了套运行环境，分数就从 52.8% 飙到了 66.5%，排名从全球第 30 跳到第 5。

**你的瓶颈不在模型，在环境。**

这个结论引出了 2026 年 AI 工程领域最核心的新范式：**Harness Engineering（驾驭工程）**。

---

## 二、什么是 Harness Engineering？三个跃迁讲清楚

要理解 Harness Engineering，得先看我们是怎么走到这一步的。

| 阶段 | 核心问题 | 工程师角色 |
|------|---------|-----------|
| Prompt Engineering (2023-2024) | 该怎么问？ | 写指令的人 |
| Context Engineering (2025) | 该让模型看到什么？ | 搭信息环境的人 |
| Harness Engineering (2026-) | 整个环境该怎么运作？ | 设计运行系统的人 |

Harness Engineering 的核心公式只有一行：

```
Agent = Model + Harness
```

模型负责推理，Harness 负责模型之外的一切——工具调用、上下文管理、权限控制、状态持久化、错误恢复、可观测性。

**如果 LLM 是 CPU，那么 Harness 就是操作系统。** CPU 再强，没有操作系统管理内存、调度进程、处理中断，它什么都干不了。

这个概念由 HashiCorp 联合创始人 Mitchell Hashimoto 在 2026 年 2 月 5 日正式命名。六天后，OpenAI 发布百万行代码实验报告正式采用。Martin Fowler 随即在 ThoughtWorks 技术博客深度分析。一个月内，这个词席卷了全球工程圈。

---

## 三、大厂们在做什么？五个框架对比

### 1. OpenAI — Codex Harness（百万行代码实验）

**背景：** 2025 年 8 月到 2026 年 2 月，3 名工程师，0 行手写代码，产出 100 万行生产级代码，合并 1500 个 PR。

**核心设计：**

- **层级依赖模型：** Types → Config → Repo → Service → Runtime → UI，下层不能反向依赖上层。编码为自定义 Linter，CI 强制阻断。
- **"三明治"推理预算：** 规划和验证阶段用高推理强度，代码实现阶段用中等强度，平衡深度与速度。
- **"垃圾收集日"：** 每天下午 4-5 点，工程师不写新功能，专门审视 Agent 当天生成的代码，识别坏模式并编译成自动化检查规则。

**一句话总结：** 规则编码为系统级阻断，从根本上阻止违规。

### 2. Anthropic — 长期运行 Agent 的 Harness 设计

**背景：** 2025 年 5 月尝试让 Claude 从零开发完整 Web 应用，发现长任务的核心问题是**上下文衰减**。

**核心设计：**

- **Context Reset（上下文重置）：** 不是压缩历史，而是直接换一条新 Agent，通过结构化的"交接文件"传递状态。比摘要压缩更激进。
- **工具定义即约束：** 每个工具的描述不仅写功能，还写什么时候不该用它。
- **并行 Claude 实验：** 使用 Git 锁文件做并发隔离，多个 Worker 不能抢同一张"许可单"。

**一句话总结：** 长任务的核心瓶颈不是模型能力，而是上下文窗口的物理极限。

### 3. LangChain — Harness 架构实证研究

**背景：** 2026 年 3 月发表《The Anatomy of an Agent Harness》，用 Terminal Bench 2.0 做控制变量实验。

**实验数据：** 同一模型（gpt-5.2-codex），只改 Harness（系统提示词结构 + 工具描述方式 + 中间件），得分从 52.8% 升至 66.5%，排名从第 30 跃升至第 5。

**三项核心改进：**
1. **PreCompletionChecklistMiddleware：** Agent 完成任务前强制拦截，要求它对照原始需求逐项验证。
2. **LocalContextMiddleware：** 启动时自动注入工作目录、环境信息，帮 Agent 省去"探索环境"的时间。
3. **LoopDetectionMiddleware：** 追踪文件编辑次数，超过阈值自动提示"建议换个思路"。

**一句话总结：** 中间件比模型权重更值钱。

### 4. Cursor — 并发 Agent 的规模灾难

**背景：** 尝试让几百个 Agent 共享一个大型项目，发现了恐怖的并发问题。

**问题数据：**
- 20 个 Agent 同时工作 → 有效吞吐量仅相当于 2-3 个 Agent
- 大量 Agent 抢不到核心代码 → 开始改注释、调整空格（假装自己在工作）
- "锁机制"成为了瓶颈

**解决方案（规划者 - 执行者门控模式）：**
一个主 Agent 充当工头，派出多个 Worker，走"调研→综合→实现→验证"四步流水线。Worker 要执行危险操作（删文件、跑脚本）必须通过"邮箱"向工头请求许可。

**一句话总结：** Agent 并发不是越多越快，需要精密的编排系统。

### 5. 海致科技 — 产业级 Harness（国内实践）

**背景：** 面向金融、政务等强监管行业的 Harness Engineering 落地，解决 B 端特有的"生产悬崖"问题。

**核心问题：** B 端企业的 IT 环境是几十年积累的"技术债博物馆"——ERP、CRM、OA、财务系统、自研中台……跨系统数据对接的工作量占总项目 60% 以上。

**解决方案（三层 Harness）：**
1. **知识层：** 企业知识图谱 + 动态文档检索，解决 15%+ 的领域幻觉率
2. **编排层：** 异构系统封装为 Agent 可调用的标准化工具
3. **安全层：** 决策可追溯、过程可审计，满足监管要求

**一句话总结：** 国内企业接入 Harness Engineering，最难的不是 AI，是 IT 基础设施的复杂异构！

---

## 四、企业怎么落地？三阶段路线图

综合 OpenAI、Anthropic、LangChain 和 Martin Fowler 的实践，企业落地 Harness Engineering 推荐渐进式三阶段路线图：

### Phase 1：信息层（1-2 天）

**目标：** 从"百科全书式文档"变成"地图式索引"。

**怎么做：**
- 把你的 AGENTS.md、README、架构文档拆成小块
- 每份文档头部带元信息（最后更新时间、覆盖范围、关联模块）
- 让 Agent 根据当前任务按需检索，而非一股脑全加载

**验收标准：** Agent 能在 2 次检索内找到正确文档。

### Phase 2：约束层（3-5 天）

**目标：** 把"严禁 XXX"变成硬性检查。

**怎么做：**
- 识别你团队最重要的 3-5 条架构规则
- 编码为 Linter 规则或 CI Hook
- 失败信息包含"为什么违规"和"正确做法"，让 Agent 能自动修正

**验收标准：** 违规代码会在 CI 被阻断，Agent 自己就能改好。

### Phase 3：自动化层（1-2 周）

**目标：** 让系统能自动发现坏模式并加固。

**怎么做：**
- 建立父-子 Agent 架构（规划者 + 执行者）
- 每天固定时间执行"技术债扫描"（参考 OpenAI 的垃圾收集日）
- Agent 每翻一次车，就往 Harness 里加一个检查点

**验收标准：** 同样的错误不会重复出现两次。

---

## 五、给技术负责人的行动建议

**1. 不要一上来就搞大架构**

从最小可行 Harness 开始。选一个你最头疼的 Agent 翻车场景，写一条 Linter 规则或者一个中间件，今天就上线。

**2. 沉默是成功，噪声是失败**

这是最反直觉但也最重要的一条。测试全通过时不要给 Agent 输出 4000 行日志——给它一个 ✓ 就够了。失败时才给详细错误信息。

**3. 让 Harness 成为活的系统**

不是写一次就完了。Mitchell Hashimoto 的 Ghostty 项目的 AGENTS.md 只有几百行，每一行对应一个历史上 Agent 犯过的错。你的 Harness 应该随着每次翻车不断生长。

**4. 你的工程师角色正在变化**

2026 年的工程师不再只是"写代码的人"。你是设计 Agent 运行环境的人——决定 Agent 能看到什么、能做什么、犯错后怎么纠正。这套技能在模型能力快速趋同的当下，才是真正拉开生产力差距的地方。

> "为了获得更高的 AI 自主性，运行时必须受到更严格的约束。增加信任需要的不是更多自由，而是更多限制。"  
> — Birgitta Böckeler, ThoughtWorks

---

## 参考与延伸阅读

- OpenAI. (2026). Harness engineering: leveraging Codex in an agent-first world.
- Anthropic. (2025). Effective harnesses for long-running agents.
- Anthropic. (2025). Effective context engineering for AI agents.
- LangChain. (2026). The Anatomy of an Agent Harness.
- Mitchell Hashimoto. (2026). Harness Engineering (Personal Blog).
- Cursor. (2026). Scaling long-running autonomous coding.
- 海致科技. (2026). Harness 时代：海致科技如何定义 AI 的"驾驭工程".
- Martin Fowler / ThoughtWorks. (2026). Harness Engineering Analysis.
