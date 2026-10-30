---
id: CRN-TYPE-DESIGN
type: doctype
title: 'design：设计方案'
revision: 1
name: design
meta: false
purpose: '系统或功能的实现方案，说明怎么满足需求、在决策约束下怎么做。'
---

# design：设计方案

系统或功能的实现方案，说明怎么满足需求、在决策约束下怎么做。

## 写法

可以引用多份需求和决策，但只写本方案负责的部分。跨模块的合计、优先级放到联合设计中。

## 关系

通过 `satisfies` 指向需求，通过 `governed_by` 指向决策，通过 `depends_on` 指向所依赖的规范或其他设计。

## 例子

- 属于：OTA 方案、模块设计、数据流设计。
- 不属于：需要对外严格遵守的接口（→ specification）。
