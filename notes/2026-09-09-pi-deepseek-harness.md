# 2026-09-09｜Pi Extension 与 DeepSeek Harness 架构

> 今日目标：把 Pi Extension、DeepSeek Harness/Cordis、Tool Pipeline、依赖/生命周期、Scope/Isolation、Sandbox 串成一条完整知识链。

## 1. 先抓住最核心的一条线

Pi 和 DeepSeek Harness 都不是“扫描到一个函数就直接执行”。它们都在做：

```text
加载插件代码
  ↓
给插件一个框架 API / Context
  ↓
插件注册 Tool、Event Hook、Command 等能力
  ↓
Agent Runtime 正常运行
  ↓
运行到对应生命周期节点时，框架反向调用插件
```

区别是：

- **Pi** 更偏向“已有 Agent Runtime + 强 Extension Hook”。
- **Harness/Cordis** 更偏向“Agent Runtime 本身也由 Plugin/Service 组合出来”。

---

# 2. Pi Extension 到底是怎么加载的

插件可以写成：

```ts
export default function plugin(pi) {
  pi.on("tool_call", handler)
  pi.registerTool(tool)
}
```

这里插件文件**不是自己 import 一个全局 `pi`**。

真正方向是：

```text
Pi Runtime
  ↓
扫描 extension 文件
  ↓
import plugin.ts
  ↓
拿到 default export
  ↓
const factory = module.default
  ↓
创建 ExtensionAPI 实例 pi
  ↓
factory(pi)
```

`factory(pi)` 本质就是普通函数传参。

## 2.1 `interface ExtensionAPI` 只是类型契约

```ts
interface ExtensionAPI {
  on(...)
  registerTool(...)
  registerCommand(...)
  sendMessage(...)
}
```

这个 `interface` **没有运行时实现**。

真正运行时一定有类似：

```ts
const pi = {
  on(event, handler) {
    // 把 handler 放进事件注册表
  },

  registerTool(tool) {
    // 把 tool 放进工具注册表
  },

  sendMessage(message) {
    // 转发给 runtime
  }
}
```

所以要区分：

```text
ExtensionAPI interface
= TypeScript 静态类型契约

createExtensionAPI() 创建出来的 pi
= 真正运行时对象
```

## 2.2 插件不是“给 pi 对象加函数”

例如：

```ts
pi.on("tool_call", handler)
```

不是：

```text
把 handler 变成 pi 上的新方法
```

而是：

```text
调用 pi 已有的 on()
  ↓
把 handler 保存到内部 Registry / Handler Map
```

例如概念上：

```ts
handlers["tool_call"].push(handler)
```

以后真正发生 `tool_call`：

```ts
for (const handler of handlers["tool_call"]) {
  await handler(event, ctx)
}
```

---

# 3. Factory、Dependency Injection、Registry、Event Hook、IoC

这几个词其实描述的是上面同一套代码。

## 3.1 Factory

Pi 中：

```ts
const factory = module.default
factory(pi)
```

`factory` 是插件统一初始化入口。

这里称 Factory，是因为 Runtime 只依赖统一协议：

```ts
type ExtensionFactory = (api: ExtensionAPI) => void | Promise<void>
```

框架不用知道具体插件内部怎么实现。

> 在 Pi 这里，更准确地说是 extension factory / plugin initializer。

## 3.2 Dependency Injection（依赖注入）

DI 最简单的定义：

> **模块需要的能力，不自己创建、不自己到全局寻找，而由外部传进来。**

Pi：

```ts
factory(pi)
```

插件需要 Tool Registry、Event 系统、Runtime Action 等能力，但不直接 import Pi 内部模块，而是统一拿 `pi`。

```text
Runtime 创建能力
  ↓
把 pi 传给 Plugin
  ↓
Plugin 使用这些能力
```

这就是函数参数形式的依赖注入。

## 3.3 Registry（注册表）

Registry 回答：

> **系统现在拥有哪些能力？**

例如：

```text
Tool Registry
├── bash
├── read_file
├── search
└── custom_tool
```

插件是 Tool producer，Agent Loop 是 Tool consumer，两边通过 Registry 解耦。

```text
Plugin A ─┐
Plugin B ─┼──→ Tool Registry ←── Agent Loop
Plugin C ─┘
```

## 3.4 Event Hook

Hook 回答：

> **系统运行到某个生命周期节点时，谁需要被调用？**

例如：

```ts
pi.on("tool_call", handler)
```

加载阶段只是注册：

```text
tool_call → [handlerA, handlerB, ...]
```

运行阶段：

```text
真正发生 tool_call
  ↓
Event Dispatcher
  ↓
找到 tool_call handlers
  ↓
handler(event, ctx)
```

注意：`on()` 订阅的是 **Event**，不是 `ctx`。

- `event`：发生了什么。
- `ctx`：这一次事件发生时的 Runtime Context。

## 3.5 IoC（Inversion of Control，控制反转）

IoC 回答：

> **最后谁控制程序什么时候调用插件？**

插件只做：

```ts
pi.on("tool_call", handler)
```

插件不知道什么时候会发生 Tool Call。

真正控制调用时机的是 Runtime：

```text
Plugin 先注册
  ↓
Agent Runtime 继续运行
  ↓
发生 tool_call
  ↓
Runtime 反向调用 Plugin handler
```

所以：

```text
DI = 能力怎么进入插件
Hook = 插件怎么声明“我想参与哪里”
IoC = 最终由框架决定什么时候调用插件
```

---

# 4. Pi 的 Tool Call 不是简单 `execute()`

Pi 本身就有完整 Tool Pipeline。

可以按下面理解：

```text
LLM 产生 Tool Call
  ↓
1. Tool Resolution
   工具是否存在？
  ↓
2. Argument Normalization / Coercion
   例如 "10" 尝试转成整数 10
  ↓
3. Schema Validation
   参数是否符合 Tool Schema？
  ↓
4. beforeToolCall / tool_call Hook
   Permission
   用户确认
   参数改写
   安全检查
   Block
  ↓
5. Tool.execute()
   signal / streaming update / try-catch
  ↓
6. afterToolCall / tool_result Hook
   修改 content/details/error 等
  ↓
ToolResult 写回 Agent Context
  ↓
LLM 下一轮
```

## 4.1 参数检查

如果模型把：

```json
{"count":"10"}
```

传给要求整数的 Tool，Runtime 可以先尝试 coercion：

```text
"10" → 10
```

如果最终仍不符合 Schema，就返回参数错误，而不是直接进入工具执行。

## 4.2 Permission / Tool Interception

`tool_call` Hook 很适合做：

```text
- 需要用户确认吗？
- 是否阻止危险 shell command？
- 是否重写参数？
- 是否根据上下文修改 Tool Call？
```

例如检测：

```text
rm -rf
sudo
危险文件写入
```

需要注意：**字符串规则不是最终安全边界**，真正安全还需要 Sandbox。

## 4.3 Tool 失败不等于 Agent 崩溃

工具执行失败可以变成：

```text
ToolResult
isError = true
```

重新给模型。

模型看到：

```text
命令失败
exit code 1
stderr: ...
```

然后下一轮自己分析、修正、再次调用 Tool。

---

# 5. Harness 相比 Pi 真正多在哪里

不能简单说：

```text
Pi = 简单
Harness = 有 Pipeline
```

这是错误的。两者都有完整 Tool Pipeline。

真正区别是：

```text
Pi
= 已存在 Agent Runtime + 强 Extension Hooks

Harness/Cordis
= Plugin/Service Framework 本身组合出 Agent Runtime
```

Harness 更强调四个词：

```text
Composition
Scope
Dependency Graph
Reversible Lifecycle
```

---

# 6. Harness 的依赖管理

Harness 插件可以声明：

```ts
export const inject = ["tools", "llm"]

export function apply(ctx) {
  // ...
}
```

意思是：

> 这个插件只有在 `tools` 和 `llm` Service 都存在时才有资格运行。

形成依赖图：

```text
tools ──┐
        ├──→ Plugin A
llm ────┘
```

如果：

```text
tools ✓
llm   ✗
```

Plugin A 不应该半初始化，而保持：

```text
PENDING
```

依赖满足后：

```text
PENDING
  ↓
LOADING
  ↓
apply(ctx)
  ↓
ACTIVE
```

这已经不仅仅是 `factory(pi)` 那种“把参数传进去”，而是 **Runtime Dependency Graph Management**。

---

# 7. Harness 的生命周期管理

生命周期管理**不是**：

> Tool 最近没调用，所以把 Tool 卸掉节省内存。

真正管理的是：

> 插件现在是否仍有资格存在于当前 Runtime，以及插件退出时如何把自己造成的副作用全部撤销。

典型状态：

```text
PENDING
  ↓
LOADING
  ↓
ACTIVE
  ↓
UNLOADING
  ↓
DISPOSED
```

## 7.1 什么叫“卸载”

卸载并不是简单把 JS 文件从内存删掉。

假设插件加载时：

```ts
ctx.tools.register(myTool)
ctx.on("tools/result", handler)
```

它还可能：

```text
创建 timer
打开 websocket
打开数据库连接
注册 Service
watch filesystem
```

卸载应该撤销：

```text
Tool Registry 中的 myTool        ×
Event Bus 中的 handler           ×
timer                             ×
connection                        ×
Service                           ×
```

所以 unload 更准确是：

> **撤销注册 + 释放副作用 + 使插件失效。**

## 7.2 为什么不卸载会出问题

### 旧依赖引用

```text
Shell V1
  ↓
Plugin A
```

如果 Shell V1 已被替换成 Shell V2，但 Plugin A 不重新初始化，它可能仍然持有旧引用。

### 重复事件监听

热重载三次：

```text
handler A
handler B
handler C
```

一次 Tool Call 可能被执行三遍。

### 资源泄漏

插件每次加载：

```ts
setInterval(...)
```

没有 cleanup，重载十次就是十个 timer。

所以 Cordis 会把很多注册看成可撤销 Effect。

---

# 8. Service Isolation 与 Agent Scope

这两个不要混。

## 8.1 Isolation：到底拿到哪个 Service 实例

两个插件都写：

```ts
ctx.shell
```

但可能拿到：

```text
Group A → Shell A，timeout = 5s
Group B → Shell B，timeout = 60s
```

所以：

> **Isolation = Service instance resolution。**

DI 只声明：

```text
我要 shell
```

Context/Isolation 决定：

```text
你这个位置具体拿 Shell A 还是 Shell B
```

## 8.2 Scope：这个 Agent 能看到哪些注册项

例如：

```text
Global Tools
├── bash
├── read
└── search
```

Agent A 的 Scope：

```text
special_search
```

Agent B 的 Scope：

```text
read only restrictions
```

于是不同 Agent 虽然共用 Harness Runtime，却可以看到不同的：

```text
Tool
Prompt
Variable
Restriction
Listener
```

所以：

```text
Isolation = 我拿哪个 Service 对象？
Scope     = 我能看到哪些 Registry contribution？
```

---

# 9. Harness Tool Execution Pipeline

Harness 和 Pi 都有完整 Pipeline。

Harness 的特点是：**把 Hook 的语义拆得更明确。**

可以记成：

```text
Tool Call
  ↓
scope / visibility
  ↓
pre-execute
  可组合策略
  allow / deny / ask
  ↓
guard
  只能进一步禁止
  不能扩大权限
  ↓
execute middleware
  timeout / retry / metrics / tracing
  ↓
Tool.execute()
  ↓
post-execute
  结果检查 / 替换 / block
  ↓
finalizeContent
  ↓
tools/result
  最终只读观察 / audit
```

## 9.1 Pi 与 Harness 在 Tool Hook 上的取向

Pi：

```text
少数几个能力很强的 Extension Point
→ 插件自己组合 permission/rewrite/block 等行为
```

Harness：

```text
把不同语义拆成不同阶段
→ 框架明确规定每个阶段能做什么、不能做什么
```

例如 `guard` 的核心思想：

```text
只能收紧权限，不能放宽权限
```

这属于 monotonic security policy。

---

# 10. Sandbox 到底是什么

Sandbox 不是某一种固定技术，也不等于 VM。

它的本质：

> **给不可信代码划定一个受限制执行边界。**

可以由：

```text
OS Sandbox
Container
VM / MicroVM
Remote Sandbox
WASM 等
```

实现。

典型限制面：

```text
filesystem
network
process
system call
CPU / memory / PID
credentials / secrets
execution timeout
```

## 10.1 Container 型 Sandbox

例如宿主机：

```text
/home/user/project
```

容器内挂载成：

```text
/workspace
```

Agent 在容器内部：

```bash
cd /workspace
python app.py
```

实际操作的是显式挂载进来的项目目录。

关键点不是“容器可以操作宿主机”，而是：

> **只把允许的宿主资源映射给 Sandbox，Sandbox 内程序只能在授权边界内访问它们。**

## 10.2 Permission 与 Sandbox 的区别

例如 Agent 要执行：

```bash
rm -rf /home/user
```

Permission：

```text
应用层策略
→ allow / deny / ask user
```

Sandbox：

```text
OS / Container / VM 层强制执行
→ 即使上层错误放行，也没有权限越界
```

所以安全设计是：

```text
LLM
 ↓
Permission / Policy
 ↓
Sandbox
 ↓
OS / Virtualization Enforcement
```

这是 Defense in Depth（纵深防御）。

---

# 11. Sandbox 面试常见问题

## Q1：Sandbox 是什么？

受限执行环境，用于限制不可信代码对 filesystem、network、process、credentials、resources 的访问。

## Q2：Docker 和 VM 区别？

```text
Container
→ 共用 Host Kernel
→ 轻、启动快

VM
→ 独立 Guest Kernel
→ 隔离更强、成本更高
```

## Q3：Docker 怎么作为 Agent Sandbox？

至少考虑：

```text
workspace mount
read-only filesystem where possible
non-root user
drop capabilities
seccomp
network namespace / egress allowlist
cgroups CPU/memory/PID limits
timeout
minimal secret injection
```

## Q4：有 Permission 为什么还要 Sandbox？

Permission 是应用层策略，会存在规则遗漏或代码 bug；Sandbox 是底层 enforcement boundary，两者不能替代。

## Q5：如何防 `rm -rf`？

不要只匹配字符串。最终应该让进程在 OS 层没有权限删除 Sandbox 边界之外的文件。

## Q6：为什么普通 Docker 仍可能不够？

Container 共用宿主 Kernel；错误配置如 privileged、危险 mount、暴露 docker.sock 都可能破坏隔离。

高风险环境可考虑：

```text
gVisor
Kata Containers
Firecracker / MicroVM
独立 VM
远程临时 Sandbox
```

---

# 12. 面试时如何比较 Pi 和 Harness

可以这样回答：

> Pi 和 DeepSeek Harness 都有完整的 Tool Execution Pipeline。Pi 更倾向于提供能力较强的通用 Extension Hook，例如 `tool_call` 和 `tool_result`，由插件自己组合 Permission、参数改写、阻断和结果处理；Harness 则把这些能力进一步框架化，把工具策略拆成 pre-execute、guard、execute middleware、post-execute 和最终只读 result 等明确阶段。
>
> 更根本的区别是，Pi 更像“已有 Agent Runtime 上的扩展系统”，而 Harness/Cordis 希望连 Tool Registry、LLM、Session、Agent Loop 都能作为 Service/Plugin 被组合。因此 Harness 还需要依赖图、生命周期、Effect cleanup、Service isolation 和 Agent scope。

---

# 13. 今天形成的知识网络

```text
Agent Extension Architecture
│
├── Dynamic Module Loading
│   └── import plugin
│
├── Factory / Plugin Initializer
│   └── factory(pi) / apply(ctx)
│
├── Dependency Injection
│   └── Runtime 把能力传给 Plugin
│
├── Registry
│   └── Tool / Command / Provider
│
├── Event Hook / Observer
│   └── on(event, handler)
│
├── IoC
│   └── Runtime 决定何时反向调用 Plugin
│
├── Tool Pipeline
│   ├── resolve
│   ├── validate
│   ├── permission
│   ├── rewrite / intercept
│   ├── execute
│   └── result processing
│
├── Harness Service Graph
│   ├── inject
│   ├── dependency management
│   └── service replacement
│
├── Lifecycle
│   ├── PENDING
│   ├── LOADING
│   ├── ACTIVE
│   ├── UNLOADING
│   └── DISPOSED
│
├── Scope / Isolation
│   ├── Service instance isolation
│   └── Agent registry visibility
│
└── Security
    ├── Permission / Policy
    ├── Guard
    ├── Sandbox
    └── Defense in Depth
```

---

# 14. 后续复习建议

下一次复习不要再从定义背起，直接回答这 6 个问题：

1. `factory(pi)` 中，谁 import 谁？`pi` 到底从哪里来的？
2. DI、Event Hook、IoC 三者分别回答什么问题？
3. Pi 从模型 Tool Call 到 ToolResult 经过哪些阶段？
4. Harness 为什么还需要 `inject + lifecycle + effect`？
5. Isolation 和 Scope 的区别是什么？
6. Permission 和 Sandbox 为什么不能互相替代？

如果这 6 个问题能连续讲清楚，今天这部分就已经连起来了。

## 参考

- DeepSeek Harness 教程：https://www.runoob.com/deepseek-harness/
- DeepSeek Harness GitHub：https://github.com/deepseek-ai/deepseek-harness
- Pi Coding Agent GitHub：https://github.com/earendil-works/pi
