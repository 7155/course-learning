# 2026-09-11｜Pi Agent Runtime：从一次回车到 Tool、Session、Compaction

> 本篇是 `2026-09-09｜Pi Extension 与 DeepSeek Harness 架构` 的后续。前一篇解决 Extension / DI / Registry / Hook / Sandbox；这一篇把 Pi 本身作为一个 Agent Harness 跑通：一次用户输入到底如何变成模型请求、工具执行、文件修改、并发调度、上下文重建和压缩。

---

# 0. 最终只保留这一张总图

```text
User 按 Enter
    │
    ▼
AgentSession / SessionManager
    │
    ├── 重建当前 active branch
    │     └── 如果已经 compact：summary + retained recent messages
    │
    ├── 构造 System Prompt
    │     ├── Pi instructions
    │     ├── tool descriptions
    │     ├── AGENTS.md / project context
    │     ├── Skill: name + description + location
    │     └── cwd
    │
    ▼
Agent Core
    │
    ├── AgentMessage[]
    ├── transformContext()
    └── convertToLlm()
    │
    ▼
pi-ai / Provider Adapter
    │
    ├── system / messages
    └── native tool schemas
    │
    ▼
LLM Request #1
    │
    ├── 普通文本 → 本轮可能结束
    │
    └── tool_call
          │
          ▼
      Tool Runtime
          │
          ├── resolve tool
          ├── prepare / coerce / validate args
          ├── beforeToolCall
          ├── execute()
          ├── try/catch
          └── afterToolCall
          │
          ▼
      ToolResult
          │
          ▼
      写回 Agent Context
          │
          ▼
      LLM Request #2
          │
          └── 继续推理 / 再调工具 / 最终回答
```

一句话：

> **LLM 只负责产生下一步动作；Harness 负责上下文、协议、工具、进程、文件、权限、状态和下一轮调用。**

---

# 1. 为什么 Pi 默认只需要 4 个 Coding Tool

默认核心工具：

```text
read
bash
edit
write
```

另外还有 `grep / find / ls` 等只读工具可以按需加入，但不是理解 Pi Agent 的必要条件。

最重要的是 `bash`：它是一个能力放大器。

```text
read   → 看文件
edit   → 局部修改已有文件
write  → 创建新文件或整体重写文件
bash   → git / npm / python / test / curl / shell ...
```

所以 Pi 的策略不是给模型 100 个业务按钮，而是给少量通用原语，让模型组合。

## write 与 edit

不要简单记成：

```text
write = 只写新文件
edit  = 修改文件
```

更准确：

```text
write
= create or whole-file rewrite

edit
= exact-text local replacement
```

`edit` 的典型输入：

```json
{
  "path": "src/app.ts",
  "oldText": "const port = 3000",
  "newText": "const port = 8080"
}
```

模型没有改磁盘，它只是输出修改意图。

---

# 2. Tool Call Pipeline：模型输出以后到底发生什么

一批 tool calls 的核心流程：

```text
AssistantMessage
│
├── toolCall A
├── toolCall B
└── toolCall C
        │
        ▼
executeToolCalls()
        │
        ├── batch scheduling
        │
        ▼
每个 ToolCall：
        │
        ├── 1. 查找工具是否存在
        ├── 2. prepareArguments（如果工具定义了）
        ├── 3. 参数 coercion + schema validation
        ├── 4. beforeToolCall
        ├── 5. tool.execute()
        │      └── try/catch
        ├── 6. afterToolCall
        └── 7. ToolResult
```

## 2.1 参数类型转换与 Schema 校验

例如工具要求：

```text
timeout: number
```

模型给：

```json
{"timeout":"10"}
```

Runtime 可以尝试：

```text
"10" → 10
```

再检查 Schema。

但 Schema 只检查“形式是否合法”：

```text
path 是不是 string？
```

它不知道：

```text
这个文件是否存在？
当前进程有没有权限？
这个命令执行后是否成功？
```

这些属于 Runtime / OS 层。

## 2.2 beforeToolCall

位置：

```text
resolve
  ↓
normalize / validate
  ↓
beforeToolCall
  ↓
execute
```

适合：

```text
permission
policy
user confirmation
audit
block
```

Pi 默认没有 Claude Code 那种内建 permission system，但 Extension 可以在这里实现 permission gate。

## 2.3 execute + try/catch

Tool 失败不等于 Agent 崩溃。

```text
tool.execute()
    ↓
throw error
    ↓
catch
    ↓
ToolResult(isError=true)
    ↓
下一次 LLM
    ↓
模型看到错误后重新决策
```

所以“Agent 自己纠错”本质是：

```text
运行时错误
  ↓
变成 observation
  ↓
下一次模型推理
```

## 2.4 afterToolCall

适合：

```text
redact sensitive data
sanitize
result transform
audit metadata
```

可以理解：

```text
beforeToolCall → 管输入和是否允许执行
afterToolCall  → 管输出和后处理
```

---

# 3. Native Tool Calling 本质是什么

工具调用不是模型拿到了一个 JS / Python 函数指针。

模型本质仍然只是在产生结构：

```json
{
  "name": "read",
  "arguments": {
    "path": "package.json"
  }
}
```

然后 Harness 执行。

## 3.1 不用 Provider native tools 也能实现 Tool Calling

只在 System Prompt 里写：

```text
你有 read 工具。
需要调用时，请输出：
<tool_call>{...}</tool_call>
```

模型也可以输出一段普通文本 JSON；Harness 自己解析、校验、dispatch。

因此：

```text
Tool Calling
≠
必须有 Provider tools API
```

## 3.2 Native Tool Calling 多了什么

Provider 的 `tools` API 把协议标准化：

```text
普通文本通道
        或
native tool-call 通道
    ├── tool name
    ├── call id
    └── arguments
```

未注册的工具，即使模型可以在普通文字里“写出”它，也不会成为这一轮 Provider 认可的 native tool-call block。

所以准确区分：

```text
prompt-only tool call
= 普通文本 + Harness parser

native tool call
= Provider 结构化协议 + Harness execute
```

Provider 内部到底是特殊 token、训练、grammar constrained decoding 还是其他机制，不从 API 表面武断推断。

---

# 4. 为什么一次 read 常常对应两次模型请求

问题：

```text
User:
“读取 package.json，然后告诉我项目名。”
```

第一次模型调用时，文件内容还不存在于 context：

```text
LLM Request #1
    ↓
模型决定：read(package.json)
    ↓
本次模型调用结束
```

然后 Pi 在本地：

```text
validate
  ↓
beforeToolCall
  ↓
read.execute()
  ↓
fs.readFile(...)
  ↓
afterToolCall
  ↓
ToolResult
```

现在文件内容才出现。

要让模型理解结果，只能再调用一次：

```text
LLM Request #2
    ↓
模型看到 ToolResult
    ↓
“项目名是 xxx”
```

所以根因不是五个 Tool Pipeline 步骤，而是：

> **第一次模型请求结束时，工具结果还不存在。**

因此：

```text
一个 User Turn
≠ 一次模型 API Request
```

可能是：

```text
LLM #1 → read
LLM #2 → bash
LLM #3 → edit
LLM #4 → final answer
```

---

# 5. 用户按 Enter 后，第一条模型请求怎么装出来

不要把“Session ID”与“上下文重建”混在一起。

Pi Session 本身是树：

```text
A ─ B ─ C ─ D
        \
         E ─ F  ← current leaf
```

如果当前 leaf 是 F，active branch 是：

```text
A → B → C → E → F
```

D 不会进入本轮模型 context。

## 5.1 三种上下文情况

```text
新会话
→ system + 当前 user message

普通续聊
→ system + active branch 历史 + 当前消息

压缩后续聊
→ system + compaction summary + retained recent messages + 后续消息
```

## 5.2 System Prompt 里有哪些东西

概念上：

```text
System Prompt
├── Pi 自身 instructions
├── tool descriptions
├── project context（如 AGENTS.md）
├── available skills
│    ├── name
│    ├── description
│    └── location
└── cwd
```

Skill 使用 progressive disclosure：启动时不把所有 SKILL.md 全文塞进 context，只放元信息；需要时再通过 `read` 加载完整 Skill。

## 5.3 Tool 有两种 representation

不要把它们混为一谈：

```text
Prompt representation
= 给模型的人类可读说明

Provider tool schema
= native tool calling 所需的机器可读 schema
```

## 5.4 AgentMessage 到 Provider Message

核心链：

```text
AgentMessage[]
    ↓
transformContext()
    ↓
AgentMessage[]
    ↓
convertToLlm()
    ↓
Message[]
    ↓
pi-ai Provider Adapter
    ↓
OpenAI / Anthropic / ... request
```

应用内部可以有 UI-only/custom messages；真正发给 LLM 前必须转换成 Provider 能理解的消息。

---

# 6. 模型不会改磁盘：真正修改文件的是本地 Runtime

模型输出：

```text
edit(path, oldText, newText)
```

Pi 才真正执行：

```text
LLM
 │ 结构化修改意图
 ▼
Pi Tool Runtime
 │
 ▼
edit.execute()
 │
 ├── read current file
 ├── locate oldText
 ├── replace
 └── write file
 │
 ▼
Operating System / filesystem
```

因此：

```text
LLM 没有 filesystem 权限
Pi process 才有 OS identity / permissions
```

`edit` 通过 exact-text matching 可以让一部分 stale edit fail-fast；Pi 内部还有针对文件 mutation 的串行化保护，但这不是跨进程 CAS / version locking。另一个 Agent、IDE 或进程在 read-write 窗口修改同一文件，仍然可能发生外部并发冲突。

---

# 7. 同一轮多个 Tool Call：真并发还是假并发

Pi 当前默认允许 parallel tool execution。

```text
read A ─────┐
read B ─────┼── concurrent execute
read C ─────┘
```

但 Tool 可以声明：

```text
executionMode = sequential
```

关键点：这是 **batch-level fallback**。

只要这一批里有一个必须 sequential：

```text
read A
special sequential tool
read B
```

整批都会走串行：

```text
A → special → B
```

而不是只让 special 自己排队。

## 7.1 Parallel 也有 barrier

```text
A 100ms ─✓
B 3s    ─────────✓
C 500ms ───✓
```

A/C 可以先发 execution completion event，但下一次 LLM request 要等当前 batch 全部完成。

```text
execute = 真并发
next LLM = batch barrier 后继续
```

持久化 ToolResult 仍然按原始 assistant tool-call source order 对齐，每条通过 `toolCallId` 关联原 call。

---

# 8. Tool 输出为什么要截断

50KB / 2000 lines 不是 Agent Core 对所有 Tool 统一做的一次 final truncate，而主要是具体 Coding Tool 自己的 output policy。

## read：保留头部

```text
文件很长
    ↓
保留开头
    ↓
提示模型用 offset 继续读
```

因为读文件通常先需要 header / imports / top-level structure。

## bash：保留尾部

```text
npm test
...
...
最后：FAIL / stack trace
```

Bash 输出通常结尾最重要，所以保留 tail；完整超长输出可以落临时文件，让模型之后再用 `read` 按需取。

这是 context backpressure：

> **工具可以产生海量数据，但每轮只把最有价值的一小段塞给模型。**

---

# 9. Python / TypeScript 运行 Bash 的本质

Python：

```text
subprocess
```

Node/TypeScript：

```text
child_process
```

概念上都是：

```text
当前进程
  ↓
OS 创建 child process
  ↓
运行 executable + arguments
```

例如：

```text
/bin/sh -c "git status && npm test"
```

拆成：

```text
/bin/sh
= Shell 解释器 executable

-c
= “把后面的字符串当 Shell 代码执行”

"git status && npm test"
= 交给 sh 解释的代码
```

而：

```text
/usr/bin/git status
```

更像：

```text
executable = git
args = ["status"]
```

Shell 的价值在于解释：

```text
&&  ||  |  >  >>  $HOME  $(...)
```

## 权限

Python 与 Node 本身没有“谁权限更大”的区别。

子进程默认继承父进程环境中的：

```text
uid/gid
cwd
env
文件权限
container / namespace / sandbox 限制
```

所以：

```text
LLM 没权限
Pi process 有权限
child process 继承 Pi/运行环境的权限
```

---

# 10. Background Bash：Pi 默认没做，但可以怎么做

普通 Bash：

```text
spawn process
  ↓
await exit
  ↓
ToolResult
  ↓
Agent Loop 继续
```

如果命令是：

```text
npm run dev
```

它可能长期不退出，于是当前 Tool Call 长期 pending。

真正的后台任务要改成：

```text
background_bash(command)
      ↓
spawn process
      ↓
保存 jobId / pid / state
      ↓
立即返回 ToolResult：
{ jobId, status: running }
      ↓
Agent 继续
```

后台 Task Manager 再维护：

```text
RUNNING
  ├── process exit 0     → SUCCEEDED
  ├── process exit != 0  → FAILED
  ├── deadline reached   → TIMED_OUT
  └── user cancel        → CANCELLED
```

Timeout 不是理论上必须，但工程上至少必须有生命周期管理：status、cancel/kill、stdout/stderr、deadline（可选）、退出事件。

Server 型任务完全可以没有 timeout；否则必须能够显式 cancel。

## 10.1 Background completion 不是第二个 ToolResult

原 Tool Call 已经闭合：

```text
ToolCall call_1
     ↓
ToolResult call_1:
job started, jobId=123
```

几分钟后的：

```text
job_123 completed
```

是新的异步 observation，不应该伪造为第二个 `ToolResult(call_1)`。

更干净的设计：

```text
BackgroundCompletionEvent
        ↓
Agent inbox / steering queue
        ↓
下一个 LLM boundary 注入
```

Subagent completion 可以复用完全相同的模型：

```text
Async Task
├── background bash
├── subagent
└── remote job
      ↓
completion event
      ↓
parent inbox / steer
```

---

# 11. Steering、Follow-up 与 Stream/Event 生命周期

这是本轮结束前最后补齐的核心点。

## 11.1 一次 prompt 的事件流

```text
agent_start
  ↓
turn_start
  ↓
message_start(user)
message_end(user)
  ↓
message_start(assistant)
message_update*       ← token / stream delta
message_end(assistant)
  ↓
如果有工具：
tool_execution_start
tool_execution_update*
tool_execution_end
message_start/end(toolResult)
  ↓
turn_end
  ↓
如果还要继续 → 下一个 turn
否则 → agent_end
```

所以 UI 上看到的流式输出、工具进度、spinner，本质都来自事件流，不是另一个 Agent Loop。

## 11.2 steer

`steer` 是：

> 当前 run 尚未结束时，给 Agent 一个“下一步请考虑这个”的新消息。

但不是把 token 插进正在进行的同一次 Transformer generation。

正确模型：

```text
当前 assistant/tool batch
      ↓
完成到一个 turn boundary
      ↓
检查 steering queue
      ↓
注入 steer message
      ↓
下一次 LLM request
```

所以 background completion / subagent completion 可以通过这类 inbox 机制通知父 Agent。

## 11.3 follow-up

`followUp` 是：

> Agent 原本已经没有 tool call / steer，准备结束时，再追加一件工作。

因此：

```text
steer
= 改变正在进行的 run 的下一步

follow-up
= 当前任务本来要结束时，再追加任务
```

---

# 12. Session、Compaction、Branch Summary

Pi Session 会持久化为树，不等于每轮都把 JSONL 全发给模型。

## 12.1 Compaction 为什么存在

Context Window 有限：

```text
历史越来越长
     ↓
快接近 context window
     ↓
不能无限把原始历史继续发送
```

Pi 的解法：

```text
老历史  ───────────────┐
                       ├── LLM summary
最近历史 ──────────────┘   （最近一段保留原文）
```

当前默认思路：

```text
reserveTokens ≈ 16384
keepRecentTokens ≈ 20000
```

触发近似：

```text
contextTokens > contextWindow - reserveTokens
```

Compaction：

```text
1. 找 cut point
2. 老历史序列化
3. 额外调用 LLM 生成 structured summary
4. 写 CompactionEntry
5. 下一次 context = summary + recent messages
```

注意：

> **Compaction 不等于把 Session 历史删除。**

原 Session Tree 仍然存在；只是重建给 LLM 的 context 时，用 summary 替代早期原文。

## 12.2 为什么不能随便从 ToolResult 中间切

ToolResult 必须和对应 ToolCall 保持语义完整，因此 cut point 不能随便落在 tool result 上。

如果单个 user turn 本身就特别长，可能出现 split turn：Pi 对早期 turn prefix 额外做摘要，再与旧 summary 合并。

## 12.3 Branch Summary

当 Session Tree 从一个 branch 跳到另一个 branch：

```text
        B ─ C ─ D   old leaf
      /
A ───
      \
        E ─ F       target
```

Pi 可以把离开的 `B/C/D` 总结，然后把摘要带到新分支：

```text
A ─ E ─ F ─ [summary of B/C/D]
```

因此：

```text
Compaction
= 同一 active branch 上，为 context window 压缩旧历史

Branch Summary
= branch 切换时，把离开路径的重要状态带到新路径
```

---

# 13. “Pi 故意少做”到底是什么意思

Pi 的 minimalism 不是“没做完”。

核心思想：

```text
Core 保持小
+ Extension / Skill / Package 扩展行为
+ 真正的安全边界交给 OS/container/sandbox
```

所以很多能力不一定进入 Core：

```text
permission UI
background task runtime
subagent orchestration
plan/todo workflow
复杂 memory policy
```

不是这些能力不能实现，而是避免把特定产品策略绑死在 Agent Core。

这也解释了 Pi 为什么特别适合用来学习 Harness：它把最基础的机制暴露得比较清楚。

---

# 14. 到这里，Pi Agent Harness 还缺什么？

作为 **Agent 原理 / 求职面试** 学习，本轮可以结束。

已经掌握的主干：

```text
[x] minimal tool set
[x] tool schema / native calling
[x] tool call pipeline
[x] before/after hook
[x] tool error → observation → self-correction
[x] read 为什么产生下一次 LLM request
[x] request/context assembly
[x] session tree / active branch
[x] file edit/write 的真实执行边界
[x] parallel vs sequential tool batch
[x] output truncation / context backpressure
[x] process / shell / permissions
[x] sandbox vs permission
[x] background task 的正确抽象
[x] steer / follow-up
[x] event streaming
[x] compaction / branch summary
[x] extension / registry / DI / IoC（见 09-09 笔记）
```

暂时不用继续深挖：

```text
- TUI / keybindings / theme
- OAuth / provider 登录细节
- llama.cpp 接入
- prompt template UI
- package 分发
- RPC / JSON CLI 协议细节
- custom provider 的每个兼容字段
```

这些是 Pi 产品/集成层，而不是理解 Agent Harness 的必要前置。

如果以后 PAW 需要对应能力，再按需求返回来学。

---

# 15. 面试压缩版

## 一句话

> Pi 是一个 minimal coding harness：Agent Core 维护消息和 Agent Loop，pi-ai 负责 Provider 协议，Coding Agent 层负责 Session/System Prompt/Tools/Skills；模型只产生结构化动作，本地 Runtime 才真正执行文件、Shell 和权限相关副作用。

## 30 秒

> 用户输入后，Pi 先从 Session Tree 重建当前 active branch，构造 System Prompt 和可用 Tool/Skill 上下文，经 pi-ai 转为 Provider 请求。模型如果返回 Tool Call，Agent Core 会进行工具解析、参数转换与校验、before hook、execute、错误捕获和 after hook，再把 ToolResult 写回 context 发起下一次模型请求。多个工具默认可以并行，但存在 batch barrier；文件和 Bash 真正由本地进程执行，所以权限属于 Pi/OS，而不是 LLM。上下文过长时通过 Compaction 用 summary 替换旧历史、保留最近消息。

## 深挖时最容易被问的几个点

1. 为什么一次 read 通常需要第二次模型请求？
2. prompt-only Tool Call 和 native Tool Call 有什么区别？
3. schema validation 为什么不能替代 Runtime permission？
4. Pi 的 parallel tools 为什么仍然存在 barrier？
5. `edit` 为什么不能完全解决多进程并发修改？
6. background Bash 为什么不能在任务完成时再次伪造原 ToolResult？
7. steer 与 follow-up 的 injection timing 有什么区别？
8. Compaction 为什么不能从 ToolResult 中间切？
9. Permission Hook 与 OS Sandbox 为什么是两层安全？
10. `/bin/sh -c` 与直接 `spawn("git", ["status"])` 的区别是什么？

---

# 16. 反向自测

如果下面这些可以不看答案自己推出来，Pi 这一章就算真正学完：

```text
Q1：模型根本不知道磁盘，为什么还能 edit？

Q2：如果第一次 LLM 已经返回 read tool_call，文件内容什么时候才第一次进入模型 context？

Q3：三个 tool call 中有一个 sequential，为什么另外两个也不能继续并行？

Q4：为什么 read 截头、bash 截尾？

Q5：一个后台 npm dev server 没有 timeout，系统需要哪些机制才能不失控？

Q6：Session 文件保存完整历史，为什么 LLM 仍然只看到 summary + recent messages？

Q7：另一个 IDE 在 Pi edit 的 read/write 窗口修改文件，为什么 path-level queue 不一定救得了？

Q8：只在 System Prompt 里定义 read，不走 Provider tools API，为什么仍然可以实现 Agent Tool Calling？
```

---

# 17. 与 PAW 的直接映射

这轮 Pi 学习最终不是为了背 Pi 源码，而是为了反推 PAW 的 Harness：

```text
Pi Agent Loop
→ PAW Runtime Loop

Pi Tool Registry + hooks
→ PAW Tool Runtime / Policy Pipeline

Pi Session Tree
→ PAW Session / Branch State

Pi Compaction
→ PAW Context Manager

Pi Bash child process
→ PAW Local / Remote Worker Process Runtime

Pi steering queue
→ PAW Async Task / Subagent Inbox

Pi Extension
→ PAW Plugin / Capability extension

OS sandbox
→ PAW Worker Isolation Boundary
```

下一阶段如果继续学习，应从“理解 Pi”切换为：

> **把这些机制映射回 PAW，检查 PAW 哪些已经有、哪些只是 demo、哪些需要生产级补齐。**

---

## 上游源码/文档入口

- `packages/agent/README.md`：Agent Loop、Events、Tool execution、Steering / Follow-up
- `packages/agent/src/agent-loop.ts`：Tool batch、preflight、parallel/sequential、下一 turn
- `packages/coding-agent/src/core/session-manager.ts`：Session Tree / CompactionEntry
- `packages/coding-agent/docs/sessions.md`：Session/branch/tree
- `packages/coding-agent/docs/compaction.md`：Compaction / Branch Summary
- `packages/coding-agent/src/core/system-prompt.ts`：System Prompt / project context / skills
- `packages/coding-agent/src/core/tools/`：read / bash / edit / write / truncate
- `packages/coding-agent/docs/extensions.md`：Extension API / hooks
- `packages/coding-agent/docs/security.md`：permission 与 sandbox 边界
