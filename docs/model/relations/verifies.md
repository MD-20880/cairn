---
id: CRN-REL-008
type: reltype
revision: 1
name: verifies
shape: arrow
from: [verification]
to: ["*"]
view: coverage
declared_in: file
---

# verifies：验证

验证文档证明终点（需求、规范或设计）被满足。

## 声明

验证文档在 meta 中写 `verifies: [ID]`，或正文写 `@verifies(ID, 被验证的版本)`。

## 失效语义

终点修订前进时，验证需要重做。

## 视图

覆盖：从被验证对象看，列出所有验证它的文档；没有时显示"未建立追溯"。

## 约束

起点只能是 verification。
