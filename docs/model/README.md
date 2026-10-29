# cairn 元模型

本目录是 cairn 元模型的唯一来源，由两部分组成：

- [`types/`](types/)：文档类型，每种类型一个文件。
- [`relations/`](relations/)：关系类型，每种关系一个文件。

每个文件的 frontmatter 是机器可读的定义，正文是语义说明。cairn 运行时从这里读取元模型，不另行维护一份副本。

## 类型描述的字段

| 字段 | 含义 |
|---|---|
| `title` | 给人看的简短名称 |
| `name` | 类型名，小写，在登记表 `type` 字段中使用 |
| `meta` | 是否为元类型（registry、doctype、reltype） |
| `purpose` | 一句话说明用途，对应登记表 `types` 中的 `purpose` |

## 关系描述的字段

| 字段 | 含义 |
|---|---|
| `title` | 给人看的简短名称 |
| `name` | 关系名，在文件 meta 的 `relations` 和正文 `@rel()` 标记中使用 |
| `shape` | `arrow`（有向）或 `line`（无向） |
| `from`、`to` | 允许的起点、终点类型列表。`"*"` 表示任意类型，`"!x"` 表示排除类型 x，`same` 表示与起点同类型 |
| `view` | `show` 展开该关系的算法：`chain`、`coverage`、`dag`、`neighbors`、`count`、`hidden` |
| `declared_in` | 关系在哪里声明：`file`（文件 meta 或正文标记）或 `registry`（登记表） |

## 方向规则

`declared_in: file` 的 arrow 关系，一律由较晚写成的文档声明、指向较早的文档。已冻结的文档不因新增关系而修改。`declared_in: registry` 的关系由登记表声明，不受此规则约束。

## 扩展

项目可以在自己的仓库中追加类型或关系，写法与本目录相同，但不能改写这里已有定义的语义。
