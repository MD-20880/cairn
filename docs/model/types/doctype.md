---
id: CRN-TYP-010
type: doctype
revision: 1
name: doctype
meta: true
purpose: '元模型中一种文档类型的定义，即本目录下的文件。'
---

# doctype：类型描述

元模型中一种文档类型的定义，即本目录下的文件。

## 写法

文件 meta 中的 `name`、`purpose` 是机器可读的定义，正文是语义说明。修改语义视为元模型变更。

## 关系

被 `relations/` 中关系描述的起点、终点约束引用。

## 例子

- 属于：`docs/model/types/*.md`。
- 不属于：项目自行追加的类型说明（写在项目中，不进 cairn）。
