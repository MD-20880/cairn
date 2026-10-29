---
id: CRN-TYP-011
type: doctype
revision: 1
name: reltype
meta: true
purpose: '元模型中一种关系类型的定义，即 `docs/model/relations/` 下的文件。'
---

# reltype：关系描述

元模型中一种关系类型的定义，即 `docs/model/relations/` 下的文件。

## 写法

文件 meta 中的 `name`、`shape`、`from`、`to`、`view`、`declared_in` 是机器可读的定义，正文说明语义、失效语义和示例。

## 关系

引用 `types/` 中的类型名作为端点约束。

## 例子

- 属于：`docs/model/relations/*.md`。
- 不属于：某个项目中具体的一条关系（写在文档里，不是类型）。
