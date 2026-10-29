---
id: CRN-REL-002
type: reltype
revision: 1
name: includes
shape: arrow
from: [registry]
to: [registry]
view: hidden
declared_in: registry
---

# includes：包含

一份登记表把另一份登记表文件登记的全部文档并入本项目。

## 声明

登记表的 `include` 列表，每项是一个相对本登记表文件所在目录的文件路径。目标是文件，不是文档 ID；文件名不限。

## 失效语义

不适用。被包含登记表的条目变化，直接反映为本项目的变化。

## 视图

默认隐藏。

## 约束

路径相对各自登记表解析；目标不得跨出根目录；同一文档被多处登记时，离根近者优先，同距且属性不同报错；成环报错；读取配置只看根。
