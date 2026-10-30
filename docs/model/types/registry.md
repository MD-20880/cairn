---
id: CRN-TYPE-REGISTRY
type: doctype
title: 'registry：登记表'
revision: 1
name: registry
meta: true
purpose: '资产登记表本身。根登记表（默认文件名 `cairn.yaml`）和被包含的登记表文件都属于此类型。'
---

# registry：登记表

资产登记表本身。根登记表（默认文件名 `cairn.yaml`）和被包含的登记表文件都属于此类型。

## 写法

登记表记录文档在项目中的属性：路径、状态、适用范围、批准依据、完整性校验值。登记表以文件路径为身份，没有文档 ID，不出现在任何 `artifacts` 条目中。它不登记自己，是持续维护的文件，不冻结，不适用"较新指向较旧"的方向规则。

## 关系

只能作为 `registers` 和 `includes` 的起点，也只有它能发出这两种关系。只能作为 `includes` 的终点。

## 例子

- 属于：`cairn.yaml`、已有仓库的 `docs/engineering-artifacts.yaml`、任意名字的子登记表文件。
- 不属于：普通的文档索引页（不纳管，或作为 glossary/design）。
