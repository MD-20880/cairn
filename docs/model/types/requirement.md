---
id: CRN-TYP-001
type: doctype
title: 'requirement：需求'
revision: 1
name: requirement
meta: false
purpose: '产品或系统必须满足的要求，是设计与验证的追溯起点。'
---

# requirement：需求

产品或系统必须满足的要求，是设计与验证的追溯起点。

## 写法

说明"必须做到什么"，不说明"怎么做"。条目应可验证、可编号。

## 关系

设计方案、规范通过 `satisfies` 指向它；验证记录通过 `verifies` 指向它。没有任何 `satisfies` 入边的需求，显示"未建立追溯"。

## 例子

- 属于：产品需求、系统要求、硬件要求清单。
- 不属于：实现方案（→ design）、接口约定（→ specification）。
