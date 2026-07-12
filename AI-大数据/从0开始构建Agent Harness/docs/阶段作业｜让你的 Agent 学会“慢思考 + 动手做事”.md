# 阶段作业｜让你的 Agent 学会“慢思考 + 动手做事”

**作者：Tony Bai**

> 以输出倒逼输入，希望你可以自己动手做出专属的Agent

课程更新到第 06 讲，我们已经一起完成了：

* Main Loop（Agent 如何持续运行）
* 慢思考（Thinking 阶段拆分）
* Provider 抽象（接入大模型）
* Tool Registry（工具注册与分发）
* YOLO 工具哲学（用最少工具完成最多事情）

接下来，是时候把这些能力“合起来用一次”了👇

## 🎯 **作业目标（当前阶段）**

基于你已有的 Agent Harness，实现一个：  
👉 **“能慢思考、会用工具、能推进任务”的 Agent**

---

### 🟢 **Level 1（目前可以完成｜基于前6讲）**

让你的 Agent具备以下能力：

1️⃣ **慢思考（Thinking 阶段）**

* 在 Main Loop 中，显式区分 Thinking / Acting
* 先思考“下一步做什么”，再执行

2️⃣ **工具调用（Tool Registry）**

* 至少接入 1 个工具（如：文件读取 / 简单编辑）
* 通过统一注册与分发机制调用（而不是写死）

3️⃣ **YOLO 执行一次完整任务**

* 给定一个目标（如：读取一个文件并总结）
* Agent 自己决定调用工具并完成任务

👉 关键词：**慢思考 → 动手做 → 得到结果**

---

### 🟡 **Level 2（进阶｜可提前探索）**

在 Level 1 基础上尝试：

* 让 Thinking 阶段更结构化（比如输出“计划”而不是一句话）
* 支持多步执行（一次任务不一定一步完成）
* 尝试让 Agent 根据情况选择不同工具（而不是固定一个）

👉 关键词：**让 Agent 开始“连续行动”**

---

### 🔴 **Level 3（挑战｜对标业界Claude Code最佳实践）**

如果你有兴趣，可以结合 Claude Code 设计思路，尝试：

* 做一个更清晰的循环：Thinking → Acting → Observation → 再思考
* 让 Agent 根据执行结果调整下一步行为
* 尝试优化 Prompt 结构（思考 vs 执行分离）

👉 关键词：**让 Agent 开始“像一个系统在运转”**

**Claude Code 源码泄漏存档：**<https://github.com/FlyAIBox/civil-engineering-cloud-claude-code-source-v2.1.88>

---

## 📦 如何**提交 & 提交内容与格式**

按以下结构提交到本讲留言区（缺一不可）：

1️⃣ **代码仓库（必需）**

* 提供 GitHub / Gitee 链接
* 至少包含：Main Loop、Thinking、Tool 调用核心代码

2️⃣ **运行演示（必需）**

* 一段真实任务执行过程（建议贴日志）
* 示例格式：

  + 用户输入
  + Thinking 输出
  + Tool 调用
  + 最终结果

3️⃣ **设计说明（必需）**

请用 3～5 句话说明：

* 你的 Main Loop 是怎么设计的？
* Thinking 和 Acting 是如何拆分的？
* Tool Registry 是如何组织的？

👉 不要求长，但要讲“你的取舍”

---

## 🧪 **我们会重点关注什么？（评分标准）**

不是看“功能多不多”，而是看这 3 点：

✅ **结构是否清晰**（有没有真正拆出 Thinking）

✅ **是否解耦**（Tool 有没有通过 Registry 管理）

✅ **是否可扩展**（能不能自然加新工具 / 新步骤）

💬 **加分项（可选）**

* 有清晰的日志输出（能看到 Agent 的“思考过程”）
* 有失败案例（以及你是怎么修的）
* 有自己的小优化（哪怕很简单）

📌 **说明**

* 不要求一步到位，可以随着课程进度不断升级
* 欢迎把你的执行过程（Thinking / Tool 调用 / 输出）贴出来
* 我们会挑选一些实现做点评（重点看“设计”，不只是结果）

**课程结课后一周，我们会选择其中最佳的5个设计方案，给予奖励并在课程内公示。**

奖励：价值99元的课程阅码，现有课程任选

好记性不如烂笔头！如果你已经学完更新的内容，希望你可以好好完成以上作业，**以输出倒逼输入**，相信会有意想不到的收获～

也欢迎大家在评论区说说目前卡在哪一块，我们会优先在后面的内容里重点补上。

👉 **好课推荐**：[《AI原生开发工作流实战》](http://gk.link/a/12IZI)**Tony Bai 出品，已完结**  
——Claude Code + SDD，打造你的自动化 AI 工作流

![](_assets/972842_img_001.jpg)

---

## 精选评论

**向往**: 1️⃣ 代码仓库

GitHub: https://github.com/skytodmoon/go-tiny-claw

核心结构:

plaintext

go-tiny-claw/
├── internal/
│   ├── engine/loop.go        # Main Loop核心
│   ├── tools/registry.go     # 工具注册管理
│   ├── tools/read_file.go    # 读文件工具
│   ├── tools/write_file.go   # 写文件工具
│   ├── tools/edit_file.go    # 编辑文件工具
│   ├── tools/bash.go         # Bash命令工具
│   ├── schema/message.go     # 消息与工具调用定义
│   └── logger/logger.go      # 日志系统
├── cmd/claw/main.go          # 主入口
└── integration/level3_integration_test.go  # 集成测试

2️⃣ 运行演示

用户输入: 读取 README.md 并创建 SUMMARY.md 说明文档

执行日志: 3 轮循环完成任务，成功调用 read_file、write_file 工具，总回合 3 次，工具调用 2 次全成功

3️⃣ 设计说明

Main Loop 设计

四阶段循环：Thinking→Acting→Observation→Re-thinking

1. Thinking：分析状态、规划行动、选择工具
2. Acting：调用 LLM 获取指令并执行工具
3. Observation：收集工具结果并记录
4. Re-thinking：调整策略，决定循环 / 结束

Thinking/Acting 拆分

动态系统提示词实现分离，共享上下文，指令不同


🧪 评分标准对照

✅ 结构清晰：四阶段分离，Thinking 独立于 Acting

✅ 解耦：Registry 管理工具，引擎不依赖具体实现

✅ 可扩展：实现接口即可新增工具

💬 加分项

1. 日志系统：多级日志、彩色控制台 + 纯文本文件、模块化标识
2. 问题修复：移除日志颜色代码、注册 BashTool
3. 优化：动态上下文组装压缩、200K token 预算、执行摘要统计

📌 版本历史

v0.1 (2026-05-01)：Level1 完成慢思考 + 工具调用

v0.3 (2026-05-04)：Level3 完成四阶段循环 + 上下文管理

v0.3.1 (2026-05-04)：日志系统 + BashTool 支持

提交人: Bob 提交日期: 2026-05-04 课程: AI Agent 实战 Level3
具体可以看代码仓库下面HOMEWORD_SUBMISSION.md

> **作者回复**: 👍


---

**利**: 老师是否可以讲下反思相关的实践

> **作者回复**: reflection我认为算是harness高级实践。在专栏中，我们在第 14 讲（Error Recovery 注入模板）和第 15 讲（System Reminders 斩断死循环）其实已经通过被动触发的方式实现了基础的“反思”。但是更复杂的、更系统的反思机制，尚未覆盖。这个看看后续是否值得在加餐里补充聊一下[手动抱拳]


---

**利**: 老师，Reporter 和 hooks定位是一样的么？是否可以讲下hooks

> **作者回复**: Reporter 和 Hooks 在机制上是相似的，都在核心循环的特定节点触发，但在设计定位上是不同的。
> 
> Reporter是单向输出的。它的唯一职责是将引擎内部的状态翻译给“人类”或 UI 界面看（比如打印终端颜色、发飞书卡片）。它绝对不应该去干预引擎的执行逻辑。
> 
> Hooks（钩子） 是双向的。我们在第 16 讲实现的 Middleware 就是一种典型的Hook。它不仅能感知上下文，还能主动返回 (allowed bool, rejectReason string) 去阻断工具的执行，甚至可以在 PostToolUse 时强行改写工具的返回值。
> 
> 在工业级 Harness 中，Hooks 是系统扩展的灵魂，比如你可以在 PostCompact Hook 中把被压缩掉的历史保存到数据库中，或者在
> PreToolUse Hook 中自动续期某个 API Token。
> 


---

**Jaising**: 1、代码仓库
git@github.com:JaisingZ/java-tiny-claw.git

2、运行日志
PS D:\WorkSpace\java-tiny-claw> java -cp $cp com.jaising.agent.app.AgentApplication run --debug --max-steps 5 --prompt "请创建文件 target/Hello.java，用 bash 编译并运行它，要求输出 Hello，Java！"
2026/05/28 10:23:24 [Registry] 成功挂载工具: read_file ......
引擎启动，锁定工作区: D:\WorkSpace\java-tiny-claw
模型: Qwen/Qwen3-8B
慢思考模式 (Thinking Phase): false
可用工具: [read_file, write_file, edit_file, bash]

2026/05/28 10:23:25 ========== [Turn 1] 开始 ==========
[Engine][Phase 2] 恢复工具挂载，等待模型采取行动...
[Engine][Phase 2] 可见工具: [read_file, write_file, edit_file, bash]
......
2026/05/28 10:23:46   -> ✅ 工具执行成功 (返回 46 字节, 耗时 3ms)

2026/05/28 10:23:46 ========== [Turn 2] 开始 ==========
......
========== [Provider][ACTION] Parsed Decision ==========
ToolDecision tool=bash args={command=javac target/Hello.java; if ($LASTEXITCODE -eq 0) { java -cp target Hello } else { exit $LASTEXITCODE }}
[Engine] 模型请求调用工具: bash
2026/05/28 10:24:04   -> 🛠️ 执行工具: bash, 参数: {command=javac target/Hello.java; if ($LASTEXITCODE -eq 0) { java -cp target Hello } else { exit $LASTEXITCODE }}
2026/05/28 10:24:06   -> ✅ 工具执行成功 (返回 17 字节, 耗时 1604ms)

2026/05/28 10:24:06 ========== [Turn 3] 开始 ==========
......
========== [Provider][ACTION] Parsed Decision ==========
FinishDecision answer=最终回答：已成功编译并运行 target/Hello.java，输出结果为 "Hello，Java！"。
2026/05/28 10:24:27 [Engine] 模型未请求调用工具，任务宣告完成。
🤖 [最终回复]:
最终回答：已成功编译并运行 target/Hello.java，输出结果为 "Hello，Java！"。

3、设计说明
Task -> AgentEngine -> ModelProvider -> Decision
ModelProvider:
ThinkingDecision: 只记录思考，不执行工具
ToolDecision: Middleware 检查后交给 ToolRegistry 执行
FinishDecision: 保存最终答案并结束任务

ToolDecision -> ToolRegistry -> Tool(read_file/write_file/edit_file/bash) -> ToolResult
ToolResult -> AgentState observation -> 下一轮 Main Loop

> **作者回复**: 👍


---

**微风**: 老师加油更新呀

> **编辑回复**: 老师正在努力码字中 ✍️


---

**梅子**: 老师有没有学习交流群啊

> **作者回复**: 编辑老师那有一个微信交流群。


---

**shuff1e**: • 1️⃣ 代码仓库（必需）

  代码仓库：https://github.com/shuff1e/go-nanoclaw

  核心代码：
  - Main Loop：internal/agent/agent.go
  - Thinking / Provider 抽象：internal/brain/brain.go、internal/brain/openai.go、internal/brain/anthropic.go
  - Tool Registry / Tool 调用：internal/hands/registry.go、internal/hands/hands.go、internal/hands/builtins.go

  2️⃣ 运行演示（必需）

  用户输入：
  请执行一个 Level 3 Agent 演示任务：总结当前项目的 Agent 架构。

  Thinking 输出：
  先观察 workspace，确认项目结构和可读文件，而不是直接猜测。

  第一次 Tool 调用：
  list_workspace
  args: {"path":"."}

  第一次 Observation：
  目录中发现 README.md、cmd/、internal/ 等，README.md 最适合作为项目能力入口。

  Re-thinking 输出：
  根据目录结果，下一步应读取 README.md 来确认项目定位、架构和已有能力。

  第二次 Tool 调用：
  read_workspace_file
  args: {"path":"README.md"}

  第二次 Observation：
  README 显示项目是 Go 实现的轻量级 Agent Runtime，包含 Main Loop、Provider、Tool Registry、执行模式、安全策略、持久化和可观测性。

  最终结果：
  当前项目已经具备 Agent Runtime 的核心闭环：用户输入进入 Main Loop，模型先 Thinking，再通过 Tool Registry 选择工具执行，工具结果作为 Observation 回到上下文，模型再根据 Observation 调整下一步行动并输出最
  终总结。

  3️⃣ 设计说明（必需）

  我的 Main Loop 在 `internal/agent/agent.go` 中实现：用户输入会先构建 request frame，然后调用 Brain；如果 Brain 返回 ToolCalls，就进入工具循环，执行工具后把 `tool_result` 注入上下文，再次调用 Brain。

  Thinking 和 Acting 的拆分体现在模型响应中：模型先说明下一步计划，再通过 ToolCalls 发起 Acting；Observation 由工具结果提供，并驱动下一轮 Re-thinking。

  Tool Registry 由 `internal/hands` 管理，工具 schema 和执行入口统一放在 `Hands.GetToolSchemas` / `Hands.ExecuteStructured`，Main Loop 不依赖具体工具实现。

  我的取舍是保留通用 Agent Loop，不为单个任务写死流程，而是通过 Prompt 约束真实模型完成多轮观察和行动，从而展示 Level 3 的系统运转感。



---

**Geek_7919a1**: 更新有点慢啊


