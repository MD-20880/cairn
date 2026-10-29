---
id: CRN-REL-005
type: reltype
revision: 1
name: satisfies
shape: arrow
from: [design, specification, verification]
to: [requirement]
view: coverage
declared_in: file
---

# satisfies：满足

源文档满足某项需求，计入需求覆盖。

## 声明

meta 中写 `satisfies: [需求 ID]`，或正文写 `@satisfies(需求 ID, 条目编号)`。

## 失效语义

需求修订或被取代时，源文档需要复审。

## 视图

覆盖：从需求看，列出所有满足它的文档；没有时显示"未建立追溯"。

## 约束

终点只能是 requirement。只是提到某需求时用 `mentions`。
