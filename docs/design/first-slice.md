---
id: CRN-DSN-001
type: design
revision: 3
relations:
  satisfies: [CRN-REQ-001]
---

# cairn 第一阶段设计：ls / show / check

## 范围

三个命令 `ls`、`show`、`check`，共用一次构建的内存图：

- `ls` 选择节点，不涉及边；
- `show` 展示单个节点的邻域，按关系类型选择展开算法；
- `check` 报告不一致之处。

本阶段不包含：全文检索、查询语言、推断边、缓存，以及对目标仓库的任何写入。

## 属性归属

| 层 | 权威来源 | 属性 |
|---|---|---|
| 文件 | 文件 meta、正文标记 | `id`、`type`、`revision`、关系 |
| 项目 | 资产登记表 | `id`、`type`、`path`、`status`、`scope`、`sha256` |
| 派生 | cairn 计算 | 有效状态、反向关系、取代链、视图 |

`id` 和 `type` 两层都有：登记表需要它们建立 ID 到路径的映射，文件需要它们说明自己是什么。两者不一致时 `check` 报错。

## 元模型

元模型是 cairn 自带的受控定义，放在 `cairn/model/metamodel.yaml`，规定文档类型和关系类型。项目可以在映射配置中追加类型，不能改写已有类型的语义。

### 文档类型

`requirement`、`decision`、`design`、`specification`、`verification`、`standard`、`reference`、`glossary`。类型名一律小写，与 embedded-skills 的 artifact-conventions 一致。登记表 `types` 中声明的类型必须在元模型中存在。

### 关系类型

| rel | 形状 | 起点类型 | 终点类型 | 终点变化时，源文档应当 | 视图 |
|---|---|---|---|---|---|
| supersedes | arrow | 任意 | 与起点同类型 | 不适用（终点退出现行） | 链 |
| amends | arrow | 任意 | 与起点同类型 | 终点仍现行，阅读终点时须叠加本文 | 链 |
| satisfies | arrow | design、specification、verification | requirement | 复审，并计入需求覆盖 | 覆盖 |
| governed_by | arrow | 任意 | decision、standard | 指向 decision 时，决策被取代则源文档过时；指向 standard 时，按规则复核合规 | 邻域 |
| depends_on | arrow | 任意 | specification、design | 终点修订前进时，确认兼容 | 依赖图 |
| verifies | arrow | verification | 任意 | 终点修订前进时，验证需要重做 | 覆盖 |
| related | line | 任意 | 任意 | 不传播 | 邻域，默认不展开 |
| mentions | arrow | 任意 | 任意 | 不传播 | 只计数 |

arrow 的方向一律从较晚写成的文档指向较早的文档，关系由前者声明。line 不区分两端，只需在一端声明。

### 视图

视图决定 `show` 怎样沿一种关系展开：

| 视图 | 算法 |
|---|---|
| 链 | 沿关系双向走到底，按链上顺序输出 |
| 覆盖 | 从终点看，列出所有声明了该关系的源文档；没有源文档时标为"未建立追溯" |
| 依赖图 | 沿出边走一层；`--depth N` 可以继续展开，遇到环停止并标出 |
| 邻域 | 只展开一层 |
| 只计数 | 默认只显示条数，加 `--all` 时逐条列出 |

## 输入

- `--root`：目标仓库根目录，默认为当前目录。
- `--config`：项目映射配置文件。

映射配置示例：

```yaml
registry: docs/engineering-artifacts.yaml
id_pattern: '\b(?:XX-[A-Z]+-\d{3}|ADR-\d{4})\b'
compat:                     # 兼容层，可选
  field_relations:          # 字段名 → 关系类型
    需求依据: satisfies
    决策依据: governed_by
```

## 文件 meta

受控的 Markdown 文件以 YAML frontmatter 开头：

```yaml
---
id: XX-DSN-001
type: design
revision: 2
relations:
  satisfies: [XX-REQ-001]
  governed_by:
    - to: ADR-0014
      note: A1、A2 独立升级
  supersedes: [XX-DSN-000]
---
```

- `relations` 以关系类型为键。值是 ID 列表；需要携带内容时，列表项写成 `{to, note}`。
- `revision` 的格式由文档类型决定，cairn 只做字符串比较是否变化，不解释大小。
- 非 Markdown 文件（PDF、图片等）没有 meta，它们只能作为关系的终点。

## 正文标记

正文中可以用 `@rel(ID)` 或 `@rel(ID, 说明)` 声明关系，例如 `@governed_by(ADR-0014, 第 2 节)`。规则：

- `rel` 必须是元模型中的关系类型，否则 `check` 报 `E_REL_TYPE`。
- 代码块和行内代码中的标记不解析，这样文档可以引用语法示例。
- 同一条关系在 meta 和正文中都声明时，合并为一条边，两处都进入 `origin`。

## 数据模型

### Node

字段：`id`、`type`、`path`、`status`（登记表）、`status_effective`（派生）、`revision`、`scope`、`sha256`、`title`。

- `title` 取文件中的第一个 H1。
- `scope` 统一为字符串列表；登记表写成单个字符串时按一项处理。

### 有效状态

登记表的 `status` 表示文档在项目中的接受状态：`draft`（未接受）、`active`（已接受）、`archived`（退出使用）。有效状态在此基础上推导：

1. 存在一个 `active` 文档以 `supersedes` 指向本文档时，本文档的有效状态为 `superseded`。
2. 否则取登记表的 `status`。

登记表直接写了 `superseded` 时，与推导结果比较，不一致报 `W_SUPERSEDES`。

### Edge

字段：`src`、`rel`、`dst`、`basis`、`origin`（`文件:行号` 列表）、`note`（可选）。line 类型的边存储时按 `(min(src, dst), max(src, dst))` 规范化。

`basis` 的取值：

| basis | 来源 |
|---|---|
| declared | 文件 meta、正文标记 |
| field | 兼容层：登记表中的 `supersedes`、按 `compat.field_relations` 抽取的字段行 |
| inferred | 预留，本阶段不产生 |
| mention | Markdown 链接，或正文中出现的 ID |

同一 `(src, rel, dst)` 有多个来源时，保留 basis 最强的一条，其余来源并入 `origin` 列表。

## 兼容层

兼容层读取已有文控仓库的既有写法，产生 basis 为 `field` 的边：

1. 登记表条目的 `supersedes` 字段（单值或列表）。
2. 字段行：`字段名[:：] 值`（字段名可以加粗），或两列表格行 `| 字段名 | 值 |`。只认 `compat.field_relations` 中列出的字段名。值中每个匹配 `id_pattern` 的 ID、每个能解析到登记路径的链接，各生成一条边。

兼容层不读取正文中的状态声明（例如 ADR 的 `## Status`）。状态以登记表为准。

## 引用抽取

Markdown 链接相对所在文件解析：去掉锚点，统一使用 `/` 分隔，匹配登记表中的路径后得到 ID，生成 mention 边。无法匹配的链接记为未解析链接。

## 跨平台一致

完整性校验值与 embedded-skills artifact-conventions 的 `registry_check.py` 逐字节一致：

| 文件 | 计算方式 |
|---|---|
| Git LFS 指针文件 | 取指针中的 `oid sha256:` |
| LFS 跟踪的文件、含 NUL 字节的文件 | 原始字节 |
| 其他文本文件 | CRLF、CR 统一为 LF 后计算 |

解析文本时按 UTF-8 解码，换行统一为 LF。路径一律用 `/`，排序按 Unicode 码位，不受系统区域设置影响。

## 命令语义

### `cairn ls [ID-glob] [--type T] [--status S] [--scope X] [-a] [-l] [--json]`

- 只列出登记表中的节点，按 ID 排序；多个过滤条件之间是"且"的关系。`--status` 按有效状态过滤。
- 默认显示有效状态为 draft 和 active 的节点；加 `-a` 时同时显示 superseded 和 archived。
- 输出列：ID、类型、有效状态、修订、标题。
- 加 `-l` 时增加路径和出入度两列（不计 mention）。
- 结果为空时，退出码仍为 0。

### `cairn show <ID> [--all] [--depth N] [--json]`

- 头部：节点元数据，含登记表状态和有效状态。
- 关系：按关系类型分组，每组按该类型的视图展开，出边和入边都列出。
- 每行显示：关系、方向、对端 ID、对端有效状态、basis、origin、note。

### `cairn check [--all] [--json]`

每条发现包含 code、级别、节点、origin 和说明。存在 E 级发现时，退出码非 0。I 级发现默认按 code 汇总条数，加 `--all` 时逐条列出。

| code | 含义 |
|---|---|
| E_DUP_ID | 登记表中 ID 重复 |
| E_MISSING_FILE | 登记路径下没有文件 |
| E_SHA | 文件的完整性校验值与登记值不符 |
| E_META | meta 中的 `id` 或 `type` 与登记表不符，或 meta 无法解析 |
| E_DANGLING | 非 mention 边指向不存在的 ID |
| E_REL_TYPE | 关系类型不在元模型中 |
| E_TYPE_CONSTRAINT | 关系两端的文档类型不符合元模型 |
| W_NO_META | 受控 Markdown 文件没有 meta |
| W_SUPERSEDES | 登记表状态与推导出的有效状态不一致 |
| W_STALE_BASIS | active 节点有非 mention 边指向有效状态为 superseded 或 archived 的节点 |
| W_CYCLE | depends_on 或 supersedes 出现环 |
| I_UNRESOLVED_LINK | 链接指向未登记的路径 |

## 模块

使用 Python 3.10 及以上版本，外部依赖只有 PyYAML，命令行解析用 argparse。

| 模块 | 职责 |
|---|---|
| `model/metamodel.yaml` | 元模型定义 |
| `metamodel.py` | 读取并校验元模型 |
| `config.py` | 读取并校验映射配置 |
| `registry.py` | 读取登记表，生成 Node |
| `meta.py` | 解析 frontmatter 和正文标记 |
| `compat.py` | 兼容层抽取 |
| `extract.py` | 汇总各来源的 Edge，抽取引用 |
| `integrity.py` | 计算完整性校验值 |
| `graph.py` | 建立索引，推导有效状态 |
| `views.py` | 各视图的展开算法 |
| `checks.py` | 运行各项检查，输出发现 |
| `render.py` | 渲染文本和 JSON |
| `cli.py` | 命令入口 |

抽取与检查都实现为纯函数。每次调用都从零重建图。

## 输出确定性

- JSON 的键按字母排序。节点按 `id` 排序，边按 `(src, rel, dst, origin)` 排序。
- 输出中的路径一律使用 `/`，换行一律使用 `\n`，编码为 UTF-8。

## 测试样例

`tests/fixtures/basic/` 是一个合成仓库，包含一份登记表和约 10 份文档：

- **正常部分**：REQ；构成取代关系的新旧两份 ADR；被另一份 ADR amends 的 ADR；用 meta 声明关系的 SPEC；用正文标记声明关系的 DSN；带 verifies 的 VER。
- **反例**，每个只触发一个 check code：悬空 ID；校验值不符；meta 与登记表类型不符；未知关系类型；关系两端类型不符；active 规范引用已取代的 ADR；登记为 superseded 但后继仍是 draft；指向未登记文件的链接；没有 meta 的受控文件。
- **跨平台**：一份 CRLF 换行的文本文件和一个 LFS 指针文件，校验值与 LF、原始内容的结果相同。

`tests/fixtures/compat/` 用字段行和登记表 `supersedes` 声明关系，验证兼容层。`E_DUP_ID` 放在单独的 fixture 中测试。

## 验收

1. 三个命令在 fixture 上的 `--json` 输出与预期输出文件逐字节一致，在 Linux 和 Windows 上都成立。
2. 在 cairn 仓库自身上运行 `cairn show CRN-DSN-001`，显示一条指向 CRN-REQ-001 的 satisfies 边，basis 为 declared；运行 `cairn check` 没有发现。
3. 在一个真实的文控仓库上，借助兼容层，`show` 能还原一张事先手工整理的关系表，并正确标出每条边的 basis；`check` 报告的每条发现，要么能解释原因，要么据此修正映射配置。

## 待确认

- 兼容层是否需要读取 ADR 的 Nygard 行首声明（`Supersedes [..](..)`、`Amends [..](..)`），还是由目标仓库逐步改为 meta。
- 字段值中混有追溯性引用时（例如"与其他文档的关系"），由哪些字段进入 `compat.field_relations`。
- 关系携带的内容除 `note` 外是否需要结构化字段，例如所依据的章节。
