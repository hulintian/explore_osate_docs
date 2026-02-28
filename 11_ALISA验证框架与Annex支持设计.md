# ALISA 验证框架与 Annex 支持设计

本文档基于源码分析，描述 OSATE2 的 ALISA 验证框架和 Annex 子语言支持架构。

## Part A：ALISA 验证框架

### 1. ALISA 概览

ALISA（Architecture-Led Incremental System Assurance）是 OSATE2 的验证保证框架，实现从需求定义到验证执行到保证案例的完整流水线。

```
ReqSpec (需求规格) → Verify (验证计划) → Alisa (保证案例) → Assure (执行结果)
```

框架由四个 Xtext 语言和一个执行引擎组成：

| 模块 | 位置 | 职责 |
|------|------|------|
| `org.osate.alisa.common` | 基础语法 | 共享类型系统、表达式、描述元素 |
| `org.osate.reqspec` | ReqSpec.xtext | 需求和目标定义 |
| `org.osate.verify` | Verify.xtext | 验证计划和方法定义 |
| `org.osate.alisa.workbench` | Alisa.xtext | 保证案例编排 |
| `org.osate.assure` | 执行引擎 | 验证执行和结果收集 |

### 2. Common 基础语法

位置：`alisa/org.osate.alisa.common/src/.../Common.xtext`

所有 ALISA 语言共享的基础设施：

#### 2.1 描述与理由

```
Description — 富文本描述
├── text: STRING              — 文本段
├── thisTarget?: boolean      — 引用 'this'（当前模型元素）
├── image: ImageReference     — 图片引用
└── showValue: AExpression    — 带单位转换的值表达式

Rationale — 设计理由
└── description: Description

Uncertainty — 不确定性度量
├── volatility: int    — 变更频率
├── precedence: int    — 优先级
└── impact: int        — 影响范围
```

#### 2.2 类型系统

```
TypeRef — 支持的类型
├── boolean, integer, real, string
├── model element — AADL 模型元素引用
├── PropertyRef → AADL Property 类型
├── 整数/实数支持单位规格
└── Range 类型: [type] 语法
```

#### 2.3 表达式系统

```
AExpression — 完整表达式语言
├── 算术: +, -, *, /, div, mod
├── 比较: ==, !=, <, <=, >, >=, >< (范围)
├── 逻辑: and/&&, or/||, not
├── 函数: 限定函数调用带参数
├── 条件: if-then-else-endif
├── 范围: [min..max] 和 [min..max delta step]
├── 单位转换: % (转换) 和 in (去除单位)
├── 属性访问: # 运算符
└── 模型遍历: . 运算符
```

#### 2.4 变量声明

```
ValDeclaration — 类型化值（带初始化）
ComputeDeclaration — 类型化计算值（无初始值，验证时计算）
```

### 3. ReqSpec 需求规格语法

位置：`alisa/org.osate.reqspec/src/.../ReqSpec.xtext`

#### 3.1 需求组织

```
ReqSpec — 顶层容器
├── SystemRequirementSet — 绑定到 ComponentClassifier 的需求集
│   ├── name, title
│   ├── target → ComponentClassifier
│   ├── requirements: Requirement[]
│   ├── constants: ValDeclaration[]
│   ├── computes: ComputeDeclaration[]
│   └── stakeholderGoals → StakeholderGoals[]
│
├── GlobalRequirementSet — 跨组件的全局需求
│   └── requirements: GlobalRequirement[]
│
├── StakeholderGoals — 利益相关者目标
│   ├── target → ComponentClassifier
│   └── goals: Goal[]
│
├── ReqDocument — 需求文档容器
│   └── content: (Goal | Requirement | ReqDocument)[]
│
└── GlobalConstants — 全局常量定义
```

#### 3.2 需求定义

```
Requirement — 核心需求元素
├── name, title
├── target → ComponentClassifier
├── targetElement → NamedElement     — 精确到模型元素
├── category: Category[]             — 语义分类（如 Quality.Mass）
├── componentCategory: ComponentCategory — 适用组件类型
│
├── description: Description
├── rationale: Rationale
├── uncertainty: Uncertainty
│
├── predicate: ReqPredicate          — 形式化断言
│   └── 如: MaximumWeight == #SEI::WeightLimit
│
├── constants: ValDeclaration[]      — 局部常量
├── computes: ComputeDeclaration[]   — 计算声明
│
├── 需求关系:
│   ├── refinesReference → Requirement[]    — 细化
│   ├── decomposesReference → Requirement[] — 分解
│   ├── inheritsReference → Requirement     — 继承
│   ├── evolvesReference → Requirement[]    — 演化
│   ├── conflictsReference → Requirement[]  — 冲突
│   ├── goalReference → Goal[]              — 关联目标
│   └── requirementReference → Requirement[] — 交叉引用
│
├── dropped?: boolean               — 标记已废弃（附理由）
├── issues: String[]                — 已知问题
└── stakeholderReference → Stakeholder[]   — 责任归属
```

#### 3.3 目标定义

```
Goal — 利益相关者目标
├── name, title
├── target → ComponentClassifier
├── description, rationale, uncertainty
├── refinesReference → Goal[]        — 目标精化
└── stakeholderReference → Stakeholder[]
```

### 4. Verify 验证计划语法

位置：`alisa/org.osate.verify/src/.../Verify.xtext`

#### 4.1 验证计划

```
VerificationPlan — 验证计划
├── name, title
├── requirementSet → RequirementSet  — 被验证的需求集
├── description: Description
├── claims: Claim[]                  — 断言集合
├── rationale: Rationale
└── issues: String[]
```

#### 4.2 断言与活动

```
Claim — 验证断言
├── requirement → Requirement (可选)  — 关联需求
├── title: String
├── activities: VerificationActivity[]
├── assert: ArgumentExpr             — 断言逻辑表达式
├── rationale: Rationale
├── weight: int                      — 优先权重
├── subclaim: Claim[]                — 嵌套子断言
└── issues: String[]

VerificationActivity — 验证活动
├── name, title
├── computes → ComputeDeclaration[]  — 产出计算值
├── method → VerificationMethod      — 执行方法
├── actuals: AExpression[]           — 传入参数
├── propertyValues: Property[]       — AADL 属性上下文
├── category: Category[]
├── timeout: AIntegerTerm            — 超时限制
└── weight: int                      — 活动权重
```

#### 4.3 验证方法

```
VerificationMethod — 可执行的验证方法
├── name, title
├── targetType: TargetType           — 适用目标类型
│   └── COMPONENT | FEATURE | CONNECTION | FLOW | MODE | ELEMENT | ROOT
├── target → ComponentClassifier     — 可选组件限制
├── formals: FormalParameter[]       — 形式参数
├── properties: Property[]           — 引用的 AADL 属性
├── results: FormalParameter[]       — 返回值参数
├── isPredicate: boolean             — 是否返回布尔值
├── isResultReport: boolean          — 是否返回报告
│
├── methodKind: MethodKind           — 方法类型（见下）
├── precondition → VerificationPrecondition   — 前置条件
└── validation → VerificationValidation       — 后置验证
```

#### 4.4 方法类型

| MethodKind | 说明 |
|------------|------|
| `ResoluteMethod` | Resolute 定理证明器引用 |
| `JavaMethod` | 反射调用 Java 方法（带可选参数） |
| `ManualMethod` | 人工验证（附对话框 ID） |
| `PluginMethod` | 插件验证（通过 methodID） |
| `AgreeMethod` | AGREE 假设-保证验证（单层/全层） |
| `JUnit4Method` | JUnit 4 测试类执行 |
| `PythonMethod` | Python 脚本验证 |

#### 4.5 证据链表达式

```
ArgumentExpr — 证据链逻辑
├── ThenEvidenceExpr — 顺序执行（A; then B）
├── ElseEvidenceExpr — 失败时替代
│   ├── normal execution
│   ├── fail branch — 活动失败时执行
│   ├── timeout branch — 活动超时时执行
│   └── error branch — 活动出错时执行
├── CompositeEvidenceExpr — 括号或量化表达式
├── QuantifiedEvidenceExpr — 'all' 量化
└── VAReference — 引用验证活动
```

#### 4.6 验证方法注册表

```
VerificationMethodRegistry — 方法库
├── name, title
├── description: Description
└── methods: VerificationMethod[]
```

### 5. Alisa 保证案例语法

位置：`alisa/org.osate.alisa.workbench/src/.../Alisa.xtext`

```
AssuranceCase — 保证案例（顶层）
├── name, title
├── system → ComponentType          — 被保证的根系统
├── description: Description
├── assurancePlans: AssurancePlan[] — 保证计划（1个或多个）
└── tasks: AssuranceTask[]          — 可选任务

AssurancePlan — 保证计划
├── name, title
├── target → ComponentImplementation — 目标实现
├── assure → VerificationPlan[]      — 验证计划引用
├── assureGlobal → VerificationPlan[] — 全局验证计划
├── assureSubsystems: Subcomponent[] | all — 保证哪些子系统
├── assumeSubsystems: Subcomponent[] | all — 假设哪些子系统
└── issues: String[]

AssuranceTask — 保证任务
├── name, title
├── category: Category[]            — 任务类别
├── anyCategory?: boolean           — "any" 类别匹配
├── description: Description
└── issues: String[]
```

**示例**：
```
assurance case SCSCase for SimpleControlSystem::SCS [
    assurance plan SCSPlan for SimpleControlSystem::SCS.tier1 [
        assure scsvplan scstier0vplan
        assure subsystem dcs
        assume subsystem sensor1 sensor2 actuator
    ]
    assurance task SCSWeight [
        category Quality.Mass
    ]
]
```

### 6. Assure 执行引擎

位置：`alisa/org.osate.assure/src/.../assure/`

#### 6.1 结果元模型

```
AssuranceCaseResult — 保证案例结果（根）
├── name, message
└── modelResult: ModelResult[]     — 每个验证模型元素的结果

ModelResult — 模型结果
├── 包含结果度量和子系统结果

ClaimResult — 断言结果
├── 聚合 VerificationActivityResult
└── subclaim: ClaimResult[]        — 嵌套断言结果

VerificationActivityResult — 单个验证活动的执行结果
├── activity → VerificationActivity
├── 成功/失败/错误状态
├── 实际返回值
├── 执行时间
└── 错误消息

ThenResult / ElseResult — 证据链执行跟踪
├── 后继结果引用

ValidationResult / PreconditionResult — 方法条件检查结果
PredicateResult — 需求谓词验证结果
```

#### 6.2 度量体系

```
Metrics — 全面的覆盖跟踪
├── tbdCount              — 待定项数
├── successCount          — 成功验证数
├── failCount             — 失败验证数
├── errorCount            — 执行错误数
├── didelseCount          — "else" 分支执行数
├── thenskipCount         — "then" 跳过数
├── preconditionfailCount — 前置条件失败数
├── validationfailCount   — 后置验证失败数
├── featuresCount         — 已验证特征数
├── featuresRequirementsCount         — 特征上的需求数
├── qualityCategoryRequirementsCount  — 质量类别需求数
├── totalQualityCategoryCount         — 质量类别总数
├── requirementsWithoutPlanClaimCount — 无验证计划的需求数
├── noVerificationPlansCount          — 无计划的需求数
├── requirementsCount                 — 总需求数
├── exceptionsCount                   — 异常数
├── reqTargetHasEMV2SubclauseCount    — 含错误模型的需求数
├── featuresRequiringClassifierCount  — 需要分类器的特征数
├── featuresWithRequiredClassifierCount — 已有分类器的特征数
├── weight                            — 总断言权重
└── executionTime                     — 总执行时间(ms)
```

### 7. 完整验证流水线

```
┌─────────────────────────────────────────────────────────┐
│                   ReqSpec (需求定义)                       │
│  SystemRequirementSet (如 SCS.reqspec)                    │
│  ├── Requirement R1: "SCS weight limit"                  │
│  │   ├── predicate: MaximumWeight == #SEI::WeightLimit   │
│  │   └── category: Quality.Mass                          │
│  └── include GlobalRequirements                          │
└───────────────────────────┬─────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   Verify (验证计划)                        │
│  VerificationPlan (如 SCSVerification.verify)             │
│  ├── Claim R1                                            │
│  │   ├── activities: checkWeight: PropertyComparison()   │
│  │   └── assert: checkWeight [success]                   │
│  └── VerificationMethodRegistry                          │
│      └── VerificationMethod PropertyComparison           │
│          ├── java: com.example.PropertyCheck              │
│          └── formals: [expected: real, actual: real]      │
└───────────────────────────┬─────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   Alisa (保证案例)                         │
│  AssuranceCase SCSCase                                   │
│  ├── for SimpleControlSystem::SCS                        │
│  ├── AssurancePlan SCSPlan                               │
│  │   ├── for SCS.tier1                                   │
│  │   ├── assure scsvplan scstier0vplan                   │
│  │   └── assure subsystem dcs                            │
│  └── AssuranceTask SCSWeight                             │
│      └── category Quality.Mass                           │
└───────────────────────────┬─────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   Assure (执行引擎)                        │
│  1. 加载 ReqSpec 需求                                     │
│  2. 映射 Claim 到 Requirement                             │
│  3. 创建保证案例结构                                       │
│  4. 执行验证方法:                                          │
│     ├── Resolute: 形式化证明                               │
│     ├── Java: ExecuteJavaUtil 反射调用                     │
│     ├── Manual: 人工审查对话框                              │
│     ├── Plugin: 插件验证 Handler                           │
│     ├── AGREE: 假设-保证验证                                │
│     ├── JUnit4: 测试框架执行                               │
│     └── Python: Python 脚本                               │
│  5. 收集 AssuranceCaseResult                              │
│  6. 计算 Metrics 覆盖率                                   │
│  7. 需求→证据可追溯性                                      │
└─────────────────────────────────────────────────────────┘
```

### 8. 需求追溯机制

| 追溯方式 | 实现 | 说明 |
|---------|------|------|
| **直接元素引用** | `Requirement.targetElement` | 需求适用于特定模型元素 |
| **类别追溯** | `Requirement.category` + `AssuranceTask.category` | 语义分组追溯 |
| **断言-需求映射** | `Claim.requirement` | 显式可追溯链接 |
| **计算跟踪** | `ComputeDeclaration` + `VerificationActivity.computes` | 计算值回传 |
| **目标-需求链接** | `Requirement.goalReference` | 需求关联利益相关者目标 |
| **需求关系图** | refines/decomposes/inherits/evolves/conflicts | 需求依赖图 |
| **子系统分解** | `AssurancePlan.assureSubsystems` | 层次化保证 |

---

## Part B：Annex 子语言支持架构

### 9. Annex 注册表体系

位置：`core/org.osate.annexsupport/`

OSATE 提供完整的插件框架，用于集成领域特定语言（DSL）作为 AADL Annex。

#### 9.1 扩展点定义

在 `plugin.xml` 中定义了 8 个扩展点：

| 扩展点 ID | 接口 | 职责 |
|-----------|------|------|
| `parser` | `AnnexParser` | 文本 → AST（Annex 语法解析） |
| `unparser` | `AnnexUnparser` | AST → 文本（反序列化） |
| `resolver` | `AnnexResolver` | 交叉引用解析 |
| `linkingservice` | `AnnexLinkingService` | 作用域与名称绑定 |
| `textpositionresolver` | `AnnexTextPositionResolver` | 文本位置映射 |
| `instantiator` | `AnnexInstantiator` | 实例模型转换 |
| `highlighter` | `AnnexHighlighter` | 语法高亮 |
| `contentassist` | `AnnexContentAssist` | 代码补全 |

#### 9.2 注册表架构

```java
AnnexRegistry (abstract) — 所有注册表基类
├── 静态常量: ANNEX_PARSER_EXT_ID, ANNEX_UNPARSER_EXT_ID, ...
├── getRegistry(extensionId) — 工厂方法
└── registerAnnex(...) — 无头模式手动注册

具体注册表:
├── AnnexParserRegistry → getAnnexParser(annexName)
├── AnnexUnparserRegistry → getAnnexUnparser(annexName)
├── AnnexResolverRegistry → getAnnexResolver(annexName)
├── AnnexLinkingServiceRegistry → getAnnexLinkingService(annexName)
├── AnnexInstantiatorRegistry → getAnnexInstantiator(annexName)
├── AnnexHighlighterRegistry → getAnnexHighlighter(annexName)
├── AnnexContentAssistRegistry → getAnnexContentAssist(annexName)
└── AnnexTextPositionResolverRegistry → getAnnexTextPositionResolver(annexName)
```

#### 9.3 AnnexParser 接口

```java
public interface AnnexParser {
    AnnexLibrary parseAnnexLibrary(
        String annexName, String source, String filename,
        int line, int column, ParseErrorReporter errReporter);

    AnnexSubclause parseAnnexSubclause(
        String annexName, String source, String filename,
        int line, int column, ParseErrorReporter errReporter);

    default String getFileExtension() { return null; }
}
```

#### 9.4 两种运行模式

| 模式 | 注册方式 | 使用场景 |
|------|---------|---------|
| **Eclipse 模式** | 自动从 `plugin.xml` 加载扩展 | 正常 IDE 运行 |
| **无头模式** | 通过 `registerAnnex()` 手动注册 | 命令行/测试环境 |

#### 9.5 命名空间管理

- Annex 名称注册附带命名空间 URI
- 支持通配符 `"*"` 作为默认 Handler
- 大小写不敏感查找，回退到通配符
- `DefaultAnnexParser` / `DefaultAnnexUnparser` 作为无特定 Handler 时的后备

### 10. Annex 集成流程

```
AADL 文本中 'annex <name> {** ... **}'
    ↓
AnnexParserAgent（绑定为 ILinker）捕获 ANNEXTEXT
    ↓
AnnexParserRegistry.getAnnexParser("<name>")
    ↓ 找到对应解析器
解析器（如 EMV2AnnexParser）解析为 EMF 模型
    ↓
AnnexLinkingServiceRegistry → 解析交叉引用
    ↓
AnnexResolverRegistry → 解析引用
    ↓
AnnexHighlighterRegistry → 语法高亮
    ↓
AnnexContentAssistRegistry → 代码补全
    ↓
AnnexInstantiatorRegistry → 实例化
```

### 11. 已注册的 Annex 子语言

| Annex 名称 | 模块 | 功能 |
|-----------|------|------|
| `error_model` | EMV2 | 错误模型（错误类型、传播、行为状态机） |
| `behavior_specification` | BA | 行为附件（状态机、动作序列） |

### 12. Plugin Support 基础设施

位置：`core/org.osate.pluginsupport/`

#### 12.1 扩展点

| 扩展点 | 用途 |
|--------|------|
| `org.osate.pluginsupport.aadlcontribution` | 贡献 AADL 包/属性集 |
| `org.osate.pluginsupport.registeredjavaclasses` | 注册 Java 方法供验证调用 |

#### 12.2 Java 方法动态调用

```java
ExecuteJavaUtil {
    // 通用反射调用
    invokeJavaMethod(String qualifiedMethodName, EObject argument)
    invokeJavaMethod(String qualifiedMethodName, Class<?>[] parameterTypes, Object[] arguments)
    getJavaMethod(String qualifiedMethodName, Class<?>[] parameterTypes)
}
```

**特性**：
- 通过 Eclipse 扩展注册表发现注册的 Java 类
- 支持单参数方法（EObject）或多参数方法
- 反射方法查找与调用
- 配置元素缓存提升性能
- 供 ALISA 的 `JavaMethod` 验证类型使用

#### 12.3 AADL 资源贡献

```java
PluginSupportUtil {
    getContributedAadl()             // 平台插件 URI
    getContributedAadlAsClasspath()  // 类路径 URI
    getContributedPropertySets()     // 属性集映射
}
```

插件可通过此机制贡献标准或自定义属性集和 AADL 包。

### 13. 完整扩展点索引

#### Annex 支持（8 个扩展点）

1. `org.osate.annexsupport.parser` — 自定义 Annex 语法解析
2. `org.osate.annexsupport.unparser` — 自定义 Annex 序列化
3. `org.osate.annexsupport.resolver` — Annex 交叉引用解析
4. `org.osate.annexsupport.linkingservice` — 作用域与名称绑定
5. `org.osate.annexsupport.textpositionresolver` — 位置跟踪
6. `org.osate.annexsupport.instantiator` — 实例模型转换
7. `org.osate.annexsupport.highlighter` — 语法高亮
8. `org.osate.annexsupport.contentassist` — 代码补全

#### 插件支持（2 个扩展点）

1. `org.osate.pluginsupport.aadlcontribution` — AADL 包/属性集贡献
2. `org.osate.pluginsupport.registeredjavaclasses` — Java 方法注册

#### ALISA 工作台（EMF 包注册）

- ReqSpec 元模型、Verify 元模型、Assure 元模型
- Alisa（工作台）元模型、Categories 元模型、Organization 元模型

#### UI 扩展

- `org.eclipse.ui.editors` — Xtext 编辑器
- `org.eclipse.ui.views` — Assurance View, Requirements Coverage View
- `org.eclipse.ui.newWizards` — 保证计划文件创建向导
- `org.eclipse.xtext.builder.participant` — 增量构建集成

## 14. org.osate.reqtrace 需求追踪矩阵生成

### 14.1 模块定位与现状

**源码路径：** `/home/hlt/project/explore_osate/osate2/alisa/org.osate.reqtrace/`

`org.osate.reqtrace` 是 ALISA 验证框架体系中专门用于生成**需求追踪矩阵（Requirements Traceability Matrix, RTM）** 的独立插件模块。其 Maven 构建配置文件如下：

```xml
<!-- 文件：/home/hlt/project/explore_osate/osate2/alisa/org.osate.reqtrace/pom.xml -->
<project>
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

需要特别注意的是：当前仓库中 `org.osate.reqtrace` 目录内**仅包含 `pom.xml` 文件**，没有任何 Java 源文件、`plugin.xml`、`META-INF/MANIFEST.MF` 或其他资源文件。这一现象意味着：

1. **模块处于预规划/占位阶段**：`pom.xml` 中的版本号 `2.0.3-SNAPSHOT` 表明它已被纳入项目的版本管理体系，但功能实现尚未合并到当前代码分支。
2. **从父 POM 中移除**：`alisa/pom.xml` 的 `<modules>` 列表中并未包含 `org.osate.reqtrace`，当前构建流程会跳过此模块。
3. **功能意图明确**：从命名和位置（位于 ALISA 验证框架 `alisa/` 目录下）可以推断其功能定位是对 `org.osate.reqspec`（需求规格）、`org.osate.verify`（验证计划）、`org.osate.assure`（保证结果）等模块的数据进行横向整合，生成可导出的追踪矩阵。

### 14.2 需求追踪矩阵的概念背景

在基于模型的系统工程（MBSE）流程中，需求追踪矩阵（RTM）是核对"每项需求都被验证了吗？每项验证都源自需求吗？"的关键文档。OSATE2/ALISA 框架的完整数据模型天然具备生成 RTM 的所有数据：

```
需求层（ReqSpec）
  SystemRequirementSet / GlobalRequirementSet
    └── Requirement (id, title, description, category, targetElement)

验证层（Verify）
  VerificationPlan
    └── Claim
         └── VerificationActivity (method, precondition, computes)

保证层（Assure）
  AssuranceCaseResult
    └── ModelResult
         └── ClaimResult
              └── VerificationActivityResult (resultType: success/fail/...)
                   └── Metrics (tbdCount, successCount, failCount, ...)
```

`org.osate.reqtrace` 的预期功能是将上述三层数据关联为如下矩阵结构：

| 需求 ID | 需求描述 | 目标组件 | 验证方法 | 验证状态 | 覆盖率 |
|---------|---------|---------|---------|---------|--------|
| SR-001  | 响应时间 < 10ms | FlightControl.impl | checkLatency | PASS | 100% |
| SR-002  | 故障安全隔离 | WBS_inst | EMV2Analysis | FAIL | 100% |
| SR-003  | 冗余切换  | ControlSys.impl | RedundancyCheck | TBD | 0% |

### 14.3 与 ALISA 体系各模块的数据关联

`org.osate.reqtrace` 需要读取以下模块产生的模型数据：

#### 14.3.1 来自 org.osate.reqspec 的需求数据

`ReqSpec.xtext`（位于 `/home/hlt/project/explore_osate/osate2/alisa/org.osate.reqspec/src/org/osate/reqspec/ReqSpec.xtext`）定义了需求的核心语法结构：

```xtext
SystemRequirementSet returns RequirementSet:
  'system' 'requirements' name=QualifiedName
  'for' target=[aadl2::ComponentClassifier|AadlClassifierReference]
  '[' (requirements+=SystemRequirement | ...)* ']'
;

SystemRequirement returns Requirement:
  'requirement' name=ID (':' title=STRING)?
  ('for' targetElement=[aadl2::NamedElement|ID])?
  '[' (
      ('category' category+=[categories::Category|QualifiedName]+)
    | description=Description
    | ('refines' refinesReference+=[Requirement|QualifiedName]+)
    | ('decomposes' decomposesReference+=[Requirement|QualifiedName]+)
    | ('see' 'goal' goalReference+=[Goal|QualifiedName]+)
    | ...
  )* ']'
;
```

RTM 生成需要从此处提取：需求 ID (`name`)、标题 (`title`)、目标组件 (`target`)、目标元素 (`targetElement`)、类别 (`category`)、层级关系 (`refinesReference`/`decomposesReference`)。

#### 14.3.2 来自 org.osate.verify 的验证计划数据

验证层记录了每条需求对应的验证活动（`VerificationActivity`），包括使用的验证方法（`JavaMethod`/`PluginMethod`/`AgreeMethod`/`ResoluteMethod`/`ManualMethod`）。追踪矩阵需从此处提取方法名称和参数。

#### 14.3.3 来自 org.osate.assure 的执行结果数据

`org.osate.assure` 的 EMF 模型（位于 `/home/hlt/project/explore_osate/osate2/alisa/org.osate.assure/src/org/osate/assure/assure/`）提供运行时结果：

```java
// VerificationResult.java — 验证结果枚举
// 包路径：org.osate.assure.assure
// 可能的状态值：SUCCESS, FAIL, TIMEOUT, ERROR, TBD, SKIPPED

// Metrics.java — 覆盖率指标聚合
// 包路径：org.osate.assure.assure
// 字段：tbdCount, successCount, failCount, errorCount, timeoutCount,
//       skippedCount, didelCount, claimCount, requirementCount

// ClaimResult.java — 每条 Claim 的执行快照
// ModelResult.java — 每个实例模型的整体结果快照
// AssuranceCaseResult.java — 整个保证案例的顶层结果
```

### 14.4 预期的功能架构设计

根据 ALISA 框架的架构模式，`org.osate.reqtrace` 预期实现如下功能组件：

#### 14.4.1 数据聚合器（Traceability Aggregator）

```
RequirementsTraceabilityAggregator
  ├── loadRequirements(IProject project)
  │     从工作区加载所有 .reqspec 文件，收集 SystemRequirementSet
  ├── loadVerificationPlans(IProject project)
  │     从工作区加载所有 .verify 文件，建立 Requirement→Claim 映射
  ├── loadAssuranceResults(IProject project)
  │     从工作区加载所有 .assure 文件，获取 ClaimResult 列表
  └── buildMatrix()
        输出 List<TraceabilityRow>，每行对应一个 (Requirement, Claim, Result) 三元组
```

#### 14.4.2 矩阵模型

```java
// 预期的 TraceabilityRow 数据结构
class TraceabilityRow {
    String requirementId;          // 来自 Requirement.name
    String requirementTitle;       // 来自 Requirement.title
    ComponentClassifier target;    // 来自 RequirementSet.target
    NamedElement targetElement;    // 来自 Requirement.targetElement
    List<Category> categories;     // 来自 Requirement.category
    String verificationMethod;     // 来自 VerificationActivity.method
    VerificationResult resultType; // 来自 VerificationActivityResult.resultType
    boolean covered;               // 是否存在对应的验证活动
}
```

#### 14.4.3 导出格式（预期支持）

基于 OSATE2 其他报告工具的惯例，RTM 工具预期支持以下导出格式：

| 格式 | 用途 |
|------|------|
| CSV  | 通用电子表格导入（Excel/LibreOffice） |
| HTML | 可在浏览器中直接查看的追踪表格 |
| Eclipse Marker | 在 IDE 中标注未覆盖的需求 |
| Markdown | 文档集成（与 ReqDocument 关联） |

### 14.5 与现有追踪机制的关系

`org.osate.reqtrace` 并不重新实现追踪链接，而是**消费并可视化**已有的追踪关系。ALISA 框架中现有的追踪机制包括：

| 追踪方式 | 定义位置 | reqtrace 的消费方式 |
|---------|---------|-------------------|
| `Requirement.targetElement` | `ReqSpec.xtext` | 矩阵"目标元素"列 |
| `Claim.requirement` | `Verify.xtext` | 建立需求→验证映射 |
| `VerificationActivityResult` | `Assure` 执行结果 | 填充"状态"列 |
| `Metrics.requirementCount` | `Metrics.java` | 计算需求覆盖率 |
| `Requirement.category` | `ReqSpec.xtext` | 支持按类别过滤矩阵 |
| `Requirement.refinesReference` | `ReqSpec.xtext` | 需求层级展开 |

### 14.6 模块在 ALISA 工作流中的位置

```
┌────────────────────────────────────────────────────────────────────┐
│                     ALISA 完整工作流                                │
│                                                                    │
│  1. [org.osate.reqspec]     编写需求规格 (.reqspec)                 │
│         ↓                                                          │
│  2. [org.osate.verify]      定义验证计划 (.verify)                  │
│         ↓                                                          │
│  3. [org.osate.alisa.workbench] 组织保证案例 (.alisa)               │
│         ↓                                                          │
│  4. [org.osate.assure]      执行验证，生成结果 (.assure)             │
│         ↓                                                          │
│  5. [org.osate.reqtrace]    ← 当前模块                             │
│         读取前四步的全部数据                                          │
│         生成需求追踪矩阵（RTM）                                       │
│         支持需求覆盖率报告和差距分析                                   │
└────────────────────────────────────────────────────────────────────┘
```

### 14.7 构建状态与版本信息

| 属性 | 值 |
|------|----|
| Maven GroupId | `org.osate` |
| Maven ArtifactId | `org.osate.reqtrace` |
| 当前版本 | `2.0.3-SNAPSHOT` |
| 父构建版本 | ALISA 2.11.0.vfinal（pom.xml 中引用） |
| 打包类型 | `eclipse-plugin` |
| 在父 POM 中的状态 | **未激活**（`alisa/pom.xml` modules 列表中未包含） |
| 源文件状态 | 仅 `pom.xml`，无 Java 源文件 |
| 实际源码路径 | `/home/hlt/project/explore_osate/osate2/alisa/org.osate.reqtrace/pom.xml` |

这一状态表明 `org.osate.reqtrace` 是已规划但尚未在本代码分支实现的功能模块，其在 ALISA 体系中的数据来源和集成点已由 `reqspec`、`verify`、`assure` 三个模块的成熟接口明确定义，实现上有清晰的路径可循。
