# 2026-09-19｜近期学习总览与本地教材衔接

范围：2026-09-09 至 2026-09-19 的 7 篇仓库学习笔记，加上月初 PAW 可视化学习项目的精选内容。远端阅读基线为 `dc0c05ed8fe206bf8c73d199da8ba142bdcf3e27`。

这里整理的是“记录覆盖了什么”，不是宣告个人已掌握全部内容。笔记里的产品状态和模型能力描述，也没有在本次逐项重新核验。

## 最近的学习主线

最近的主题可以归纳为一个问题：怎样把模型能力组织成可执行、可恢复、可衡量的 Agent 系统？

| 日期 | 已有笔记 | 本篇最值得保留的理解 | 后续复习检查 |
| --- | --- | --- | --- |
| 09-09 | [Pi Extension 与 Harness](2026-09-09-pi-deepseek-harness.md) | Factory、DI、Registry、Hook、生命周期分别解决不同问题 | 能否解释插件退出后谁撤销监听和资源 |
| 09-11 | [Pi Agent Runtime](2026-09-11-pi-agent-runtime-deep-dive.md) | 从用户输入、上下文构造到 ToolResult 回填；模型与执行层分工 | 能否讲清一次 read 为什么要后续模型请求 |
| 09-11 | [Browser 与 Code Mode](2026-09-11-browser-control-and-code-mode.md) | 观察、定位、行动、验证；确定性控制流可交给程序 | 能否区分工具次数、模型轮次和任务完成 |
| 09-13 | [Agent Sandbox](2026-09-13-agent-sandbox-runtime.md) | 应用策略与底层执行隔离互补；Execution Runtime 抽象 | 能否说明只隔离 bash 留下哪些路径 |
| 09-18 | [GIS 数据与计算放置](2026-09-18-paw-gis-agent-data-compute-placement.md) | 按数据位置、规模和下游需求选择 Local/GEE | 能否区分 Tile、Evaluate 和 Export 的产物 |
| 09-18 | [Jev 语义决策](2026-09-18-jev-system-one-agent-decision-layer.md) | 确定性代码、有限语义判断、开放生成的分工 | 能否说明 typed output 为什么不保证判断正确 |
| 09-19 | [DeepDOC PDF 解析](2026-09-19-ragflow-deepdoc-pdf-parsing.md) | 文本识别之外，还要恢复布局、表格与阅读顺序 | 能否解释原生文本可用时哪些视觉计算仍有价值 |

## 把零散主题串起来

```text
资料进入：PDF → 结构 → chunk → 检索与排序 → 上下文
                                          ↓
任务运行：Session → 模型/语义决策 → Tool Dispatcher
                                  ↓
                         Browser / GIS / 代码执行
                                  ↓
                        Observation → 下一轮决策
                                  ↓
                     持久化、压缩、恢复与效果评测
```

Extension/Harness 解释能力怎样接进来；Runtime 解释能力怎样被调用；Browser 和 GIS 解释不同执行环境；Sandbox 解释执行边界；DeepDOC 与 RAG 解释知识怎样进入上下文；Jev 提供一种有限语义决策的研究方向。

## 从月初 PAW 可视化学习项目补入的五块

1. [RAG 检索、排序与评测](../topics/rag-retrieval-and-evaluation.md)：接在 DeepDOC 后，补上结构解析如何影响 chunk、召回和答案，以及如何通过消融验证优化。
2. [工具发现与上下文预算](../topics/tool-discovery-and-context-budget.md)：连接 Extension、Registry 和 Code Mode，区分工具可见、schema 加载与真正执行资格。
3. [请求接受与恢复](../topics/session-recovery-and-memory.md)：连接 Session 与 Sandbox，区分超时、接受未知、原请求恢复和重新执行。
4. [前后端事件与快照同步](../topics/frontend-event-snapshot-recovery.md)：补足断线、迟到响应与终态保护。
5. [质量优先的 Agent 评测](../topics/quality-first-agent-evaluation.md)：区分失败、未知与可比较候选。

原项目还有 Room、输入法训练、App 交付与分布式基础等课，本次不全量复制。

本地材料只作为历史学习来源。筛选范围与来源版本见[来源说明](../sources/local-learning-site.md)。

## 建议的复习顺序

先复述一次完整 Tool Loop，再复习工具发现与执行边界；然后串起 PDF→RAG→证据回答；最后讨论长任务恢复和语义决策的取舍。GIS 用作综合案例：输入数据在哪里、工具在哪里运行、返回什么产物、失败后如何确认结果。

每次只验证一个机制：给出具体输入，说明谁拥有状态、哪个模块产生副作用、失败如何回传、用什么证据确认完成。能脱稿解释并用真实源码或小实验验证后，再记录为“已验证掌握”。

## 三个贯穿自测

<details><summary>为什么 click 成功、Tile 出现和上传请求发出，都不能直接算任务完成？</summary>

这些只是中间动作或表示。任务可能要求页面业务状态变化、本地 GeoTIFF 文件或持久化上传结果。需要检查用户目标对应的产物与真实状态，而不是把中间工具成功替代最终验收。

</details>

<details><summary>减少 token、模型轮次和搬运数据量，共同需要什么前提？</summary>

先明确下游真正需要什么信息和产物，再转移确定性工作、按需加载能力或只返回小结果；同时保持授权、证据和恢复路径。节省成本后仍需验证最终任务质量没有下降。

</details>

<details><summary>如何区分“整理了课程”与“已经掌握”？</summary>

文档覆盖说明材料存在。掌握需要独立复述目的、数据流、所有权、失败与取舍，并能用可重复实验或源码定位支持解释。本次没有进行个人掌握度测验。

</details>
