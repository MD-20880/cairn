---
id: CRN-REL-009
type: reltype
revision: 1
name: related
shape: line
from: ["*"]
to: ["*"]
view: neighbors
declared_in: file
---

# related：相关

两份文档相关，但不声明具体的依赖语义。

## 声明

在任意一端的 meta 中写 `related: [ID]`，或正文写 `@related(ID)`。只需在一端声明。

## 失效语义

不传播。

## 视图

邻域，默认不展开，加 `--all` 时显示。

## 约束

无向。能说清依赖方向时，改用有方向的关系。
