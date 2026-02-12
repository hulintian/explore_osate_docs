# AADL2 元模型设计

本文档基于源码分析，深入描述 OSATE2 的 AADL2 Ecore 元模型和实例模型的设计。

## 1. 元模型概览

AADL2 元模型定义在 `core/org.osate.aadl2/model/aadl2.ecore` 中：
- **nsURI**: `http://aadl.info/AADL/2.0`
- **nsPrefix**: `aadl2`
- 约 4700 行 Ecore XML，包含数百个 EClass

## 2. 核心类层次

### 2.1 根基类

```
Element (abstract) — 所有模型元素的基类
├── Comment — 文本注解
└── NamedElement (abstract) — 具有 name/qualifiedName/propertyAssociations
    ├── Type (abstract) — 约束类型化元素的值
    │   ├── PropertyType (abstract) — 属性类型基类
    │   ├── SubcomponentType (abstract)
    │   └── FeatureType (abstract)
    │
    ├── Namespace (abstract) — 可包含 NamedElement 的容器
    │   ├── Classifier (abstract) — 描述具有共同特性的实例集合
    │   ├── PackageSection (abstract)
    │   │   ├── PublicPackageSection
    │   │   └── PrivatePackageSection
    │   └── PropertySet — 属性定义的命名空间
    │
    ├── ClassifierFeature (abstract) — 分类器的特征
    │   ├── ModeFeature (abstract)
    │   │   ├── Mode
    │   │   └── ModeTransition
    │   ├── StructuralFeature (abstract)
    │   │   ├── Feature (abstract), Prototype (abstract), Connection (abstract)
    │   └── BehavioralFeature (abstract)
    │       ├── SubprogramCallSequence, SubprogramCall
    │
    └── RefinableElement (abstract) — 支持细化的元素
        └── StructuralFeature
```

### 2.2 组件类型与实现

这是 AADL2 元模型最核心的设计模式——**类型-实现分离**：

```
Classifier (abstract)
└── ComponentClassifier (abstract)
    ├── ComponentType (abstract) — 定义外部接口（features, flow specs）
    │   ├── AbstractType, SystemType, ProcessType, ThreadType
    │   ├── ThreadGroupType, ProcessorType, MemoryType
    │   ├── DeviceType, BusType, VirtualBusType, VirtualProcessorType
    │   ├── SubprogramType, SubprogramGroupType, DataType
    │
    └── ComponentImplementation (abstract) — 定义内部结构（subcomponents, connections）
        ├── AbstractImplementation, SystemImplementation, ProcessImplementation
        ├── ThreadImplementation, ThreadGroupImplementation, ...（与上对应）
```

**关键关系**：
- `ComponentImplementation` 通过 `ownedRealization: Realization` 关联到其 `ComponentType`
- `ComponentType` 可通过 `ownedExtension: TypeExtension` 扩展另一个同类型
- `ComponentImplementation` 可通过 `ownedExtension: ImplementationExtension` 扩展另一个实现

**设计意义**：
- Type 定义接口（对外可见的 features）
- Implementation 定义结构（内部 subcomponents、connections、flow implementations）
- 一个 Type 可以有多个 Implementation
- 支持继承和精化(refinement)

## 3. Feature（接口特征）建模

### 3.1 Feature 层次

```
Feature (abstract) — 组件的接口点
├── DirectedFeature (abstract) — 有方向的特征
│   ├── Port (abstract) — 类型化的连接点
│   │   ├── DataPort — 同步数据交换
│   │   ├── EventPort — 事件信号
│   │   └── EventDataPort — 带数据负载的事件
│   │
│   ├── Access (abstract) — 访问特征
│   │   ├── BusAccess — 总线访问
│   │   ├── DataAccess — 数据访问（共享数据）
│   │   ├── SubprogramAccess — 子程序访问
│   │   └── SubprogramGroupAccess
│   │
│   ├── Parameter — 子程序参数
│   ├── FeatureGroup — 特征组（捆绑多个特征）
│   └── AbstractFeature — 通用连接器
│
└── InternalFeature (abstract) — 仅在实现内部可见
    ├── EventSource, EventDataSource
    └── PortProxy, SubprogramProxy
```

### 3.2 关键 Feature API

```java
Feature.getAllFeatureRefinements()  // 返回 feature 及所有被精化的祖先链
Feature.getAllClassifier()          // 获取分类器（考虑精化链）
Feature.getAllPrototype()           // 获取原型（考虑精化链）
```

## 4. Connection（连接）建模

### 4.1 连接类型

```
Connection (abstract)
├── AccessConnection — 连接 Access 特征
├── ParameterConnection — 连接参数到端口
├── PortConnection — 连接数据/事件端口
├── FeatureConnection — 通用特征连接
└── FeatureGroupConnection — 连接特征组
```

### 4.2 连接结构

```
Connection
├── source: ConnectedElement (containment)
│   └── element → NamedElement (Context 或 Feature)
│   └── next → ConnectedElement (多跳路径)
├── destination: ConnectedElement (containment)
├── bidirectional: boolean
├── refined: Connection (optional, 精化链)
└── inMode: Mode[] (仅在特定模式下有效)
```

## 5. Subcomponent（子组件）建模

```
Subcomponent (abstract)
├── SystemSubcomponent, ProcessSubcomponent, ThreadSubcomponent, ...
│
├── subcomponentType: SubcomponentType (derived union)
│   └── 可以是 ComponentClassifier 或 ComponentPrototype
├── ownedPrototypeBindings: PrototypeBinding[] — 参数化绑定
├── ownedModeBindings: ModeBinding[]
├── inMode: Mode[] — 仅在这些模式中存在
├── arrayDimensions: ArrayDimension[] — 数组支持
└── refined: Subcomponent (optional)
```

## 6. Flow（流）建模

```
FlowSpecification (在 Type 中定义)
├── kind: FlowKind (SOURCE, SINK, PATH)
├── inEnd: FlowEnd (入口特征)
└── outEnd: FlowEnd (出口特征)

FlowImplementation (在 Implementation 中定义)
├── specification → FlowSpecification
├── flowElements: Connection/FlowSpecification 链

EndToEndFlow (在 Implementation 中定义)
├── flowElements: FlowElement[] (规格、实现、连接的有序链)
└── inMode: Mode[]
```

## 7. Property（属性）系统

### 7.1 属性定义

```
PropertySet (属性定义的命名空间)
├── ownedPropertyTypes: PropertyType[]
│   ├── AadlBoolean, AadlInteger, AadlReal, AadlString
│   ├── EnumerationType, RangeType, RecordType, ArrayType
│   ├── ClassifierType, UnitsType, ReferenceType
│
├── ownedProperties: Property[]
│   ├── propertyType → PropertyType
│   ├── defaultValue → PropertyExpression
│   ├── appliesToMetaclass: EClass[] — 适用于哪些元类
│   ├── inherit: boolean — 是否可继承
│   └── appliesToClassifier: Classifier[]
│
└── ownedPropertyConstants: PropertyConstant[]
```

### 7.2 属性应用

```
PropertyAssociation (出现在任何 NamedElement 上)
├── property → Property (引用哪个属性定义)
├── ownedValue: ModalPropertyValue[]
│   ├── value → PropertyExpression (实际值)
│   └── inMode: Mode (模式特定值)
├── appliesTo: ContainedNamedElement[] — 针对哪个子元素
├── append: boolean — 追加到继承值
└── inBinding: Classifier[] — 在哪些绑定上下文中适用
```

## 8. Instance（实例）模型

实例模型定义在 `core/org.osate.aadl2/model/instance.ecore`，是声明式模型的**展平运行时表示**。

### 8.1 核心实例类

```
SystemInstance (根，extends ComponentInstance)
├── componentImplementation → ComponentImplementation
├── systemOperationMode: SystemOperationMode[]

ComponentInstance
├── classifier → ComponentClassifier
├── subcomponent → Subcomponent (创建它的声明式子组件)
├── featureInstance: FeatureInstance[]
├── componentInstance: ComponentInstance[] (子组件树)
├── modeInstance: ModeInstance[]
├── modeTransitionInstance: ModeTransitionInstance[]
├── connectionInstance: ConnectionInstance[]
├── flowSpecification: FlowSpecificationInstance[]
└── endToEndFlow: EndToEndFlowInstance[]

ConnectionInstance
├── source/destination → ConnectionInstanceEnd
├── connectionReference: ConnectionReference[] (声明式连接段链)
│   ├── connection → Connection (声明式)
│   ├── context → ComponentInstance
│   └── reverse: boolean
├── complete: boolean
├── bidirectional: boolean
└── inSystemOperationMode: SystemOperationMode[]
```

### 8.2 声明式 vs 实例模型对比

| 声明式模型 | 实例模型 |
|-----------|---------|
| ComponentType/Implementation | ComponentInstance |
| Feature | FeatureInstance |
| Connection (层次内/跨层) | ConnectionInstance (端到端) |
| FlowSpecification/FlowImplementation | FlowSpecificationInstance |
| EndToEndFlow | EndToEndFlowInstance |
| Mode | ModeInstance |
| PropertyAssociation | 缓存的属性值 |

**关键区别**：声明式模型是可复用模板（类型+实现），实例模型是完全展开的具体系统树。分析工具主要操作实例模型。

## 9. 关键 EMF 设计模式

### 9.1 工厂模式
```java
Aadl2Factory.eINSTANCE.createSystemType();
Aadl2Factory.eINSTANCE.createDataPort();
```

### 9.2 Derived/Union 特征
- `Feature.featureClassifier` — classifier 和 prototype 的 union
- `Classifier.classifierFeature` — 收集所有特征的 union
- `ComponentImplementation.ownedSubcomponent` — 具体子组件类型的 union

### 9.3 Containment vs Reference
- **Containment**: ownedFeature, ownedSubcomponent, ownedConnection（生命周期管理）
- **Reference**: featureClassifier, type, refined（跨元素引用）
- **Bidirectional**: featuringClassifier ⟷ classifierFeature (eOpposite)

### 9.4 getAllXxx() 继承收集
```java
ComponentImplementation.getAllSubcomponents()   // 包含继承链上的所有子组件
ComponentType.getAllFeatures()                   // 包含扩展链上的所有特征
```

## 10. 核心设计决策总结

| 决策 | 说明 |
|------|------|
| **Type-Implementation 分离** | 接口与实现分离，支持多实现 |
| **Context 概念** | 子组件、端口、特征组等实现 Context 接口，支持嵌套作用域 |
| **Prototype/Binding** | 支持参数化设计模板（类似泛型） |
| **Array 支持** | ArrayableElement mixin 支持多维数组 |
| **Modal 语义** | 几乎所有元素支持 inMode 限制 |
| **Refinement 链** | 特征、连接、子组件都支持精化继承 |
| **noXxx 标志** | 显式声明 noFeatures/noSubcomponents 防止误用 |
