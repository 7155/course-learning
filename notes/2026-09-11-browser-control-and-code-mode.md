# 2026-09-11｜Browser Control 与 Code Mode：从 CDP 到 Agent Tool SDK

> 本篇记录浏览器控制与 Code Mode 的学习主线：网页在浏览器里到底变成什么、Playwright / Ego / CDP 分别在哪一层、Browser Agent 为什么还需要 Semantic Snapshot，以及 Code Mode 为什么能减少 LLM 推理轮次、如何在 Pi / PAW 上实现而不绕过原有权限边界。

---

# 0. 最终只保留两张总图

## 浏览器控制栈

```text
LLM / Agent
    │
    ▼
Agent Browser Layer
Ego / Browser Use / agent-browser
    │
    │ semantic snapshot / refs / task space
    ▼
Browser Automation Layer
Playwright / Puppeteer / Selenium
    │
    │ locator / wait / frame / context / actionability
    ▼
Browser Protocol
CDP / WebDriver / WebDriver BiDi
    │
    ▼
Browser Engine
DOM / AX Tree / Layout / JS / Network / Input
    │
    ▼
Pixels
```

## Code Mode 栈

```text
LLM
 │
 │ 一段 JavaScript
 ▼
exec(code)
 │
 ▼
JavaScript Runtime
 │
 ├── tools.read()
 ├── tools.edit()
 ├── tools.write()
 ├── tools.bash()
 └── tools.browser()
       │
       ▼
Pi Tool Dispatcher
       │
       ▼
Permission / Workspace / Approval / Trace
       │
       ▼
Real Tools
```

一句话：

> **浏览器 Agent 的关键是“观察、定位、执行、验证”；Code Mode 的关键是“把确定性控制流从 LLM 搬到程序里，但真正权限仍然留在 Tool Gateway”。**

---

# 1. 网页最终不是“只有 HTML”

HTML 只是浏览器的输入之一。

```text
HTML
  ↓ parse
DOM

CSS
  ↓ parse
CSSOM

DOM + CSSOM
  ↓
Style Calculation
  ↓
Layout
  ↓
Paint
  ↓
Compositing
  ↓
Pixels
```

JavaScript 会在运行时继续修改 DOM、样式和应用状态：

```text
JavaScript
   ├── 改 DOM
   ├── 改 CSS
   ├── 发网络请求
   └── 绑定事件
```

因此需要区分：

```text
HTML
= 初始结构描述

DOM
= 当前运行时结构

Layout
= 元素最终位置和尺寸

AX Tree
= 浏览器提供的人类语义 / 可访问性结构

Pixels
= 屏幕最终显示结果
```

现代 React / Vue 页面尤其不能只看服务器返回的原始 HTML，因为很多真实内容是在 JS 执行后才进入 DOM。

---

# 2. 为什么 Agent 不直接吃完整 HTML

完整 DOM / HTML 对程序很有用，但对 LLM 往往过于冗长：

```html
<div class="css-16abc">
  <div class="css-298xx">
    <span>
      <svg>...</svg>
      登录
    </span>
  </div>
</div>
```

Agent 实际更关心：

```text
textbox "邮箱"
textbox "密码"
button "登录"
link "忘记密码"
```

所以 Browser Agent 常把浏览器的 Accessibility Tree（AX Tree）压缩为 Semantic Snapshot。

例如：

```text
@1 heading "账号登录"
@2 textbox "邮箱"
@3 textbox "密码"
@4 button "登录"
```

内部可以保存：

```text
@4
 ↓
backendNodeId = 973
role = button
name = 登录
```

于是对 LLM 暴露的是稳定、低 token 的语义表示，而不是整个 DOM。

---

# 3. `@4 button "登录"` 到真正点击发生了什么

最小链路：

```text
Agent
 │
 │ click("@4")
 ▼
Ref Map
 │
 │ backendNodeId = 973
 ▼
DOM.getBoxModel
 │
 │ 当前几何位置
 ▼
Input.dispatchMouseEvent
 │
 ▼
Chromium
```

所以浏览器内部确实维护：

```text
逻辑页面节点
    ↓
Layout Box
    ↓
屏幕几何位置
```

但高层 Agent 不必自己维护坐标。

---

# 4. CDP 是什么

CDP = Chrome DevTools Protocol。

可以把它先理解为 Chromium 暴露的一组 RPC Domain：

```text
Target.*
Page.*
DOM.*
Runtime.*
Input.*
Network.*
Accessibility.*
Browser.*
```

典型调用：

```text
Page.navigate
→ 导航页面

Runtime.evaluate
→ 在页面里执行 JavaScript

DOM.getBoxModel
→ 获取某个 DOM 节点的位置和尺寸

Input.dispatchMouseEvent
→ 注入鼠标事件

Accessibility.getFullAXTree
→ 获取可访问性树
```

所以大量 Chromium 自动化框架最终都会直接或间接使用 CDP 提供的能力。

但不要把：

```text
所有浏览器自动化 = CDP wrapper
```

当成绝对结论。

Playwright 支持 Chromium、Firefox、WebKit，它提供的是跨浏览器统一抽象，不等同于单纯包装 CDP。

---

# 5. 为什么有 CDP 还需要 Playwright

CDP 给的是原始能力。

假设想点一个按钮，直接 CDP 需要自己处理：

```text
元素出现了吗？
元素是否 visible？
还在动画吗？
被 overlay 挡住了吗？
需要 scroll into view 吗？
React 是否刚刚重建了 DOM？
iframe 在哪里？
点击后如何等待 navigation？
```

Playwright 把这些问题包装成更稳定的自动化语义：

```text
Locator
Auto-wait
Actionability
Page
Frame
BrowserContext
Event
Popup
Network wait
```

所以：

> **CDP 解决“浏览器能做什么”；Playwright 解决“程序如何可靠地做”。**

---

# 6. 广告为什么不一定会让 Playwright 点错

如果使用：

```js
await page.getByRole('button', { name: '提交' }).click();
```

Locator 不是：

```text
提交按钮 = (500, 300)
```

而更像：

```text
当前页面中 role=button 且 accessible name=提交 的元素
```

所以广告把按钮从 y=300 挤到 y=700，通常不影响 Locator。

真正容易出现问题的是：

```text
1. 纯坐标操作
2. 广告 / modal overlay 挡住元素
3. selector 写得太宽导致歧义
4. Browser Agent 自己判断错了目标
5. DOM 动态重建导致旧 element handle / ref stale
```

Playwright 能解决大量“可靠执行”问题，但不会替 Agent 决定：

```text
当前应该关广告？
接受 Cookie？
登录？
绕过弹窗？
这个按钮是否符合用户目标？
```

所以：

> **Playwright 解决“怎么点”；Browser Agent 还要解决“现在该点什么，以及点完以后任务是否成功”。**

---

# 7. Playwright 与 Ego 的区别

最小模型：

```text
Playwright
= Browser Automation Library

Ego
= Agent-oriented Browser Runtime / Control Layer
```

## Playwright

通常是程序员已经知道目标：

```js
page.getByRole('button', { name: '登录' })
```

```text
程序员
  ↓
Playwright
  ↓
Browser
```

## Ego

Agent 一开始只知道用户目标，不知道页面有哪些控件：

```text
LLM
 ↓
Semantic Snapshot
 ↓
@1 textbox "Email"
@2 button "Login"
 ↓
Agent 决定目标
 ↓
执行动作
```

Ego 还增加了 Agent Runtime 语义：

```text
Task Space
ownership
handoff
takeover
persistent tabs
human-in-the-loop
```

所以 Ego 很多 API 看起来像 Playwright，并不代表 Ego = Playwright。

可以理解为：

```text
Playwright
→ 给程序员可靠控制浏览器

Ego
→ 给 Agent 眼睛、手、工作空间和控制权模型
```

---

# 8. Browser Agent 的标准闭环

不要只记 click / fill。

真正完整链路是：

```text
Observe
  ↓
Decide
  ↓
Locate
  ↓
Act
  ↓
Wait
  ↓
Verify
```

尤其：

```text
click() resolve
≠
业务任务成功
```

例如：

```text
点击登录
  ↓
服务器可能返回
├── success
├── 密码错误
├── captcha
└── timeout
```

所以必须在 Action 后验证业务后置条件。

---

# 9. Code Mode 为什么存在

传统 Tool Calling 的控制流：

```text
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
LLM
```

每当 Tool 返回结果，如果下一步需要根据结果决定动作，就必须再次调用模型。

例如：

> 读 `a.ts` 和 `b.ts`；如果 `a.ts` 包含 `foo`，把它改成 `bar`。

普通模式：

```text
LLM #1
 ↓
read a + read b
 ↓
Tool Results
 ↓
LLM #2
 │ 看到 a 中有 foo
 ↓
edit a
 ↓
Tool Result
 ↓
LLM #3
 ↓
final answer
```

Code Mode：

```js
const a = await tools.read({ path: 'a.ts' });
const b = await tools.read({ path: 'b.ts' });

if (a.includes('foo')) {
  await tools.edit({
    path: 'a.ts',
    oldText: 'foo',
    newText: 'bar'
  });
}

text({ a, b });
```

执行：

```text
LLM #1
 ↓
JavaScript Runtime
 ├── read a
 ├── read b
 ├── if
 └── edit a
 ↓
LLM #2
 ↓
final answer
```

所以真正减少的是：

```text
Tool Calls
10 → 仍可能是 10

LLM Inference Rounds
6 → 可能变成 2
```

核心：

> **Code Mode 不减少真实 Tool 执行次数；它减少 Tool 之间需要 LLM 再次参与控制决策的次数。**

---

# 10. Code Mode 什么时候真正有价值

特别适合确定性控制流：

```text
if / else
for / while
Promise.all
批处理
分页
过滤
排序
数据转换
重试
聚合
```

例如搜索 100 个文件：

```js
const hits = await tools.grep({ pattern: 'TODO', path: 'src' });
const files = unique(hits.map(x => x.path));

const contents = await Promise.all(
  files.map(path => tools.read({ path }))
);

const targets = contents.filter(countTodosGreaterThan10);

for (const target of targets) {
  await tools.edit(...);
}
```

这期间可以有很多 Tool Calls，但没有必要每一步都重新调用模型。

不适合替代：

```text
开放式语义理解
复杂判断
需要新的创造性计划
```

这类场景 ToolResult 返回后仍需要重新让 LLM 思考。

---

# 11. 为什么 Code Mode 比“直接让模型写 Bash”更通用

写 Bash 也能减少 LLM turn：

```text
LLM
 ↓
一段 shell script
 ↓
几十个系统步骤
```

但 Bash 的问题是：

```text
shell
 ↓
直接拥有 OS 能力
```

Code Mode 可以做到：

```text
JavaScript
 ↓
tools.read()
tools.edit()
tools.browser()
 ↓
Tool Gateway
 ↓
权限 / Schema / Trace / Sandbox
```

因此它把：

> “一次写程序完成很多步骤”

扩展到任意 Tool，同时保留 Tool Runtime 的安全边界。

---

# 12. `tools.read()` 不是文件 SDK，它只是 Proxy

不要实现成：

```js
tools.read = fs.readFile;
tools.bash = child_process.exec;
```

否则会绕过原来的 Agent 权限系统。

正确模型：

```text
tools.read({ path })
       │
       │ nested tool request
       ▼
Pi Tool Dispatcher
       │
       ├── tool exists?
       ├── schema valid?
       ├── session allowed?
       ├── workspace path allowed?
       ├── approval required?
       └── trace
       │
       ▼
原来的 read.execute()
```

所以：

```text
模型能写出危险参数
≠
危险操作能够执行
```

权限边界必须仍在 Host / Tool Gateway。

---

# 13. Tool Registry 如何变成动态 JS SDK

Pi 本来就已经有 Tool Registry：

```text
ToolSpec
├── name
├── description
├── parameters/schema
└── execute()
```

Code Mode 可以把允许暴露的工具映射成 JS Proxy：

```js
const tools = new Proxy({}, {
  get(_, toolName) {
    return async (args) => {
      return callHostTool(toolName, args);
    };
  }
});
```

所以：

```js
tools.read(...)
tools.edit(...)
tools.browser(...)
```

不是提前生成几百个真实 JS SDK 文件，而是：

```text
Tool Registry
     ↓
Exposure Filter
     ↓
Dynamic Proxy
     ↓
Nested Tool Call
```

同时可以暴露：

```js
ALL_TOOLS
```

只作为工具目录元数据，用于搜索当前可用能力。

---

# 14. 为什么不是所有 Tool 都暴露进 Code Mode

最好分三层：

```text
Global Tool Registry
      ↓
Session Policy
      ↓
Session Tools
      ↓
Code Mode Exposure Policy
      ↓
Code Mode Tools
```

例如：

```text
read            → Code Mode 可用
edit            → Code Mode 可用
browser         → Code Mode 可用
send_email      → 可能 Direct Only
payment         → Direct Only / Require User Approval
delete_account  → 禁止 Code Mode
```

Code Mode 只改变编排方式，不能扩大 Session 权限。

---

# 15. 用 Pi 基础工具实现 Code Mode

Pi 的基础工具：

```text
read
edit
write
bash
```

增加一个：

```text
exec
```

模型只需要调用：

```text
exec({ code })
```

概念实现：

```ts
pi.registerTool({
  name: 'exec',
  description: 'Run JavaScript that orchestrates allowed Pi tools',
  parameters: {
    type: 'object',
    properties: {
      code: { type: 'string' }
    },
    required: ['code']
  },

  async execute(toolCallId, params, signal, ctx) {
    return codeModeHost.run({
      code: params.code,
      sessionId: ctx.sessionId
    });
  }
});
```

JS Runtime 内最小只暴露：

```text
tools
ALL_TOOLS
text()
```

不要直接暴露：

```text
fs
child_process
任意 network
完整 process
```

否则 Tool Gateway 会失去意义。

---

# 16. Pi Code Mode 最关键的内部接口

理想链路：

```text
exec
 ↓
JS Runtime
 ↓
tools.read()
 ↓
invokeNestedTool('read', args)
 ↓
Pi 原 Tool Dispatcher
 ↓
原 Tool pipeline
```

因此 Pi Core 最值得提供的是一个很薄的内部 seam：

```ts
ctx.invokeTool(name, args)
```

或者：

```text
ToolDispatcher.invokeNestedTool()
```

绝不能为了 Code Mode 再实现一套：

```text
read2
edit2
bash2
permission2
```

否则会产生两个 Authority / Tool Runtime。

---

# 17. PAW 为什么比 Codex 截图里的 Browser 链更简单

Codex 例子里出现：

```text
exec
 ↓
node_repl
 ↓
Browser Skill
 ↓
iab
 ↓
tab.playwright
 ↓
Browser
```

这是因为它需要通过一个持久 Node REPL 加载自己的 In-App Browser SDK。

PAW 已经有：

```text
browser tool
 ↓
BrowserControlService
 ↓
ego-browser
 ↓
globalThis.ego
 ↓
CDP
 ↓
PAW Chromium
```

所以 PAW Code Mode 可以直接：

```js
await tools.browser({
  op: 'run',
  script: `
    const task = await taskSpaces.useOrCreate('search baidu');
    await browser.openOrReuseTab('https://www.baidu.com', { wait: true });
    console.log(await page.info());
  `
});
```

不需要复制 Codex 的 `node_repl → iab → tab.playwright` 路径。

---

# 18. 浏览器与 Code Mode 的共同设计思想

Ego Browser 和 Code Mode 看起来是两个话题，其实核心思想很接近：

```text
原子 Tool 模式：

LLM → click
LLM ← result
LLM → fill
LLM ← result
LLM → wait

程序化 Runtime：

LLM → 一段 JS
      ├── click
      ├── fill
      ├── wait
      └── verify
    ← 一次结果
```

本质都是：

> **让 LLM 做开放式规划，把已经确定的机械控制流交给程序执行。**

这能减少：

```text
模型调用次数
token 往返
延迟
重复决策
```

同时程序层还能提供：

```text
并行
条件控制
重试
超时
状态检查
```

---

# 19. 面试压缩版

## Playwright vs Ego

> Playwright 主要解决程序如何可靠操作浏览器，它提供 Locator、Auto-wait、Actionability、Frame、Context 等自动化抽象；Ego 面向 Agent，又增加了 Semantic Snapshot、Ref、Task Space、Ownership 和 Human Handoff，让模型可以先理解页面，再动态决定下一步动作。

## Code Mode

> Code Mode 不是把十次工具调用变成一次，而是把十次工具调用之间原本需要模型参与的 if、循环、过滤和批处理控制流放进一个 JavaScript Runtime。真实 Tool Call 仍然经过原 Tool Dispatcher，因此权限、Schema、Approval、Trace 和 Sandbox 不应该被绕过。

## 为什么减少 Turn

> 普通 Tool Calling 每次拿到结果后，如果下一步依赖这个结果，通常需要重新调用模型。Code Mode 让 JavaScript 可以直接根据 ToolResult 做确定性判断，所以 Tool Calls 数量可能不变，但 LLM inference rounds 可以明显减少。

---

# 20. 反向自测

```text
Q1：为什么 React 页面不能只读取服务器返回的 HTML？

Q2：DOM、AX Tree、Layout、Pixels 分别解决什么问题？

Q3：为什么 Locator 不等于屏幕坐标？

Q4：广告把按钮位置改变，为什么 Playwright 通常不会点错？

Q5：为什么 click() 成功仍然不能证明任务完成？

Q6：Playwright 和 Ego 分别解决哪一层问题？

Q7：为什么有 CDP 以后仍然需要 Playwright？

Q8：Code Mode 为什么不一定减少 Tool Call 数量，却能减少 LLM Turn？

Q9：为什么 tools.read() 不能直接实现成 fs.readFile()？

Q10：Code Mode 为什么必须重新进入原来的 Pi Tool Dispatcher？

Q11：哪些工具应该只允许 Direct Call，而不应该进入 Code Mode？

Q12：为什么 PAW 不需要复制 Codex 的 node_repl + IAB 链路？
```

---

# 21. 下一步学习

按这个顺序继续：

```text
Browser Page / Context / Frame / Tab
        ↓
Playwright Locator + Auto-wait
        ↓
CDP Session / Target / Domain
        ↓
AX Tree / Snapshot / Ref Resolver
        ↓
Ego Task Space / Ownership / Handoff
        ↓
Browser Agent 并发与隔离
        ↓
Code Mode Nested Tool Runtime
        ↓
Pi / PAW 中实现安全的 Code Mode Host
```
