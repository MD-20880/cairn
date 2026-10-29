---
id: CRN-REL-006
type: reltype
title: 'governed_by：受约束于'
revision: 1
name: governed_by
shape: arrow
from: ["*"]
to: [decision, standard]
view: neighbors
declared_in: file
---

# governed_by：受约束于

源文档受某项决策或工程规则约束。

## 声明

meta 中写 `governed_by: [ADR 或规则 ID]`，或正文写 `@governed_by(ID, 说明)`。

## 失效语义

指向 decision 时，决策被取代则源文档过时，需要复审；指向 standard 时，规则修订后按新规则复核合规。

## 视图

邻域：展开一层；终点是 decision 时一并显示它的取代链。

## 约束

终点只能是 decision 或 standard。
