---
id: CRN-REL-001
type: reltype
revision: 1
name: registers
shape: arrow
from: [registry]
to: ["*", "!registry"]
view: hidden
declared_in: registry
---

# registers：登记

登记表把一份文档纳入本项目，并携带它在本项目中的属性：路径、状态、适用范围、批准依据、完整性校验值。

## 声明

登记表的每个 `artifacts` 条目就是一条 registers 边。

## 失效语义

不适用。被登记文档变化时，由完整性校验值反映。

## 视图

不在 `show` 的关系列表中出现，携带的项目属性显示在节点头部。

## 约束

只能从登记表指向非登记表文档。登记表之间用 `includes`。
