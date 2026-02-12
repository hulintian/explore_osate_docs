# Xtext 语言集成设计

本文档基于源码分析，描述 OSATE2 如何使用 Xtext 框架实现 AADL2 语言支持。

## 1. 非标准 Xtext 架构

OSATE2 的 Xtext 用法与典型项目有重大差异：**它使用预先存在的 Ecore 元模型**，而不是从语法生成元模型。

```
典型 Xtext：Grammar.xtext → 自动生成 → model.ecore → 生成 Java API
OSATE2：    aadl2.ecore (预先手工设计) ← 语法映射 ← Aadl2.xtext
```

关键语句：
```xtext
grammar org.osate.xtext.aadl2.Aadl2 with org.osate.xtext.aadl2.properties.Properties
import "http://aadl.info/AADL/2.0" as aadl2    // 引用外部 Ecore，不生成
```

## 2. 语法结构

### 2.1 双语法层次

**Properties.xtext**（基础层）
- 位置：`core/org.osate.xtext.aadl2.properties/src/.../Properties.xtext`
- 定义属性相关规则：PropertyAssociation, PropertyExpression, 各种值类型
- 隐藏元素：空白和单行注释

**Aadl2.xtext**（继承 Properties）
- 位置：`core/org.osate.xtext.aadl2/src/.../Aadl2.xtext`
- 约 2512 行
- 定义完整 AADL2 语法

### 2.2 核心语法规则

```xtext
Model returns aadl2::ModelUnit:        // 入口规则
    AadlPackage | PropertySet;

AadlPackage returns aadl2::AadlPackage:
    'package' name=PNAME ...;

PublicPackageSection / PrivatePackageSection:
    ('with' importedUnit+=[aadl2::ModelUnit|PNAME] ...)  // 包导入
    (ownedClassifier+=Classifier ...)*;

ComponentType (抽象):
    AbstractType | SystemType | ProcessType | ThreadType | ...
    每个类型有独立规则，均返回对应 Ecore 类

ComponentImplementation:
    AbstractImplementation | SystemImplementation | ...
```

### 2.3 特殊终端符号

| Token | 用途 |
|-------|------|
| `PNAME` | 限定包名 (package::name) |
| `QCREF` | 限定组件引用 |
| `QPREF` | 限定属性引用 |
| `ANNEXTEXT` | Annex 原始文本内容 |
| `REFINEDNAME` | 精化名称引用 |

## 3. 作用域（Scoping）策略

OSATE2 使用多层作用域架构进行名称解析。

### 3.1 作用域层次

```
EClassGlobalScopeProvider (全局作用域)
    ↓
Aadl2ImportedNamespaceAwareLocalScopeProvider (导入的命名空间)
    ↓
Aadl2ScopeProvider (声明式局部作用域)
    ↓ extends
PropertiesScopeProvider (属性作用域)
```

### 3.2 声明式作用域方法

`Aadl2ScopeProvider` 使用 `scope_<EClass>_<EReference>` 命名模式：

| 方法 | 解析内容 |
|------|---------|
| `scope_TypeExtension_extended()` | 类型扩展的基类型 |
| `scope_ComponentImplementation()` | 分类器作用域 |
| `scope_FeatureGroupType_inverse()` | 逆特征组类型 |
| `scope_SubprogramCall_calledSubprogram()` | 子程序调用目标 |
| `scope_Subcomponent_subcomponentType()` | 子组件类型 |

### 3.3 全局作用域

`EClassGlobalScopeProvider`（在 `org.osate.aadl2.modelsupport.scoping` 中）：
- 提供 EClass 感知的全局元素搜索
- 接口：`IEClassGlobalScopeProvider`
- 支持按 EClass 过滤的全局查找

### 3.4 导入命名空间处理

`Aadl2ImportedNamespaceAwareLocalScopeProvider`：
- 处理 `with` 子句的包导入解析
- 绑定为 `AbstractDeclarativeScopeProvider.NAMED_DELEGATE`

### 3.5 关键作用域模式

- **Prototype 作用域**：解析原型及其约束
- **Refinement 感知**：过滤精化/非精化元素（`filterRefined` 扩展）
- **层次作用域**：处理嵌套组件层次
- **模式特定作用域**：约束元素到特定操作模式
- **包重命名**：支持导入分类器的本地重命名

## 4. 链接服务

### 4.1 Aadl2LinkingService

位置：`core/org.osate.xtext.aadl2/src/.../linking/Aadl2LinkingService.java`

继承 `PropertiesLinkingService`，处理复杂交叉引用：
- 类型层次解析（原型精化）
- 特征/连接端点验证
- 子程序调用上下文解析
- 流规格元素链接
- Annex 链接服务集成

### 4.2 命名提供器

**Aadl2QualifiedNameProvider**：
- 自定义限定名生成用于索引
- 处理 Annex 元素命名（委托给注册的 Annex 链接服务）
- 为 AadlPackage, Classifier, Property, PropertySet 等生成限定名

**Aadl2QualifiedNameFragmentProvider**：
- 基于限定名的 URI 片段构建
- 支持健壮的跨文件引用解析

## 5. 验证架构

### 5.1 Aadl2Validator

使用 `@Check` 注解的声明式验证，自定义错误码：

| 错误码 | 检查内容 |
|--------|---------|
| `MISMATCHED_BEGINNING_AND_ENDING_IDENTIFIERS` | 分类器/包首尾标识符匹配 |
| `DUPLICATE_COMPONENT_TYPE_NAME` | 类型名唯一性 |
| `DUPLICATE_LITERAL_IN_ENUMERATION` | 枚举值唯一性 |
| `MODE_NOT_DEFINED_IN_CONTAINER` | 模式引用有效性 |
| `SELF_NOT_ALLOWED` | self 引用合法性 |

### 5.2 其他验证器

- **Aadl2NamesAreUniqueValidationHelper**：命名唯一性检查
- **Aadl2ConcreteSyntaxValidator**：仅对 aadl2 和 EMV2 包进行语法约束验证

## 6. RuntimeModule 自定义绑定

`Aadl2RuntimeModule` 是 Guice 模块，定义了大量自定义绑定：

```java
// 链接与引用
bindILinkingService()      → Aadl2LinkingService
bindILinker()              → AnnexParserAgent (特殊！处理 Annex 解析)

// 命名
bindIQualifiedNameProvider()  → Aadl2QualifiedNameProvider
bindIQualifiedNameConverter() → Aadl2QualifiedNameConverter
bindIFragmentProvider()       → Aadl2QualifiedNameFragmentProvider

// 作用域
bindIScopeProvider()        → Aadl2ScopeProvider
configureDelegate()         → Aadl2ImportedNamespaceAwareLocalScopeProvider
bindIGlobalScopeProvider()  → EClassGlobalScopeProvider

// 资源处理
bindXtextResource()               → NoCacheDerivedStateAwareResource
bindIDerivedStateComputer()       → Aadl2DerivedStateComputer
bindIResourceStorageFacade()      → Aadl2ResourceStorageFacade

// 序列化
bindICrossReferenceSerializer()   → Aadl2CrossReferenceSerializer
bindITransientValueService()      → Aadl2TransientValueService

// 文档
bindIEObjectDocumentationProvider() → Aadl2DocumentationProvider
```

### 6.1 关键技术决策

**禁用索引片段的延迟链接**：
```java
configureUseIndexFragmentsForLazyLinking(binder) {
    binder.bind(Boolean.TYPE)
        .annotatedWith(Names.named(USE_INDEXED_FRAGMENTS_BINDING))
        .toInstance(Boolean.FALSE);
}
```
原因：索引片段会破坏 OSATE 中的代理解析。

## 7. Annex 子语言集成

### 7.1 Annex 语法规则

```xtext
AnnexLibrary returns aadl2::AnnexLibrary:
    DefaultAnnexLibrary;

DefaultAnnexLibrary returns aadl2::DefaultAnnexLibrary:
    'annex' name=ID sourceText=ANNEXTEXT ';';

DefaultAnnexSubclause returns aadl2::DefaultAnnexSubclause:
    'annex' name=ID sourceText=ANNEXTEXT
    (InModesKeywords '(' inMode+=[aadl2::Mode|ID] ... ')')?  ';';
```

### 7.2 AnnexParserAgent（关键集成点）

位置：`core/org.osate.xtext.aadl2/src/.../parsing/AnnexParserAgent.java`

**绑定为 `ILinker`**（替换标准链接器），职责：
1. 捕获原始 ANNEXTEXT token 内容
2. 模型链接后调用注册的 Annex 解析器（通过 `AnnexParserRegistry`）
3. 将 Annex 解析为 EMF 模型
4. 调用 Annex 链接服务（`AnnexLinkingServiceRegistry`）
5. 调用 Annex 验证器（`AnnexValidatorRegistry`）

### 7.3 Annex 注册表体系

位于 `core/org.osate.annexsupport/`：

| 注册表 | 扩展点 | 用途 |
|--------|--------|------|
| `AnnexParserRegistry` | annexParser | 文本→AST |
| `AnnexLinkingServiceRegistry` | annexLinkingService | 交叉引用解析 |
| `AnnexResolverRegistry` | annexResolver | 引用解析 |
| `AnnexValidatorRegistry` | annexValidator | 语义验证 |
| `AnnexUnparserRegistry` | annexUnparser | AST→文本 |
| `AnnexInstantiatorRegistry` | annexInstantiator | 实例化 |
| `AnnexHighlighterRegistry` | annexHighlighter | 语法高亮 |
| `AnnexContentAssistRegistry` | annexContentAssist | 内容辅助 |
| `AnnexTextPositionResolverRegistry` | annexTextPositionResolver | 位置映射 |

所有注册表继承自 `AnnexRegistry` 抽象类，通过 OSGi 扩展点发现。

### 7.4 EMV2 集成示例

EMV2 作为 Annex 子语言的集成路径：
```
AADL 文本中 'annex error_model {** ... **}'
  → AnnexParserAgent 捕获 ANNEXTEXT
  → AnnexParserRegistry 查找 "error_model" 解析器
  → EMV2AnnexParser 解析为 ErrorModel EMF 模型
  → EMV2AnnexLinkingService 解析引用
  → EMV2AnnexValidator 验证语义
```

## 8. UI 层集成

### 8.1 内容辅助

`AnnexAwareContentAssistProcessor`：
- 扩展 `XtextContentAssistProcessor`
- 检测光标是否在 Annex 区域
- 如在 Annex 中，委托给注册的 Annex 内容辅助提供器

### 8.2 格式化

`Aadl2Formatter`（Xtend，约 122KB）：
- 使用 Xtext Formatting2 API
- 声明式格式化规则

### 8.3 快速修复

`Aadl2QuickfixProvider`：
- 提供常见错误的自动修复
- 例如：匹配首尾标识符、创建缺失的类型/实现

## 9. 架构总结

```
用户编辑 AADL 文本
    ↓
Xtext Parser (Aadl2.xtext / Properties.xtext)
    ↓
AST → Ecore 模型 (aadl2.ecore 中的类)
    ↓
AnnexParserAgent 检测 Annex 块
    ├→ EMV2AnnexParser → ErrorModel Ecore
    ├→ BAAnnexParser → BehaviorAnnex Ecore
    └→ 其他注册的 Annex 解析器
    ↓
Aadl2LinkingService 解析交叉引用
    ↓
Aadl2ScopeProvider 提供名称作用域
    ↓
Aadl2Validator 执行语义验证
    ↓
Aadl2Formatter / ContentAssist / Quickfix (IDE 功能)
```
