# 2026-09-18｜Jev System One：从可编程 Reranker 到 Agent Decision Layer

> 核心问题：Agent 里的每一步都需要生成式 LLM 吗？
>
> Jev 给出的答案是：不一定。很多步骤不是“创造一个答案”，而是“在已经存在的候选空间里做快速语义判断”。这类任务可以从主 LLM 中拆出来，交给一个低延迟、受约束、带概率输出的 Decision Model。

---

# 0. 最终总图

~~~text
                         Agent System
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
               Code          Jev          LLM
          deterministic   decision       generation
                 │            │            │
          数学 / 权限      选择 / 打分      推理 / 生成
          状态 / 事务      路由 / gate      规划 / 写代码
                 │            │            │
                 └────────────┼────────────┘
                              ▼
                           Harness
                              │
                     Tool / Browser / Memory
                              │
                           Observation
~~~

一句话：

> **代码处理确定性规则，Jev 处理“答案空间已知”的模糊语义判断，生成式 LLM 处理需要创造、规划和多步推理的问题。**

---

# 1. Jev 不是“小号 Chat LLM”

普通 LLM：

~~~text
messages / context
        ↓
       LLM
        ↓
next text / tool call
~~~

它解决：

> 下一步应该“说什么 / 做什么”？

Jev：

~~~text
state
+
questions
+
预定义答案空间
        ↓
       Jev
        ↓
typed answers + probabilities
~~~

它解决：

> 关于当前 state，我列出的这些判断分别是什么？

如果只定义 q1 到 q100，正常输出就对应 a1 到 a100。它不会自己扩展 q101、q102，也不会额外写一段自由解释。

工程上仍应验证缺失答案、异常字段、非法概率，并在异常时 fail-open / fallback。

---

# 2. 三种基础判断

## Noul：一个命题成立多少？

~~~text
“这个 Tool Result 未来还需要完整保留吗？”

→ 0.87
~~~

适合：

- 是否相关；
- 是否危险；
- 是否需要保留；
- 是否支持某个 claim；
- 是否值得升级到强模型。

## Choice：有限候选里选哪个？

~~~text
当前任务更适合：

systematic-debugging
implementation-planning
test-driven-implementation
none
~~~

输出是候选概率分布。

适合 Skill routing、Tool routing、Browser element selection、Model routing 和分类。

## Score：处于哪个等级？

例如把“回答覆盖程度”定义成 0～3 四个等级，Jev 返回各等级分布，再得到一个期望分数。

---

# 3. 为什么 Jev 不只是“LLM 强制 JSON”

TypeSafe 当前公开的核心区别：

~~~text
普通 LLM
→ token-by-token autoregressive generation

Jev
→ parallel decision outputs
→ 不生成自由字符串
~~~

官方将 Jev 描述为：

~~~text
new architecture
+
parallel sampler
+
RLCD
Reinforcement Learning for Calibrated Decisions
~~~

目前没有公开：

~~~text
参数量
Transformer 层数
hidden size
attention heads
具体 base model
decision head 结构
RLCD reward 公式
calibration loss 公式
完整训练数据组成
模型权重
~~~

因此不能说它“就是某个 LLM 后面接 Linear + Softmax”。

但可以确认：

> **它不是靠自回归把一大段 JSON 一个 token 一个 token 写出来。**

---

# 4. Jev 和 Reranker 的关系

传统 cross-encoder reranker：

~~~text
(query, document)
        ↓
    Transformer
        ↓
 relevance score
~~~

它高度特化于：

> “这个 document 与 query 有多相关？”

Jev 更像：

# Programmable Semantic Scorer

或者：

# Generalized Reranker

~~~text
传统 Reranker

固定 semantic relation
query ↔ document relevance
        ↓
一个 score
~~~

~~~text
Jev

state
+
运行时自然语言定义 relation
        ↓
relevant?
keep?
dangerous?
supports claim?
matches skill?
needs review?
        ↓
typed probabilities
~~~

因此 reranking 是 Jev 可以表达的一种特殊任务；但固定检索任务上，专用本地 reranker 仍可能更快、更便宜、更容易微调。

---

# 5. 最关键案例：Pi 的 Jev Compaction

问题：

~~~text
Agent 跑了很久
Context ≈ 200K+
~~~

Jev 本身吃不下这么大的原始历史。

真正流程：

~~~text
200K 原始 Agent History
        │
        │ deterministic preprocessing
        ▼
~20K Structured Skeleton
        │
        ▼
Jev 批量判断候选
        │
        ▼
KEEP / DROP_RESULT / DROP_PAIR
        │
        ▼
修改下一次发给主 LLM 的 Context
~~~

## 5.1 200K → 20K 不是 Jev 做的

Harness 先构造 Skeleton。

原来：

~~~text
t17 read("src/auth.ts")

result:
██████████████████
20K tokens 文件内容
██████████████████
~~~

Skeleton 里变成：

~~~text
t17
tool = read
input = {"path":"src/auth.ts"}
result = "ok, 80000 chars (omitted)"
~~~

最大的 Tool Result 正文根本不发送给 Jev。

如果 Skeleton 仍然过大，再按代码规则：

~~~text
缩短 tool arguments
        ↓
缩短旧 message text
        ↓
collapse 很老的 message
        ↓
仍然过大
        ↓
跳过 Jev
        ↓
Pi 正常 summarization
~~~

这是结构裁剪，不是语义摘要。

---

# 6. 没看到正文，Jev 为什么还能判断 Tool Result？

Jev 没看到 t17 的 20K 正文，但会看到任务轨迹：

~~~text
User:
修 auth bug

t17 read auth.ts
→ ok, 80000 chars omitted

Assistant:
发现 refreshToken 可能有问题

t18 grep refreshToken
→ ok

t19 read refresh.ts
→ ok

t20 edit refresh.ts
→ ok

t21 test
→ passed
~~~

它判断的是：

> **从整个后续轨迹来看，未来是否还需要重新看到 t17 的完整结果？**

而不是：

> “t17 正文内部到底有哪些事实？”

所以这种方案适合大量已经被后续行动覆盖的 Tool Result。

但如果某份长文件里藏着一个后续没有重复出现的重要约束，Skeleton 可能无法知道。

因此必须配套：

~~~text
recent messages pin
instruction / plan file protection
副作用工具保护
原始结果仍持久化
recall / restore
Jev error → fallback
~~~

---

# 7. 50 个 Tool Call 怎么批量判断？

假设：

~~~text
state ≈ 18K Skeleton

candidate:
t1 ... t50
~~~

当前 Pi Compaction 对每个 Tool Pair 问两个 Noul：

~~~text
t1:
  keepCall?
  keepResult?

t2:
  keepCall?
  keepResult?

...

t50:
  keepCall?
  keepResult?
~~~

即：

~~~text
50 candidates
→ 100 semantic questions
~~~

Jev 可以针对同一个 state 批量回答很多 question：

~~~text
                  one state
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   call_t1?      result_t1?    call_t2? ...
       │             │             │
      .91           .12           .05
~~~

代码再转换：

~~~text
keepResult >= threshold
→ KEEP

否则 keepCall >= threshold
→ DROP_RESULT

否则
→ DROP_PAIR
~~~

当前实现会限制每批候选数量，因此 50 个候选可能拆成多个 request；每个 request 重复同一个 Skeleton state。

---

# 8. 为什么不是直接 KEEP / TRUNCATE / DROP？

完全可以用 Choice 做三分类。

但两个 Noul 多保留一层信息：

~~~text
P(call still matters)
P(full result still matters)
~~~

例如：

~~~text
edit a.ts

keepCall   = 0.96
keepResult = 0.05
~~~

表示：

> 修改发生过很重要，但工具返回正文不重要。

最终执行策略仍然由代码控制：

~~~text
Jev
→ semantic signal

Policy Code
→ actual action
~~~

---

# 9. State 必须结构化吗？

不必须固定成 context / goal / history。

Jev 的 state 可以是字符串或 JSON-like structured state。

Pi 插件选择结构化，是因为 Harness 已经先完成：

~~~text
原始 transcript
   ↓
配对 tool call/result
   ↓
过滤不能裁剪的候选
   ↓
敏感信息处理
   ↓
候选编号
   ↓
移除大块 result body
   ↓
生成 structured state
~~~

因此送给 Jev 的已经不是“脏的 Agent 历史”，而是一个任务明确、候选明确的 decision problem。

---

# 10. Jev 自己不维护 Agent Context

Jev 可以抽象成：

~~~text
a_t = f(state_t, questions_t)
~~~

它没有自己的 Agent Loop、Tool Runtime、Memory 或 Session。

连续调用：

~~~text
state_t
   ↓
Jev
   ↓
action_t
   ↓
Environment
   ↓
observation_t+1
   ↓
Harness 构造 state_t+1
   ↓
Jev
~~~

所以：

> **长期状态和语义记忆属于 Harness，Jev 只是 stateless decision function。**

---

# 11. 浏览器控制：Planner + Jev Policy

Browser Agent 可以拆成：

~~~text
             Strong LLM
          规划目标 / 子目标
                │
                ▼
              Harness
        维护 browser state
                │
                ▼
       当前 DOM / 可操作元素
                │
                ▼
               Jev
         choose next action
                │
       ┌────────┴────────┐
       ▼                 ▼
CLICK / SELECT       TYPE_TEXT
直接可执行             │
                      ▼
                   Small LLM
                生成自由字符串
                      │
                      ▼
                   Browser
~~~

Jev 很适合：

~~~text
点哪个按钮？
选哪个 DOM element？
当前是否完成？
是否应该 wait / back / scroll？
~~~

不适合：

~~~text
搜索框具体填什么？
邮件正文写什么？
SQL 怎么写？
下一阶段复杂计划是什么？
~~~

因为这些答案不是从候选中选择，而是需要创造新字符串。

---

# 12. 机器人也可以，但目前主要是高层离散 Policy

机器人高层动作：

~~~text
GRASP
RELEASE
INSPECT
MOVE_TO_A
STOP
~~~

可以是 Jev 场景。

但：

~~~text
(x, y, z)
joint torque
grip force
motor PWM
continuous trajectory
~~~

属于连续控制空间，不适合当前这种离散 typed decision。

更合理：

~~~text
High-level Planner
       ↓
      Jev
 discrete semantic policy
       ↓
GRASP / INSPECT / STOP
       ↓
robot controller /
continuous learned policy
       ↓
motors
~~~

目前 Jev 还是云服务，因此更适合高层动作，不适合 100Hz / 1kHz 实时闭环。

真正大的想象空间会出现在：

> **可本地部署的 Jev-like System-One Model。**

如果未来能在本地 NPU / edge GPU 上几毫秒到几十毫秒运行，就可能进入桌面自动化、机器人高层控制、AR、游戏 NPC、智能家居等高频策略层。

---

# 13. 当前已经出现的应用

## Context Compaction

过去的 Tool Result → 未来还需要吗？

Pi Compaction 已经在做。

## Memory Semantic Recall

query + memory candidates → 哪些与当前问题语义相关？

Pi ecosystem 已经在做 Jev semantic recall ranking。

## Automatic Review Triage

当前工作 → 值不值得启动昂贵的 Buddy Reviewer？

明显低价值 review 可以跳过，复杂、高风险、不确定则升级到完整 reviewer。

## RAG Passage Classification

对 passage 同时判断：

~~~text
relevant?
provides evidence?
contradicts premise?
contains injection?
~~~

## Skill / Tool / Model Routing

~~~text
task
  ↓
Jev
  ↓
哪个 Skill？
哪个 Tool？
cheap model 还是 strong model？
~~~

## Browser / Computer Policy

~~~text
current page
+
available elements
→ next discrete action
~~~

## Guard / Verify / Gate

~~~text
Tool Call
→ destructive?
→ irreversible?
→ suspicious?
→ needs review?
~~~

安全硬规则仍然由代码和 Sandbox enforce。

## 大规模 Semantic Feature / Map-Reduce

同一个对象一次问很多独立问题：

~~~text
document / event / trace
        ↓
几十个 semantic properties
        ↓
structured features
~~~

---

# 14. 最大想象空间：Agent Semantic Decision Layer

把这些场景放一起：

~~~text
RAG reranker
Memory scorer
Skill router
Model router
Context pruner
Safety guard
Browser selector
Review gate
~~~

表面不同，本质都是：

~~~text
当前 State
+
一组候选 / 一组命题
        ↓
语义判断
        ↓
概率
        ↓
Code Policy
~~~

所以更大的抽象是：

# Semantic Decision Layer

~~~text
                      Harness
                         │
              Semantic Decision API
                         │
        ┌────────────────┼────────────────┐
        │                │                │
       Jev         Local small model   Reranker
        │                │                │
        └────────────────┼────────────────┘
                         │
                  typed probabilities
                         │
                         ▼
                     Code Policy
~~~

真正值得沉淀的不是：

> “PAW 支持 Jev。”

而是：

> **PAW 能把 Agent 任务拆成 deterministic logic、semantic decision 和 generative reasoning，并用 Lab 验证哪一层最合适。**

---

# 15. 能力边界：不要把 System One 当 System Two

可以把 Jev 的“智商形状”理解成：

~~~text
单跳语义判断      强
分类 / routing    强
相关性 / 匹配     强
局部常识判断      强

复杂多跳推理      明显弱
精确数学          不适合
日期计算          不适合
长链规划          不适合
自由文本生成      不支持
长期状态维护      不属于模型本身
~~~

官方发布材料称，在 System-One-shaped workflow 上 Jev 可以与强 LLM 做可比判断，但这不是通用智力等价。

正确说法：

> **答案空间提前定义、一个人看一眼就能判断的任务，是 Jev 的甜区；需要连续推导、创造新内容和长期计划时，仍然需要生成式 / reasoning model。**

---

# 16. 真正的竞争对手：小模型 + 工程约束

应该比较：

~~~text
Jev
vs
Luna / Flash-class small LLM
vs
local Qwen small model
vs
专用 reranker / classifier
vs
deterministic rules
~~~

如果一个 Flash / Luna：

~~~text
同样准确
同样快
更便宜
还能自由生成
~~~

那没有理由为了 Jev 而 Jev。

反过来，如果 Jev 能做到：

~~~text
更低 latency
更低 decision cost
一次并行几十/上百判断
稳定 typed output
概率校准更好
~~~

才适合进入 Harness hot path。

所以真正看的是：

~~~text
Decision Quality
×
Latency
×
Cost
×
Calibration
×
Failure Rate
~~~

---

# 17. PAW 应该怎么实验

先放 Lab，不重构 Runtime：

~~~text
同一批 Decision Cases
          │
    ┌─────┼────────────┐
    ▼     ▼            ▼
   Jev   Luna      local model
    │     │            │
    └─────┼────────────┘
          ▼
       Comparison
~~~

优先：

~~~text
1. Context keep/drop
2. RAG claim-evidence support
3. Skill routing
4. Review / retry gate
5. Memory recall
~~~

指标：

~~~text
Accuracy / Macro F1
False-positive / false-drop rate
Calibration / Brier / ECE
P50 / P95 latency
input token usage
cost / 1k decisions
fallback rate
最终 Agent task success
~~~

Compaction 尤其不能只看 context reduction：

> **必须看 bad-prune 后 Agent 成功率有没有下降。**

---

# 18. 最终判断

Jev 不会替代：

~~~text
GPT / Claude / Qwen reasoning
代码生成
复杂规划
自由文本 Agent
连续控制模型
~~~

它更可能侵蚀的是：

~~~text
reranker
classifier
router
guard model
tool selector
context scorer
memory scorer
review gate
~~~

真正值得继续追踪的问题是：

> **Agent 系统里到底有多少昂贵 LLM 调用，本质上只是“看一下然后选一个”？**

这些调用才是 Jev 类模型真正可能拿走的计算量。

---

# 19. 面试压缩版

## 一句话

> **Jev 是一种面向程序决策的 System One 模型：输入 state 和预定义问题，直接并行输出 typed probabilities，而不是自回归生成文本；它更像自然语言可编程的 generalized reranker，适合 Agent 的 routing、context pruning、RAG scoring、memory recall、guard 和离散控制。**

## 30 秒版本

> 普通 LLM 解决“下一步生成什么”，Jev 解决“对于当前状态，我列出的这些判断分别是什么”。比如 Pi Compaction 会先把 200K 历史通过确定性代码变成约 20K 的 Skeleton，再让 Jev 对每个 Tool Call 的 call/result 是否继续保留批量打概率，Policy Code 决定 keep、drop result 或 drop pair。它更像可编程 reranker，而不是小号 GPT。

## 深挖追问

1. 为什么 200K context 不能直接交给 Jev？Skeleton 做了什么？
2. 没读取完整 Tool Result，Jev 为什么还能判断是否保留？风险是什么？
3. 为什么每个 Tool Pair 问两个 Noul，而不是一个三分类 Choice？
4. Jev 和传统 reranker 最核心的区别是什么？
5. 为什么 typed output 不代表 semantic correctness？
6. Browser Agent 里为什么 Jev 能选按钮，却不能自己生成搜索词？
7. 为什么 Jev 不应该替代 permission / sandbox？
8. 为什么长期状态应该留在 Harness？
9. 什么情况下 Luna / Flash 比 Jev 更合适？
10. PAW Lab 应该如何证明 Jev 有真实价值？

---

# 20. 最终心智模型

~~~text
Reranker
= 固定 semantic relation
→ score


Jev
= runtime-programmable semantic relation
→ typed probability


LLM
= open-ended reasoning + generation
→ new information / action


Harness
= state owner + policy owner + execution owner
~~~

最后只记一句：

> **Jev 的价值不是“更会思考”，而是把大量不需要生成的语义判断变成足够快、足够便宜、足够受约束的程序组件。**

---

# 参考

- TypeSafe AI：Introducing System One Models and Jev  
  https://typesafe.ai/blog/introducing-system-one-models-and-jev
- TypeSafe AI Models  
  https://docs.typesafe.ai/models
- TypeSafe System One  
  https://docs.typesafe.ai/concepts/system-one
- Choice  
  https://docs.typesafe.ai/primitives/choice
- Noul  
  https://docs.typesafe.ai/primitives/noul
- Score  
  https://docs.typesafe.ai/primitives/score
- TypeSafe RAG Passage Classification  
  https://docs.typesafe.ai/cookbooks/classifying_rag_passages
- TypeSafe Skill Suggestion  
  https://docs.typesafe.ai/cookbooks/skill_suggestion
- Pi Ecosystem Jev Integration  
  https://github.com/season179/pi-ecosystem/blob/main/docs/JEV.md
- Pi Jev Compaction  
  https://github.com/season179/pi-ecosystem/tree/main/packages/pi-compaction
- Pi Memory Semantic Recall  
  https://github.com/season179/pi-ecosystem/tree/main/packages/pi-memory
- Pi Buddy Jev Triage  
  https://github.com/season179/pi-ecosystem/tree/main/packages/pi-buddy
