# 03｜慢思考与自省：在 ReAct 循环中剥离独立的 Thinking 阶段

**作者：Tony Bai**



> 我们来做慢思考这个部分

你好，我是Tony Bai。欢迎来到《从0开始构建 Agent Harness》专栏的第三讲。

在上一讲中，我们从学术界的 ReAct 论文出发，用 Go 语言手写了 `go-tiny-claw` 的心脏——Main Loop。我们成功地让一个“假肢”模型在“思考-行动-观察”的无尽循环中跑了起来。

看似我们已经掌握了智能体运行的终极密码。但是，如果现在就把真实的前沿大模型（如 Claude 4.x 或 GPT-5.x）接入这个基础循环，并且给它挂载上能够修改本地代码的 `edit` 和 `bash` 工具，你会遭遇一个极其普遍但又令人抓狂的现象：**大模型变得极其“冲动”。**

当你给它一个复杂的任务：“帮我分析整个订单模块的并发逻辑，并重构它”时。它可能连其他文件都没看，瞬间就发出了一个 `edit` 工具的调用请求，去盲目修改它看到的第一个文件。

为什么会这样？在这一讲中，我们将深入探讨大模型在面对工具时的“认知陷阱”，并学习顶级驾驭工程（Harness Engineering）是如何通过**在底层架构上强制剥离出一个独立的 Thinking（慢思考）阶段**，来让 Agent 从“莽夫”蜕变为“架构师”的。

## 大模型的“快思考”与“慢思考”

诺贝尔经济学奖得主丹尼尔·卡尼曼在《思考，快与慢》中提出，人类的大脑包含两套系统：

* **系统 1（快思考）**：直觉的、本能的、自动的。比如你看到 `2+2` 立刻就能想到 `4`。
* **系统 2（慢思考）**：逻辑的、深思熟虑的、需要消耗精力的。比如让你计算 `17 × 24`，你必须拿出一张草稿纸，一步步推导。

大语言模型的本质是预测下一个 Token。从架构上看，它天生就是一个完美的“系统 1”。

当你向大模型发起一次普通的 API 请求时，它只能顺着当前的上下文，凭借概率直觉“一口气”把答案生成出来。如果问题极其复杂，它无法在生成第一个字之前，在脑海里预演几十步的完整计划。

### 提示词工程的破产与驾驭工程的解法

为了激发大模型的“系统 2（慢思考）”，学术界发明了 **Chain of Thought（思维链，CoT）** 技术。我们会在提示词里加上一句咒语：“Let’s think step by step（让我们一步步思考）*”*。这就相当于给了大模型一张“草稿纸”，让它在输出最终答案前，先把中间推理过程写出来。

但在 Agent 的工具调用（Function Calling）场景下，这种单纯的提示词工程彻底破产了。因为 AI 工程师在构建长程 Coding Agent 时，发现了一个致命的规律：

> *“When tools are available, models tend to act quickly rather than think deeply.”*  
> （当工具可用时，模型倾向于迅速采取行动，而不是深入思考。）

如果你在系统提示词里写：“请你先仔细规划，然后再调用工具”。大模型往往会无视这句话。只要它在上下文的 Schema 里看到了诱人的 `bash` 或 `edit` 工具，它的预测概率就会瞬间坍塌，转而生成一段 JSON 参数去调用工具。

如何解决呢？既然提示词管不住它的“手”，那我们就用架构锁住它的“手”！驾驭工程（Harness Engineering）给出的解法是：**机制决定行为**。

在每一次大模型采取行动前，Harness 引擎会向它发起一次没有附带任何工具 Schema 的纯文本 API 请求。在这个绝对没有工具诱惑的“小黑屋”里，模型别无选择，只能乖乖地输出一段纯文本的深度推理与规划。

等它想清楚了，Harness 会把这段推理记录追加到上下文中，然后再发起第二次附带工具的请求，让它去执行。

这就是工业级 Agent 循环中的 Two-Stage ReAct（两阶段 ReAct 循环）。

### 架构演进：Two-Stage ReAct 循环

我们可以用一张示意图，对比一下我们在上一讲实现的基础循环，与今天我们要重构的“两阶段循环”的差异：

![图片](_assets/967578_img_001.jpg)  
通过这种物理层面的隔离，我们将 Agent 的“谋”与“动”彻底分开了。

## 代码实战：在 Main Loop 中剥离 Thinking 阶段

理解了理论，落实到 Go 语言的代码层面其实非常简单。

得益于我们在上一讲中设计的 `LLMProvider` 接口极其纯粹（`Generate(ctx, msgs, tools)`），我们只需要在传参时动一点手脚：如果不传 `tools` 数组，大模型自然就只能输出文本了。

### 目录结构回顾与更新

我们今天的核心修改全部集中在 `internal/engine/loop.go` 和测试入口 `cmd/claw/main.go` 中。

```plain
go-tiny-claw/
├── cmd/
│   └── claw/
│       └── main.go          # 【修改】升级 Mock Provider，支持区分推理和行动请求
├── internal/
│   ├── engine/
│   │   └── loop.go          # 【核心修改】引入 EnableThinking 开关，重构为两阶段循环
│   ├── provider/
│   │   └── interface.go     # 保持不变
│   ├── schema/
│   │   └── message.go       # 保持不变
│   └── tools/
│       └── registry.go      # 保持不变
├── go.mod
└── README.md
```

### 第 1 步：改造 AgentEngine，增加思维开关

打开 `internal/engine/loop.go`，我们为引擎增加一个 `EnableThinking` 的配置项。在处理简单任务（比如只问个天气）时，我们可以关闭它以节省 Token；但在处理复杂代码任务时，我们将其打开。

```go
// internal/engine/loop.go
package engine

import (
    "context"
    "fmt"
    "log"

    "github.com/yourname/go-tiny-claw/internal/provider"
    "github.com/yourname/go-tiny-claw/internal/schema"
    "github.com/yourname/go-tiny-claw/internal/tools"
)

type AgentEngine struct {
    provider       provider.LLMProvider
    registry       tools.Registry
    WorkDir        string
    EnableThinking bool // 【新增】慢思考模式开关
}

func NewAgentEngine(p provider.LLMProvider, r tools.Registry, workDir string, enableThinking bool) *AgentEngine {
    return &AgentEngine{
        provider:       p,
        registry:       r,
        WorkDir:        workDir,
        EnableThinking: enableThinking,
    }
}
```

### 第 2 步：重构 Main Loop，实现两阶段流转

接下来，我们修改 `Run` 方法。在原来的向大模型发起请求的地方，将其拆分为明显的两步：Phase 1（Thinking）和 Phase 2（Action）。

```go
// internal/engine/loop.go (续)

func (e *AgentEngine) Run(ctx context.Context, userPrompt string) error {
    log.Printf("[Engine] 引擎启动，锁定工作区: %s\n", e.WorkDir)
    log.Printf("[Engine] 慢思考模式 (Thinking Phase): %v\n", e.EnableThinking)

    contextHistory := []schema.Message{
        {
            Role:    schema.RoleSystem,
            Content: "You are go-tiny-claw, an expert coding assistant. You have full access to tools in the workspace.",
        },
        {
            Role:    schema.RoleUser,
            Content: userPrompt,
        },
    }

    turnCount := 0

    for {
        turnCount++
        log.Printf("\n========== [Turn %d] 开始 ==========\n", turnCount)

        // 获取当前挂载的所有工具定义
        availableTools := e.registry.GetAvailableTools()

        // ====================================================================
        // Phase 1: 慢思考阶段 (Thinking) - 剥夺工具，强制规划
        // ====================================================================
        if e.EnableThinking {
            log.Println("[Engine][Phase 1] 剥夺工具访问权，强制进入慢思考与规划阶段...")

            // 核心机制：传入的 availableTools 为 nil！
            // 大模型看不到任何 JSON Schema，被迫只能输出纯文本的思考过程。
            thinkResp, err := e.provider.Generate(ctx, contextHistory, nil)
            if err != nil {
                return fmt.Errorf("Thinking 阶段生成失败: %w", err)
            }

            // 如果模型输出了思考过程，我们将其作为 Assistant 消息追加到上下文中
            if thinkResp.Content != "" {
                fmt.Printf("🧠 [内部思考 Trace]: %s\n", thinkResp.Content)
                contextHistory = append(contextHistory, *thinkResp)
            }
        }

        // ====================================================================
        // Phase 2: 行动阶段 (Action) - 恢复工具，顺着规划执行
        // ====================================================================
        log.Println("[Engine][Phase 2] 恢复工具挂载，等待模型采取行动...")

        // 此时的 contextHistory 中已经包含了上一阶段模型自己的 Thinking Trace。
        // 模型会顺着自己的逻辑，结合恢复的 availableTools 发起精准的工具调用。
        actionResp, err := e.provider.Generate(ctx, contextHistory, availableTools)
        if err != nil {
            return fmt.Errorf("Action 阶段生成失败: %w", err)
        }

        contextHistory = append(contextHistory, *actionResp)

        if actionResp.Content != "" {
            fmt.Printf("🤖 [对外回复]: %s\n", actionResp.Content)
        }

        // ====================================================================
        // 退出与执行逻辑 (与上一讲保持一致)
        // ====================================================================
        if len(actionResp.ToolCalls) == 0 {
            log.Println("[Engine] 模型未请求调用工具，任务宣告完成。")
            break
        }

        log.Printf("[Engine] 模型请求调用 %d 个工具...\n", len(actionResp.ToolCalls))

        for _, toolCall := range actionResp.ToolCalls {
            log.Printf("  -> 🛠️ 执行工具: %s, 参数: %s\n", toolCall.Name, string(toolCall.Arguments))

            result := e.registry.Execute(ctx, toolCall)

            if result.IsError {
                log.Printf("  -> ❌ 工具执行报错: %s\n", result.Output)
            } else {
                log.Printf("  -> ✅ 工具执行成功 (返回 %d 字节)\n", len(result.Output))
            }

            // 将工具执行的观察结果追加到 Context，准备进入下一轮
            observationMsg := schema.Message{
                Role:       schema.RoleUser,
                Content:    result.Output,
                ToolCallID: toolCall.ID,
            }
            contextHistory = append(contextHistory, observationMsg)
        }
    }

    return nil
}
```

请注意看 Phase 1 中的 `e.provider.Generate(ctx, contextHistory, nil)`。这个 `nil` 就是驾驭工程中四两拨千斤的魔法。

由于大模型是自回归（Auto-regressive）的，当它在 Phase 1 自己输出了“我应该先用 bash 看看系统日志”这句话并被存入 `contextHistory` 后，在 Phase 2 时，它自己看到自己说的话，就会顺理成章、毫无幻觉地生成一个调用 `bash` 的 JSON。这极大降低了模型瞎调工具的概率。

## 运行与验证：升级 Mock Provider

为了验证这个两阶段架构，我们需要升级一下 `cmd/claw/main.go` 中的 `mockProvider`。

我们要让这个“假肢”变得智能一点：当它发现 `availableTools == nil` 时，它必须假装在进行慢思考；当它发现传入了工具时，它才返回 `ToolCall`。

打开 `cmd/claw/main.go`：

```go
package main

import (
    "context"
    "log"
    "os"

    "github.com/yourname/go-tiny-claw/internal/engine"
    "github.com/yourname/go-tiny-claw/internal/provider"
    "github.com/yourname/go-tiny-claw/internal/schema"
    "github.com/yourname/go-tiny-claw/internal/tools"
)

// 升级版 Mock Provider
type mockProvider struct {
    turn int
}

func (m *mockProvider) Generate(ctx context.Context, msgs []schema.Message, tools []schema.ToolDefinition) (*schema.Message, error) {
    // 如果工具列表为空，说明这是引擎发起的 Phase 1: Thinking 阶段
    if len(tools) == 0 {
        return &schema.Message{
            Role:    schema.RoleAssistant,
            Content: "【推理中】目标是检查文件。我不能直接盲猜，我需要先调用 bash 工具执行 ls 命令，看看当前目录下有什么，然后再做定夺。",
        }, nil
    }

    // 如果工具列表不为空，说明这是 Phase 2: Action 阶段
    m.turn++
    if m.turn == 1 {
        // 第一轮 Action：顺着刚才的 Thinking，精准调用工具
        return &schema.Message{
            Role:    schema.RoleAssistant,
            Content: "我要执行我刚才计划的步骤了。",
            ToolCalls: []schema.ToolCall{
                {ID: "call_123", Name: "bash", Arguments: []byte(`{"command": "ls -la"}`)},
            },
        }, nil
    }

    // 第二轮 Action：直接总结退出
    return &schema.Message{
        Role:    schema.RoleAssistant,
        Content: "根据工具返回的结果，我看到了 main.go，任务圆满完成！",
    }, nil
}

type mockRegistry struct{}

func (m *mockRegistry) GetAvailableTools() []schema.ToolDefinition { 
    // 为了让 Phase 2 能检测到工具，这里返回一个伪造的工具定义数组
    return []schema.ToolDefinition{{Name: "bash"}} 
}

func (m *mockRegistry) Execute(ctx context.Context, call schema.ToolCall) schema.ToolResult {
    return schema.ToolResult{
        ToolCallID: call.ID,
        Output:     "-rw-r--r--  1 user group  234 Oct 24 10:00 main.go\n",
        IsError:    false,
    }
}

func main() {
    workDir, _ := os.Getwd()

    p := &mockProvider{}
    r := &mockRegistry{}

    // 实例化引擎，开启 EnableThinking = true
    eng := engine.NewAgentEngine(p, r, workDir, true)

    err := eng.Run(context.Background(), "帮我检查当前目录的文件")
    if err != nil {
        log.Fatalf("引擎崩溃: %v", err)
    }
}
```

### 运行步骤与预期输出

在终端中执行启动命令：

```bash
go run cmd/claw/main.go
```

你将看到如下极具层次感的执行日志：

```plain
2026/03/29 14:37:57 [Engine] 引擎启动，锁定工作区: build-agent-harness-from-scratch/part1/source/ch03/go-tiny-claw
2026/03/29 14:37:57 [Engine] 慢思考模式 (Thinking Phase): true
2026/03/29 14:37:57 
========== [Turn 1] 开始 ==========
2026/03/29 14:37:57 [Engine][Phase 1] 剥夺工具访问权，强制进入慢思考与规划阶段...
🧠 [内部思考 Trace]: 【推理中】目标是检查文件。我不能直接盲猜，我需要先调用 bash 工具执行 ls 命令，看看当前目录下有什么，然后再做定夺。
2026/03/29 14:37:57 [Engine][Phase 2] 恢复工具挂载，等待模型采取行动...
🤖 [对外回复]: 我要执行我刚才计划的步骤了。
2026/03/29 14:37:57 [Engine] 模型请求调用 1 个工具...
2026/03/29 14:37:57   -> 🛠️ 执行工具: bash, 参数: {"command": "ls -la"}
2026/03/29 14:37:57   -> ✅ 工具执行成功 (返回 51 字节)
2026/03/29 14:37:57 
========== [Turn 2] 开始 ==========
2026/03/29 14:37:57 [Engine][Phase 1] 剥夺工具访问权，强制进入慢思考与规划阶段...
🧠 [内部思考 Trace]: 【推理中】目标是检查文件。我不能直接盲猜，我需要先调用 bash 工具执行 ls 命令，看看当前目录下有什么，然后再做定夺。
2026/03/29 14:37:57 [Engine][Phase 2] 恢复工具挂载，等待模型采取行动...
🤖 [对外回复]: 根据工具返回的结果，我看到了 main.go，任务圆满完成！
2026/03/29 14:37:57 [Engine] 模型未请求调用工具，任务宣告完成。
```

注：在 Turn 2 中的 Thinking 输出与 Turn 1 一致，是因为我们的 mock 逻辑比较简单。在真实的 LLM 接入后，它会在 Turn 2 思考：“我已经看到了结果，下一步应该总结输出了”。

看到这里，你是否体会到了 Harness 驾驭工程的魅力？我们没有在 Prompt 里写一句“求求你先思考再行动”，仅仅是通过**在架构层面做了一次精妙的剥离拦截**，大模型的行为范式就发生了根本性的翻覆。

### 反思：静态慢思考的局限性

细心的同学可能已经发现，我们目前在 `AgentEngine` 中引入的 `EnableThinking` 是一个**静态的全局开关**。

只要在启动时传了 `true`，大模型在未来的几十轮（Turn）交互中，每一轮都必须先被关进“小黑屋”强制做一次纯文本的推理与规划（Phase 1），然后再去执行动作（Phase 2）。

对于诸如“帮我写一个 Web 服务器”这样复杂的开局任务，强制慢思考确实能极大地提升成功率。但是，当任务进行到中后期，大模型仅仅只是需要逐个完成一些简单微小的任务时，强制它做一次慢思考，不仅显得极其啰嗦，还会白白浪费大量的 API Token 成本和响应时间。

那么，如何才能让大模型在宏观上拥有长程的规划能力，而在微观执行上又能在简单任务时做到“免思考、极速响应”呢？

这个解法，将在本专栏的 **第 13 讲（记忆沉淀：引入 Plan Mode）** 中为你揭晓！我们将探讨“慢思考”与“Plan 模式”的本质区别。现在，让我们带着这个疑问，继续打磨我们的基础引擎。

## 本讲小结

今天，我们在上一讲基础 Main Loop 的框架内，完成了一次非常高级的架构重构。

1. **洞察模型的“冲动”陷阱**：我们从理论上揭示了当工具 Schema 存在于请求上下文中时，大模型（作为系统 1）天生倾向于立刻采取行动，而忽视全局规划。提示词层面的约束对此收效甚微。
2. **两阶段 ReAct（Two-Stage ReAct）**：借鉴业界顶级框架的实践，我们在代码层面上强制将 `Thinking` 和 `Action` 物理拆分。
3. **驾驭工程的四两拨千斤**：在具体实现时，我们通过在第一次调用 Provider 时传入 `availableTools = nil`，极其优雅地“逼迫”大模型输出纯文本的思考轨迹（Thinking Trace），并将其固化为接下来的上下文，利用其自回归特性引导精准的行动。

现在，这台引擎不仅有强健的心跳，更拥有了深邃的思考能力。但是，一直依赖 Mock 的假肢是没有成就感的。

在下一讲中，我们将正式拔掉这个 `mockProvider`，通过抽象的适配层，全面接入官方的 **Claude 大模型**和 **OpenAI兼容的国内大模型**。届时，我们将用真实的 AI 大脑，来检验这套两阶段引擎在面对真实环境时的澎湃算力！

> 注：本讲的示例代码，可以在[这里](https://github.com/bigwhite/publication/tree/master/column/timegeek/build-agent-harness-from-scratch/ch03)下载。

## 思考题

在当前的实现中，我们在 Phase 1（Thinking）得到的 `thinkResp` 被作为一条普通的 `schema.RoleAssistant` 消息追加到了上下文中。

但在真实的工业级大模型调用中，这不仅会消耗不菲的 Token，还会暴露模型内部极其冗长的“自言自语”给终端用户（在某些产品设计中，我们希望对用户隐藏这些内部 Trace）。

不仅如此，如果模型在 Phase 1 想出的计划本身就是荒谬或错误的，我们在 Phase 2 直接让它去执行，依然会导致失败。结合这些痛点，你认为在驾驭工程中，我们能不能在 Phase 1 之后、Phase 2 之前，再插入一个“自我审查”的微循环？如果有，它的逻辑该怎么写？

欢迎在留言区分享你的架构洞察。我们下一讲见！

---

## 精选评论

**lJ**: 思考题：

可以插入一个自我审查的微循环。Phase 1 得到 thinkResp 后，不急着把它追加到主 context，而是单独发起一次 Critic 调用——用独立的 system prompt 让模型以"审判者"身份评审这份计划，输出通过/拒绝结论。如果计划被否，就重新触发 Phase 1 重新规划（设置最大重试次数防止死循环）；只有通过审查后，才把思考结果追加到 context 并进入 Phase 2 执行。这样既解决了 Token 浪费（有缺陷的 thinking trace 不进主 context），也解决了内容泄露（整个审查过程内部流转），更重要的是加了一道防止盲目执行的防火墙，本质上是给慢思考再套一层元认知。

疑问：

两阶段 ReAct 的核心是"机制决定行为"，用架构锁住模型的冲动。但现在有些前沿模型宣称已具备原生慢思考能力。请问老师，在工程选型上，我们如何判断一个模型的原生 Thinking 是否可靠到足以弱化两阶段架构的必要性？还是说即便模型能力足够，两阶段架构依然有其不可替代的工程价值——比如 trace 可观测性、Critic 介入点、以及对任意模型的通用性？

在两阶段架构中，每一 Turn 的 thinking trace 都被追加到 contextHistory。短期来看，当前 Turn 的 trace 对 Phase 2 几乎肯定有益（自回归锚定，模型顺着自己的逻辑走）。但多 Turn 之后，早期规划可能已经过时甚至是错误的，积累在 context 里会不会反而锚定模型、产生路径依赖？工程上是否需要对历史 thinking trace 做老化处理或摘要压缩？

> **作者回复**: 关于疑问1，我觉得 现阶段，即使模型具备原生慢思考，两阶段脚骨依然有一定工程价值。它提供了介入点（人类可以拦截计划）、可观测性（你可以看到并审计思考过程，而原生思维往往是黑盒）以及多模型兼容性。
> 
> 关于疑问2，这正是第 12 讲“内存压缩”要解决的问题。早期的 Thinking Trace  在任务推移后确实会成为干扰，我们需要像清理内存一样，对其进行策略性的处理。


---

**我是你的杨先森**: 老师请教个问题，在开启慢思考模式下，第一步为什么不能动态的把有什么工具的描述放到prompt里，这样ai在思考时就可以知道有哪些工具可以用，而不是随意发挥了

> **作者回复**: 问题非常好。
> 
> 其实，大模型在 Phase 1 (Thinking) 时，确实需要知道自己有哪些“武器”，否则它的规划就会脱离实际。
> 
> 但关键在于“如何告诉它”。
> 
> 在 API 层面，通常有两种方式告诉大模型工具的存在：
> 
> 1.  通过 tools 字段传入结构化的 JSON Schema：这就是我们在 Phase 2 (Action) 做的。一旦传入，模型底层的 Function Calling 机制就会被激活，它会产生强烈的冲动去直接输出调用 JSON，从而破坏我们期望的纯文本推理。
> 2.  通过 System Prompt 传入自然语言描述：比如在 Prompt 里写上“你拥有读取文件、执行 bash 等能力”。
> 
> 在这一讲中我们选择 availableTools = nil 是为了实现最彻底的“无诱惑思考”。
> 
> 但在后续第 10 讲中，你会发现：
>              
> 在 Phase 1 时，我们将工具的宏观描述（自然语言）作为 System Prompt 或上下文的一部分发给大模型，让它知道自己有什么能力去“做规划”；
> 但是，我们故意不传结构化的 tools 字段。这就相当于给模型戴上了“手铐”，逼迫它必须用纯文本把步骤想清楚，而无法直接“扣动扳机”。
> 
> 到了 Phase 2，我们再把完整的 tools 字段传过去，它就会顺着自己刚才的规划，精准地执行动作了。
> 
> 随着后续章节（尤其是第 10 讲 Prompt Composer）的展开，你会看到这种“分离认知与行动”的机制是如何落地的。
> 
> 


---

**jarvis**: 可靠 Agent =
目标理解
+ 计划生成
+ 计划审查
+ 工具调用
+ 权限控制
+ 观察反馈
+ 错误恢复
+ 上下文压缩
+ 成本控制
+ 用户确认

> **作者回复**: 👍


---

**晴天了**: 思考题:  

每次循环到 phase 1后, 即把phase1 的消息返回给用户让用户确认
如果用户确认, 继续走phase 2 
如果用户否认, 把phase 1 的消息 和用户否认的消息连同加入到 上下文中. 并拿着上下文 重新走 phase 1的循环. 

> **作者回复**: 👍


---

**LL**: 思考题 可以插入 自我审查完成之后 审查不通过带着审察信息返回phase1
 审查通过进入phase2

> **作者回复**: 👍


---

**Dragon baby**: 老师，这文章提到大语言模型是冲动的，当有工具的时候 会经常忽略思考 直接进行调用的请求。这块是否有相关文章的依据，我看你引用了 When tools are available, models tend to act quickly rather than think deeply.”（当工具可用时，模型倾向于迅速采取行动，而不是深入思考。） 这段话

> **作者回复**: 有一些paper做了相关研究，比如https://arxiv.org/abs/2512.07497 ，这一现象在学术界被归类为 'premature action without grounding'。这个在这一讲的留言里也有说过。可以翻看一下。


---

**gsz**: 老师，关于Main Loop有两点疑问请教下：
1.如果availableTools本身就是为null，是不是没有必要two-stage
2.ReAct每一轮循环都包含Phase1和Phase2，为什么不将Phase1单独抽出来，只针对Phase2进行循环呢

> **作者回复**: 1.  是的，如果这个系统从一开始就没打算给模型任何工具（纯 Chat 机器人），那么直接调一次大模型就够了，不存在 ReAct，也不需要什么慢
> 思考。
> 2.  为什么 Phase 1 也在循环里？这是因为这是一个持续不断解决长程任务的过程。 比如修一个 Bug，模型可能需要： (Turn 1 / Phase 1) 思考：我要找 Bug，先去读 main.go。 (Turn 1 / Phase 2) 行动：执行 read_file。 (Turn 2 / Phase 1) 思考：我读完了，发现少了 import，我决定去写代码。 (Turn 2 / Phase 2) 行动：执行 edit_file。
> 
> 每一次底层物理世界状态的改变（Observation 注入后），模型都需要重新审时度势，所以思考（Phase 1）必须在每一轮 Turn 中都发生。
> 
> 后面也提到的更优的方案，那就是自适应的开关慢思考。
> 
> 


---

**Dragon baby**: 老师，目前市面上什么agent应用是这种两段式的，我看openclaw的源码里面好像没这块逻辑

> **作者回复**: OpenClaw 确实非常极简，没有在代码里强制硬切分阶段。这一讲也是harness的一种探索，后续结合13讲的plan mode，会有更好的结果。


---

**飞翔的傻瓜**: ClaudeCode 这种 Agent 也是这样设计的么两阶段ReAct，老师要是能对比下 OpenClaw 和 Claude Code 这样的代码就更棒了

> **作者回复**: 我在编写这一讲时，cc还没有leak source。现在倒是有各种对cc的分析了。这个可能在加餐篇中进行对比剖析。


---

**水**: 用自己开发的agent执行skill时，还真是遇到有些时候模型并未按照最开始的要求：必须按照步骤定义执行，而是自己调用了MCP接口，并给出了最终结论

虽然也看过《思考的快与慢》，但根本没联想到，看了这篇文章才恍然大悟

> **作者回复**: [手动抱拳]

---

**Jaising**: Harness 就是干这个的，约束模型将模型的推理、审查、行动分别关进不同阶段：

User Task
  -> Phase 1: Think / Plan
  -> Phase 1.5: Self Review / Critique
      -> 通过：压缩成 Approved Plan，进入 Action
      -> 不通过：带着 Review 意见重新 Plan
      -> 多次失败：停止执行，要求补充信息或人工确认
  -> Phase 2: Action

> **作者回复**: 👍


---

**Dragon baby**: 老师我还有个疑问，这种两段式的是否有其他的弊端，因为它没有拿到真实tool调用的结果，所以第一段推理的过程并不是很准确，第二段按照推理的逻辑去执行是不是就容易出错。那是否可以这样设计，比如边思考 边执行 拿到结果再判断接下来要如何执行，而不是固定的两段式

> **作者回复**: 慢思考的确有弊端。本讲其他类似留言都有回答，可以读一下。


---

**佳佳的爸**: 加入 自我审查的代码，片段如下：
for {
		// ========== Phase 1：思考 & 生成计划 ==========
		tools := r.GetAvailableTools()
		resp, err := p.Generate(context.Background(), messages, tools)
		if err != nil {
			log.Fatal(err)
		}
		messages = append(messages, *resp)
		log.Println("🤖 思考结果:", resp.Content)

		// 无工具调用 = 结束
		if len(resp.ToolCalls) == 0 {
			log.Println("✅ 任务完成")
			break
		}

		// ========== 【插入：自我审查微循环】 ==========
		log.Println("\n==== 启动自我审查 ====")
		reviewMsgs := []schema.Message{
			{Role: schema.RoleUser, Content: selfReviewPrompt},
			*resp,
		}
		reviewResult, err := p.Generate(context.Background(), reviewMsgs, nil)
		if err != nil {
			log.Fatal(err)
		}
		log.Println("审查结果:", reviewResult.Content)

		if reviewResult.Content != "APPROVED" {
			log.Println("❌ 计划被拒绝，重新思考...")
			messages = messages[:len(messages)-1]
			continue
		}
		log.Println("✅ 审查通过！\n")

> **作者回复**: 👍


---

**金龟**: 没理解，authropic是严格user，assiant这种。慢思考user:用户输入，assiant:规划，user:这里应该填什么呢?还是用户输入?

> **作者回复**: 这个在后续长程任务时的确会有一些问题。13讲会有一次针对智谱模型api的错误的修正。


---

**Harrison**: 慢思考设计的很精妙，但是我有一个问题：如果直接把思考作为 assistant 消息放到 context，phase2 再次生成回复的时候，不需要把用户 提问再次追加一个 user message 到 context 中吗？

> **作者回复**: 如果再塞入一个 User: 帮我查天气，反而会打乱对话的时序逻辑，让模型感到困惑。


---

**davix**: 模型的“冲动”可否看成模型的不成熟？未來模型會按需覺得是否先要慢思考？

> **作者回复**: 冲动是工程界的一种观察结果。
> 
> 目前就像你说的，“自适应推理”是一个研究方向。


---

**晴天了**: 老师 我想问一下 大模型自带的thinking 和文中代码中的thinking 有什么区别？？？

> **作者回复**: 原生 Thinking（如 OpenAI 模型）是模型在黑盒里自己进行的强化学习思考（隐藏的思维链）。它依然带着所有工具的权限在思考，且思考过程对人类往往是不可见的（或者不可干预的）。 
> 
> 而这一讲驾驭层的  Thinking（代码隔离）是我们在harness架构上强行切断了它的工具挂载（Tools=nil）。这可以逼迫它把战术全盘写成清晰可见的纯文本，防范大模型因为看到 bash 工具而产生“扣动扳机”的“神经反射冲动”。


---

**Hector**: 这样是否会降低大模型的cache的命中率，第一次是没带tool的schema去请求，第二次带了之后cache丢失

> **作者回复**: 👍。你的问题涉及到了目前最前沿的 API 成本问题——Prompt Caching！ 这是一个不可避免的Trade-off（折中）。
> 
> 确实，由于 Phase 1 (无 tools) 和 Phase 2 (带 tools) 的
> Request 前缀不一致，在某些模型上会导致 Cache
> 破裂。但在极度复杂的长程任务中，为了防止模型“冲动修改代码”带来的翻车成本（一旦改错，你要多花几十轮循环的
> Token 才能救回来），第一次无 Cache 的 Thinking Token 消耗是绝对划算的。在后续，这可以结合“动态开启开关”来缓解。


---

**看戏**: 之前也遇到工具不推理这个疑惑问题，看到直接传递nil感觉很妙，如果在第一步提示词里面再增加一个工具description纯文本告诉它，它已经具备这个能力，再让他推理，第二步时候调用工具，会不会更好一些

> **作者回复**: 很好的想法。这也是在第 10 讲（Prompt 动态组装）中要做的。在 Phase 1 时，我们可以通过System Prompt 告诉模型“你有读写文件能力”，但在 API 的 tools 字段中故意传 nil（不激活 Function Calling扳机）。这样模型既能完美规划，又不会被触发冲动行动。


---

**TheOne**: 有几个问题想问一下老师
1. 为什么大模型在提供工具之后，会渴望使用工具，不太明白


2. 加入的慢思考，假如用户本身就有这个习惯，让llm先把方案输出到上下文里，那这个 loop 里面的慢思考是不是就和用户的冲突了？用户自己有意识的组织了上下文，agent 又组织了一遍，这对于最终的输出会产生一些影响吧？

还有就是关于在 agent 内部加入慢思考，就像老师说用机制让 agent 的输出更可靠，但是随着 llm 的能力提升，这种机制有可能成为枷锁吗？

> **作者回复**: 非常好的问题
> 
> 关于“为什么大模型在提供工具之后，会渴望使用工具，并急于行动”，这是一些业界的工程观察。当然也有一些paper做了相关研究，比如https://arxiv.org/abs/2512.07497 ，这一现象在学术界被归类为 'premature action without grounding'.
> 
> 关于问题2的冲突，我理解可能并不冲突，反而可能是互补。用户在 Context 中提供的“方案”多是业务级的，而慢思考（Thinking）是模型在决定“用哪些 Tool来实现方案”时的“战术推演”。
> 
> 最后你的问题也非常深刻。驾驭工程最深刻的命题——“Harness 是活的”。业界一些大模型厂商的研究也说明了，当模型更聪明时，过度的脚手架（Scaffolding）可能会成为枷锁。所以引擎必须能根据大模型返回的元数据动态开关这些防线（即自适应推理, Adaptive Reasoning）。
> 慢思考是否能取得更好效果，与大模型能力、任务类型都有一定关系。并非什么任务都需要开启慢思考的。
> 
>

---

**jarvis**: 用户任务
    ↓
contextHistory
    ↓
Turn 1 开始
    ↓
Phase 1：不给工具
    ↓
模型只能输出规划文本
    ↓
把规划文本加入 contextHistory
    ↓
Phase 2：恢复工具
    ↓
模型根据刚才的规划调用工具
    ↓
工具执行结果加入 contextHistory
    ↓
Turn 2 继续

> **作者回复**: 👍


---

**兔子高**: 请教一个问题：文章说的这句话，大模型在未来的几十轮（Turn）交互中，每一轮都必须先被关进“小黑屋”强制做一次纯文本的推理与规划（Phase 1），然后再去执行动作（Phase 2）。 在看代码里，对应的是 m.turn++ 和 if m.turn == 1 。这个 m 在 engine 初始化的时候只会初始化一次，也就是这个 turn 是一直累加的，所以  if m.turn == 1 里的逻辑只会执行一次，是否合理呢？


> **作者回复**: 代码看得非常仔细！
> 
>  需要说明的是，m.turn == 1 是我们在 MockProvider（模拟大脑） 中为了演示“思考 -> 行动 ->结束”这一个特定序列而写的硬编码。 在真实的 LLM实现中，这个判断逻辑会被替换为模型自身的推理状态。只要模型认为任务没完，循环就会一直跑，不存在只跑一次的问题。
> 
> 


---

**毁灭吧**: 又看完了，老师快点更新吧～～～恨不得让老师一天加更 10 篇！

> **作者回复**: [手动抱拳] 


---

**游来游去的W**: // internal/engine/loop.go
  const maxRefineAttempts = 2

  if e.EnableThinking {
      // ===== Phase 1a: 自由思考 =====
      thinkResp, err := e.provider.Generate(ctx, contextHistory, nil)
      if err != nil { return err }
      log.Printf("🧠 [内部 Trace 仅日志可见]: %s\n", thinkResp.Content)
      // ⚠️ 注意：thinkResp 此时不进 contextHistory

      // ===== Phase 1b: 自我审查 + 精炼微循环 =====
      var compressedPlan string
      for attempt := 0; attempt < maxRefineAttempts; attempt++ {
          critique, err := e.critiqueAndCompress(ctx, contextHistory, thinkResp.Content)
          if err != nil { return err }

          if critique.Valid {
              compressedPlan = critique.Plan
              break
          }

          // 计划有问题，把审查意见喂回去重新思考
          log.Printf("⚠️ [审查未通过] 问题: %s, 重试中...\n", critique.Issue)
          retryMsgs := append(contextHistory,
              schema.Message{Role: schema.RoleAssistant, Content: thinkResp.Content},
              schema.Message{Role: schema.RoleUser,
                  Content: fmt.Sprintf("你刚才的计划有问题：%s。请重新规划。", critique.Issue)},
          )
          thinkResp, _ = e.provider.Generate(ctx, retryMsgs, nil)
      }

      // 兜底：审查多次仍未通过，退而求其次用最后一次的原始 Trace
      if compressedPlan == "" {
          compressedPlan = thinkResp.Content
      }

      // 只把精炼版 Plan 放入 Context
      contextHistory = append(contextHistory, schema.Message{
          Role:    schema.RoleAssistant,
          Content: "[Plan]\n" + compressedPlan,
      })
  }



---

**甜甜的咸鱼**: 慢思考模式 也有缺点，不把工具包 报漏给大模型，大模型怎么能重复使用工具包



---

**Void**: 看上面内容的时候第一想到的是Plan Mode，静待13章吧。

不过从代码上看，如果是判断tools，那么这个EnableThinking这个静态变量实际意义不大，而且是启动时候加入的全局变量。


