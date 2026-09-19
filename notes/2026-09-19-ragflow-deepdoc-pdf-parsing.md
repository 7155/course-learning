# 2026-09-19｜RAGFlow DeepDOC：从 PDF Box 到 OCR、Layout 与阅读顺序

> 目标：不是记住 RAGFlow 调了哪些函数，而是能从问题重新推导出为什么需要 Box、什么时候 OCR、为什么还要 Layout，以及这些额外计算什么时候值得。

---

## 1. 第一性原理：PDF 解析真正难的不是“认字”，而是恢复文档结构

一个双栏论文页面：

~~~text
┌────────────────────────────────────────┐
│             3 Method                   │
│                                        │
│ A1：We propose...    B1：The result... │
│ A2：The model...     B2：Compared...   │
│ A3：It contains...   B3：Therefore...  │
└────────────────────────────────────────┘
~~~

人类阅读顺序是：

~~~text
3 Method
↓
A1 → A2 → A3
↓
B1 → B2 → B3
~~~

但 PDF 更像一个“二维绘图指令集合”，可能只是在不同坐标画文字：

~~~python
draw_text("A1", x=60,  y=120)
draw_text("B1", x=320, y=120)
draw_text("A2", x=60,  y=140)
draw_text("B2", x=320, y=140)
~~~

如果直接相信 PDF 内部的文本顺序，可能得到 A1 → B1 → A2 → B2。  
所以真正的问题是：

> **如何把 PDF 的二维页面重新恢复成人类的一维阅读顺序？**

---

## 2. 为什么需要 Box：把“字符串”升级成“空间对象”

DeepDOC 的核心中间表示不是纯字符串，而是类似：

~~~python
box = {
    "text": "The encoder is composed of six layers.",
    "page_number": 3,
    "x0": 61,
    "x1": 285,
    "top": 143,
    "bottom": 162,
}
~~~

有了 Box，后面才能回答：

- 它在哪一页？
- 属于左栏还是右栏？
- 上一个 Box 和它是否连续？
- 它落在哪个 Layout 区域？
- 它是否属于表格？
- 哪些 Box 应该合并成一个 paragraph？

因此：

~~~text
纯文本
→ 只知道“写了什么”

Box
→ 知道“写了什么 + 在哪里”
~~~

**结论：Box 不是 OCR 的副产品，而是整个文档理解链路的统一空间 IR（Intermediate Representation）。**

---

## 3. 为什么不直接对所有 PDF 做完整 OCR？

很多 born-digital PDF 本身已经有 text layer。例如 PDF 原生字符可以直接得到字符、坐标、字体等信息。

如果这些字符已经正确，再重新 OCR：

~~~text
PDF native text
→ 已经正确

再 OCR
→ 额外推理成本
→ 还可能识别错
~~~

所以 DeepDOC 的设计不是“所有文字重新 OCR”，而更接近：

~~~text
PDF native chars
+
OCR Detection 得到统一空间 Box
        ↓
优先把 native chars 填到 Box
        ↓
native text 不可信时
        ↓
才 OCR Recognition
~~~

---

## 4. OCR 必须拆成 Detection 和 Recognition

### Detection：哪里有字？

~~~text
页面图像
↓
Text Detector
↓
bbox1 / bbox2 / bbox3
~~~

输出主要是空间位置，例如：

~~~python
{
    "x0": 70,
    "x1": 190,
    "top": 98,
    "bottom": 122,
    "text": ""
}
~~~

### Recognition：这些字是什么？

输入已经裁好的 Box：

~~~text
┌─────────────────┐
│ 3.2 Attention   │
└─────────────────┘
~~~

输出文字和识别置信度。

因此：

~~~text
Detection = where
Recognition = what
~~~

在当前 DeepDOC PDF parser 中，先用 Detection 建立 Box；如果 PDF 原生文字可用，就不必对这个 Box 再跑 Recognition。

---

## 5. 一个 PDF 的真正决策树：什么时候 OCR，什么时候不用？

~~~text
                    PDF 第 N 页
                        │
            ┌───────────┴───────────┐
            │                       │
      PDF native chars          page image
            │                       │
            │                  OCR.detect()
            │                  找 text boxes
            │                       │
            └──────坐标 overlap─────┘
                        │
                给 Box 填 native text
                        │
              ┌─────────▼─────────┐
              │ native text 可用？│
              └─────────┬─────────┘
                    /           \
                  YES            NO
                   │              │
                   │       crop box image
                   │              │
                   │     OCR.recognize_batch()
                   │              │
                   └──────┬───────┘
                          ▼
                 统一可信 Text Boxes
                          │
                          ▼
                   Layout Analysis
~~~

### 什么情况下 native text 不可信？

典型情况：

1. 根本没有 text layer，例如扫描 PDF；
2. 大量 PUA / CID / replacement / control 等异常字符；
3. font encoding 映射异常；
4. native chars 无法正常填入 Box。

典型处理模式：

~~~python
if garbled_count / total_count >= threshold:
    b["text"] = ""
~~~

把 text 清空不是 OCR 本身，而是：

~~~text
当前结果质量检查失败
↓
invalidate
↓
统一走 fallback
~~~

后面再统一判断：

~~~python
if not b["text"]:
    # OCR recognition fallback
~~~

这是值得学习的工程模式：**先把不可信结果标记为 invalid，再统一进入 fallback。**

---

## 6. 为什么即使有 native text，还值得跑 Text Detection？

这是成本 / 收益 trade-off。

最便宜方案：

~~~text
PDF
↓
pdfplumber / PyMuPDF
↓
native text
~~~

优势是快，对干净 born-digital PDF 很合适。

问题是：

- 双栏阅读顺序可能混乱；
- 扫描页没有 text layer；
- 混合 PDF 难处理；
- 表格和复杂 Layout 容易碎；
- 不同 PDF 的内部文本顺序不统一。

DeepDOC 多做一步 Detection：

~~~text
页面 image
↓
Text Detection
↓
统一空间 boxes
~~~

额外成本：

- PDF rasterization；
- detection model inference；
- 后处理；
- 更多内存和实现复杂度。

换来的收益：

- 统一处理扫描 PDF / born-digital PDF / 混合 PDF；
- 恢复双栏阅读顺序；
- 为 Layout overlap 提供空间锚点；
- 为 paragraph merge 提供几何依据；
- 为 Table TSR 提供文字坐标。

因此真正应该比较的是：

~~~text
额外视觉推理成本
        VS
错误阅读顺序 / 错误 chunk 对 RAG 质量的损失
~~~

如果不用 Detection 导致 A1 B1 A2 B2 被直接嵌入成一个错误 chunk，那么即使 ingestion 更快，后面的 retrieval 已经建立在错误文本上。

---

## 7. 200 篇论文：什么时候应该走 DeepDOC？

### 场景 A：来源很干净

- 全是 arXiv born-digital PDF；
- text layer 完整；
- 编码正常；
- 版式比较规范；
- 表格和图片不是主要检索目标。

此时可以考虑：

~~~text
Native PDF parser
↓
native chars + 坐标
↓
轻量 reading-order / heading 恢复
↓
chunk
~~~

全量 DeepDOC 的额外视觉推理未必划算。

### 场景 B：来源很杂

~~~text
200 papers
├─ 50 arXiv
├─ 40 扫描老论文
├─ 30 中文期刊
├─ 30 双栏复杂排版
├─ 20 text layer 损坏
└─ 30 表格/图片很多
~~~

这时 DeepDOC 的统一链路更有价值：

~~~text
OCR Detection
+
native text reuse
+
OCR fallback
+
Layout
+
TSR
~~~

### 更成熟的生产方案：Cheap Preflight

~~~text
                     PDF
                      │
               Cheap Preflight
                      │
           ┌──────────┴──────────┐
           │                     │
        简单干净               复杂/异常
           │                     │
           ▼                     ▼
      Native fast path       DeepDOC path
           │                     │
           └──────────┬──────────┘
                      ▼
              unified document IR
~~~

工程上可以检查：

~~~python
def choose_parser(pdf):
    if text_coverage < 0.8:
        return "deepdoc"

    if garbled_ratio > 0.1:
        return "deepdoc"

    if scanned_page_ratio > 0.2:
        return "deepdoc"

    if layout_complexity > threshold:
        return "deepdoc"

    return "native"
~~~

这些阈值只是设计示意，应该通过自己的数据集评测确定。

---

## 8. 为什么 OCR 完了还必须有 Layout？

OCR 只回答“这里写了什么”，它不知道：

~~~text
这是标题？
正文？
表格？
图片 caption？
页眉？
reference？
equation？
~~~

例如 OCR 得到：

~~~python
[
    {"text": "3 Model Architecture", "bbox": "..."},
    {"text": "Most competitive neural...", "bbox": "..."},
    {"text": "Layer", "bbox": "..."},
    {"text": "Complexity", "bbox": "..."},
]
~~~

Layout 再赋予文档角色：

~~~text
3 Model Architecture        → title
Most competitive neural...  → text
Layer / Complexity          → table
~~~

所以：

~~~text
OCR
“字是什么？”
   ↓
Layout
“这个区域在文档里扮演什么角色？”
~~~

两者不是重复工作。

---

## 9. Layout 后为什么 Table 还需要 TSR？

Layout 只能告诉你“这一大片是 Table”。

OCR 可能只有：

~~~text
Model
Params
BLEU
Transformer
65M
28.4
~~~

还不知道：

~~~text
        C0           C1       C2

R0    Model        Params    BLEU
R1 Transformer      65M      28.4
~~~

所以还需要 TSR（Table Structure Recognition）：

~~~text
table crop
↓
row / column / header / spanning cell
↓
把已经识别出的文字映射回表格结构
~~~

TSR 主要是在恢复**表格结构**，不是再把所有表格文字重新 OCR 一遍。

---

## 10. 双栏为什么需要 Box？

假设 Box 左边界为：

~~~python
x0 = [
    62, 61, 64,
    314, 317, 313
]
~~~

明显形成两组：

~~~text
≈ 60   → 左栏
≈ 315  → 右栏
~~~

DeepDOC 后续可以根据空间特征做 column assignment。当前实现中可看到 KMeans / silhouette score 一类方法用于栏识别。

得到：

~~~python
A1["col_id"] = 0
A2["col_id"] = 0
A3["col_id"] = 0

B1["col_id"] = 1
B2["col_id"] = 1
B3["col_id"] = 1
~~~

于是阅读顺序不能只按 top 排序，而需要考虑：

~~~text
page
↓
column
↓
top
~~~

恢复成：

~~~text
A1
A2
A3
B1
B2
B3
~~~

---

## 11. Text Merge 为什么还需要？

PDF 可能把一句话拆成两个 Box：

~~~python
box1 = {
    "text": "The encoder is composed of",
    "page": 3,
    "col_id": 0,
    "top": 100,
}

box2 = {
    "text": "a stack of six identical layers.",
    "page": 3,
    "col_id": 0,
    "top": 115,
}
~~~

不能只因为相邻就合并，还要检查：

~~~text
同页？
↓
同栏？
↓
同 layout？
↓
垂直距离合理？
↓
标点/上下文支持连接？
↓
merge
~~~

最终恢复成完整 paragraph。

---

## 12. 论文标题层级：正则负责什么？

经过 OCR / Layout / Merge 后，假设得到：

~~~text
1 Introduction
3 Model Architecture
3.1 Encoder and Decoder Stacks
3.2 Attention
3.2.1 Scaled Dot-Product Attention
~~~

可以从零写：

~~~python
import re

heading_re = re.compile(
    r"^(?P<number>\d+(?:\.\d+)*)\s+(?P<title>.+)$"
)
~~~

对于：

~~~text
3.2.1 Scaled Dot-Product Attention
~~~

得到：

~~~text
number = 3.2.1
title  = Scaled Dot-Product Attention
~~~

层级可以直接：

~~~python
level = number.count(".") + 1
~~~

于是：

~~~text
3       → level 1
3.2     → level 2
3.2.1   → level 3
~~~

### 正则拆解

~~~regex
^(?P<number>\d+(?:\.\d+)*)\s+(?P<title>.+)$
~~~

- ^：字符串开始；
- \d+：一个或多个数字；
- (?:\.\d+)*：“.数字”整体重复 0 次或多次；
- \s+：一个或多个空白；
- (?P<title>.+)：命名捕获剩余标题文字；
- $：字符串结束。

要区分三个问题：

~~~text
正则 pattern 深度
→ 判断一级 / 二级 / 三级

文档出现顺序
→ 判断谁是谁的父节点

frequency / pivot
→ 决定主要在哪一级切 chunk
~~~

---

## 13. 整条 DeepDOC → Paper Chunking 心智模型

~~~text
PDF
│
├─ native chars
│
└─ page image
      │
      ▼
OCR Detection
“文字区域在哪里？”
      │
      ▼
统一 Text Boxes
      │
      ├─ native text 正常 → 直接填
      │
      └─ 缺失/乱码 → OCR Recognition fallback
      │
      ▼
可信 Text Boxes
      │
      ▼
Layout Analysis
“这些区域是什么？”
      │
      ├─ title
      ├─ text
      ├─ table
      ├─ figure
      └─ equation
      │
      ├─ table → TSR
      ├─ text  → column + text merge
      └─ title → heading regex
                     │
                     ▼
                 title level
                     │
                     ▼
                  hierarchy
                     │
                     ▼
                   pivot
                     │
                     ▼
                 token cap
                     │
                     ▼
                   chunks
~~~

---

## 14. 成本模型：以后看任何 Parser 都先算这笔账

~~~text
T_total
≈ T_pdf_parse
+ N_page × T_detection
+ N_bad_box × T_recognition
+ N_page × T_layout
+ N_table × T_tsr
+ T_merge
~~~

这不是精确 benchmark，而是用来判断设计是否值得的心智模型。

收益：

- 阅读顺序更准；
- 扫描件可用；
- Layout 更准；
- 表格结构可恢复；
- RAG chunk 质量提高。

成本：

- 模型推理；
- PDF rasterization；
- 内存；
- 实现复杂度；
- ingestion 延迟。

原则：

~~~text
收益很小 + 成本明显增加
→ 不一定值得走重 parser

不用就会产生错误 chunk
→ 收益远大于成本
→ 值得走 DeepDOC
~~~

---

## 15. 源码阅读入口

重点：

~~~text
deepdoc/parser/pdf_parser.py
  __ocr()
  _layouts_rec()
  _table_transformer_job()
  _text_merge()
  _assign_column()

deepdoc/vision/ocr.py
  detect()
  recognize_batch()
  __call__()

deepdoc/vision/layout_recognizer.py

rag/app/paper.py

rag/nlp/__init__.py
  BULLET_PATTERN
  bullets_category()
  title_frequency()
~~~

推荐阅读顺序：

~~~text
__ocr()
↓
Detection / native text / Recognition fallback

_assign_column()
↓
Box 如何恢复双栏

_layouts_rec()
↓
Box 如何映射 title/text/table

_table_transformer_job()
↓
为什么 Layout 后还需要 TSR

_text_merge()
↓
Box 如何恢复 paragraph

paper.py
↓
论文 section / pivot / chunk

BULLET_PATTERN
↓
标题正则和 hierarchy
~~~

---

## 16. 反向验证问题

1. 为什么 born-digital PDF 已经有文字，还要做 Text Detection？
2. Detection 和 Recognition 为什么拆开？
3. 为什么不应该对所有 native text 都重新 OCR？
4. 为什么 OCR 之后仍然必须有 Layout？
5. Box 的存在除了双栏排序，还解决了什么？
6. Layout 已识别 Table，为什么还需要 TSR？
7. 什么情况下应该走 Native fast path，而不是 DeepDOC？
8. 3.2.1 的标题 level 是如何得到的？
9. 标题 level、父子关系、pivot frequency 分别解决什么问题？
10. 如果 Detection 很慢，但数据全是标准 arXiv PDF，是否还应该全量使用？为什么？

---

## 一句话总结

> **DeepDOC 的关键不是“用 OCR 把 PDF 读出来”，而是用 Text Box 建立统一的二维文档表示：原生文字能用就复用，不能用才 OCR；Box 再支撑 reading order、Layout、TSR 和 text merge。是否值得走这条重解析链，最终取决于它提升的结构解析质量是否值得额外的视觉推理成本。**
