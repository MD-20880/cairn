---
id: CRN-REL-AMENDS
type: reltype
title: 'amends：修订补充'
revision: 1
name: amends
shape: arrow
from: ["*"]
to: same
view: chain
declared_in: file
---

# amends：修订补充

新文档修改旧文档的一部分，旧文档仍然现行，阅读时必须与新文档叠加。

## 声明

新文档在 meta 中写 `amends: [旧 ID]`，或正文写 `@amends(旧 ID, 修改的章节)`。

## 失效语义

终点仍现行；终点被取代时，修订补充随之需要复审。

## 视图

链。

## 约束

两端类型必须相同。整体替代时用 `supersedes`。
