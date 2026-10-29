---
id: CRN-TYP-004
type: doctype
title: 'specification：规范'
revision: 1
name: specification
meta: false
purpose: '必须严格遵守的接口契约或协议，是不同模块、板卡、系统之间的约定。'
---

# specification：规范

必须严格遵守的接口契约或协议，是不同模块、板卡、系统之间的约定。

## 写法

内容是规范性的：读者按它实现，并按它判断对错。版本兼容规则写在规范内。

## 关系

实现方被 `depends_on` 指向它；验证记录通过 `verifies` 指向它。规范修订前进时，依赖方需要确认兼容。

## 例子

- 属于：板间链路协议、云端接口、文件格式。
- 不属于：实现说明或背景解释（→ design）。
