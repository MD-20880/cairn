# cairn 第一阶段设计：ls / show / check

**需求依据**：[CRN-REQ-001](../requirements.md)

## 范围

三个命令 `ls`、`show`、`check`，共用一次构建的内存图：

- `ls` 选择节点，不涉及边；
- `show` 展示单个节点的邻域；
- `check` 报告不一致之处。

本阶段不包含：全文检索、查询语言、`@rel()` 显式声明语法、推断边、缓存，以及对目标仓库的任何写入。

## 输入

- `--root`：目标仓库根目录，默认为当前目录。
- `--config`：项目映射配置文件。

映射配置示例：

```yaml
registry: docs/engineering-artifacts.yaml
id_pattern: '\b(?:XX-[A-Z]+-\d{3}|ADR-\d{4})\b'
field_relations:          # 字段名 → 关系类型
  需求依据: satisfies
  决策依据: governed_by
  依赖: depends_on
body_status:              # 状态以正文为准的文档类型，及提取规则
  Decision: '^\s*[-*]?\s*\**状态\**\s*[:：]\s*(\w+)'
```

## 数据模型

### Node

字段：`id`、`type`、`path`、`status_registry`、`status_body`（可选）、`revision`、`scope`、`sha256`、`title`。

- `title` 取文件中的第一个 H1。
- 有效状态：文档类型列在 `body_status` 中时取 `status_body`，否则取 `status_registry`。

### Edge

字段：`src`、`rel`、`dst`、`basis`、`origin`（`文件:行号` 列表）、`raw`（原文片段）。

`basis` 的取值：

| basis | 来源 |
|---|---|
| declared | 登记表中的 `supersedes` |
| field | 按 `field_relations` 从字段行抽取 |
| inferred | 预留，本阶段不产生 |
| mention | Markdown 链接，或正文中出现的 ID |

### 关系类型

方向一律从较晚写成的文档指向较早的文档，关系由前者声明。

| rel | 终点 | 终点变化时，源文档应当 |
|---|---|---|
| satisfies | Requirement | 复审，并计入需求覆盖 |
| governed_by | Decision / Standard | 指向 Decision 时，决策被取代则源文档过时；指向 Standard 时，按规则复核合规 |
| depends_on | Specification / Design | 终点修订前进时，确认兼容 |
| verifies | 任意受控文档 | 终点修订前进时，验证需要重做 |
| supersedes | 同类型文档 | 不适用（终点已退出现行） |
| related | 任意 | 不传播；默认不参与展开 |
| mentions | 任意 | 不传播 |

## 抽取规则

1. **字段行**支持两种写法：`字段名[:：] 值`（字段名可以加粗），以及两列表格行 `| 字段名 | 值 |`。值中每个匹配 `id_pattern` 的 ID、每个能解析到登记路径的链接，各生成一条边。
2. **Markdown 链接**相对所在文件解析：去掉锚点，统一使用 `/` 分隔，匹配登记表中的路径后得到 ID，生成 mention 边。无法匹配的链接记为未解析链接。
3. 同一 `(src, rel, dst)` 有多个来源时，保留 basis 最强的一条，其余来源并入 `origin` 列表。
4. sha256 按文件的原始字节计算。解析时按 UTF-8 解码并统一换行符。

## 命令语义

### `cairn ls [ID-glob] [--type T] [--status S] [--scope X] [-a] [-l] [--json]`

- 只列出登记表中的节点，按 ID 排序；多个过滤条件之间是"且"的关系。
- 默认显示 draft 和 active；加 `-a` 时同时显示 superseded 和 archived。
- 输出列：ID、类型、有效状态、修订、标题。登记表状态与正文状态不一致时，在状态后加 `!`。
- 加 `-l` 时增加路径和出入度两列（不计 mention）。
- 结果为空时，退出码仍为 0。

### `cairn show <ID> [--all] [--json]`

- 头部：节点元数据。
- 上游：出边，按关系类型分组。
- 下游：入边，由出边反向计算得到。
- 取代链：沿 `supersedes` 双向走到底。
- 每行显示：关系、对端 ID、对端有效状态、basis、origin。
- mention 默认只显示条数，加 `--all` 时逐条展开。

### `cairn check [--json]`

每条发现包含 code、级别、节点、origin 和说明。存在 E 级发现时，退出码非 0。

| code | 含义 |
|---|---|
| E_DUP_ID | 登记表中 ID 重复 |
| E_MISSING_FILE | 登记路径下没有文件 |
| E_SHA | 文件的 sha256 与登记值不符 |
| E_DANGLING | 非 mention 边指向不存在的 ID |
| W_STATUS_MISMATCH | 正文状态与登记表状态不一致 |
| W_SUPERSEDES | 被取代文档的有效状态不是 superseded |
| W_STALE_BASIS | active 节点有非 mention 边指向 superseded 或 archived 的节点 |
| I_UNRESOLVED_LINK | 链接指向未登记的路径 |

## 模块

使用 Python 3.10 及以上版本，外部依赖只有 PyYAML，命令行解析用 argparse。

| 模块 | 职责 |
|---|---|
| `config.py` | 读取并校验映射配置 |
| `registry.py` | 读取登记表，生成 Node |
| `extract.py` | 逐个文件抽取 Edge |
| `graph.py` | 建立出边/入边索引，计算取代链 |
| `checks.py` | 运行各项检查，输出发现 |
| `render.py` | 渲染文本和 JSON |
| `cli.py` | 命令入口 |

抽取与检查都实现为纯函数。每次调用都从零重建图。

## 输出确定性

- JSON 的键按字母排序。节点按 `id` 排序，边按 `(src, rel, dst, origin)` 排序。
- 输出中的路径一律使用 `/`，换行一律使用 `\n`。

## 测试样例

`tests/fixtures/basic/` 是一个合成仓库，包含一份登记表和约 8 份文档：

- **正常部分**：REQ；构成取代关系的新旧两份 ADR；带字段的 SPEC；IF；带 verifies 的 VER。
- **反例**，每个只触发一个 check code：悬空 ID；sha256 不符；ADR 正文状态与登记表不一致；active 规范引用已取代的 ADR；指向未登记文件的链接。

`E_DUP_ID` 放在单独的 fixture 中测试。

## 验收

1. 三个命令在 fixture 上的 `--json` 输出与预期输出文件逐字节一致，在 Linux 和 Windows 上都成立。
2. 在 cairn 仓库自身上运行 `cairn show CRN-DSN-001`，显示一条指向 CRN-REQ-001 的 satisfies 边，basis 为 field；运行 `cairn check` 没有发现。
3. 在一个真实的文控仓库上，`show` 能还原一张事先手工整理的关系表，并正确标出每条边的 basis；`check` 报告的每条发现，要么能解释原因，要么据此修正映射配置。

## 待确认

以下问题需要在实现前读取真实文件确认：

- 字段行的实际写法。抽取 3 到 5 份文档确认后，再确定正则。
- 目标仓库文控规则中 sha256 的计算对象：原始字节，还是规范化后的内容。
- ADR 正文中状态行的写法。
- 字段值中带有说明文字或多个 ID 时，是否每个 ID 都生成一条边。
