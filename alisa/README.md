# ALISA 子系统

ALISA（Architecture-Led Incremental System Assurance，架构主导的增量式系统保证）是 OSATE 中的顶层验证与确认框架。它将系统需求、验证计划和保证案例统一构建在 AADL 架构模型之上，通过一组相互依赖的领域特定语言（DSL）形成完整的闭环证据链。

源码位置：`osate2/alisa/`，版本 2.11.0（参见 `alisa.releng` 中的父 POM）。

---

## 1. 模块概述

ALISA 子系统包含 27 个 Eclipse 插件，按功能分为以下六组：

### 公共基础
| 插件 | 职责 |
|---|---|
| `org.osate.alisa.common` | 公共表达式语言（Common DSL）的 Xtext 语法与 EMF 模型 |
| `org.osate.alisa.common.ui` | Common DSL 的 Eclipse 编辑器支持 |

### 分类与组织
| 插件 | 职责 |
|---|---|
| `org.osate.categories` | 分类定义语言（Categories DSL）的语法与模型 |
| `org.osate.categories.ui` | Categories 编辑器 |
| `org.osate.categories.tests` | Categories 单元测试 |
| `org.osate.organization` | 组织与利益相关者角色 DSL |
| `org.osate.organization.ui` | Organization 编辑器 |
| `org.osate.organization.tests` | Organization 单元测试 |

### 需求规范
| 插件 | 职责 |
|---|---|
| `org.osate.reqspec` | ReqSpec DSL 核心：语法、EMF 模型、验证器、作用域 |
| `org.osate.reqspec.ui` | ReqSpec 编辑器与快速修复 |
| `org.osate.reqspec.tests` | ReqSpec 单元测试 |
| `org.osate.reqtrace` | 需求追踪矩阵插件（占位声明，见第 8 节） |

### 验证计划
| 插件 | 职责 |
|---|---|
| `org.osate.verify` | Verify DSL：验证计划、验证方法注册表 |
| `org.osate.verify.ui` | Verify 编辑器 |
| `org.osate.verify.tests` | Verify 单元测试 |

### 保证执行
| 插件 | 职责 |
|---|---|
| `org.osate.assure` | 保证案例构造器、执行引擎、EMF 结果模型 |
| `org.osate.assure.tests` | Assure 单元测试 |
| `org.osate.assure.resolute.tests` | Assure 与 Resolute 集成测试 |

### 工作台与集成
| 插件 | 职责 |
|---|---|
| `org.osate.alisa.workbench` | ALISA 顶层 DSL（`.alisa` 文件）及工作台集成 |
| `org.osate.alisa.workbench.ui` | 工作台 Eclipse UI |
| `org.osate.alisa.workbench.sdk` | SDK 导出包 |
| `org.osate.alisa.workbench.tests` | 工作台集成测试 |
| `org.osate.alisa.contribution` | OSATE 贡献点注册 |
| `org.osate.alisa.help` | Eclipse 帮助文档内容 |

---

## 2. 六层语言架构

ALISA 的 DSL 形成严格的依赖层次。下游语言通过 Xtext 的 `with` 和 `import` 机制重用上游语言的产生式和 EMF 类型：

```
┌─────────────────────────────────────────────────────────┐
│   org.osate.alisa.workbench / Alisa.xtext               │  顶层工作台
│   (AssuranceCase, AssurancePlan, AssuranceTask)          │
└─────────────────────┬───────────────────────────────────┘
                      │ with Common; import Verify, Categories
┌─────────────────────▼───────────────────────────────────┐
│   org.osate.verify / Verify.xtext                        │  验证计划
│   (VerificationPlan, VerificationMethod, MethodKind...)  │
└─────────────────────┬───────────────────────────────────┘
                      │ with Common; import ReqSpec, Categories
┌─────────────────────▼───────────────────────────────────┐
│   org.osate.reqspec / ReqSpec.xtext                      │  需求规范
│   (SystemRequirementSet, Goal, Requirement...)           │
└─────────────────────┬───────────────────────────────────┘
                      │ import Categories, Organization, Common
┌──────────┬──────────▼──────────────────────────────────┐
│Categories│  Organization   │   org.osate.alisa.common   │  基础层
│ DSL      │  DSL            │   Common.xtext             │
└──────────┴─────────────────┴────────────────────────────┘
```

每一层的语法文件如下：
- `Common.xtext` — `grammar org.osate.alisa.common.Common with org.eclipse.xtext.common.Terminals`
- `Categories.xtext` — `grammar org.osate.categories.Categories with org.eclipse.xtext.common.Terminals`
- `Organization.xtext` — `grammar org.osate.organization.Organization with org.eclipse.xtext.common.Terminals`
- `ReqSpec.xtext` — `grammar org.osate.reqspec.ReqSpec with org.osate.alisa.common.Common`
- `Verify.xtext` — `grammar org.osate.verify.Verify with org.osate.alisa.common.Common`
- `Alisa.xtext` — `grammar org.osate.alisa.workbench.Alisa with org.osate.alisa.common.Common`

---

## 3. Common 公共表达式语言（org.osate.alisa.common）

**源码**：`org.osate.alisa.common/src/org/osate/alisa/common/Common.xtext`

Common 语言为所有上层 DSL 提供共享的表达式基础设施，其 EMF 命名空间为 `http://www.osate.org/alisa/common/Common`。

### 3.1 变量声明

Common 定义了两种变量：

```
// 常量声明：编译期已知值
val MaxLatency = 100 ms
val Threshold : real units Time_Units = 50.0 ms

// 计算变量：运行期由验证方法写入
compute ActualLatency : real units Time_Units
```

语法产生式：

```xtext
ValDeclaration returns AVariableDeclaration:
  {ValDeclaration} 'val' name=ID
    (':' (type=TypeRef | 'typeof' type=PropertyRef
          | range?='[' (type=TypeRef | 'typeof' type=PropertyRef) ']') )?
    '=' value=AExpression
;

ComputeDeclaration returns AVariableDeclaration:
  {ComputeDeclaration}
  'compute' name=ID ':' (type=TypeRef | 'typeof' type=PropertyRef
                          | range?='[' (type=TypeRef | 'typeof' type=PropertyRef) ']')
;
```

### 3.2 类型系统

Common 通过 `TypeRef` 产生式将 AADL2 属性类型直接嵌入：

```xtext
TypeRef returns aadl2::PropertyType:
    {aadl2::AadlBoolean} 'boolean'
  | {aadl2::AadlInteger} 'integer' ('units' referencedUnitsType=[aadl2::UnitsType|AADLPROPERTYREFERENCE])?
  | {aadl2::AadlReal}    'real'    ('units' referencedUnitsType=[aadl2::UnitsType|AADLPROPERTYREFERENCE])?
  | {aadl2::AadlString}  'string'
  | {ModelRef}           'model' 'element'
  | {TypeRef}            ref=[aadl2::PropertyType|AADLPROPERTYREFERENCE]
;
```

`PropertyRef` 额外支持用 `typeof` 关键字从已有 AADL 属性定义推导类型。

### 3.3 AExpression 表达式层次

表达式遵循标准的算符优先级层次（从低到高）：

```
AExpression
  └─ AOrExpression        (or / ||)
       └─ AAndExpression  (and / &&)
            └─ AEqualityExpression  (== / !=)
                 └─ ARelationalExpression  (>= / <= / > / < / ><)
                      └─ AAdditiveExpression       (+ / -)
                           └─ AMultiplicativeExpression  (* / / / div / mod)
                                └─ AUnaryOperation   (not / - / +)
                                     └─ AUnitExpression
                                          └─ APrimaryExpression
```

`APrimaryExpression` 可以是以下节点之一：
- `ALiteral` — 布尔、整数、实数、字符串字面量
- `AVariableReference` — 对 `AVariableDeclaration` 的引用
- `AModelOrPropertyReference` — 模型或属性引用
- `AFunctionCall` — 限定名函数调用，如 `mylib.computeLoad(x, y)`
- `ARangeExpression` — `[min .. max delta d]`
- `AIfExpression` — `if cond then e1 else e2 endif`
- `AParenthesizedExpression` — 括号分组

### 3.4 模型引用与属性访问

这是 Common 语言最具特色的部分，允许在表达式中直接引用 AADL 模型元素：

```xtext
// 模型路径遍历：用 '.' 链式访问
AModelReference:
  modelElement=[aadl2::NamedElement|ThisKeyword]
  ({AModelReference.prev=current} '.' modelElement=[aadl2::NamedElement|ID])*
;

// AADL 属性访问：用 '#' 操作符
AModelOrPropertyReference returns AExpression:
  AModelReference
    (=>({APropertyReference.modelElementReference=current} '#')
       property=[aadl2::AbstractNamedValue|AADLPROPERTYREFERENCE])?
| APropertyReference
;

// 全局属性引用（不绑定模型元素）
APropertyReference returns APropertyReference:
  {APropertyReference} '#' property=[aadl2::AbstractNamedValue|AADLPROPERTYREFERENCE]
;
```

示例：
```
// 访问 this 的 AADL 属性
#Timing_Properties::Period

// 遍历模型后访问属性
this.sensor.dataPort#Data_Model::Data_Representation

// 单位转换：% 表示 convert（转为目标单位数值），in 表示 drop（丢弃当前单位取数值）
val latencyMs = someLatency % ms
```

### 3.5 Operation 枚举

`enum Operation` 定义所有二元与一元算符：

```xtext
enum Operation:
  OR='or' | ALT_OR='||'
  | AND='and' | ALT_AND='&&'
  | EQ='==' | NEQ='!='
  | GEQ='>=' | LEQ='<=' | GT='>' | LT='<' | IN='><'
  | PLUS='+' | MINUS='-'
  | MULT='*' | DIV='/' | INTDIV='div' | MOD='mod'
  | NOT='not'
;
```

其中 `IN='><'` 是区间包含操作符，用于判断某个值是否在某个范围内。

### 3.6 Uncertainty（不确定性描述）

Common 提供了 `Uncertainty` 块，允许在需求上标注变更的不确定性量化指标：

```xtext
Uncertainty:
  {Uncertainty} 'uncertainty' '['
    ('volatility'  volatility=INT)?
  & ('precedence'  precedence=INT)?
  & ('impact'      impact=INT)?
  ']'
;
```

三个整数字段分别表示变更频率、决策优先级和影响范围。

---

## 4. Categories 分类语言（org.osate.categories）

**源码**：`org.osate.categories/src/org/osate/categories/Categories.xtext`
**EMF 命名空间**：`http://www.osate.org/categories/Categories`

Categories 是一个极简的分类定义语言，其主要作用是为需求和验证活动提供正交的分类维度，以支持过滤和统计。

### 4.1 语法结构

```xtext
CategoriesDefinitions :
  categories+=Categories* categoryFilters+=CategoryFilter*
;

Categories returns Categories:
  name=ID '[' category += Category+ ']'
;

Category returns Category:
  name = ID
;

CategoryFilter returns CategoryFilter:
  'filter' name=ID '['
    category+=[Category|CatRef]* anyCategory?='any'?
  ']'
;

// 分类引用格式：分类组名.分类名
CatRef: ID '.' ID ;
```

### 4.2 典型 .cat 文件示例

```
// 需求质量属性分类
quality [
  Safety
  Security
  Performance
  Reliability
]

// 例外处理分类
exception [
  OutOfScope
  Waived
  Deferred
]

// 过滤器：选取安全和可靠性相关需求
filter SafetyFilter [
  quality.Safety
  quality.Reliability
]

// 过滤器：接受任何分类
filter AllFilter [ any ]
```

### 4.3 在上层 DSL 中的用法

Categories 通过 `import` 被 ReqSpec 和 Verify 引用。在 ReqSpec 的 `Requirement` 中：

```xtext
// 来自 ReqSpec.xtext
'category' category+=[categories::Category|QualifiedName]+
```

在 Verify 的 `VerificationActivity` 中同样存在 `category` 字段，这使得可以按分类过滤需求和验证活动。

`AssuranceTask` 在 Alisa.xtext 中直接继承 `CategoryFilter`：

```xtext
AssuranceTask returns categories::CategoryFilter :
  {AssuranceTask} 'assurance' 'task' name=ID (':' title=STRING)?
  '[' (
    (description=Description)?
    & ('category' category+=[categories::Category|QualifiedName]+ anyCategory?='any'? )?
    & ('issues' issues+=STRING+ )?
  ) ']'
;
```

这意味着一个 `assurance task` 就是一个带名字的分类过滤器，执行引擎（`AssureProcessor`）在遍历保证案例时将其作为 `CategoryFilter` 参数传递，从而选择性地只执行特定分类的验证活动。

---

## 5. Organization 组织角色语言（org.osate.organization）

**源码**：`org.osate.organization/src/org/osate/organization/Organization.xtext`
**EMF 命名空间**：`http://www.osate.org/organization/Organization`

Organization DSL 用于定义项目的组织结构和利益相关者，为需求的归属和责任链提供形式化描述。

### 5.1 完整语法

```xtext
Organization:
  'organization' name = ID
  stakeholder += Stakeholder+
;

Stakeholder:
  'stakeholder' name=ID
  '[' (
      ('full' 'name' fullname=STRING)?
    & ('title'       title=STRING)?
    & ('description' description=STRING)?
    & ('role'        role=STRING)?
    & ('email'       email=STRING)?
    & ('phone'       phone=STRING)?
  )
    & ('supervisor' supervisor=[Stakeholder|QID])?
  ']'
;

QID : ID ('.' ID)?;
```

所有属性使用无序合并（`&` 算符），因此在块内出现顺序任意。`supervisor` 字段支持跨文件引用（通过限定名 `QID`）来表达层级汇报关系。

### 5.2 示例

```
organization MyProject

  stakeholder sys_engineer [
    full name "Alice Chen"
    title "Lead Systems Engineer"
    role "System Requirements Author"
    email "alice@example.com"
  ]

  stakeholder safety_lead [
    full name "Bob Liu"
    title "Safety Lead"
    role "Safety Requirements Review"
    supervisor sys_engineer
  ]
```

### 5.3 与 ReqSpec 的集成

在 ReqSpec 的 `Goal` 和 `Requirement` 中，利益相关者通过以下字段关联：

```xtext
// Goal 中
'stakeholder' stakeholderReference+=[org::Stakeholder|QualifiedName]+

// SystemRequirement 中
'development' 'stakeholder' developmentStakeholder+=[org::Stakeholder|QualifiedName]+
```

注意两者语义不同：`stakeholderReference` 在 Goal 中表示"谁提出了这个目标"，而 `developmentStakeholder` 在 SystemRequirement 中表示"谁负责开发/实现这个需求"。

---

## 6. ReqSpec 需求规范语言（org.osate.reqspec）

**源码**：`org.osate.reqspec/src/org/osate/reqspec/ReqSpec.xtext`
**EMF 命名空间**：`http://www.osate.org/reqspec/ReqSpec`

ReqSpec 是 ALISA 中最核心的 DSL 之一，用于将系统需求与 AADL 架构模型精确绑定。

### 6.1 顶层结构

一个 `.reqspec` 文件的顶层（`ReqSpec`）可以包含以下四种文档类型：

```xtext
ReqSpec: parts+=(SystemRequirementSet | GlobalRequirementSet
                | StakeholderGoals | ReqDocument | GlobalConstants)+;
```

| 类型 | 关键字 | 说明 |
|---|---|---|
| `SystemRequirementSet` | `system requirements` | 绑定到特定 AADL 分类器的需求集 |
| `GlobalRequirementSet` | `global requirements` | 可复用的通用需求集（不绑定具体组件） |
| `StakeholderGoals` | `stakeholder goals` | 利益相关者目标（高层次意图陈述） |
| `ReqDocument` | `document` | 嵌套文档结构（含 `section` 和层级需求） |
| `GlobalConstants` | `constants` | 跨文件共享的常量定义 |

### 6.2 StakeholderGoals 与 SystemRequirementSet 的区别

**StakeholderGoals** 表达"系统应该做什么"的意图，绑定到组件分类器（可以是抽象的）：

```xtext
StakeholderGoals:
  'stakeholder' 'goals' name=QualifiedName (':' title=STRING)?
  ('for' (target=[aadl2::ComponentClassifier|AadlClassifierReference]
          | componentCategory+=ComponentCategory+))
  ('use' 'constants' importConstants+=[GlobalConstants|QualifiedName]+)?
  '[' (
    description=Description?
    & constants+=ValDeclaration*
    & goals+=Goal*
    & ('see' 'document' docReference+=ExternalDocument+)?
    & ('issues' issues+=STRING+)?
  ) ']'
;
```

**SystemRequirementSet** 表达"系统必须满足的约束"，必须精确绑定到 `ComponentClassifier`（`for` 字段为必填）：

```xtext
SystemRequirementSet returns RequirementSet:
  {SystemRequirementSet} 'system' 'requirements' name=QualifiedName (':' title=STRING)?
  'for' target=[aadl2::ComponentClassifier|AadlClassifierReference]
  ...
```

### 6.3 Goal 的字段结构

Goal 代表一条利益相关者目标。关键字段（来自 `Goal` 产生式）：

| 字段 | 语法 | 语义 |
|---|---|---|
| `category` | `category quality.Safety` | 分类标签（引用 Categories） |
| `description` | `description "..."` | 描述文本（`DescriptionElement+`） |
| `whencondition` | `when in mode Degraded` | 适用条件 |
| `rationale` | `rationale "..."` | 原因说明 |
| `changeUncertainty` | `uncertainty [volatility 2 ...]` | 变更不确定性量化 |
| `refinesReference` | `refines parent.g1` | 精化（继承并细化）父目标 |
| `conflictsReference` | `conflicts with g2` | 与其他目标冲突 |
| `evolvesReference` | `evolves req.r1` | 从某需求演化而来 |
| `dropped` | `dropped "reason"` | 标记为已废弃 |
| `stakeholderReference` | `stakeholder org.alice` | 目标提出者 |
| `goalReference` | `see goal parent.g0` | 相关目标的追溯链接 |
| `docReference` | `see document spec/req.pdf#section2` | 外部文档引用 |

### 6.4 SystemRequirement 的字段结构

`SystemRequirement` 在 Goal 的基础上增加了更强的绑定性和可量化谓词：

| 字段 | 语法 | 语义 |
|---|---|---|
| `for` | `for dataPort` | 需求针对的目标模型元素（可选） |
| `predicate` | `value predicate x <= MaxLatency` | 形式化可验证谓词 |
| `mitigates` | `mitigates SafetyException` | 需求用于缓解的危害/例外 |
| `inherits` | `inherits global.r_base` | 继承另一个需求的所有字段 |
| `decomposes` | `decomposes parent.r1` | 分解（细化自父需求） |
| `computes` | `compute ActualValue : real` | 由验证结果填充的计算变量 |
| `developmentStakeholder` | `development stakeholder org.bob` | 负责实现的干系人 |
| `requirementReference` | `see requirement global.r2` | 相关需求追踪 |

### 6.5 ReqPredicate：可量化谓词

需求可以携带形式化谓词，这是与传统文本需求工具最大的不同点：

```xtext
ReqPredicate: InformalPredicate | ValuePredicate;

InformalPredicate: 'informal' 'predicate' description=STRING;

ValuePredicate:
  'value' 'predicate' xpression=AAndExpression
  ('with' desiredValue+=DesiredValue+)?
;

DesiredValue:
  desired=AVariableReference (upto?='upto' | 'downto') value=AExpression
;
```

完整示例：

```
system requirements latency_reqs for FlightControl.impl [
  val MaxLatency = 100 ms

  requirement r_latency : "End-to-end latency shall not exceed MaxLatency" [
    category quality.Performance
    description "The system shall respond to all sensor inputs within MaxLatency."
    value predicate ActualLatency <= MaxLatency
      with ActualLatency upto MaxLatency
    rationale "FAA DO-178C requires deterministic response time."
    development stakeholder org.sys_engineer
    see goal flightGoals.g_performance
  ]

  requirement r_safety : "No single point of failure" [
    category quality.Safety
    mitigates SinglePointFailure
    see document hazard/fha.pdf#2.3
    issues "Pending EMV2 fault model completion"
  ]
]
```

### 6.6 GlobalRequirement：通用需求

`GlobalRequirement` 不绑定具体组件，可通过 `for componentCategory` 或 `for TargetType` 限定适用范围：

```xtext
GlobalRequirement returns Requirement:
  'requirement' name=ID (':' title=STRING)?
  ('for' (componentCategory+=ComponentCategory+ | targetType=TargetType))?
  ...
```

`TargetType` 枚举定义于 Common：
```xtext
enum TargetType:
  COMPONENT='component' | FEATURE='feature' | CONNECTION='connection'
  | FLOW='flow' | MODE='mode' | ELEMENT='element' | ROOT='root';
```

通过 `include` 机制可将全局需求引入系统需求集：

```xtext
IncludeGlobalRequirement:
  'include' include=[ecore::EObject|QualifiedName]
  ('for' (local?='self' | targetElement=[aadl2::NamedElement|ID]))?
;
```

---

## 7. Verify 验证计划语言（org.osate.verify）

**源码**：`org.osate.verify/src/org/osate/verify/Verify.xtext`
**EMF 命名空间**：`http://www.osate.org/verify/Verify`

Verify DSL 定义"如何验证需求"，将抽象的验证方法与具体活动绑定到需求。

### 7.1 顶层结构

```xtext
Verification:
  contents+=(VerificationPlan | VerificationMethodRegistry)+;
```

一个 `.verify` 文件可以包含：
- `VerificationPlan`：对某一个 `RequirementSet` 中的所有需求制定验证计划
- `VerificationMethodRegistry`：定义可复用的验证方法库

### 7.2 VerificationPlan

```xtext
VerificationPlan returns VerificationPlan:
  'verification' 'plan' name=QualifiedName (':' title=STRING)?
  'for' requirementSet=[ReqSpec::RequirementSet|QualifiedName]
  '[' (
    (description=Description)?
    & claim+=Claim*
    & rationale=Rationale?
    & ('issues' issues+=STRING+)?
  ) ']';
```

### 7.3 Claim：声明与证据链

`Claim` 是 VerificationPlan 的核心元素，将一个需求与其验证活动及论证表达式关联：

```xtext
Claim:
  {Claim} 'claim' requirement=[ReqSpec::Requirement|QualifiedName]? (':' title=STRING)?
  '[' (
      ('activities' activities+=VerificationActivity*)?
    & ('assert'  assert=ArgumentExpr)?
    & rationale=Rationale?
    & ('weight'  weight=INT)?
    & subclaim+=Claim*
    & ('issues'  issues+=STRING+)?
  ) ']'
;
```

`assert` 字段携带一个 `ArgumentExpr`，这是一个结构化的证据组合表达式：

```xtext
// then：先做第一批，成功后才做第二批
ThenEvidenceExpr: ElseEvidenceExpr (=> ({ThenExpr.left=current} 'then') successor=ThenEvidenceExpr)*;

// else：第一批失败后执行备选方案
SingleElseEvidenceExpr: VAReference (=> ({ElseExpr.left=current} 'else')
    (error=ElseEvidenceExpr |
     '[' ('fail': fail=...) ('timeout': timeout=...) ('error': error=...) ']'
    ))*;

// all：要求所有子活动都通过
QuantifiedEvidenceExpr: 'all' {AllExpr} '[' elements+=ThenEvidenceExpr+ ']';
```

### 7.4 VerificationActivity：具体验证活动

```xtext
VerificationActivity:
  name=ID (':' title=STRING)?
  ':'
  (computes+=ComputeRef (',' computes+=ComputeRef)* '=')?
  method=[VerificationMethod|QualifiedName]
  '(' (actuals+=AExpression (',' actuals+=AExpression)*)? ')'
  ('property' 'values' '(' (propertyValues+=AExpression (',' propertyValues+=AExpression)*)? ')')?
  ('[' (
    ('category' category+=[categories::Category|QualifiedName]+)?
    & ('timeout' timeout=AIntegerTerm)?
    & ('weight'  weight=INT)?
  ) ']')?
;
```

其中：
- `computes` 列表中的变量在活动执行后会被写入（对应 ReqSpec 中的 `compute` 声明）
- `actuals` 是传递给验证方法的实参（`AExpression` 类型，支持所有 Common 表达式）
- `property values` 用于传递 AADL 属性值
- `timeout` 以毫秒为单位，执行引擎在超时后将结果置为 ERROR

### 7.5 VerificationMethod：方法注册

`VerificationMethod` 是验证方法的规范声明，包含形参列表、属性列表、返回值描述和方法类型：

```xtext
VerificationMethod:
  'method' name=ID
  (
    '(' ((targetType=TargetType)? | (formals+=FormalParameter+) | ...)? ')'
    ('properties' '(' properties+=[aadl2::Property|AADLPROPERTYREFERENCE]* ')')?
    ('returns'    '(' results+=FormalParameter* ')')?
    (isPredicate?='boolean' | isResultReport?='report')?
  )?
  (':' title=STRING)?
  ('for' (target=[aadl2::ComponentClassifier|AadlClassifierReference]
          | componentCategory+=ComponentCategory+))?
  '[' (
    methodKind=MethodKind
    & description=Description?
    & precondition=VerificationPrecondition?
    & validation=VerificationValidation?
    & ('category' category+=[categories::Category|QualifiedName]+)?
  ) ']'
;
```

### 7.6 MethodKind 枚举：七种验证方式

`MethodKind` 是 Verify 语言中最关键的设计点，定义了七种可用于验证的技术手段：

```xtext
MethodKind:
  ResoluteMethod | JavaMethod | ManualMethod | PluginMethod
  | AgreeMethod | JUnit4Method | PythonMethod
;
```

| 类型 | 语法关键字 | 实现说明 |
|---|---|---|
| `ResoluteMethod` | `resolute MyLib.myAssertion` | 通过 Resolute 断言工具验证（引用 EObject） |
| `JavaMethod` | `java my.pkg.MyAnalysis.check(String name)` | 反射调用 Java 静态/实例方法 |
| `ManualMethod` | `manual dialogs.ReviewDialog` | 弹出对话框要求人工确认 |
| `PluginMethod` | `plugin MassAnalysis` | 调用已注册的 OSATE 分析插件 |
| `AgreeMethod` | `agree single` 或 `agree all` | 调用 AGREE 形式化验证（单层/全部子系统） |
| `JUnit4Method` | `junit my.pkg.MyTests` | 通过 JUnitCore 执行 JUnit4 测试类 |
| `PythonMethod` | `python scripts.validate` | 调用 Python 脚本（限定名路径） |

各类型的具体产生式：

```xtext
ResoluteMethod: 'resolute' methodReference=[ecore::EObject|QualifiedName];
JavaMethod:     'java'     methodPath=QualifiedName
                           ('(' (params+=JavaParameter (',' params+=JavaParameter)*)? ')')?;
PythonMethod:   'python'   methodPath=QualifiedName;
ManualMethod:   'manual'   {ManualMethod} dialogID=QualifiedName;
PluginMethod:   'plugin'   methodID=ID;
AgreeMethod:    'agree'    (singleLayer?='single' | all?='all');
JUnit4Method:   'junit'    classPath=QualifiedName;
```

### 7.7 VerificationPrecondition 与 VerificationValidation

方法可以附带前置条件和后置验证。前置条件失败时，活动直接跳过（记录为 `PRECONDITIONFAIL`）：

```xtext
VerificationPrecondition returns VerificationCondition:
  'precondition' {VerificationPrecondition}
    method=[VerificationMethod|QualifiedName]
    '(' (parameters+=[FormalParameter|ID])* ')'
;

VerificationValidation returns VerificationCondition:
  'validation' {VerificationValidation}
    method=[VerificationMethod|QualifiedName]
    '(' (parameters+=[FormalParameter|ID])* ')'
;
```

### 7.8 VerificationMethodRegistry 中已注册的插件方法

`VerificationMethodDispatchers`（Xtend 类）通过 `switch` 语句硬编码了当前支持的内置插件方法：

```xtend
def Object dispatchVerificationMethod(PluginMethod vm,
    InstanceObject target, List<PropertyExpression> parameters) {
  switch (vm.methodID) {
    case "A429Consistency":          target.A429Consistency
    case "ConnectionBindingConsistency": target.ConnectionBindingConsistency
    case "PortDataConsistency":      target.PortDataConsistency
    case "MassAnalysis":             target.MassAnalysis
    case "BoundResourceAnalysis":    target.BoundResourceAnalysis
    case "NetworkBandwidthAnalysis": target.NetworkBandWidthAnalysis
    case "PowerAnalysis":            target.PowerAnalysis
    case "ResourceBudgets":          target.ResourceBudget
    default: null
  }
}
```

---

## 8. Assure 保证案例执行（org.osate.assure）

**源码目录**：`org.osate.assure/src/org/osate/assure/`

Assure 模块负责根据 `.alisa` 工作台文件构造和执行完整的保证案例，产生可持久化的结果模型（`.assure` 文件）。

### 8.1 顶层工作台语法（Alisa.xtext）

```xtext
AssuranceCase:
  'assurance' 'case' name=QualifiedName (':' title=STRING)?
  'for' system=[aadl2::ComponentType|AadlClassifierReference]
  '[' description=Description?
      assurancePlans+=AssurancePlan+
      tasks+=AssuranceTask*
  ']'
;

AssurancePlan:
  'assurance' 'plan' name=ID (':' title=STRING)?
  'for' target=[aadl2::ComponentImplementation|AadlClassifierReference]
  '[' (
    (description=Description)?
    & ('assure'           assure+=        [Verify::VerificationPlan|QualifiedName]+)?
    & ('assure' 'global'  assureGlobal+=  [Verify::VerificationPlan|QualifiedName]+)?
    & ('assure' 'subsystem' (assureSubsystems+=... | assureAll?='all'))?
    & ('assume' 'subsystem' (assumeSubsystems+=... | assumeAll?='all'))?
    & ('issues' issues+=STRING+)?
  ) ']'
;
```

注意 `AssuranceCase` 绑定到 `ComponentType`（接口），而 `AssurancePlan` 绑定到 `ComponentImplementation`（实现），这体现了 ALISA 的架构主导思想。

### 8.2 EMF 结果模型类层次

执行引擎遍历保证案例后，将结果存入以下 EMF 对象树（包 `org.osate.assure.assure`）：

```
AssureResult (基接口，含 Metrics)
├── AssuranceCaseResult
│     - name: String
│     - message: String
│     - modelResult: EList<ModelResult>
│
├── ModelResult
│     - plan: AssurancePlan (引用)
│     - target: ComponentImplementation (引用)
│     - claimResult: EList<ClaimResult>
│     - subsystemResult: EList<SubsystemResult>
│     - subAssuranceCase: EList<AssuranceCaseResult>
│
├── SubsystemResult
│     - targetSystem: Subcomponent (引用)
│     - claimResult: EList<ClaimResult>
│     - subsystemResult: EList<SubsystemResult>   (递归)
│
├── ClaimResult
│     - targetReference: QualifiedClaimReference
│     - modelElement: NamedElement (引用)
│     - message: String
│     - subClaimResult: EList<ClaimResult>        (递归)
│     - verificationActivityResult: EList<VerificationExpr>
│     - predicateResult: VerificationResult
│
└── VerificationResult (也是 AssureResult)
      - type: ResultType  (SUCCESS | FAILURE | ERROR | TBD)
      - issues: EList<Diagnostic>
      - results: EList<Result>
      - message: String
      - analysisresult: EList<AnalysisResult>
      │
      ├── VerificationActivityResult
      │     - targetReference: QualifiedVAReference
      │     - preconditionResult: VerificationResult
      │     - validationResult: VerificationResult
      │
      ├── ValidationResult
      ├── PreconditionResult
      └── PredicateResult
```

验证活动结果（`VerificationExpr`，非 `AssureResult`）的类层次：
```
VerificationExpr
├── VerificationResult (同时也是 AssureResult)
├── ThenResult
│     - first: EList<VerificationExpr>
│     - second: EList<VerificationExpr>
│     - didThenFail: boolean
└── ElseResult
      - first: EList<VerificationExpr>
      - error: EList<VerificationExpr>
      - fail: EList<VerificationExpr>
      - timeout: EList<VerificationExpr>
      - didElse: ResultType
```

### 8.3 Metrics：度量指标

每个 `AssureResult` 都持有一个 `Metrics` 对象，汇总统计信息：

```java
// org.osate.assure.assure.Metrics
int tbdCount;                         // 尚未执行
int successCount;                     // 成功
int failCount;                        // 失败
int errorCount;                       // 执行错误
int didelseCount;                     // 触发了 else 分支
int thenskipCount;                    // then 分支因前置失败被跳过
int preconditionfailCount;            // 前置条件失败
int validationfailCount;              // 后置验证失败
int featuresCount;                    // 目标组件的特征数量
int featuresRequirementsCount;        // 有需求覆盖的特征数
int qualityCategoryRequirementsCount; // 质量分类需求数
int totalQualityCategoryCount;        // 质量类别总数
int requirementsWithoutPlanClaimCount;// 无声明覆盖的需求数
int noVerificationPlansCount;         // 无验证计划的声明数
int requirementsCount;                // 总需求数
int exceptionsCount;                  // 例外/缓解需求数
int reqTargetHasEMV2SubclauseCount;   // 目标有 EMV2 子句的需求数
int featuresRequiringClassifierCount; // 需要分类器的特征数
int featuresWithRequiredClassifierCount; // 已有分类器的特征数
int weight;                           // 声明权重
long executionTime;                   // 执行耗时（毫秒）
```

### 8.4 AssureConstructor：结果树构造器

`AssureConstructor`（Xtend 类，`org.osate.assure.generator`）在执行前从 `AssuranceCase` 规格构造空的结果树，所有 `VerificationResult.type` 初始化为 `ResultType.TBD`。

### 8.5 AssureProcessor：执行引擎

`AssureProcessor`（Xtend 类，实现 `IAssureProcessor`）是 ALISA 的核心执行引擎，通过 Xtend 的 `dispatch void process(...)` 多分派模式递归遍历结果树：

```xtend
// 接口定义（@ImplementedBy 绑定到 AssureProcessor）
@ImplementedBy(AssureProcessor)
interface IAssureProcessor {
  def void processCase(AssuranceCaseResult assureResult,
                       CategoryFilter filter,
                       IProgressMonitor monitor, boolean save);
  def void setProgressUpdater((URI)=>void progressUpdater)
  def void setRequirementsCoverageUpdater(=>void requirementsCoverageUpdater)
}
```

执行流程（dispatch 调用链）：

```
processCase(AssuranceCaseResult, filter, monitor)
  └─ process(AssuranceCaseResult)
       └─ forEach modelResult → process(ModelResult)
            ├─ forEach claimResult → process(ClaimResult)
            │    ├─ [过滤：evaluateRequirementFilter(filter)]
            │    ├─ forEach vaResult → process(VerificationActivityResult)
            │    │    ├─ [过滤：evaluateVerificationActivityFilter + evaluateVerificationMethodFilter]
            │    │    ├─ process(PreconditionResult) [若存在]
            │    │    │    └─ runVerificationMethod(preconditionResult)
            │    │    ├─ runVerificationMethod(vaResult)  ← 核心执行点
            │    │    └─ process(ValidationResult)  [若存在]
            │    ├─ process(PredicateResult)
            │    └─ forEach subClaimResult → process(ClaimResult) [递归]
            ├─ forEach subsystemResult → process(SubsystemResult) [递归]
            └─ forEach subAssuranceCase → process(AssuranceCaseResult) [递归]
```

**参数绑定机制**：`runVerificationMethod` 中通过 `CommonInterpreter`（XSemantics 规则引擎）对实参 `AExpression` 求值，结果存入 `RuleEnvironment`（key 为 `"vals"` 和 `"computes"`）。单位不匹配时自动调用 `AssureUtilExtension.convertValueToUnit` 进行转换。

**结果处理**：

| 方法类型 | 返回值类型 | 结果设置逻辑 |
|---|---|---|
| `PluginMethod` | `String`（marker 类型 ID）或 `AnalysisResult` 或 `Result` | 遍历 `AnalysisResult.results`，匹配 `modelElement == target` |
| `JavaMethod` | 任意 Java 对象 | 通过 `VerifyJavaUtil` 反射调用，解析返回值 |
| `JUnit4Method` | `org.junit.runner.Result` | `failureCount == 0` 则 SUCCESS，否则 FAIL |
| `ResoluteMethod` | boolean | 通过 `ResoluteUtil` 调用 Resolute 验证 |
| `AgreeMethod` | 不支持 | 直接 `setToError("Execution of AGREE methods is not supported")` |
| `ManualMethod` | 无操作 | 保持 TBD，由用户手工在 UI 中标记 |

### 8.6 AssureRequirementMetricsProcessor：覆盖率计算

`AssureRequirementMetricsProcessor` 在执行完成后独立计算需求覆盖率指标，通过 `IReqspecGlobalReferenceFinder` 跨文件查找系统需求集，计算如下统计量：

- `requirementsCount`：目标组件的全部系统需求数量
- `requirementsWithoutPlanClaimCount`：未被任何 `Claim` 覆盖的需求数（追踪缺口）
- `qualityCategoryRequirementsCount`：属于 `quality` 分类的非特征需求数
- `featuresRequirementsCount`：被需求约束的特征（Feature）数量
- `exceptionsCount`：`exception` 类别需求数 + `mitigates` 字段非空的需求数

这些统计量最终显示在 ALISA 工作台的 Dashboard 视图中，用于评估系统保证的完整性。

---

## 9. org.osate.reqtrace 需求追踪矩阵

**源码状态**：在当前代码库（版本 2.11.0）中，`org.osate.reqtrace` 插件目录下**仅存在一个 `pom.xml` 文件**，没有任何 Java、Xtend 或 Xtext 源码：

```xml
<!-- org.osate.reqtrace/pom.xml -->
<project ...>
  <parent>
    <groupId>org.osate</groupId>
    <artifactId>alisa.parent</artifactId>
    <version>2.11.0.vfinal</version>
  </parent>
  <groupId>org.osate</groupId>
  <artifactId>org.osate.reqtrace</artifactId>
  <version>2.0.3-SNAPSHOT</version>
  <packaging>eclipse-plugin</packaging>
</project>
```

版本号 `2.0.3-SNAPSHOT` 与父 POM 的 `2.11.0` 明显不同，且后缀为 `-SNAPSHOT`，表明该模块**处于早期开发或规划阶段**，尚未实现任何功能。

### 9.1 设计意图（从模块名称推断）

根据模块名称 `reqtrace`（requirements traceability）以及 ALISA 的整体架构，该模块预计提供的功能应包括：

1. **需求追踪矩阵视图**：将 `StakeholderGoals.Goal` → `SystemRequirement` → `VerificationActivity` → `AssuranceCaseResult` 的四层链接可视化为矩阵形式
2. **双向追踪查询**：给定一个需求，找到覆盖它的所有验证活动；反之，给定一个验证活动，找到它所覆盖的需求
3. **覆盖率报告**：基于 `Metrics.requirementsWithoutPlanClaimCount` 等指标生成标准化追踪报告

### 9.2 现有的追踪能力

虽然 `org.osate.reqtrace` 尚未实现，ALISA 在以下位置已内置了需求追踪的数据基础：

- **需求间追踪**：`Requirement.refinesReference`、`decomposesReference`、`evolvesReference`、`goalReference`、`requirementReference` 字段
- **需求到文档**：`docReference` → `ExternalDocument`（`DOCPATH#QualifiedName` 格式）
- **需求到验证**：`VerificationPlan.claim[].requirement` 实现 N:1 的显式链接
- **验证到结果**：`ClaimResult.targetReference` 和 `VerificationActivityResult.targetReference` 实现运行时追踪链
- **覆盖率度量**：`AssureRequirementMetricsProcessor` 中的 `requirementsWithoutPlanClaimCount` 直接量化追踪缺口

---

## 10. 扩展点与集成

### 10.1 插件验证方法扩展

外部插件可以通过 `AnalysisPluginInterface`（`org.osate.verify.analysisplugins`）注册新的分析方法，由 `VerificationMethodDispatchers.dispatchVerificationMethod` 调度。当前内置方法包括：

| 方法 ID | 功能 |
|---|---|
| `A429Consistency` | ARINC 429 总线一致性检查 |
| `ConnectionBindingConsistency` | 连接绑定一致性 |
| `PortDataConsistency` | 端口数据一致性 |
| `MassAnalysis` | 质量/重量分析 |
| `BoundResourceAnalysis` | 资源绑定分析 |
| `NetworkBandwidthAnalysis` | 网络带宽分析 |
| `PowerAnalysis` | 电功耗分析 |
| `ResourceBudgets` | 资源预算分析 |

### 10.2 与 Resolute 集成

`AssureProcessor` 导入 `org.osate.resolute.ResoluteUtil`，在处理 `ResoluteMethod` 类型时调用 Resolute 的推理引擎。Resolute 方法通过 `methodReference=[ecore::EObject|QualifiedName]` 引用 Resolute 定义中的 `ResoluteFunction`。

### 10.3 与 AGREE 的关系

Verify 语法中定义了 `AgreeMethod`（含 `single` 和 `all` 两种模式），但在当前 `AssureProcessor.xtend` 实现中被直接设为 ERROR：

```xtend
AgreeMethod: {
  setToError(verificationResult, "Execution of AGREE methods is not supported")
}
```

这意味着 AGREE 方法的 UI 声明已就绪，但实际执行器尚未实现。

### 10.4 结果模型与 org.osate.result

`VerificationResult` 依赖 `org.osate.result` 包中的类型：
- `ResultType`：`SUCCESS | FAILURE | ERROR | TBD`
- `Diagnostic`（含 `DiagnosticType`）：问题诊断条目
- `Result`：单个分析结果，携带 `modelElement` 引用
- `AnalysisResult`：一次分析运行的全部结果集合

这使得 ALISA 的保证结果可以与 OSATE 的通用分析结果框架无缝集成。

### 10.5 参数类型转换

`AssureProcessor` 内置了 AADL2 属性值到 Java 类型的双向映射：

```
AadlBoolean  ↔  Boolean (Java)
AadlInteger  ↔  Integer / Long
AadlReal     ↔  Double
AadlString   ↔  String
BooleanValue ↔  BooleanLiteral (AADL2)
IntegerValue ↔  IntegerLiteral (带可选单位)
RealValue    ↔  RealLiteral (带可选单位)
StringValue  ↔  StringLiteral
```

单位自动转换：当形参（`FormalParameter.unit`）与实参单位不同时，通过 `AssureUtilExtension.convertValueToUnit` 先将实参转换为统一基础单位再传递。

---

## 11. 典型完整示例

以下示例展示 ALISA 四层文件如何协同工作：

**1. 利益相关者（Organization）**

```
// my_project.org
organization MyProject

  stakeholder sys_lead [
    full name "Alice Chen"
    role "Systems Engineer"
    email "alice@example.com"
  ]
```

**2. 分类定义（Categories）**

```
// my_project.cat
quality [ Safety Performance Reliability ]
phase   [ Concept Detailed Certification ]

filter CertFilter [ quality.Safety phase.Certification ]
```

**3. 利益相关者目标（ReqSpec）**

```
// my_goals.goals
stakeholder goals flight_goals for FlightControl
  use constants global_consts [
  description "Goals for the flight control system"

  goal g_latency : "System shall have bounded response time" [
    category quality.Performance
    description "All sensor inputs shall be processed within deadline."
    stakeholder MyProject.sys_lead
  ]

  goal g_safety : "No catastrophic failure mode" [
    category quality.Safety
    rationale "DO-178C Level A requirements."
    stakeholder MyProject.sys_lead
  ]
]
```

**4. 系统需求（ReqSpec）**

```
// fc_reqs.reqspec
system requirements fc_reqs for FlightControl.Impl
  use constants global_consts [
  val MaxE2ELatency = 100 ms

  requirement r_latency : "End-to-end latency bound" [
    category quality.Performance
    compute ActualLatency : real units Time_Units
    value predicate ActualLatency <= MaxE2ELatency
      with ActualLatency upto MaxE2ELatency
    development stakeholder MyProject.sys_lead
    see goal flight_goals.g_latency
  ]

  requirement r_no_spof : "No single point of failure on critical path" [
    category quality.Safety
    mitigates CriticalPathFailure
    see goal flight_goals.g_safety
    see document safety/fha.pdf#section_3
  ]
]
```

**5. 验证计划（Verify）**

```
// fc_verify.verify
verification plan fc_vplan for fc_reqs [
  description "Verification plan for FlightControl requirements"

  claim r_latency [
    activities
      a1: ActualLatency = FlowAnalysis.computeE2ELatency()
    assert a1
  ]

  claim r_no_spof [
    activities
      a2: SafetyAnalysis.checkFTA() [ category quality.Safety ]
      a3: manual dialogs.ReviewDialog
    assert a2 then a3
  ]
]

verification methods fc_methods [
  method checkFTA (component) returns (passed : boolean) : "Fault Tree Analysis" [
    plugin BoundResourceAnalysis
    description "Performs FTA on the target component"
  ]
]
```

**6. 保证案例（Alisa 工作台）**

```
// fc_assurance.alisa
assurance case fc_case : "FlightControl Assurance Case"
  for FlightControl [
  description "Full assurance case for the flight control system"

  assurance plan fc_plan : "Main Assurance Plan"
    for FlightControl.Impl [
    assure fc_vplan
    assure subsystem all
  ]

  assurance task cert_task : "Certification Task" [
    category quality.Safety phase.Certification
  ]
]
```

---

## 12. 依赖关系

### ALISA 依赖的 OSATE 模块

| 依赖模块 | 用途 |
|---|---|
| `core/org.osate.aadl2` | AADL2 元模型（`ComponentClassifier`、`NamedElement`、`Property` 等） |
| `core/org.osate.aadl2.instance` | 实例模型（`ComponentInstance`、`InstanceObject` 等） |
| `core/org.osate.result` | 通用分析结果框架（`ResultType`、`Diagnostic`、`AnalysisResult`） |
| `analyses/*` | 各类分析插件（通过 `PluginMethod` 调用） |
| `resolute/org.osate.resolute` | Resolute 形式化验证引擎 |
| `pluginsupport/org.osate.pluginsupport` | `ExecuteJavaUtil` 反射工具 |

### 被 ALISA 依赖的外部框架

| 框架 | 用途 |
|---|---|
| Eclipse Xtext | DSL 语法解析、作用域、代码补全 |
| Eclipse EMF | 元模型（EObject、EList、EcoreUtil） |
| XSemantics | `CommonInterpreter`（表达式求值规则引擎） |
| JUnit4 | `JUnitCore` 执行测试方法 |
| Google Guice | 依赖注入（`@ImplementedBy`、`@Inject`） |
