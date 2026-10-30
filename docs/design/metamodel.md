---
id: CRN-DESIGN-METAMODEL
type: design
title: 'cairn 元模型设计：文档类型与关系'
revision: 2
relations:
  satisfies: [CRN-REQUIREMENTS]
---

# cairn 元模型设计：文档类型与关系

本文定义文档类型和关系类型怎样建模：分几层、每层由什么定义、定义里有哪些机器可读的字段、这些字段驱动哪些算法。它是 [`docs/model/`](../model/README.md) 的设计依据。本文被接受后，`docs/model/` 下的定义文件按本文改写。

## 现状的问题

逐份检查 `docs/model/` 中 11 种类型和 10 种关系后，发现以下问题：

1. **层次混在一起。** `registry`、`doctype`、`reltype` 和 `requirement` 等业务类型放在同一个列表里，只靠一个 `meta` 布尔值区分。但 `registry` 以路径为身份、不被登记，行为和其他类型完全不同。`doctype` 用 `doctype` 定义自己，没有说明这个自引用怎样起步。
2. **类型几乎没有机器可读的内容。** 类型定义只有 `name` 和 `purpose`。决策被接受后能不能再改、外部资料能不能声明关系，这些影响校验的性质都只写在正文里。
3. **关系的行为写在正文里。** 失效语义、取代对状态的影响、不能成环、两端同类型，都是正文说明。cairn 只能按关系名写死这些算法，新增一种关系时只写定义文件是不够的。
4. **端点写法是临时拼的。** `"*"`、`"!x"`、`same` 混在一起，有时是列表，有时是单值。
5. **同一关系出现两套失效语义。** `governed_by` 指向 decision 时"被取代才失效"，指向 standard 时"修订就要复核"。这说明失效规则不该按关系写死，而应由关系和终点类型共同决定。
6. **satisfies 和 verifies 重叠。** verification 既可以 `satisfies` 需求，也可以 `verifies` 需求，"需求被覆盖"因此分不清是有了方案还是有了证据。
7. **"修订后失效"无法判断。** 规则说"终点修订后源文档需要复审"，但关系没有记录声明时终点处于哪个修订，cairn 无从比较。
8. **方向规则说的是时间，不是职责。** "所有 file 关系都由较晚的文档指向较早的文档"是为了保护冻结文档，但它没有说清一条关系该由谁写。需求、设计这类持续修订的文档并不冻结，按时间先后定方向对它们没有意义。
9. **项目扩展没有落点。** "项目可以追加类型"只有一句话，没说定义放在哪里、怎样被找到、能不能细化已有类型。

## 分层

元模型分三层：

| 层 | 内容 | 由什么定义 | 能否修改 |
|---|---|---|---|
| 内核 | 类型 `registry`、`doctype`、`reltype`；关系 `registers`、`includes`、`mentions` | cairn 代码 | 不能。`docs/model/` 中对应的文件只是说明 |
| 标准模型 | `requirement`、`design` 等业务类型，`satisfies`、`supersedes` 等业务关系 | `docs/model/` 中的定义文件，随 cairn 发布 | 项目不能改写，只能细化 |
| 项目扩展 | 项目自定义的类型和关系 | 项目仓库中登记过的 doctype、reltype 文档 | 由项目按文控流程修改 |

内核解决自举问题：cairn 先用代码认出登记表、类型描述和关系描述，再读取它们去理解其他一切。内核关系的行为也写在代码里，`registers` 和 `includes` 的规则见检索设计的"项目根"一节，`mentions` 由引用抽取产生。

**类型本身是受控文档。** 一份已登记、未归档的 doctype 文档定义一种类型，一份 reltype 文档定义一种关系。一个项目可用的类型和关系，等于标准模型加上从根登记表推导出的全部 doctype、reltype 文档。类型的新增、修订和取代因此同样走文控。登记表中不再单独列出 `types`，只保留一个来源。

类型和关系按 `name` 引用，不按 ID。登记的 doctype、reltype 文档按 ID 分两种情况处理：

1. **与随 cairn 发布的定义同 ID**（例如 `CRN-TYPE-DESIGN`）：视为同一份定义，不算项目扩展，也不重复加载。这正是 cairn 仓库自身的情况，它登记的就是标准模型的原件。此时以仓库中的文件为准，便于在 cairn 仓库里修改元模型；内容与发布版不同时，后续 `check` 报 `W_MODEL_DRIFT`。
2. **其他 ID**：属于项目扩展。它的 `name` 与内核或标准模型重名时报错，不允许覆盖。

## 类型定义

doctype 文档的 frontmatter：

```yaml
id: XX-TYPE-TEST-RECORD
type: doctype
title: '测试记录'
revision: 1
name: test_record
extends: verification     # 可选，细化一个已有类型
lifecycle: frozen         # frozen / living / external
content: markdown         # markdown / binary
purpose: '一次测试的执行记录：对象版本、环境、结论。'
```

### lifecycle：修改方式

| 值 | 含义 | 被接受后内容变化 | 关系怎样声明 |
|---|---|---|---|
| `frozen` | 接受后冻结，要改就写新文档取代或修订它 | 错误 | 只在创建时声明，之后不再增加 |
| `living` | 原地修订，每次修订更新 `revision` 并重新批准 | 正常，需要重新批准 | 随时可以增删 |
| `external` | 外部资料，原样保存，只能整体替换 | 视为换了新版本，在登记表记录版本 | 不能作为关系起点 |

lifecycle 只决定一份文档自己能不能再增加关系，谁来声明关系由"声明规则"一节决定。

### content：内容形式

`markdown` 的文档有文件 meta，可以作为关系起点。`binary` 的文档（PDF、图片、资料包）没有 meta，只能作为关系终点。

### extends：细化

项目可以用 `extends` 从已有类型派生子类型，例如把 verification 细分为测试计划和测试记录。规则：

1. 只允许单继承，不能成环。
2. 关系端点写父类型时，子类型自动满足。`satisfies` 的起点写 `design`，`design` 的子类型也可以声明 `satisfies`。
3. 子类型只能收紧，不能放宽：`living` 可以收紧为 `frozen`，反之不行；`content` 必须与父类型相同。
4. 按类型检索时包含子类型：`ls --type verification` 会列出测试记录。

这样标准类型可以保持粗粒度，项目需要更细的区分时自己细化，不必为了一个项目修改标准模型。

## 关系定义

reltype 文档的 frontmatter：

```yaml
id: CRN-REL-SATISFIES
type: reltype
title: 'satisfies：满足'
revision: 2
name: satisfies
inverse: satisfied_by       # 反向显示名
shape: arrow                # arrow / line
from: [design, specification]
to: [requirement]
same_type: false            # 两端是否必须同类型
acyclic: false              # 是否禁止成环
retires_target: false       # 起点被接受后，终点是否退出现行
stale_on: [revised, retired]
attrs: [note, section, at]  # 关系可以携带的内容
view: coverage
```

### 字段

| 字段 | 取值 | 驱动的算法 |
|---|---|---|
| `inverse` | 名称 | `show` 中反向边的显示名 |
| `shape` | `arrow` / `line` | 边的存储与展开方向 |
| `from`、`to` | 类型名列表，或 `["*"]` | 端点检查。`*` 表示任意非内核类型；写父类型时子类型也符合 |
| `same_type` | 布尔 | 两端须为同一类型或同一父类型的子类型 |
| `acyclic` | 布尔 | 成环检查 |
| `retires_target` | 布尔 | 有效状态推导：起点被接受过时，终点为 superseded |
| `stale_on` | `revised`、`retired` 的子集 | 失效推导：终点发生这些事件时，起点需要复审 |
| `attrs` | `note`、`section`、`at` 的子集 | 关系可以携带的内容 |
| `view` | `chain`、`coverage`、`dag`、`neighbors`、`count`、`hidden` | `show` 的展开算法 |

字段是固定词汇，不是表达式。固定词汇在每个平台上的行为相同，能逐项校验，cairn 也能为每个字段写一套确定的算法。将来出现一种关系、固定字段描述不了时，再考虑扩展。

原来的 `declared_in` 去掉：内核关系由代码处理，其余关系一律在文件中声明。

### 声明规则

**关系由主动方声明。** arrow 关系的 `from` 端就是主动方，即动作的发出方：设计满足需求，验证记录验证规范，新决策取代旧决策，模块依赖规范。主动方在自己的文件 meta 或正文标记中声明这条关系。

1. 被动方不声明。`inverse` 只用于显示，不能写在文件中；在文件中写 `satisfied_by` 之类的反向名时，`check` 报 `E_REL_TYPE`。
2. line 关系没有主动方，由任一方声明一次即可。两边都声明时合并为一条边。
3. 内核关系同样适用：登记表是 `registers`、`includes` 的主动方，所以写在登记表里。
4. 定义关系类型时，`from` 必须选主动方。判断方法是问"这条关系变了，该改哪份文件"：答案就是 `from` 端。

这条规则和 lifecycle 配合，自然保护冻结文档：主动方通常是后写成的文档，冻结文档作为被动方永远不需要改动。主动方本身已冻结、又需要补一条关系时，不能改它，只能写一份新文档 `amends` 它。

### 两个通用推导

有了上面的字段，原来写死在 `supersedes` 上的规则变成通用规则：

1. **退出现行。** 任何 `retires_target: true` 的关系，只要起点被接受过，终点的有效状态就是 superseded。"被接受过"的定义不变，见检索设计的"有效状态"。
2. **失效。** 终点的事件：`retired` 指有效状态变为 superseded 或 archived；`revised` 指终点当前的 `revision` 与关系上记录的 `at` 不同。终点发生 `stale_on` 中的事件时，这条边标为失效，起点需要复审。

`governed_by` 的两套语义由此统一为一条 `stale_on: [revised, retired]`。决策是 frozen，不会出现 revised，所以只有被取代才失效；规则是 living，修订就会触发复核。区别来自终点的 lifecycle，不需要写在关系里。

### `at`：声明时的终点修订

要判断"终点修订后失效"，关系必须记录声明时终点的修订：

```yaml
relations:
  satisfies:
    - {to: CRN-REQUIREMENTS, at: 3, section: R-14}
```

`at` 可以不写。不写时，cairn 只能判断 `retired`，`revised` 显示为"未知"。手写 `at` 容易漏，所以由编辑插件在插入关系时自动填入终点当前的修订。

## 标准模型

### 类型

| name | lifecycle | content | 说明 |
|---|---|---|---|
| requirement | living | markdown | 需求，追溯起点 |
| decision | frozen | markdown | 决策记录（ADR） |
| design | living | markdown | 设计方案 |
| specification | living | markdown | 接口与协议规范。不兼容的变化写新规范取代旧规范 |
| verification | living | markdown | 验证计划与记录。项目需要冻结记录时，派生 frozen 子类型 |
| standard | living | markdown | 工程规则 |
| reference | external | binary | 外部资料 |
| glossary | living | markdown | 术语表 |

### 关系

| name | inverse | from → to | same_type | acyclic | retires_target | stale_on | view |
|---|---|---|---|---|---|---|---|
| supersedes | superseded_by | * → * | 是 | 是 | 是 | — | chain |
| amends | amended_by | * → * | 是 | 是 | 否 | retired | neighbors |
| satisfies | satisfied_by | design, specification → requirement | 否 | 否 | 否 | revised, retired | coverage |
| verifies | verified_by | verification → * | 否 | 否 | 否 | revised, retired | coverage |
| governed_by | governs | * → decision, standard | 否 | 否 | 否 | revised, retired | neighbors |
| depends_on | needed_by | * → specification, design, reference | 否 | 是 | 否 | revised, retired | dag |
| related | related | * — * | 否 | 否 | 否 | — | neighbors |

相对现状的改动：

- `satisfies` 的起点去掉 verification。需求的覆盖视图分两栏：`satisfies` 表示有了方案，`verifies` 表示有了证据。
- `amends` 的视图从 chain 改为 neighbors：修订补充不构成链，显示在被修订文档的旁边。
- 每种关系都写明 `inverse` 和 `stale_on`。

### 内核

| name | 说明 |
|---|---|
| registry | 登记表，以文件路径为身份，没有 ID，不被登记 |
| doctype | 类型描述 |
| reltype | 关系描述 |
| registers | 登记表 → 非登记表文档，携带项目属性 |
| includes | 登记表 → 登记表文件 |
| mentions | 由链接和正文中的 ID 自动抽取，只计数，不参与检查 |

## 对第一阶段的影响

第一阶段只做检索，用到的字段是 `name`、`extends`、`content`、`inverse`、`shape`、`view`，以及有效状态推导用到的 `retires_target`。`lifecycle`、`same_type`、`acyclic`、`stale_on`、`at` 在第一阶段只读入、不检查，由后续的 `check` 使用；但它们决定数据格式，所以现在一并定下。

## 待确认

- 是否引入 `extends`。不引入的话，项目只能新增平级类型，关系端点要逐个列出。
- `lifecycle` 是否就这三种，verification 默认 living 还是 frozen。
- `at` 记录终点的 `revision` 还是 `sha256`。revision 便于阅读，sha256 不依赖作者是否记得更新修订号。
- `satisfies` 去掉 verification 后，现有仓库中验证记录的 `satisfies` 由兼容层改读为 `verifies`，还是报告给作者修改。
- 标准模型总是加载，还是由根登记表显式声明使用哪个版本。
