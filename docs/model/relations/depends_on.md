---
id: CRN-REL-007
type: reltype
title: 'depends_on：依赖'
revision: 1
name: depends_on
shape: arrow
from: ["*"]
to: [specification, design, reference]
view: dag
declared_in: file
---

# depends_on：依赖

源文档的正确性依赖终点的内容，例如实现某个协议、基于某个设计、引用某个芯片手册。

## 声明

meta 中写 `depends_on: [ID]`，或正文写 `@depends_on(ID, 版本或章节)`。

## 失效语义

终点修订前进时，源文档需要确认兼容。

## 视图

依赖图：沿出边展开，`--depth N` 继续展开，遇到环停止并标出。

## 约束

只是参考、不影响正确性时用 `mentions` 或 `related`。
