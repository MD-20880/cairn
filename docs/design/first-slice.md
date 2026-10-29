---
id: CRN-DSN-001
type: design
revision: 3
relations:
  satisfies: [CRN-REQ-001]
---

# cairn 第一阶段设计：检索

## 范围

第一阶段只做检索：给定一个问题，能找到相关文档，沿受控关系展开，并标出每份文档的现行状态和来源。三个命令共用一次构建的内存图：

- `find` 按关键词找到候选文档；
- `ls` 按类型、状态、范围列出文档；
- `show` 展示单个文档的关系，按关系类型选择展开算法。

三个命令都从项目根出发构建图（见"项目根"），都是只读的。

本阶段不包含：
- 文档管理：一致性检查 `check`、批准记录的核对、受管块、`init`、`sync`。这些的语义已在本设计中定义，见"后续阶段"，以免数据格式日后返工。
- 查询语言、相关度排序、推断边、缓存。

## 属性归属

| 层 | 权威来源 | 属性 |
|---|---|---|
| 文件 | 文件 meta、正文标记 | `id`、`type`、`revision`、关系 |
| 项目 | 登记关系 `registers`（登记表 → 文档）携带的内容 | `path`、`status`、`scope`、`approval`、`sha256` |
| 副本 | 文件 meta 中的受管块 `project` | 项目属性的副本，由工具写入，不是权威 |
| 派生 | cairn 计算 | 有效状态、反向关系、取代链、视图 |

项目属性既不属于文档，也不属于登记表，而属于两者之间的登记关系。同一份文档可以被多个登记表登记，各自携带不同的状态和范围。

`id` 和 `type` 在登记表条目和文件 meta 中都出现：登记表需要它们建立 ID 到路径的映射，文件需要它们说明自己是什么。两者不一致时 `check` 报错。

## 项目根

推导从一个根登记表出发：

1. 沿 `includes` 展开被包含的登记表，再沿各登记表的 `registers` 确定哪些文档属于本项目，以及它们的项目属性。图的节点集合由此确定。
2. 文档之间的关系从各文件的 meta 和正文标记中读取。
3. 有效状态、悬空引用、未登记链接由两者合并推导。

换一个根登记表，就是换一个项目视角：节点范围和项目状态可以不同，文档之间的关系不变。

根登记表就是 `cairn.yaml`。它是一份完整的登记表，可以自己登记所有文档，也可以包含其他登记表，同时承载读取配置。查找规则：

- 从当前目录开始向上查找最近的 `cairn.yaml`，找到即停。
- 一直找到文件系统根仍没有时，报错并退出，提示运行 `cairn init`。
- 环境变量 `CAIRN_CEILING_DIRS`（路径列表，分隔符同系统 `PATH`）可以限定查找上限，到达其中任一目录后不再向上。
- 最近的 `cairn.yaml` 就是根。子目录可以有自己的 `cairn.yaml`，在其中运行得到该子项目的视角；上级的 `cairn.yaml` 可以 `include` 它，两个视角由此相连。
- `--root DIR` 直接指定 `cairn.yaml` 所在目录，跳过查找。
- 所有 JSON 输出带 `root` 字段，值为 `cairn.yaml` 所在目录，使结果可以复现。

### 包含规则

1. **路径**：每份登记表中的路径，相对该登记表自己所在的目录解析。被包含的登记表单独使用时结果不变。
2. **范围**：`include` 的目标必须位于根目录之内，不跨出根目录。
3. **冲突**：同一份文档被多份登记表登记时，取 include 链上离根最近的那一份。距离相同且项目属性不同时报错。include 成环时报错。
4. **配置**：`id_pattern`、`compat` 等读取配置只从根 `cairn.yaml` 读取，被包含登记表中的配置忽略。
5. 被包含的登记表可以是任何符合 artifact-conventions 格式的登记表，例如已有仓库的 `docs/engineering-artifacts.yaml`。已有仓库只需在根目录新建一个包含它的 `cairn.yaml`，原登记表不用修改。

查找、读取、推导和校验只使用文件系统，不调用 git。

## 元模型

元模型的唯一来源是 [`docs/model/`](../model/README.md)：`types/` 中每个文件定义一种文档类型，`relations/` 中每个文件定义一种关系类型。文件的 frontmatter 是机器可读的定义，正文是语义说明。cairn 发布时把这两个目录打包进去，运行时从中读取，不另行维护副本。项目可以在自己的仓库中追加类型或关系，不能改写已有定义的语义。

- 文档类型：requirement、decision、design、specification、verification、standard、reference、glossary；元类型 registry、doctype、reltype。类型名一律小写，与 embedded-skills 的 artifact-conventions 一致。登记表 `types` 中声明的类型必须在元模型中存在。
- 关系类型：registers、includes、supersedes、amends、satisfies、governed_by、depends_on、verifies、related、mentions。

`cairn.yaml` 和被包含的登记表都属于元类型 registry。

### 视图

关系描述中的 `view` 决定 `show` 怎样沿该关系展开：

| view | 算法 |
|---|---|
| chain | 沿关系双向走到底，按链上顺序输出 |
| coverage | 从终点看，列出所有声明了该关系的源文档；没有源文档时标为"未建立追溯" |
| dag | 沿出边走一层；`--depth N` 可以继续展开，遇到环停止并标出 |
| neighbors | 只展开一层；`related` 默认不展开，加 `--all` 时显示 |
| count | 默认只显示条数，加 `--all` 时逐条列出 |
| hidden | 不在 `show` 的关系列表中出现，其内容作为节点的项目属性显示在头部 |

## 输入

- `--root`：`cairn.yaml` 所在目录，默认按"项目根"一节向上查找。

`cairn.yaml` 示例：

```yaml
schema_version: 1
integrity: sha256
id_pattern: '\b(?:XX-[A-Z]+-\d{3}|ADR-\d{4})\b'   # 读取配置，只在根生效
compat:                                           # 兼容层，可选
  field_relations:                                # 字段名 → 关系类型
    需求依据: satisfies
    决策依据: governed_by
include:                                          # includes 关系
  - docs/engineering-artifacts.yaml
  - components/boardcom/cairn.yaml
types:
  design:
    purpose: "设计方案"
artifacts:                                        # 每个条目是一条 registers 关系
  - id: XX-DSN-001
    type: design
    path: docs/design/link.md
    status: draft
    scope: ["project"]
    sha256: 3f9a…c21e
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
project:            # 受管块：由工具写入，手工不要修改
  status: active
  approval: "https://github.com/org/repo/pull/123"
---
```

- `relations` 以关系类型为键。值是 ID 列表；需要携带内容时，列表项写成 `{to, note}`。
- `project` 是受管块，保存项目属性的副本，让文档离开仓库时仍能看出状态。它由 `sync` 按登记表写入，内容以登记表为准。本阶段忽略它，状态一律从登记表读取。
- 文件 meta 中不出现由作者手写的状态字段。
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

### 登记条目

登记表的每个条目是一条 `registers` 边，除 `id`、`type` 外携带：`path`、`status`、`scope`、`sha256`，以及可选的 `approval`。

`approval` 记录批准依据：

```yaml
approval:
  ref: "https://github.com/org/repo/pull/123"   # 批准记录：PR、评审记录或 gate review
  sha256: 3f9a…c21e                             # 批准时内容的完整性校验值
```

`status` 为 `active` 时应当有 `approval`。批准绑定的是内容：当前校验值与 `approval.sha256` 不同，说明文档在批准后被修改过。

### 有效状态

登记表的 `status` 表示文档在项目中的接受状态：`draft`（未接受）、`active`（已接受）、`archived`（退出使用）。有效状态在此基础上推导：

1. 存在一个**被接受过**的文档以 `supersedes` 指向本文档时，本文档的有效状态为 `superseded`。被接受过，指登记表状态为 `active` 或 `superseded`（后者说明它曾被接受，之后又被更新的文档取代），或者登记条目带有 `approval`。最后一条覆盖已被接受、之后又被归档的后继：归档表示退出使用，不撤销它当初对前任的取代，前任不会因此恢复现行。没有 `approval` 的 `archived` 后继视为从未被接受。这样多代取代链 A ← B ← C 中，B 已被 C 取代，A 仍然保持 superseded。
2. 否则取登记表的 `status`；登记表写的是 `superseded` 时按 `active` 处理，因为它曾被接受，而取代它的文档还没有被接受。

推导只看关系和后继是否被接受过，不用本文档自己在登记表中的 `superseded` 来证明自己已被取代。登记表状态与推导结果不一致时报 `W_SUPERSEDES`，包括两种情况：登记为 `superseded` 但没有被接受过的后继（含后继仍是 draft，或根本没有后继）；登记为 `draft` 或 `active` 但已有被接受过的后继。

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

完整性校验值供后续的 `check` 使用，本阶段不计算。规则与 embedded-skills artifact-conventions 的 `registry_check.py` 逐字节一致：

| 文件 | 计算方式 |
|---|---|
| Git LFS 指针文件 | 取指针中的 `oid sha256:` |
| LFS 跟踪的文件、含 NUL 字节的文件 | 原始字节 |
| 其他文本文件 | 去掉 frontmatter 中的受管块后，CRLF、CR 统一为 LF 再计算 |

是否被 LFS 跟踪，通过直接解析根标记所在目录及各级子目录中的 `.gitattributes`（`filter=lfs`）判断，不调用 git。

受管块的去除规则：只在文件开头的 frontmatter 内生效；删除顶层键 `project:` 所在行，以及其后所有缩进的续行，直到下一个顶层键或 frontmatter 结束。其余字节保持不变。这样写入或更新受管块不改变校验值，冻结文件的受管块也可以由 `sync` 更新。`registry_check.py` 需要采用同一规则，在此之前，带受管块的文件两边的校验值会不一致。

解析文本时按 UTF-8 解码，换行统一为 LF。路径一律用 `/`，排序按 Unicode 码位，不受系统区域设置影响。

## 命令语义

### `cairn find <词>... [--type T] [--status S] [-a] [--expand] [--json]`

- 在登记节点的 ID、标题和正文中查找。多个词之间是"且"的关系；匹配不区分大小写（Unicode casefold），不做分词和词形还原。
- 结果分三档，依次为：ID 完全等于某个词；标题包含全部词；正文包含全部词。同一档内按正文命中次数降序，再按 ID 排序。排序只依赖文本，不引入向量或相关度模型。
- 默认只返回有效状态为 draft 和 active 的文档；加 `-a` 时包括 superseded 和 archived。
- 每条结果显示：ID、类型、有效状态、标题，以及最多 3 处正文命中的 `文件:行号` 和该行文字。
- 加 `--expand` 时，每条结果附带上下游各一层的非 mention 关系，相当于对每条结果做一次精简的 `show`。
- 没有结果时，退出码仍为 0。

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
- 对端 ID 未登记时，对端状态显示为"未登记"；关系类型不在元模型中时，该边仍然列出，并标为"未知关系"。检索不因数据不一致而中断，系统性的检查留给后续的 `check`。
- 某种覆盖类关系（satisfies、verifies）在本节点上没有任何入边时，显示"未建立追溯"。

## 模块

使用 Python 3.10 及以上版本，外部依赖只有 PyYAML，命令行解析用 argparse。

| 模块 | 职责 |
|---|---|
| `metamodel.py` | 从 `docs/model/` 读取并校验元模型 |
| `discover.py` | 查找项目根标记 |
| `config.py` | 读取并校验映射配置 |
| `registry.py` | 读取登记表，生成 Node |
| `meta.py` | 解析 frontmatter 和正文标记 |
| `compat.py` | 兼容层抽取 |
| `extract.py` | 汇总各来源的 Edge，抽取引用 |
| `graph.py` | 建立索引，推导有效状态 |
| `views.py` | 各视图的展开算法 |
| `search.py` | 关键词查找 |
| `render.py` | 渲染文本和 JSON |
| `cli.py` | 命令入口 |

抽取与检索都实现为纯函数。每次调用都从零重建图。

## 输出确定性

- JSON 输出带 `root` 字段，键按字母排序。节点按 `id` 排序，边按 `(src, rel, dst, origin)` 排序。
- 输出中的路径一律使用 `/`，换行一律使用 `\n`，编码为 UTF-8。

## 测试样例

`tests/fixtures/basic/` 是一个合成仓库，包含一份登记表和约 10 份文档：

- REQ；构成三代取代链的三份 ADR（中间一代已被取代）；被另一份 ADR amends 的 ADR；用 meta 声明关系的 SPEC；用正文标记声明关系的 DSN；带 verifies 的 VER；一份没有任何 satisfies 入边的 REQ。
- 登记为 superseded 但后继仍是 draft 的文档，验证有效状态推导。
- 一条指向未登记 ID 的关系、一个未知关系类型，验证 `show` 不中断。
- 一份 CRLF 换行的文件，验证解析结果与 LF 相同。
- **find**：同一个词分别出现在 ID、标题和正文中，验证分档和排序；中英文混合的词。
- **包含**：根 `cairn.yaml` 包含一份子登记表，验证路径相对子登记表解析、离根近者优先、同距冲突报错、include 成环报错、跨出根目录报错。
- **根查找**：在 fixture 的子目录中运行，找到上级的 `cairn.yaml`；设置 `CAIRN_CEILING_DIRS` 后，在上限以内没有 `cairn.yaml` 的目录中运行，报错退出。

`tests/fixtures/compat/` 用字段行和登记表 `supersedes` 声明关系，验证兼容层。

## 验收

1. 三个命令在 fixture 上的 `--json` 输出与预期输出文件逐字节一致，在 Linux 和 Windows 上都成立。
2. 在 cairn 仓库自身上运行 `cairn show CRN-DSN-001`，显示一条指向 CRN-REQ-001 的 satisfies 边，basis 为 declared。
3. 在一个真实的文控仓库上，借助兼容层，`show` 能还原一张事先手工整理的关系表，并正确标出每条边的 basis。
4. 在同一个仓库上回答"A2 OTA 前向兼容的决策依据是什么？现行方案、协议与验证记录分别在哪里？"：`find A2 OTA --expand` 加上若干次 `show`，能找到相关的需求、ADR 和协议，标出旧方案已被取代、现行方案是否已被接受，没有验证关系的地方显示"未建立追溯"，并附带原始文件位置。

## 后续阶段

以下内容只定义语义，不在本阶段实现。它们会影响数据格式（登记条目字段、受管块、校验值规则），所以现在一并定下。

### `cairn check [--all] [--json]`

只读。

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
| W_SUPERSEDES | 登记表状态与推导出的取代状态不一致：登记为 superseded 但没有被接受过的后继，或已有被接受过的后继但未登记为 superseded |
| W_STALE_BASIS | active 节点有非 mention 边指向有效状态为 superseded 或 archived 的节点 |
| W_CYCLE | depends_on 或 supersedes 出现环 |
| W_NO_APPROVAL | 登记为 active 但没有 `approval` |
| W_APPROVAL_STALE | 当前校验值与 `approval.sha256` 不同，文档在批准后被修改 |
| W_MANAGED_DRIFT | 文件受管块与登记表的项目属性不一致 |
| I_UNRESOLVED_LINK | 链接指向未登记的路径 |

以下两个是写命令，也是 cairn 仅有的写操作。

### `cairn init`

在当前目录创建一份空的 `cairn.yaml`，`types` 取自元模型。目标文件已存在时拒绝执行。发现当前目录下已有 `docs/engineering-artifacts.yaml` 时，自动把它写进 `include`。只新建文件，不修改已有的受控资产。

### `cairn sync [ID...]`

按登记表把项目属性写入各文件的受管块。只改受管块，不改其他内容；由于校验值不包含受管块，写入后 `sha256` 不变。冻结文件的受管块同样更新，例如旧文档被取代后写入 `superseded`。

## 待确认

- 兼容层是否需要读取 ADR 的 Nygard 行首声明（`Supersedes [..](..)`、`Amends [..](..)`），还是由目标仓库逐步改为 meta。
- 字段值中混有追溯性引用时（例如"与其他文档的关系"），由哪些字段进入 `compat.field_relations`。
- 关系携带的内容除 `note` 外是否需要结构化字段，例如所依据的章节。
- 是否允许 include 其他仓库的登记表。允许的话需要锁定对方版本，本阶段不支持。
- 受管块记录哪些项目属性：只记 `status`，还是同时记 `scope` 和 `approval`。
- 文档被多个登记表登记时，受管块记录哪个项目的属性。
- `registry_check.py` 迁入 cairn，还是保留在 embedded-skills 中并同步采用受管块去除规则。
