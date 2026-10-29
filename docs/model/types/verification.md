---
id: CRN-TYP-005
type: doctype
revision: 1
name: verification
meta: false
purpose: '验证计划、验收标准或验证记录，说明某项需求或规范是否被满足、怎么证明。'
---

# verification：验证

验证计划、验收标准或验证记录，说明某项需求或规范是否被满足、怎么证明。

## 写法

区分计划与记录；记录要写明被验证对象的版本、环境和结论。

## 关系

通过 `verifies` 指向被验证的文档，通过 `satisfies` 计入需求覆盖。被验证对象修订前进时，验证需要重做。

## 例子

- 属于：测试计划、实板验收记录、gate review 结论。
- 不属于：测试代码本身（不纳管）。
