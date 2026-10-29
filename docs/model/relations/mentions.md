---
id: CRN-REL-010
type: reltype
revision: 1
name: mentions
shape: arrow
from: ["*"]
to: ["*"]
view: count
declared_in: file
---

# mentions：提及

源文档提到了终点，例如 Markdown 链接或正文中出现的 ID。不表示工程依赖。

## 声明

不需要声明，由 cairn 从链接和正文自动抽取，basis 为 mention。

## 失效语义

不传播。

## 视图

只计数：默认只显示条数，加 `--all` 时逐条列出。

## 约束

不参与任何强制性检查。
