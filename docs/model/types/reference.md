---
id: CRN-TYP-007
type: doctype
title: 'reference：外部依据'
revision: 1
name: reference
meta: false
purpose: '原样保存的外部资料，作为工程依据使用。'
---

# reference：外部依据

原样保存的外部资料，作为工程依据使用。

## 写法

内容不可编辑，只能整体替换为新版本。通常是 PDF、图片等非 Markdown 文件，没有文件 meta，只能作为关系的终点。

## 关系

其他文档通过 `depends_on` 或 `mentions` 指向它。登记条目中用 `revision` 记录厂商版本。

## 例子

- 属于：芯片手册、原理图、PCB 资料包、外部标准。
- 不属于：自己写的解读或摘要（→ design 或 glossary）。
