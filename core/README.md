# OSATE2 Core 模块深度架构分析

> 基于源码实际阅读的架构设计文档
> 源码位置：`/home/hlt/project/explore_osate/osate2/core/`

---

## 目录

1. [模块概览与整体架构](#1-模块概览与整体架构)
2. [org.osate.annexsupport — Annex扩展点体系](#2-orgosateannexsupport--annex扩展点体系)
   - 2.1 [扩展点注册表架构](#21-扩展点注册表架构)
   - 2.2 [核心接口族](#22-核心接口族)
   - 2.3 [代理模式与懒加载](#23-代理模式与懒加载)
   - 2.4 [headless模式支持](#24-headless模式支持)
   - 2.5 [AnnexParseUtil：解析坐标映射](#25-annexparseutil解析坐标映射)
   - 2.6 [AnnexValidator：委托验证器](#26-annexvalidator委托验证器)
3. [org.osate.pluginsupport — 插件贡献与AADL资源管理](#3-orgosatepluginsupport--插件贡献与aadl资源管理)
   - 3.1 [AADL贡献扩展点](#31-aadl贡献扩展点)
   - 3.2 [PredeclaredProperties：工作空间覆盖机制](#32-predeclaredproperties工作空间覆盖机制)
   - 3.3 [ExecuteJavaUtil：反射调用Java方法](#33-executejavautil反射调用java方法)
   - 3.4 [ScopeFunctions：Xtend风格的Java替代API](#34-scopefunctionsxtend风格的java替代api)
4. [org.osate.contribution.sei — SEI标准属性集](#4-orgosatecontributionsei--sei标准属性集)
   - 4.1 [属性集注册机制](#41-属性集注册机制)
   - 4.2 [SEI.aadl：资源预算与分析属性集](#42-seiaadl资源预算与分析属性集)
   - 4.3 [Data_Model.aadl：数据建模属性集](#43-data_modelaadl数据建模属性集)
   - 4.4 [其他属性集和包](#44-其他属性集和包)
5. [org.osate.slicer — 模型切片工具](#5-orgosateslicer--模型切片工具)
   - 5.1 [设计动机与核心思想](#51-设计动机与核心思想)
   - 5.2 [OsateSlicerVertex：图顶点类型](#52-osateslicervertex图顶点类型)
   - 5.3 [SlicerRepresentation：图构建与切片算法](#53-slicerrepresentation图构建与切片算法)
   - 5.4 [前向与后向可达性](#54-前向与后向可达性)
   - 5.5 [UnusedElementDetector：假设检查](#55-unusedelementdetector假设检查)
   - 5.6 [与EMV2的集成](#56-与emv2的集成)
6. [org.osate.resolute — Resolute验证语言集成](#6-orgosateresolute--resolute验证语言集成)
   - 6.1 [设计模式：运行时桥接](#61-设计模式运行时桥接)
   - 6.2 [ResoluteAccess接口](#62-resoluteaccess接口)
   - 6.3 [与org.osate.result的关联](#63-与orgosateresult的关联)
7. [org.osate.results — 统一分析结果模型](#7-orgosateresults--统一分析结果模型)
   - 7.1 [EMF元模型结构](#71-emf元模型结构)
   - 7.2 [AnalysisResult：顶层结果容器](#72-analysisresult顶层结果容器)
   - 7.3 [Result：分层结果节点](#73-result分层结果节点)
   - 7.4 [Diagnostic与DiagnosticType](#74-diagnostic与diagnostictype)
   - 7.5 [Value类型层次](#75-value类型层次)
8. [org.osate.aadl2.modelsupport — 错误报告基础设施与模型遍历](#8-orgosateaadl2modelsupport--错误报告基础设施与模型遍历)
   - 8.1 [错误报告器层次架构](#81-错误报告器层次架构)
   - 8.2 [AnalysisErrorReporterManager：跨资源管理](#82-analysiserrorreportermanager跨资源管理)
   - 8.3 [MarkerAnalysisErrorReporter：Eclipse标记集成](#83-markeranalysiserrorreportereclipse标记集成)
   - 8.4 [模型遍历框架](#84-模型遍历框架)
9. [org.osate.aadl2.instance.textual — 实例模型文本语法](#9-orgosateaadl2instancetextual--实例模型文本语法)
   - 9.1 [Xtext语法架构](#91-xtext语法架构)
   - 9.2 [核心语法规则](#92-核心语法规则)
   - 9.3 [跨资源引用解析](#93-跨资源引用解析)
10. [ease-scripts — EASE自动化脚本](#10-ease-scripts--ease自动化脚本)
11. [模块间依赖关系图](#11-模块间依赖关系图)
12. [关键设计模式总结](#12-关键设计模式总结)

---

## 1. 模块概览与整体架构

Core 模块是 OSATE2 的核心基础模块，包含 38 个子模块（截至 2.18.0-SNAPSHOT 版本）。整体架构分为以下几层：

```
┌──────────────────────────────────────────────────────────────┐
│                    工具层 / 分析层                             │
│  org.osate.slicer  org.osate.resolute  org.osate.results      │
├──────────────────────────────────────────────────────────────┤
│                    语言支持层                                   │
│  org.osate.aadl2.instance.textual                             │
│  org.osate.xtext.aadl2  org.osate.xtext.aadl2.properties     │
├──────────────────────────────────────────────────────────────┤
│                    扩展点/插件基础设施层                         │
│  org.osate.annexsupport  org.osate.pluginsupport              │
├──────────────────────────────────────────────────────────────┤
│                    模型支持层                                   │
│  org.osate.aadl2.modelsupport                                 │
├──────────────────────────────────────────────────────────────┤
│                    AADL元模型层                                 │
│  org.osate.aadl2  org.osate.aadl2.instance                    │
├──────────────────────────────────────────────────────────────┤
│                    属性集贡献层                                  │
│  org.osate.contribution.sei  org.osate.aadl2.contrib          │
└──────────────────────────────────────────────────────────────┘
```

所有模块均遵循 Eclipse OSGi 插件规范，通过 `plugin.xml` 声明扩展点和扩展。

---

## 2. org.osate.annexsupport — Annex扩展点体系

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.annexsupport/src/org/osate/annexsupport/`

**插件ID**：`org.osate.annexsupport`

### 2.1 扩展点注册表架构

Annex（附件）是 AADL 语言的扩展机制，允许第三方语言（如 EMV2、BA、AGREE 等）嵌入到 AADL 组件的 `annex` 块中。`org.osate.annexsupport` 模块为此提供了完整的 Eclipse 扩展点框架。

**`plugin.xml` 声明的扩展点（8个）**：

```xml
<extension-point id="parser"              name="Annex Parser"          schema="schema/annexparser.exsd"/>
<extension-point id="unparser"            name="Annex Unparser"        schema="schema/annexunparser.exsd"/>
<extension-point id="resolver"            name="Annex Resolver"        schema="schema/annexresolver.exsd"/>
<extension-point id="linkingservice"      name="Annex Linking Service" schema="schema/annexlinkingservice.exsd"/>
<extension-point id="textpositionresolver" name="Text Position Resolver" .../>
<extension-point id="instantiator"        name="Annex Instantiator"    schema="schema/annexinstantiator.exsd"/>
<extension-point id="highlighter"         name="Annex Highlighter"     schema="schema/annexhighlighter.exsd"/>
<extension-point id="contentassist"       name="Content Assist"        schema="schema/contentassist.exsd"/>
```

同时，`plugin.xml` 自身也注册了通配符默认实现：

```xml
<extension point="org.osate.annexsupport.parser">
  <parser annexName="*"
          class="org.osate.annexsupport.DefaultAnnexParser"
          id="org.osate.annexsupport.DefaultAnnexParser"
          name="Built-in default annex parser"/>
</extension>
```

**`AnnexRegistry` 类**（`AnnexRegistry.java`）是整个注册表体系的核心，定义了 8 个扩展点的 ID 常量：

```java
public static final String ANNEX_PARSER_EXT_ID                = "parser";
public static final String ANNEX_UNPARSER_EXT_ID              = "unparser";
public static final String ANNEX_RESOLVER_EXT_ID              = "resolver";
public static final String ANNEX_LINKINGSERVICE_EXT_ID        = "linkingservice";
public static final String ANNEX_TEXTPOSITIONRESOLVER_EXT_ID  = "textpositionresolver";
public static final String ANNEX_INSTANTIATOR_EXT_ID          = "instantiator";
public static final String ANNEX_HIGHLIGHTER_EXT_ID           = "highlighter";
public static final String ANNEX_CONTENT_ASSIST_EXT_ID        = "contentassist";
```

`AnnexRegistry` 是抽象类，通过工厂方法 `createRegistry(String extensionId)` 创建对应的具体注册表实例：

```java
protected static AnnexRegistry createRegistry(String extensionId) {
    if (extensionId.equalsIgnoreCase(ANNEX_PARSER_EXT_ID)) {
        return new AnnexParserRegistry();
    } else if (extensionId.equalsIgnoreCase(ANNEX_UNPARSER_EXT_ID)) {
        return new AnnexUnparserRegistry();
    } // ... 以此类推
}
```

全局共享注册表通过 `Map<String, AnnexRegistry> registries` 静态字段缓存，按需惰性创建。

### 2.2 核心接口族

每个扩展点都对应一个 Java 接口：

**`AnnexParser` 接口**（`AnnexParser.java`）：

```java
public interface AnnexParser {
    // 解析 annex library：出现在 AADL package section 中
    AnnexLibrary parseAnnexLibrary(String annexName, String source, String filename,
                                   int line, int column, ParseErrorReporter errReporter)
        throws RecognitionException;

    // 解析 annex subclause：出现在 AADL classifier 中
    AnnexSubclause parseAnnexSubclause(String annexName, String source, String filename,
                                       int line, int column, ParseErrorReporter errReporter)
        throws RecognitionException;

    // 返回 Xtext 文件扩展名（可选，since 3.0）
    default String getFileExtension() { return null; }
}
```

关键设计点：
- 解析方法接受 `line` 和 `column` 参数，用于在主 AADL 文件中精确定位 annex 文本的起始位置。这与 `AnnexParseUtil` 中的白空格填充机制配合，确保错误行号映射到原始 AADL 文件而非 annex 子文本。
- 返回类型分别是 `AnnexLibrary` 和 `AnnexSubclause`，这两个类定义在 `org.osate.aadl2` 的 EMF 元模型中。
- 错误报告通过 `ParseErrorReporter` 接口传入，与具体的错误展示实现解耦。

**`AnnexUnparser` 接口**（`AnnexUnparser.java`）：

```java
public interface AnnexUnparser {
    String unparseAnnexLibrary(AnnexLibrary library, String indent);
    String unparseAnnexSubclause(AnnexSubclause subclause, String indent);
}
```

将已解析的 annex EMF 模型对象反序列化回文本形式，用于模型保存和代码生成场景。

**`AnnexResolver` 接口**（`AnnexResolver.java`）：

```java
public interface AnnexResolver {
    void resolveAnnex(String annexName, List<?> annexElements, AnalysisErrorReporterManager errManager);
}
```

在 AADL 主模型解析完成后调用，用于解析 annex 内部的交叉引用（例如，EMV2 annex 中对 AADL 错误类型库的引用）。

**`AnnexInstantiator` 接口**（`AnnexInstantiator.java`）：

```java
public interface AnnexInstantiator {
    // 对每个组件实例逐一调用
    void instantiateAnnex(ComponentInstance instance, String annexName,
                          AnalysisErrorReporterManager errorManager);  // since 4.0

    // 在整个系统实例创建完毕后调用一次
    void instantiateAnnex(SystemInstance instance, String annexName,
                          AnalysisErrorReporterManager errorManager);  // since 4.0
}
```

这是将 annex 信息从声明式 AADL 模型"实例化"到 `SystemInstance` 的关键钩子。EMV2 实例化器正是通过实现此接口，在每个 `ComponentInstance` 上附加错误传播信息。

**`AnnexHighlighter` 接口**（`AnnexHighlighter.java`）：

```java
public interface AnnexHighlighter {
    void highlightAnnexLibrary(AnnexLibrary library, AnnexHighlighterPositionAcceptor acceptor);
    void highlightAnnexSubclause(AnnexSubclause subclause, AnnexHighlighterPositionAcceptor acceptor);
}
```

用于在 AADL 文本编辑器中对 annex 内容进行语法高亮渲染。`AnnexHighlighterPositionAcceptor` 是位置信息收集器接口，编辑器通过它接收需要高亮的区间与样式。

### 2.3 代理模式与懒加载

每种注册表类型都实现了对应的 `AnnexXxxProxy` 代理类（如 `AnnexParserProxy`、`AnnexUnparserProxy` 等）。代理类继承自抽象基类 `AnnexProxy`：

```java
abstract public class AnnexProxy {
    protected static final String ATT_ID        = "id";
    protected static final String ATT_NAME      = "name";
    protected static final String ATT_ANNEXNAME = "annexName";
    protected static final String ATT_ANNEXNSURI= "annexNSURI";
    protected static final String ATT_CLASS     = "class";

    protected final IConfigurationElement configElem;
    protected final String id;
    protected final String name;
    protected final String annexName;
    protected final String className;

    // 从 Eclipse 配置元素构造（Eclipse 模式）
    protected AnnexProxy(IConfigurationElement configElem) { ... }

    // 手工构造（standalone 模式）
    protected AnnexProxy(String id, String name, String annexName, String className) { ... }
}
```

代理类持有 `IConfigurationElement`，实际的 Java 类仅在首次被调用时通过 `configElem.createExecutableExtension("class")` 懒加载实例化，避免在 Eclipse 启动阶段加载所有 Annex 实现类。

### 2.4 headless模式支持

`AnnexRegistry` 的 `initialize()` 方法中有重要的分支逻辑：

```java
protected void initialize(String extensionId) {
    extensions = new HashMap<String, Object>();
    boolean hasExtensionPoints = false;
    final IExtensionRegistry extensionRegistry = Platform.getExtensionRegistry();
    if (extensionRegistry != null) {
        // Eclipse 模式：从 Eclipse 扩展注册表读取
        IExtensionPoint extensionPoint =
            extensionRegistry.getExtensionPoint(AnnexPlugin.PLUGIN_ID, extensionId);
        if (extensionPoint != null) {
            hasExtensionPoints = true;
            // ... 读取并注册所有已声明的扩展
        }
    }
    if (!hasExtensionPoints) {
        // headless 模式：仅注册默认处理器
        final Object defaultHandler = getDefault();
        if (defaultHandler != null) {
            extensions.put("*", defaultHandler);
        }
    }
}
```

对于 headless（无 Eclipse）应用，还提供了 `registerAnnex()` 静态工厂方法：

```java
public static void registerAnnex(final String annexName,
    final AnnexParser parser, final AnnexUnparser unparser,
    final AnnexLinkingService linkingService, final AnnexContentAssist contextAssist,
    final AnnexHighlighter highlighter, final AnnexInstantiator instantiator,
    final AnnexResolver resolver, final AnnexTextPositionResolver textPositionResolver) { ... }
```

此方法允许 standalone 应用程序（如批处理分析工具）以编程方式一次性注册某个 annex 的所有组件，而无需 Eclipse 平台支持。

### 2.5 AnnexParseUtil：解析坐标映射

`AnnexParseUtil.java` 提供了一个关键的静态工具方法 `parse()`，处理 annex 文本在主 AADL 文件中的位置偏移问题：

```java
public static EObject parse(AbstractAntlrParser parser, String editString,
        ParserRule parserRule, String filename, int line, int offset,
        ParseErrorReporter err) {
    // 在文本前填充与原始文件匹配的空白字符，使错误行号自动对齐
    editString = genWhitespace(line, offset) + editString;
    IParseResult parseResult = parser.parse(parserRule, new StringReader(editString));
    // ...
}
```

`genWhitespace(int line, int length)` 方法生成由换行符和空格组成的填充前缀，使得 annex 子解析器产生的错误行号与原始 AADL 文件中的行号直接对应，无需额外的坐标转换逻辑。

解析结果通过 `ParseResultHolder`（一个 EMF Adapter）附加到解析所得的 AST 根节点上，用于后续的语法高亮、悬停提示等 IDE 功能。

### 2.6 AnnexValidator：委托验证器

`AnnexValidator.java` 实现了 `EObjectValidator`，通过 `NO_VALIDATION_ADAPTER` 机制阻止对"原始" DefaultAnnexSection 的重复验证：

```java
public class AnnexValidator extends EObjectValidator {
    private static final Adapter NO_VALIDATION_ADAPTER = new NoValidationAdaper();

    @Override
    public boolean validate(EObject eObject, DiagnosticChain diagnostics, Map<Object, Object> context) {
        if (!isValidating(eObject)) {
            return true;  // 跳过已被替换为具体 annex 模型的默认容器
        }
        return delegateValidator.validate(eObject, diagnostics, context);
    }
}
```

当 annex 被成功解析后，`setNoValidation()` 静态方法将 `NO_VALIDATION_ADAPTER` 附加到默认 annex 节点，并将 annex 的具体 `EValidator` 注册到 `EValidator.Registry`，从而将验证职责转交给 annex 自己的验证器。

---

## 3. org.osate.pluginsupport — 插件贡献与AADL资源管理

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.pluginsupport/src/org/osate/pluginsupport/`

**插件ID**：`org.osate.pluginsupport`

### 3.1 AADL贡献扩展点

`plugin.xml` 定义了两个核心扩展点：

```xml
<!-- 允许插件贡献 AADL Package 或 Property Set 文件 -->
<extension-point id="aadlcontribution"
                 name="AADL Package / Property Set Contributor"
                 schema="schema/aadlcontribution.exsd"/>

<!-- 允许插件将 Java 类注册到 OSATE 系统 -->
<extension-point id="registeredjavaclasses"
                 name="Register Java Class"
                 schema="schema/registeredjavaclasses.exsd"/>
```

`aadlcontribution` 扩展点是 OSATE 插件分发标准属性集的标准机制。各贡献者在 `plugin.xml` 中声明 `<aadlcontribution file="...">` 元素，指向插件内的 `.aadl` 文件路径。

`PluginSupportUtil.getContributedAadl()` 静态方法读取所有注册的贡献并返回 EMF URI 列表：

```java
public static List<URI> getContributedAadl() {
    return getContributedAadl(configElem -> {
        String path = configElem.getAttribute("file");
        String fullpath = configElem.getDeclaringExtension().getContributor().getName()
                + (path.charAt(0) == '/' ? "" : "/") + path;
        return URI.createPlatformPluginURI(fullpath, false);
    });
}
```

此方法返回形如 `platform:/plugin/org.osate.contribution.sei/resources/properties/SEI.aadl` 的 URI，这些 URI 在 AADL 构建过程中被自动加入到 Xtext 资源集，使所有项目都能"看到"这些预定义属性集。

`getContributedPropertySets()` 方法则通过轻量级文本扫描（不做完整解析）区分出属性集（`property set`）和包（`package`），以字符串形式返回属性集名称：

```java
public static Map<URI, String> getContributedPropertySets() {
    // 扫描文件首行，查找 "property set <name>" 模式
    // 跳过注释（以 "--" 开头的行）
    // 返回 URI -> 属性集名称的映射
}
```

### 3.2 PredeclaredProperties：工作空间覆盖机制

`PredeclaredProperties.java` 实现了一个重要功能：允许工作空间中的 AADL 文件**覆盖**插件提供的预定义属性集，用于调试和自定义。这一机制通过 Eclipse 偏好存储（`IPreferenceStore`）持久化映射关系：

```java
// 键：被覆盖的插件资源 URI（contributed.resource.override.key.N）
// 值：工作空间中的覆盖资源 URI（contributed.resource.override.value.N）
private static final String WORKSPACE_OVERRIDE_KEY_PREFIX   = "contributed.resource.override.key.";
private static final String WORKSPACE_OVERRIDE_VALUE_PREFIX = "contributed.resource.override.value.";

// 禁用特定贡献资源（不参与构建）
private static final String WORKSPACE_DISABLED_CONTRIBUTIONS_VALUE_PREFIX = "contributed.resource.disabled.";
```

`getEffectiveContributedResources()` 方法返回经过覆盖处理后的有效资源列表（禁用的项被排除，覆盖的项用工作空间版本替换）。

覆盖关系变化时，会触发 `closeAndReopenProjects()` 方法，通过关闭再重新打开所有工作空间项目来强制触发 Xtext 全量重新构建，以更新 AADL 范围和索引。

**线程安全**：`isChanged`、`contributedResources`、`effectiveContributedResources` 等字段使用 `volatile` 修饰，`buildContributedResources()` 和 `setDisabledContributions()` 方法加 `synchronized`，保证在 UI 线程修改偏好和构建器线程读取资源时的内存可见性。

### 3.3 ExecuteJavaUtil：反射调用Java方法

`ExecuteJavaUtil.java` 允许通过 `org.osate.pluginsupport.registeredjavaclasses` 扩展点以字符串形式动态调用已注册的 Java 类方法：

```java
public final class ExecuteJavaUtil {
    private final static String EXTENSION_POINT_ID = "org.osate.pluginsupport.registeredjavaclasses";

    // 调用以 EObject 为单一参数的方法
    public static Object invokeJavaMethod(String qualifiedMethodName, EObject argument) {
        return invokeJavaMethod(qualifiedMethodName,
            new Class[] { EObject.class }, new Object[] { argument });
    }

    // 通用版本：支持任意参数类型
    public static Object invokeJavaMethod(String qualifiedMethodName,
            Class<?>[] parameterTypes, Object[] arguments) {
        // 分割 "com.example.MyClass.myMethod" 为类名和方法名
        Pair<String, String> names = splitClassAndMethodName(qualifiedMethodName);
        // 从扩展注册表获取类实例（缓存在 configElementCache 中）
        Object instance = getInstance(className);
        return instance.getClass().getMethod(methodName, parameterTypes).invoke(instance, arguments);
    }
}
```

此工具主要被 ALISA（Assurance Cases）框架用来调用在 GSNS（Goal-based Safety Net Specification）验证规则中引用的 Java 方法，实现声明式验证语言与具体 Java 分析实现之间的桥接。

### 3.4 ScopeFunctions：Xtend风格的Java替代API

`ScopeFunctions.java`（since 7.2）提供了 Kotlin/Xtend 风格的 `with()` 方法，作为 Xtend 的 `=>` 操作符的 Java 替代：

```java
public static <T> T with(T object, Consumer<T> function) {
    function.accept(object);
    return object;
}
```

典型用法是在测试代码和 SWT UI 代码中构建与对象层次结构对齐的代码块：

```java
// 替代 Xtend 的 => 写法
with((SystemImplementation) pkg.getPublicSection().getOwnedClassifiers().get(0), system -> {
    assertEquals("s.i", system.getName());
    with(system.getOwnedSubcomponents().get(0), subcomponent -> {
        assertEquals("sub", subcomponent.getName());
    });
});
```

这是 OSATE2 代码库从 Xtend 逐步迁移到纯 Java 的关键工具类之一。

---

## 4. org.osate.contribution.sei — SEI标准属性集

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.contribution.sei/`

### 4.1 属性集注册机制

`plugin.xml` 使用 `aadlcontribution` 扩展点注册了 8 个 AADL 文件：

```xml
<extension point="org.osate.pluginsupport.aadlcontribution">
  <!-- SEI 私有属性集 -->
  <aadlcontribution file="resources/properties/SEI.aadl"
                    id="org.osate.contributes.SEI.aadlcontribution1"/>
  <aadlcontribution file="resources/properties/Physical.aadl"
                    id="org.osate.contributes.SEI.aadlcontribution2"/>
  <aadlcontribution file="resources/packages/PhysicalResources.aadl"
                    id="org.osate.contributes.SEI.aadlcontribution3"/>
  <!-- SAE 标准属性集 -->
  <aadlcontribution file="resources/properties/Data_Model.aadl"
                    id="org.osate.contributes.SAE.aadlcontribution2"/>
  <aadlcontribution file="resources/packages/Base_Types.aadl"
                    id="org.osate.contributes.SAE.aadlcontribution3"/>
  <aadlcontribution file="resources/properties/ARINC653.aadl"
                    id="org.osate.contributes.SAE.aadlcontribution4"/>
  <aadlcontribution file="resources/properties/ARINC429.aadl"
                    id="org.osate.contributes.SAE.aadlcontribution5"/>
  <aadlcontribution file="resources/properties/Code_Generation_Properties.aadl"
                    id="org.osate.contributes.SAE.aadlcontribution6"/>
</extension>
```

### 4.2 SEI.aadl：资源预算与分析属性集

文件路径：`resources/properties/SEI.aadl`

`@codegen-package org.osate.contribution.sei.sei` 注解表明此属性集有对应的代码生成版本（在 `src-gen` 目录下）。

**主要属性分类**：

**安全性与可靠性属性**：
```aadl
SecurityLevel   : inherit aadlinteger applies to (thread, thread group, process, system, virtual processor);
SafetyCriticality: aadlinteger applies to (thread, thread group, process, system, virtual processor);
```

**重量分析属性**（用于物理系统建模）：
```aadl
NetWeight    : aadlreal units SEI::WeightUnits applies to (system, processor, memory, bus, device, access connection);
GrossWeight  : aadlreal units SEI::WeightUnits applies to (...);
WeightLimit  : aadlreal units SEI::WeightUnits applies to (...);
WeightUnits  : type units (kg);
```

**资源预算属性**（用于资源分配分析）：
```aadl
MIPSCapacity    : aadlreal units SEI::Processor_Speed_Units applies to (processor, system);
MIPSBudget      : aadlreal units SEI::Processor_Speed_Units applies to (thread, thread group, process, system, device, virtual processor);
RAMCapacity     : aadlreal units Size_Units applies to (memory, system, processor, virtual processor, abstract);
RAMBudget       : aadlreal units Size_Units applies to (thread, thread group, process, system, device);
ROMCapacity     : aadlreal units Size_Units applies to (memory, system, processor, virtual processor, abstract);
ROMBudget       : aadlreal units Size_Units applies to (thread, thread group, process, system, device);
PowerCapacity   : aadlreal units SEI::Power_Units applies to (bus, system, device, abstract);
PowerBudget     : aadlreal units SEI::Power_Units applies to (feature);
PowerSupply     : aadlreal units SEI::Power_Units applies to (feature);
BandWidthCapacity: aadlreal units Data_Rate_Units applies to (bus, virtual bus, system);
BandWidthBudget  : aadlreal units Data_Rate_Units applies to (connection, bus, virtual bus);
```

**单位类型定义**：
```aadl
Processor_Speed_Units: type units (KIPS, MIPS => KIPS * 1000, GIPS => MIPS * 1000);
Power_Units          : type units (mW, W => mW * 1000, KW => W * 1000);
InstructionVolumeUnits: type units (IPD, KIPD => IPD * 1000, MIPD => KIPD * 1000);
```

**分区与延迟属性**（用于分区操作系统建模）：
```aadl
Partition_Latency : Time applies to (system, process, virtual processor);
Is_Partition      : aadlboolean applies to (system, process, virtual processor);
```

**模型参考属性**（用于多工具协同）：
```aadl
Model_Reference: type record (
    Model_Type : SEI::Model_Source_Type;
    Kind       : SEI::Reference_Kind_Type;
    Filename   : aadlstring;
    Artifact   : aadlstring;);
Model_References: list of SEI::Model_Reference applies to (all);
```

### 4.3 Data_Model.aadl：数据建模属性集

文件路径：`resources/properties/Data_Model.aadl`，对应 SAE AS5506 数据建模附件规范（v15，2009年版）。

主要属性：

```aadl
Base_Type       : list of classifier(data) applies to (data, feature);
Data_Representation : enumeration
    (Array, Boolean, Character, Enum, Float, Fixed, Integer, String, Struct, Union)
    applies to (data, feature);
Dimension       : list of aadlinteger applies to (data, data port, event data port, data access);
Element_Names   : list of aadlstring applies to (data, feature);
Enumerators     : list of aadlstring applies to (data, feature);
IEEE754_Precision: enumeration (Simple, Double) applies to (data, feature);
Integer_Range   : range of aadlinteger applies to (data, feature);
Initial_Value   : list of aadlstring applies to (data, feature);
```

### 4.4 其他属性集和包

| 文件 | 用途 |
|------|------|
| `Physical.aadl` | 物理资源属性集，用于描述硬件的物理特性 |
| `PhysicalResources.aadl` | 物理资源包（数据组件），提供通用物理量的类型定义 |
| `Base_Types.aadl` | SAE 标准 Base_Types 包，定义 Integer, Float, Boolean, String 等基础数据类型 |
| `ARINC653.aadl` | ARINC 653 分区操作系统标准属性集，用于航空软件分区建模 |
| `ARINC429.aadl` | ARINC 429 数据总线标准属性集，用于航空数据总线建模 |
| `Code_Generation_Properties.aadl` | 代码生成相关属性集，用于控制源代码生成行为 |

---

## 5. org.osate.slicer — 模型切片工具

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.slicer/src/org/osate/slicer/`

### 5.1 设计动机与核心思想

模型切片（Model Slicing）是一种从复杂系统模型中提取与特定关注点相关的子模型的技术。OSATE2 的 slicer 模块针对 AADL 实例模型，将其中的组件特性（Feature）、连接（Connection）和错误传播路径抽象为一个有向图，从而支持前向（影响分析）和后向（根因分析）可达性查询。

底层图算法库使用 JGraphT（`org.jgrapht`），这是 Core 模块中唯一声明的外部图算法依赖。

### 5.2 OsateSlicerVertex：图顶点类型

`OsateSlicerVertex.java` 是切片图的顶点类型，每个顶点表示"实例模型中的某个元素 + 可选的错误类型"：

```java
public class OsateSlicerVertex {
    // 关联的错误类型 token（来自 EMV2 TypeSetElement）
    final private Optional<TypeTokenInstance> token;

    // 对 AADL 实例对象的适配器引用（多态）
    final private VertexIObjAdapter element;
```

`VertexIObjAdapter` 是内部适配器接口，有以下实现（定义在 `VertexIObjAdapter.java`）：
- `FeatureInstanceAdapter`：包装 `FeatureInstance`（端口、特性）
- `AccessPropagationAdapter`：包装通过 access 连接传播的 `ComponentInstance`
- `ErrorFlowInstanceAdapter`：包装 `ErrorFlowInstance`（错误源、错误汇、错误路径）
- `PointPropagationAdapter`：包装命名的传播点（`PointPropagation`）
- `BoundComponentInstanceAdapter`：包装绑定类型传播（如处理器绑定）

**顶点唯一性**通过 `getName()` 方法实现，名称由实例对象路径和可选的错误类型名称组成（以 `.` 分隔）：

```java
public String getName() {
    if (token.isPresent()) {
        return element.name() + "." + token.get().getFullName();
    } else {
        return element.name();
    }
}

// equals 和 hashCode 均基于 toString()（即 getName()）
```

### 5.3 SlicerRepresentation：图构建与切片算法

`SlicerRepresentation.java` 是 slicer 模块的主类，职责包括：

**内部数据结构**：
```java
// 主有向图（JGraphT DefaultDirectedGraph）
private final Graph<OsateSlicerVertex, DefaultEdge> g = new DefaultDirectedGraph<>(DefaultEdge.class);
// 反向图（用于后向可达性，EdgeReversedGraph 是 g 的视图而非副本）
private final Graph<OsateSlicerVertex, DefaultEdge> rg = new EdgeReversedGraph<>(g);
// 顶点名称 -> 顶点对象的映射缓存
private final Map<String, OsateSlicerVertex> vertexMap = new HashMap<>();
// 每个容器的输入特性集合（用于推断隐式组件内连接）
private Map<String, Set<String>> inFeats = new HashMap<>();
// 每个容器的输出特性集合
private Map<String, Set<String>> outFeats = new HashMap<>();
// 有显式流规格的组件（不添加隐式全连接）
private Set<String> hasExplicitDecomp = new HashSet<>();
```

**图构建流程** (`buildGraph(SystemInstance si)`)：

1. **阶段1 — 核心 AADL 图构建**（`SlicerSwitch`）：
   遍历 `SystemInstance` 的所有元素：
   - `caseFeatureInstance`：为每个 `FeatureInstance` 创建顶点，并按方向（IN/OUT/IN_OUT）分类记录到 `inFeats`/`outFeats`
   - `caseConnectionInstance`：为每个 `ConnectionReference` 的源和目标创建顶点，添加有向边；若连接横跨多个层次（`connectionReferences.size() > 1`），标记目标为有显式分解
   - `caseEndToEndFlowInstance`：为端到端流中的 `FlowSpecificationInstance` 添加源到目标的边，标记相关组件

2. **阶段2 — EMV2 错误传播图构建**（`Emv2SlicerSwitch`）：
   遍历 EMV2 实例对象：
   - `caseErrorSourceInstance`：创建错误源顶点和对应传播特性顶点，添加源 → 传播 的边
   - `caseErrorSinkInstance`：创建传播特性顶点和错误汇顶点，添加传播 → 汇 的边
   - `caseErrorPathInstance`：从源传播到目标传播（内部错误路径）添加边
   - `casePropagationPathInstance`：记录"可能传播"（不立即添加边，延迟到不动点计算）

3. **阶段3 — 不动点计算** (`calculateFixpoint()`)：
   迭代地从每个错误源出发，检查可达的传播点是否触发了 `possiblePropagations` 中的跨组件路径，若有则添加新边，直到没有新边产生为止：

   ```java
   do {
       edgesModified = false;
       for (OsateSlicerVertex srcVrt : sourcePropagations) {
           // 从源顶点 BFS，检查每个到达的顶点是否有可能的传播
           // 若类型集合匹配，则添加新的跨组件边
       }
   } while (edgesModified);  // 收敛（不动点）后停止
   ```

4. **阶段4 — 隐式组件内连接** (`buildIntraConnections()`)：
   对于没有显式流规格的组件，连接其所有输入特性到所有输出特性（保守近似：假设任何输入都可能影响任何输出）

5. **阶段5 — 假设检查** (`checkAssumptions()`)：
   委托给 `UnusedElementDetector`，检测孤立的错误源和不可达的错误汇

### 5.4 前向与后向可达性

```java
// 从指定顶点名出发的 BFS 可达子图
public AsSubgraph<OsateSlicerVertex, DefaultEdge> forwardReachability(String srcVert) {
    return reach(g, srcVert);  // 在正向图上 BFS
}

// 能到达指定顶点的 BFS 可达子图（通过反向图实现）
public AbstractGraph<OsateSlicerVertex, DefaultEdge> backwardReachability(String tgtVert) {
    return new EdgeReversedGraph<>(reach(rg, tgtVert));  // 在反向图上 BFS，再反转回来
}
```

`reach()` 的实现手写了 BFS（而非使用 JGraphT 内置的 `getDescendants()`），原因有三（源代码注释中明确说明）：
1. `DirectedAcyclicGraph.getDescendants()` 要求图无环，但 EMV2 传播图可以有环
2. 返回子图比返回顶点集提供更多信息
3. JGraphT 内置实现与此自定义实现的计算复杂度相当

**高级查询 — 必经路径** (`reachThrough()`)：检查从 source 到 target 的所有路径是否都经过 mid，利用**双连通分量**（`BiconnectivityInspector`）的割点检测实现：

```java
public Optional<GraphPath<OsateSlicerVertex, DefaultEdge>> reachThrough(
        Graph<OsateSlicerVertex, DefaultEdge> graph,
        String sourceName, String targetName, String midName, boolean counterexample) {
    var gForward  = reach(graph, sourceName);   // 从 source 可达的子图
    var gBackward = reach(new EdgeReversedGraph<>(gForward), targetName);  // 能到达 target 的子图
    // mid 是割点 ↔ 所有路径都必须经过 mid
    if (!new BiconnectivityInspector<>(gBackward).getCutpoints().contains(vertexMap.get(midName))) {
        // mid 不是割点，存在绕过 mid 的路径
        // 若需要反例，用 BFSShortestPath 在屏蔽掉 mid 后找到这条路径
        ...
    }
    return Optional.empty();  // mid 是割点，所有路径必经 mid
}
```

### 5.5 UnusedElementDetector：假设检查

`UnusedElementDetector` 通过深度优先遍历检测两类问题：

- **非终止的错误源**（`nonTerminatingSourceVertices`）：从错误源可达，但不能到达任何错误汇的顶点集合
- **不可达的错误汇**（`unreachableSinkVertices`）：没有错误源能到达的错误汇

结果封装在 `AssumptionCheckResult` 记录类中（Java 16+ record）：

```java
public record AssumptionCheckResult(
    HashSet<OsateSlicerVertex> nonTerminatingSourceVertices,
    HashSet<OsateSlicerVertex> unreachableSinkVertices
) {}
```

### 5.6 与EMV2的集成

Slicer 对 EMV2 实例模型的依赖体现在 `Emv2SlicerSwitch` 中，该内部类继承自 `EMV2InstanceSwitch<Void>`（来自 `org.osate.aadl2.errormodel.instance` 包），处理以下 EMV2 实例对象类型：
- `ErrorSourceInstance` / `ErrorSinkInstance`
- `ErrorPathInstance`
- `PropagationPathInstance`（包括 `ConnectionPath`、`BindingPath`、`UserDefinedPath`）
- `FeaturePropagation`、`AccessPropagation`、`BindingPropagation`、`PointPropagation`

---

## 6. org.osate.resolute — Resolute验证语言集成

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.resolute/src/org/osate/resolute/`

### 6.1 设计模式：运行时桥接

`org.osate.resolute` 是一个"桥接"模块，本身不包含 Resolute 语言的实现，而是提供接口供 OSATE 核心（特别是 ALISA 框架）在不直接依赖 Resolute 插件的情况下调用 Resolute 功能。

`ResoluteUtil.java` 实现了运行时动态发现机制：

```java
public class ResoluteUtil {
    private final static String BUNDLE_ID  = "com.rockwellcollins.atc.resolute.analysis";
    private final static String CLASS_ID   =
        "com.rockwellcollins.atc.resolute.analysis.access.ResoluteInterface";

    private static ResoluteAccess RESOLUTE = null;

    public static boolean isResoluteInstalled() {
        return getResolute() != null;
    }

    public static ResoluteAccess getResolute() {
        if (RESOLUTE == null) {
            try {
                final Bundle bundle = Platform.getBundle(BUNDLE_ID);
                final Class<?> resoluteInterface = bundle.loadClass(CLASS_ID);
                RESOLUTE = (ResoluteAccess) resoluteInterface
                    .getDeclaredConstructor().newInstance();
            } catch (Exception e) {
                // 若 Resolute 未安装，静默忽略异常
            }
        }
        return RESOLUTE;  // 若未安装则返回 null
    }
}
```

这一设计使得 OSATE 核心功能可以在没有安装 Resolute 插件时正常工作，只有当 Resolute 实际可用时才启用相关功能（通过 `isResoluteInstalled()` 检查）。

### 6.2 ResoluteAccess接口

`ResoluteAccess.java` 定义了 OSATE 核心需要使用的 Resolute 功能的最小接口集：

```java
public interface ResoluteAccess {

    // 用于 EMV2 场景：执行 Resolute 函数并返回 Diagnostic（来自 org.osate.result）
    Diagnostic executeResoluteFunctionOnce(EObject fundef,
        final SystemInstance instanceroot, final ComponentInstance targetComponent,
        final InstanceObject targetElement, List<PropertyExpression> parameterObjects);

    // 用于 ALISA（Assure）场景：执行 Resolute 函数，返回内部 EObject 结果
    EObject executeResoluteFunctionOnce(EObject fundef,
        final ComponentInstance targetComponent, final InstanceObject targetElement,
        List<PropertyExpression> parameterObjects);

    // 反射式 API：获取 ResoluteLibrary 的所有函数定义
    List<EObject> getDefinitions(EObject resoluteLibrary);

    // 类型检查
    boolean isFunctionDefinition(EObject obj);
    boolean isBaseType(EObject type);

    // 元数据访问
    List<NamedElement> getArgs(EObject functionDefinition);
    EObject getType(EObject arg);
    String getTypeName(EObject baseType);
}
```

接口使用 `EObject` 而非 Resolute 特定类型，这使得调用方无需在编译时依赖 Resolute 的 EMF 元模型，完美体现了"插件对调用方透明"的设计原则。

### 6.3 与org.osate.result的关联

`ResoluteAccess.executeResoluteFunctionOnce()` 的一个重载返回 `org.osate.result.Diagnostic`，将 Resolute 的验证结论（成功/失败/信息）统一转换为 OSATE 结果模型，从而与其他分析工具（如 ALISA）共享统一的结果展示框架。

---

## 7. org.osate.results — 统一分析结果模型

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.results/src/org/osate/result/`

### 7.1 EMF元模型结构

`org.osate.results` 提供了一个基于 EMF（Eclipse Modeling Framework）定义的统一分析结果模型，所有 OSATE 分析工具（延迟分析、资源预算分析、EMV2 分析等）均可使用此模型描述分析结果，实现结果的序列化、持久化和跨工具共享。

元模型的类层次：
```
EObject
├── AnalysisResult      // 顶层：一次分析运行的总结果
│   ├── analysis        : String       // 分析名称
│   ├── message         : String       // 总体描述信息
│   ├── modelElement    : EObject      // 被分析的模型元素（引用）
│   ├── parameters      : ObjectValue* // 分析参数
│   ├── results         : Result*      // 子结果列表
│   ├── diagnostics     : Diagnostic*  // 诊断消息列表
│   └── resultType      : ResultType   // 总体结论（默认 TBD）
│
├── Result              // 中间节点：一个分析项的结果
│   ├── message         : String
│   ├── modelElement    : EObject
│   ├── values          : Value*       // 结果值
│   ├── diagnostics     : Diagnostic*
│   ├── subResults      : Result*      // 支持嵌套结果树
│   └── resultType      : ResultType
│
├── Diagnostic          // 叶节点：单条诊断消息
│   ├── diagnosticType  : DiagnosticType
│   ├── message         : String
│   └── modelElement    : EObject
│
└── Value               // 抽象值
    ├── BooleanValue    // 布尔值
    ├── IntegerValue    // 整数值
    ├── RealValue       // 实数值
    ├── StringValue     // 字符串值
    ├── EObjectValue    // EMF 对象引用
    └── ObjectValue     // 通用 Java 对象
```

### 7.2 AnalysisResult：顶层结果容器

`AnalysisResult` 接口定义了分析运行的顶层结果：

```java
public interface AnalysisResult extends EObject {
    String getAnalysis();           // 分析工具标识符（如 "Latency Analysis"）
    void setAnalysis(String value);

    String getMessage();            // 人类可读的总体信息
    void setMessage(String value);

    EObject getModelElement();      // 被分析的模型元素（通常是 SystemInstance）
    void setModelElement(EObject value);

    EList<ObjectValue> getParameters();  // 输入参数（记录分析配置）

    EList<Result> getResults();     // 分析产生的各项结果（分层树结构）

    EList<Diagnostic> getDiagnostics();  // 分析级别的诊断消息

    ResultType getResultType();     // SUCCESS, FAILURE, ERROR, WARNING, INFO, TBD
    void setResultType(ResultType value);
}
```

### 7.3 Result：分层结果节点

`Result` 支持无限嵌套的 `subResults`，形成树形结果结构，非常适合描述层次化系统分析中的结果组织（例如：系统级 → 子系统级 → 组件级）：

```java
public interface Result extends EObject {
    EList<Value> getValues();        // 该分析项的具体数值结果
    EList<Diagnostic> getDiagnostics();
    EList<Result> getSubResults();   // 子结果（可嵌套）
    String getMessage();
    EObject getModelElement();
    ResultType getResultType();
}
```

### 7.4 Diagnostic与DiagnosticType

`Diagnostic` 表示一条具体的诊断消息，携带类型、消息文本和关联的模型元素：

```java
public interface Diagnostic extends EObject {
    DiagnosticType getDiagnosticType();  // ERROR, WARNING, INFO, TBD
    String getMessage();
    EObject getModelElement();           // 触发该诊断的模型元素
}
```

`DiagnosticType` 是 EMF `Enumerator` 枚举：

```java
public enum DiagnosticType implements Enumerator {
    TBD(0, "TBD", "TBD"),       // 未确定（初始默认值）
    ERROR(1, "ERROR", "ERROR"),
    WARNING(2, "WARNING", "WARNING"),
    INFO(3, "INFO", "INFO");
    ...
}
```

### 7.5 Value类型层次

`Value` 的各子类型用于携带不同类型的分析结果数值：

| 类型 | 用途 | 例子 |
|------|------|------|
| `BooleanValue` | 布尔型结论 | 资源预算是否满足 |
| `IntegerValue` | 整数值结果 | 错误路径数量 |
| `RealValue` | 实数值结果 | 延迟时间（ms） |
| `StringValue` | 字符串描述 | 分析状态描述 |
| `EObjectValue` | EMF 对象引用 | 关联的实例模型元素 |
| `ObjectValue` | 通用 Java 对象 | 复杂数据结构 |

---

## 8. org.osate.aadl2.modelsupport — 错误报告基础设施与模型遍历

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.aadl2.modelsupport/src/org/osate/aadl2/modelsupport/`

### 8.1 错误报告器层次架构

模块提供了一套分层的错误报告体系，区分**解析阶段**（文本 → 模型）和**分析阶段**（模型验证/分析）的错误报告：

```
ErrorReporter (接口)
├── ParseErrorReporter (接口)
│   ├── error(LocationReference loc, String message)
│   ├── error(String filename, int line, String message)
│   ├── warning(LocationReference, String)
│   └── info(LocationReference, String)
│
└── AnalysisErrorReporter (接口)
    ├── error(Element obj, String msg, String[] attrs, Object[] values)
    ├── warning(Element obj, String msg, ...)
    └── info(Element obj, String msg, ...)
```

抽象实现类：
- `AbstractParseErrorReporter`：实现计数逻辑，跟踪 error/warning/info 计数
- `AbstractAnalysisErrorReporter`：实现有限计数（超出上限时创建溢出标记）
- `AbstractErrorReporterManager`：管理多个 Reporter 实例的生命周期

具体实现类：

| 类名 | 模式 | 说明 |
|------|------|------|
| `MarkerParseErrorReporter` | Eclipse 标记 | 在 AADL 文本文件上创建 Eclipse 问题标记 |
| `MarkerAnalysisErrorReporter` | Eclipse 标记 | 在 AAXL 模型资源上创建 Eclipse 问题标记 |
| `LogParseErrorReporter` | 日志 | 写入 Eclipse 错误日志 |
| `LogAnalysisErrorReporter` | 日志 | 写入 Eclipse 错误日志 |
| `NullAnalysisErrorReporter` | 空操作 | 忽略所有错误（headless 场景） |
| `ChainedAnalysisErrorReporter` | 组合 | 将错误转发给多个 Reporter |

`NullAnalysisErrorReporter` 有一个单例工厂：`NullAnalysisErrorReporter.factory`，用于构造只忽略错误的 `AnalysisErrorReporterManager.NULL_ERROR_MANANGER`。

### 8.2 AnalysisErrorReporterManager：跨资源管理

`AnalysisErrorReporterManager.java` 解决了一个核心问题：在分析一个 `SystemInstance` 时，错误可能涉及多个来源 AADL 文件（每个文件对应一个 `Resource`），需要将错误分别汇报到正确的资源上。

```java
public final class AnalysisErrorReporterManager extends AbstractErrorReporterManager {
    // 单例空操作管理器
    public static final AnalysisErrorReporterManager NULL_ERROR_MANANGER =
        new AnalysisErrorReporterManager(NullAnalysisErrorReporter.factory);

    // Resource -> AnalysisErrorReporter 的映射
    private final Map reportersMap;

    // 所有 reporter 的有序列表（用于统计计数）
    private final List reportersList;

    // 消息前缀栈（用于为错误消息添加上下文前缀，如 "In SystemMode X: "）
    private LinkedList prefixStack = new LinkedList();
```

核心方法 `getReporter(Resource rsrc)`：首次访问某个资源时，通过工厂创建对应的 Reporter 并调用 `deleteMessages()` 清除旧标记，然后缓存供后续复用：

```java
public final AnalysisErrorReporter getReporter(final Resource rsrc) {
    AnalysisErrorReporter errReporter = (AnalysisErrorReporter) reportersMap.get(rsrc);
    if (errReporter == null) {
        errReporter = factory.getReporterFor(rsrc);
        reportersMap.put(rsrc, errReporter);
        reportersList.add(errReporter);
        errReporter.deleteMessages();  // 清除旧的分析标记
    }
    return errReporter;
}
```

**前缀栈机制**：允许分析工具为一组错误添加上下文信息。例如，在模式 X 下进行分析时：

```java
manager.addPrefix("In SystemMode X: ");
// ... 报告分析错误 ...
manager.removePrefix();
```

所有在 `addPrefix` 和 `removePrefix` 之间报告的错误消息都会自动附加前缀。

### 8.3 MarkerAnalysisErrorReporter：Eclipse标记集成

`MarkerAnalysisErrorReporter.java` 将分析错误作为 Eclipse 问题标记（`IMarker`）附加到 `.aaxl` 文件上：

```java
private void createMarker(final Element where, final String message,
        final int severity, final String[] attrs, final Object[] values) {
    IMarker marker_p = iResource.createMarker(markerType);
    marker_p.setAttribute(IMarker.SEVERITY, severity);
    marker_p.setAttribute(IMarker.MESSAGE, message);
    // 存储模型对象的 EMF URI，用于从标记导航回模型元素
    marker_p.setAttribute(AadlConstants.AADLURI, EcoreUtil.getURI(where).toString());

    // 存储分析特定的额外属性
    for (int i = 0; i < attrs.length; i++) {
        marker_p.setAttribute(markerType + "." + attrs[i], values[i]);
    }
}
```

标记类型名 `markerType` 由各分析工具自行定义，不同的分析工具使用不同的标记类型，便于在 Eclipse Problems 视图中分类过滤。内嵌 `Factory` 类在无法找到对应 `IResource` 时可选地委托给辅助工厂（如 `LogAnalysisErrorReporter.Factory`），实现优雅降级。

### 8.4 模型遍历框架

`org.osate.aadl2.modelsupport.modeltraversal` 包提供了多种模型遍历策略：

- **`ForAllElement`**：通用迭代器，支持条件过滤（`suchThat(Element)`）和自定义动作（`action(Element)`），结果收集在 `resultList` 中
- **`PreOrderTraversal`**：前序遍历（先父后子）
- **`PostOrderTraversal`**：后序遍历（先子后父）
- **`TopDownComponentImplTraversal`**：自顶向下遍历 `ComponentImplementation` 层次
- **`BottomUpComponentImplTraversal`**：自底向上遍历
- **`TopDownComponentClassifierTravseral`**：遍历 `ComponentClassifier` 层次
- **`SOMIterator`**：迭代 `SystemOperationMode`（系统操作模式）
- **`TraverseWorkspace`**：遍历整个 Eclipse 工作空间中的 AADL 文件

---

## 9. org.osate.aadl2.instance.textual — 实例模型文本语法

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/org.osate.aadl2.instance.textual/src/`

### 9.1 Xtext语法架构

AADL 实例模型（`.aaxl2` 文件）的文本序列化格式由 Xtext 语法 `Instance.xtext` 定义：

```xtext
grammar org.osate.aadl2.instance.textual.Instance
    with org.osate.xtext.aadl2.properties.Properties
```

此语法**继承**自 `org.osate.xtext.aadl2.properties.Properties`（属性语法），避免重复定义属性相关的语法规则。

导入的元模型：
```xtext
import "http://aadl.info/AADL/2.0/instance" as instance  // 实例元模型
import "http://aadl.info/AADL/2.0"         as aadl2     // 声明式 AADL 元模型
import "http://www.eclipse.org/emf/2002/Ecore" as ecore
```

### 9.2 核心语法规则

**`SystemInstance`**（系统实例，实例模型的根节点）：
```xtext
SystemInstance returns instance::SystemInstance:
    category=ComponentCategory name=ID ':'
    componentImplementation=[aadl2::ComponentImplementation|ImplRef] '{'
    (
        featureInstance+=FeatureInstance |
        componentInstance+=ComponentInstance |
        connectionInstance+=ConnectionInstance |
        flowSpecification+=FlowSpecificationInstance |
        endToEndFlow+=EndToEndFlowInstance |
        modeInstance+=ModeInstance |
        modeTransitionInstance+=ModeTransitionInstance |
        systemOperationMode+=SystemOperationMode |
        ownedPropertyAssociation+=PropertyAssociationInstance
    )* '}'
;
```

此规则说明系统实例由组件类别、名称、指向声明式 `ComponentImplementation` 的跨资源引用（`ImplRef` 格式）以及各种子元素组成。

**`ComponentInstance`**（组件实例，递归结构）：
```xtext
ComponentInstance returns instance::ComponentInstance:
    category=ComponentCategory
    classifier=[aadl2::ComponentClassifier|ClassifierRef]?
    name=ID ('[' index+=Long ']')*   // 数组下标（支持多维）
    ('in' 'modes' '(' inMode+=[instance::ModeInstance] ... ')')?
    ':' subcomponent=[aadl2::Subcomponent|DeclarativeRef]
    ('{' ( ... 同 SystemInstance 的子元素类型 ... )* '}')?
;
```

**`ConnectionInstance`**（连接实例）：
```xtext
ConnectionInstance returns instance::ConnectionInstance:
    complete?='complete'? kind=ConnectionKind name=STRING ':'
    source=[instance::ConnectionInstanceEnd|InstanceRef]
    (bidirectional?='<->' | '->')
    destination=[instance::ConnectionInstanceEnd|InstanceRef]
    ('in' 'modes' '(' ... ')')?
    ('in' 'transitions' '(' ... ')')?
    '{' (connectionReference+=ConnectionReference | ownedPropertyAssociation+=...) + '}'
;
```

**`ConnectionReference`**（单跳连接引用）：
```xtext
ConnectionReference returns instance::ConnectionReference:
    source=[instance::ConnectionInstanceEnd|InstanceRef] '->'
    destination=[instance::ConnectionInstanceEnd|InstanceRef] ':'
    (reverse?='reverse')? connection=[aadl2::Connection|DeclarativeRef]
    'in' context=[instance::ComponentInstance|InstanceRef]
;
```

**`SystemOperationMode`**（系统操作模式）：
```xtext
SystemOperationMode returns instance::SystemOperationMode:
    'som' name=STRING
    (currentMode+=[instance::ModeInstance|InstanceRef] (',' ...)* )?
;
```

**`PropertyAssociationInstance`**（属性关联实例）：
```xtext
PropertyAssociationInstance returns instance::PropertyAssociationInstance:
    property=[aadl2::Property|QPREF] '=>'
    ownedValue+=OptionalModalPropertyValue (',' ownedValue+=...)*
    ':' propertyAssociation=[aadl2::PropertyAssociation|PropertyAssociationRef]
;
```

### 9.3 跨资源引用解析

实例模型中大量使用了跨资源引用，指向声明式 AADL 模型（`.aadl` 文件）中的元素。`InstanceLinkingService.java` 处理这些引用的解析，`InstanceQualifiedNameConverter.java` 定义了引用的名称格式（如 `DeclarativeRef`、`InstanceRef`、`ClassifierRef`、`ImplRef`）。

`InstanceFormatter.java` 负责反方向的序列化格式化，`InstanceSemanticSequencer.java` 和 `InstanceSyntacticSequencer.java` 是 Xtext 自动生成的序列化器组件。

`InstanceCrossReferenceSerializer.java` 专门处理跨资源引用的序列化，确保保存的 `.aaxl2` 文件中的引用格式能被正确解析回对应的声明式模型元素。

---

## 10. ease-scripts — EASE自动化脚本

**源码路径**：`/home/hlt/project/explore_osate/osate2/core/ease-scripts/src/`

EASE（Eclipse Advanced Scripting Environment）是 Eclipse 的脚本扩展，支持在 Eclipse 内直接运行 JavaScript 或 Python 脚本。Core 模块提供了两个 EASE 脚本：

**`generate-core.js`**：自动化执行 AADL2 元模型的代码生成流程。脚本使用 SWTBot 自动化操作 Eclipse UI：

1. 在 Package Explorer 中定位 `org.osate.aadl2.metamodel/Models/AADL2.EMF.uml`
2. 用 UML Model Editor 打开
3. 另存为 `AADL2.EMF.merged.uml`
4. 执行 "Package Merge" 操作，合并 UML 包
5. 执行 "Convert To Metamodel"，将 UML 转换为 ECore 元模型
6. 重新加载 `aadl2.genmodel` 生成器模型
7. 执行 "Generate All"，生成所有 Java 代码

```javascript
importPackage(org.eclipse.swtbot.eclipse.finder);
bot = new SWTWorkbenchBot();

// 打开 UML 文件
explorerView = bot.viewByTitle("Package Explorer");
node = findTreeItem(explorer.getAllItems(), "core");
// ... 导航到 AADL2.EMF.uml 文件

// 执行包合并
umlMenu = bot.menu("UML Editor");
umlMenu.menu("Package").menu("Merge...").click();

// 执行代码生成
node.contextMenu("Reload...").click();  // 重载 genmodel
root.contextMenu("Generate All").click();  // 生成代码
```

**`generate-instance.js`**：类似的自动化脚本，用于生成 AADL 实例元模型（`instance.genmodel`）的 Java 代码。

这两个脚本用于开发阶段元模型变更后的代码再生成，通常不在日常开发中运行。

---

## 11. 模块间依赖关系图

```
org.osate.aadl2 (EMF 元模型)
    ↑
    ├── org.osate.aadl2.modelsupport
    │       ├── 提供 ParseErrorReporter, AnalysisErrorReporter 体系
    │       └── 提供模型遍历框架 (ForAllElement, traversal)
    │
    ├── org.osate.annexsupport
    │       ├── 依赖 aadl2.modelsupport (ParseErrorReporter, AnalysisErrorReporterManager)
    │       └── 使用 aadl2.AnnexLibrary, AnnexSubclause
    │
    ├── org.osate.pluginsupport
    │       └── 管理 AADL 贡献资源的发现与覆盖
    │
    ├── org.osate.contribution.sei
    │       └── 通过 pluginsupport.aadlcontribution 扩展点贡献 SEI 属性集
    │
    ├── org.osate.aadl2.instance.textual
    │       ├── 依赖 aadl2 实例元模型
    │       └── 继承 xtext.aadl2.properties 语法
    │
    ├── org.osate.results (org.osate.result)
    │       └── 独立的 EMF 元模型，被各分析工具使用
    │
    ├── org.osate.resolute
    │       └── 依赖 org.osate.result.Diagnostic
    │          运行时动态加载 com.rockwellcollins.atc.resolute.analysis
    │
    └── org.osate.slicer
            ├── 依赖 aadl2.instance (SystemInstance, ComponentInstance 等)
            ├── 依赖 aadl2.errormodel.instance (EMV2 实例类型)
            └── 依赖 org.jgrapht (JGraphT 图算法库)
```

**外部依赖**：
- Eclipse Platform（扩展注册表、资源管理、偏好存储）
- EMF（Eclipse Modeling Framework）
- Xtext（AADL 语言解析器框架）
- JGraphT（图算法，仅 slicer 使用）
- SWTBot（UI 自动化，仅 ease-scripts 使用）
- Apache Commons Lang（`StringUtils.countMatches`，仅 slicer 使用）

---

## 12. 关键设计模式总结

### 12.1 注册表 + 代理的懒加载模式（Annex扩展体系）

`AnnexRegistry` 结合 `AnnexProxy` 实现了经典的"按需加载"模式：Eclipse 平台扫描所有 `plugin.xml` 建立轻量级代理索引，只有在实际调用某个 annex 功能时才真正实例化对应的类。

```
Eclipse 启动时
    ↓
plugin.xml 扫描 → 建立 AnnexProxy（含类名字符串，未实例化）
    ↓
首次调用（如 parser.parseAnnexLibrary(...)）
    ↓
AnnexProxy.createExecutableExtension("class") → 实例化真正的 AnnexParser
```

### 12.2 双模式支持（Eclipse模式 vs headless模式）

多处代码（AnnexRegistry、PluginSupportUtil 等）通过检查 `Platform.getExtensionRegistry() != null` 来区分运行环境，在 headless 场景下提供降级实现，支持 OSATE 作为无界面批处理工具使用。

### 12.3 Eclipse Resource - EMF Resource 双层资源体系

错误报告器区分了两种资源类型：
- Eclipse `IResource`：文件系统视图，用于创建/删除问题标记
- EMF `Resource`：模型对象的容器，用于定位元素所在的"来源"文件

`AnalysisErrorReporterManager` 以 EMF `Resource` 为键缓存 Reporter，`MarkerAnalysisErrorReporter` 内部维护 `IResource` 用于标记操作，通过 `OsateResourceUtil.toIFile()` 在两者间转换。

### 12.4 不动点迭代（Slicer 模块）

`SlicerRepresentation.calculateFixpoint()` 使用迭代不动点（Fix-point Iteration）算法解决 EMV2 错误传播图的连通性问题——初始只有确定性的边，通过反复传播直到收敛，这是数据流分析中的经典技术。

### 12.5 接口隔离与运行时桥接（Resolute 集成）

`ResoluteAccess` 接口完全用 `EObject` 替代 Resolute 特定类型，配合 `Platform.getBundle()` + 反射实例化，实现了在编译时不依赖可选插件、在运行时按需启用功能的"可选依赖"模式。

### 12.6 EMF Value Object 模式（Results 模型）

`AnalysisResult` / `Result` / `Diagnostic` / `Value` 构成了一个完整的"结果数据传输对象"体系，基于 EMF 生成代码，支持 EMF 通知机制（可绑定到 Eclipse UI 视图）、序列化（保存为 XMI 文件）和内容比较，是跨工具结果共享的理想载体。

---

*文档基于源码实际阅读，版本：OSATE2 2.18.0-SNAPSHOT，日期：2026年2月。*
