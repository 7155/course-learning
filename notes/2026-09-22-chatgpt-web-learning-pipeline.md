# 2026-09-22｜ChatGPT Web 采集与 Course Learning 自动整理

> 今日主线：把“每天让模型回忆聊天”改成“聊天发生时保存证据，之后再自动整理知识”。
>
> 范围说明：本次日结尝试检索 2026-09-22 的历史会话，但历史会话检索没有成功返回结果。因此本记录基于当前可见对话与已确认的项目上下文，不把它标记为当天全部 ChatGPT 对话的完整清单。

## 今日解决的问题

目标原本是：每天自动读取 ChatGPT Web 中今天问过的问题和回答，再整理进 Course Learning。

今天明确了一个关键边界：

```text
ChatGPT 历史引用
= 按相关性召回过去信息

完整学习采集
= 真实保存每条 Q/A
+ 保证顺序
+ 可回溯
+ 可判断是否漏项
```

因此，如果目标是“尽量完整、不靠模型记忆”，需要一个浏览器侧 Collector，而不能只靠 scheduled summary。

## 最终心智模型

```text
ChatGPT Web
    ↓
Browser Collector
    ↓
Raw Conversation Evidence
    ↓
Code: 去重 / 会话边界 / 时间过滤
    ↓
JEV: 分类 / 路由 / 合并候选 / review gate
    ↓
LLM: 解释 / 教材化 / 生成练习题
    ↓
dg-ai-notes / Obsidian
    ↓
Course Atlas
```

最重要的两个分层：

```text
Raw Evidence
= 我真的问了什么、GPT 真的答了什么
= append-only

Derived Knowledge
= 我最终理解了什么
= 可以合并、修订、纠错
```

以及：

```text
authoring_progress
= 内容有没有被整理、写入、索引

learning_progress
= 是否能独立复述、实现、验证
```

今天只完成了前者，不把自动生成笔记当成“已经掌握”。

## 为什么选择事件驱动采集

不建议每天晚上自动点击 ChatGPT 左侧 Recents 再逐个爬会话，因为它容易受到虚拟列表、DOM 变化、网络加载和 SPA 路由影响。

更可靠的 V1：

```text
message 出现在页面
↓
MutationObserver
↓
解析 role / content / conversation
↓
生成稳定 identity
↓
去重
↓
assistant streaming finalize
↓
append Raw Store
```

这里要特别区分：

```text
DOM mutation
≠
一条新的逻辑消息
```

Assistant 流式回答必须先更新 temporary buffer，结束后只保存最终文本。

## JEV 在这条管线里的职责

JEV 适合有限答案空间中的语义判断，例如：

```text
learning / project / career / noise ?
已有 thread A / thread B / new ?
需要 review / 不需要 review ?
```

不适合让 JEV 替代 Raw Store，也不需要让它承担完整自由生成。

合理分工：

```text
Code → deterministic
JEV  → finite semantic decision
LLM  → open synthesis / explanation
```

这与已有 Jev Decision Layer 学习主线一致。

## 最小实验

先不要做整套系统。只做一个 conversation：

1. 捕获一个 user message；
2. 捕获一个 assistant final message；
3. 写入 JSONL；
4. 刷新、上下滚动、切换会话后检查是否重复；
5. 让 assistant 输出长文本，确认 streaming 只产生一个最终记录。

通过以后，再接每日聚类和知识整理。

## 必测失败模式

- **Streaming duplication**：一条 assistant 回答被保存几十次。
- **SPA route change**：切换 conversation 后消息归错会话。
- **Virtualized DOM**：旧消息重新进入 DOM 时被重复采集。
- **Selector drift**：ChatGPT 前端改版导致 parser 失效。
- **Local ingest unavailable**：本地服务关闭时事件被直接丢弃；应先进入 outbox。
- **Summary hallucination**：Derived Knowledge 写出 Raw Evidence 中没有的结论；重要知识必须能回溯来源。

## 与现有课程连接

这次主题可接回：

- Pi 消息系统：事件怎样成为消息并在层间传递；
- Pi 事件驱动：MutationObserver / streaming / finalize 与 Runtime 事件心智模型；
- 上下文工程：哪些历史需要进入当前 context；
- 会话管理：conversation identity、恢复与去重；
- Memory：evidence → atom / consolidated knowledge；
- PAW：把 ChatGPT Web 当成一种输入源，由 JEV 决定进入项目、学习或仅归档。

详细笔记已写入：

```text
7155/dg-ai-notes
pi-agent/docs/typescript/学习笔记-2026-09-22-ChatGPT网页采集与Course-Learning自动整理.md
```

## 今日复习题

1. 为什么“能引用过去聊天”不等于“能完整枚举所有 transcript”？
2. 为什么实时事件采集通常比晚上批量爬 Recents 更可靠？
3. 为什么每次 DOM mutation 不能直接当成一条新 assistant message？
4. Raw Conversation 为什么应该 append-only，而 Course Learning 可以修订？
5. 在这条管线里，Code、JEV、LLM 各自最适合承担什么？

## 状态

```yaml
date: 2026-09-22
source_scope: partial_current_visible_context
authoring_progress:
  status: captured
  detailed_note_written: true
  course_index_ready: true
learning_progress:
  status: unassessed
  verified_by_recall: false
  verified_by_code_experiment: false
next_learning_evidence:
  - 脱稿画出 ChatGPT Web → Collector → Raw → JEV → Course Atlas
  - 实现一个单会话 MutationObserver 采集实验
  - 通过 streaming / rerender / route-switch 去重测试
```
