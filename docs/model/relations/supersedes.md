---
id: CRN-REL-003
type: reltype
title: 'supersedes：取代'
revision: 1
name: supersedes
shape: arrow
from: ["*"]
to: same
view: chain
declared_in: file
---

# supersedes：取代

新文档整体替代旧文档。新文档被接受后，旧文档退出现行，有效状态为 superseded。

## 声明

新文档在 meta 中写 `supersedes: [旧 ID]`，或正文写 `@supersedes(旧 ID)`。旧文档不修改。

## 失效语义

不适用：终点已退出现行。

## 视图

链：沿关系双向走到底，按先后顺序列出整条取代链。

## 约束

两端类型必须相同。只替代一部分时用 `amends`。
