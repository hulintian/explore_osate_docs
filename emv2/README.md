# EMV2 错误模型

EMV2（Error Model Version 2）是 AADL 的错误模型附件（Error Model Annex）实现，通过 Xtext DSL 和 EMF 元模型，提供系统级的错误建模、错误传播分析和故障树生成功能，是进行安全性与可靠性分析的核心模块。

源码位置：`osate2/emv2/`，12 个 Eclipse 插件，90,018 行 Java + 12,497 行 Xtend + 304 个 AADL 测试文件。

---

## 1. 插件组织

| 插件 | 职责 |
|------|------|
| `org.osate.xtext.aadl2.errormodel` | Xtext 语法、EMF 元模型、语义验证、作用域 |
| `org.osate.xtext.aadl2.errormodel.ui` | 编辑器 UI、内容辅助、快速修复 |
| `org.osate.xtext.aadl2.errormodel.edit` | EMF Edit 支持（属性视图） |
| `org.osate.xtext.aadl2.errormodel.feature` | Feature 打包 |
| `org.osate.aadl2.errormodel.instance` | 错误模型实例化（EMV2AnnexInstantiator） |
| `org.osate.aadl2.errormodel.contrib` | 标准属性集贡献（EMV2.aadl 等） |
| `org.osate.aadl2.errormodel.faulttree` | 故障树 Ecore 元模型 |
| `org.osate.aadl2.errormodel.faulttree.generation` | FTAGenerator 算法实现 |
| `org.osate.aadl2.errormodel.faulttree.edit` | 故障树 EMF Edit |
| `org.osate.aadl2.errormodel.faulttree.design` | 故障树可视化（Sirius） |
| `org.osate.aadl2.errormodel.propagationgraph` | 错误传播图 Ecore + 构建工具 |
| `org.osate.aadl2.errormodel.analysis` | 错误影响正向分析 |
| `org.osate.aadl2.errormodel.help` | 帮助文档 |
| `org.osate.aadl2.errormodel.tests` | 单元测试 |

---

## 2. 语法结构（ErrorModel.xtext）

EMV2 语法定义在 `emv2/org.osate.xtext.aadl2.errormodel/src/.../ErrorModel.xtext`，作为 AADL Annex 子语言集成（通过 `AnnexParserAgent` 嵌入）。

### 2.1 顶层结构

```
EMV2Root:
    EMV2Library | EMV2Subclause[]

ErrorModelLibrary  — 可复用的错误模型库（.emv2 文件）
├── ErrorType[]               — 错误类型定义（支持继承/别名）
├── TypeSet[]                 — 错误类型集合
├── ErrorBehaviorStateMachine[] — 错误行为状态机
├── TypeMappingSet[]          — 类型映射集
└── TypeTransformationSet[]   — 类型转换集

ErrorModelSubclause  — 附加到 AADL 组件的错误规格（annex error_model {** **}）
├── ErrorPropagation[]        — 错误传播声明
├── ErrorFlow[]               — 错误流（Source / Sink / Path）
├── ErrorBehaviorTransition[] — 状态转换
├── OutgoingPropagationCondition[] — 出站传播条件
├── ErrorDetection[]          — 错误检测
├── CompositeState[]          — 组合错误行为（系统级）
└── PropagationPath[]         — 传播路径
```

### 2.2 错误类型系统

```
ErrorType
├── name: String
├── superType → ErrorType     — 继承（type X extends Y）
└── aliased  → ErrorType      — 别名（type X renames Y）

TypeSet — 错误类型集合（联合语义）
└── typeTokens: TypeToken[]

TypeToken — 类型构造器
└── type: ErrorType[]         — 用 * 组合为乘积类型
```

**设计要点**：层次化类型支持继承，TypeToken 的乘积类型支持复合错误建模（如 `{HardError * TimingError}`）。

### 2.3 错误传播与流

```
ErrorPropagation
├── direction: in | out
├── featureorPPRef → Feature / PropagationPoint
├── typeSet → TypeSet
└── not: boolean              — noerror 情况

ErrorFlow (abstract)
├── ErrorSource  — 错误产生源
│   ├── sourceModelElement → NamedElement
│   ├── failureModeReference → ErrorBehaviorState
│   ├── typeTokenConstraint → TypeSet
│   └── flowCondition → IfCondition (STRING / Resolute / Java)
├── ErrorSink    — 错误终止点（含类型约束）
└── ErrorPath    — 错误转换路径
    ├── incoming / outgoing → ErrorPropagation
    ├── typeMappingSet → TypeMappingSet
    └── targetToken → TypeToken
```

### 2.4 错误行为状态机

```
ErrorBehaviorStateMachine
├── states: ErrorBehaviorState[]
│   ├── name: String
│   ├── initial: boolean
│   └── typeSet → TypeSet (可选状态类型约束)
├── events: ErrorBehaviorEvent[]
│   ├── ErrorEvent    — 错误事件（触发状态转换）
│   ├── RepairEvent   — 修复事件
│   └── RecoverEvent  — 恢复事件
└── transitions: ErrorBehaviorTransition[]
    ├── source → ErrorBehaviorState | all
    ├── condition → ConditionExpression
    └── target → ErrorBehaviorState | same state | Branch[] (概率分支)
```

**AADL 示例**：
```aadl
error behavior Simple
  states
    Operational: initial state;
    Failed: state;
  transitions
    Operational -[Failure]-> Failed;
end behavior;
```

### 2.5 条件表达式（布尔代数）

```
ConditionExpression
├── AND, OR              — 基本逻辑运算
├── AllExpression        — 全部满足
├── OrmoreExpression     — N-of-M（至少 N 个满足）
├── OrlessExpression     — 至多 N 个满足
└── ConditionElement     — 引用事件/入站传播（带类型约束）
```

### 2.6 组合错误行为（系统级）

```
CompositeState — 基于子组件错误状态决定父组件状态
├── state → ErrorBehaviorState
└── condition → 子组件状态条件表达式

OutgoingPropagationCondition — 基于内部状态决定出站错误
├── state → ErrorBehaviorState | allStates
├── condition → ConditionExpression
└── outgoing → ErrorPropagation | all | noerror
```

---

## 3. 错误模型实例化

`EMV2AnnexInstantiator` 将文本错误规格转换为运行时实例模型（位于 `errormodel.instance` 插件）。

### 3.1 实例化流水线

```
对每个 ComponentInstance:
  1. 提取所有 ErrorModelSubclause
  2. 获取行为状态机
  3. 创建 EMV2AnnexInstance（根容器）
     ├── 4. 传播点实例化   → PropagationPointInstance
     ├── 5. 错误传播实例化 → FeaturePropagation / AccessPropagation
     │                       PointPropagation / BindingPropagation
     │                       + 匿名类型集解析（AnonymousTypeSet + TypeInstance）
     ├── 6. 错误事件实例化 → ErrorEventInstance / RepairEventInstance / RecoverEventInstance
     ├── 7. 状态实例化     → StateInstance（标记 initial）
     ├── 8. 转换实例化     → TransitionInstance
     │       ├── source: SourceStateReference | AllSources
     │       ├── condition: ConditionExpressionInstance
     │       └── destination: DestinationStateReference | SameState | Branches
     ├── 9. 出站传播条件实例化 → OutgoingPropagationConditionInstance
     └── 10. 检测实例化    → DetectionInstance（IntegerCode / StringCode / ConstantCode）

对 SystemInstance（系统级）:
  ├── 11. 连接路径实例化 → ConnectionPath
  └── 12. 绑定路径实例化 → BindingPath (processor / memory / connection / binding)
```

### 3.2 关键实例类

| 类 | 职责 |
|----|------|
| `EMV2AnnexInstance` | 根容器（extends AnnexInstance） |
| `AbstractTypeSet` | 类型集基类，提供 `flatten()` |
| `TypeSetInstance` | 引用命名类型集 |
| `AnonymousTypeSet` | 内联类型定义 |
| `TypeInstance` | 单个类型实例 |
| `TypeProductInstance` | 乘积类型实例 |
| `TransitionInstance` | 转换实例（源/条件/目标） |
| `CompositeConditionExpression` | 复合条件求值基类 |

---

## 4. 错误传播图

定义在 `errormodel.propagationgraph/model/PropagationGraph.ecore`。

```
PropagationGraph — 系统级错误传播拓扑
├── components: ComponentInstance[]
├── paths: PropagationGraphPath[]
│   ├── source / destination: PropagationPathEnd
│   │   ├── componentInstance → ComponentInstance
│   │   ├── errorPropagation  → ErrorPropagation
│   │   └── connectionInstance → ConnectionInstance (可选)
│   └── type: PropagationType (connection | binding | userDefined)
└── connections: ConnectionInstance[]
```

**生成方式**：`Util.generatePropagationGraph(root, false)` 从实例模型自动构建，遍历所有 ConnectionInstance 和 BindingInstance，将错误传播端点映射为图的边。

---

## 5. 故障树生成算法

`FTAGenerator`（位于 `faulttree.generation`）通过**反向遍历**从系统级错误状态追溯故障链。

### 5.1 算法流程

```
1. 根选择：用户选择顶层故障（错误状态或传播）

2. 反向遍历（PropagationGraphBackwardTraversal）
   ├── 从选中错误状态/传播出发
   ├── 反向追踪错误路径和 Sink
   ├── 展开入站错误的条件
   └── 递归处理子组件

3. 事件层次构建
   ├── FaultTree（根 Event）
   ├── Event 类型: Basic（叶节点）| External | Undeveloped | Intermediate（门）
   └── SubEventLogic: OR | AND | XOR | PriorityAnd | kOf | kOrmore | kOrless

4. 门优化
   ├── flattenGates()    — 合并同类嵌套门
   ├── cleanupXORGates() — 移除单事件 XOR 门
   ├── optimizeGates()   — 最小化事件树
   └── minimalCutSet()   — 计算最小割集

5. 概率计算
   ├── fillProbabilities()    — 分配/传播概率
   └── computeProbabilities() — 按门类型递归聚合概率
```

### 5.2 故障树类型

| 类型 | 说明 |
|------|------|
| `FaultTree` | 优化后的树，门已扁平化 |
| `FaultTrace` | 保留完整路径细节的反向追踪 |
| `CompositeParts` | 仅显示直接贡献者，不展开 |
| `MinimalCutSet` | 最小割集表示（MCS 分析） |

### 5.3 正向错误影响分析

`PropagateErrorSources` 执行正向影响分析（与 FTA 的反向遍历互补）：
- 从错误源出发，前向追踪影响范围
- 可配置最大追踪深度（默认 7 级）
- 使用 `visited` 集合避免循环追踪
- 使用 `alreadyTreated` 缓存避免重复处理
- 生成 CSV 格式的影响报告

### 5.4 完整分析链

```
AADL 文本 (annex error_model {** ... **})
    ↓ AnnexParserAgent → EMV2AnnexParser
ErrorModel Ecore 模型（声明式）
    ↓ EMV2AnnexInstantiator
EMV2AnnexInstance（运行时实例）
    ↓ Util.generatePropagationGraph()
PropagationGraph（传播拓扑）
    ↓ FTAGenerator / PropagateErrorSources
FaultTree / 影响报告
    ↓ 门优化 + 概率计算
分析结果（Eclipse Markers + CSV 报告）
```

---

## 6. 标准支持与应用领域

| 标准 | 对应功能 |
|------|---------|
| ARP4761 | FHA/FTA/FMEA 分析流程支持 |
| ARP4754A | 民用飞机系统开发指南中的安全需求 |
| DO-178C | 机载软件认证中的错误处理要求 |
| ISO 26262 | 道路车辆功能安全（ASIL 等级） |
| IEC 61508 | 工业安全相关系统（SIL 验证） |
| SAE-AS5506 | AADL 错误模型附件标准 |

---

## 7. 依赖关系

```
core（AADL 实例模型） ──→ emv2（EMV2AnnexInstance）
analyses（分析框架）  ──→ emv2（Handler 层次）
emv2 ──→ alisa（Verify DSL 中引用错误状态作为验证条件）
emv2 ──→ ge（GE errormodel Handler 可视化错误传播）
```
