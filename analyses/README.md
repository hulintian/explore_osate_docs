# Analyses 分析模块

## 1. 模块概述与架构

`analyses` 子项目包含 OSATE2 中用于对 AADL 实例模型执行各类量化与质量分析的所有插件。全部模块均以 OSGi Bundle 方式发布，遵循 Eclipse Plugin 扩展点机制与 OSATE2 的 `AbstractAaxlHandler` / `AadlProcessingSwitchWithProgress` 框架。

### 1.1 实际存在的 19 个 Bundle

**分析核心**
| Bundle | 说明 |
|--------|------|
| `org.osate.analysis.model` | 分析基础设施（`AnalysisModelTraversal` 等工具类） |

**架构与语义检查**
| Bundle | 说明 |
|--------|------|
| `org.osate.analysis.architecture` | 端口一致性、连接绑定、约束绑定、权重、未使用分类器 |
| `org.osate.analysis.architecture.tests` | 上述分析的 JUnit 测试 |

**流分析**
| Bundle | 说明 |
|--------|------|
| `org.osate.analysis.flows` | 端到端流延迟分析（核心） |
| `org.osate.analysis.flows.tests` | 流延迟分析的 JUnit 测试 |

**资源预算**
| Bundle | 说明 |
|--------|------|
| `org.osate.analysis.resource.budgets` | 处理器/内存/总线负载分析 |
| `org.osate.analysis.resource.budgets.tests` | 预算分析 JUnit 测试 |
| `org.osate.analysis.resource.management` | 资源管理分析（调度相关） |

**模式分析**
| Bundle | 说明 |
|--------|------|
| `org.osate.analysis.modes` | SOM 可达性分析、模式图生成 |
| `org.osate.analysis.modes.tests` | 模式分析 JUnit 测试 |
| `org.osate.analysis.modes.ui` | 模式分析 UI 对话框 |

**装箱与调度**
| Bundle | 说明 |
|--------|------|
| `org.osate.analysis.binpacking` | 软件到处理器的装箱与调度算法库 |

**模型统计**
| Bundle | 说明 |
|--------|------|
| `org.osate.modelstats` | 模型统计（组件数量、连接数量等） |
| `org.osate.modelstats.tests` | 统计 JUnit 测试 |
| `org.osate.modelstats.ui` | 统计 UI |

**代码生成检查**
| Bundle | 说明 |
|--------|------|
| `org.osate.codegen.checker` | POK/VxWorks/Deos 代码生成前置检查 |

**模型导入**
| Bundle | 说明 |
|--------|------|
| `org.osate.importer` | 通用模型导入框架（中间模型 + 状态机模型） |
| `org.osate.importer.simulink` | Simulink XML 文件导入（解析 `.mdl`/`.slx` XML） |

**发布特性**
| Bundle | 说明 |
|--------|------|
| `org.osate.plugins.feature` | 聚合 Feature 描述符 |

### 1.2 分析执行框架

所有分析都通过 Eclipse Command/Handler 机制触发，典型调用路径：

```
Eclipse UI 命令
  → Handler（继承 AbstractInstanceOrDeclarativeModelModifyHandler 或 AbstractAaxlHandler）
    → AadlProcessingSwitchWithProgress（遍历实例模型）
      → 具体 *Switch 或 *Analysis 类
        → AnalysisResult / Result（org.osate.result 包）
```

`AadlProcessingSwitchWithProgress` 的 `initSwitches()` 方法中，开发者通过覆写 `InstanceSwitch<String>` 的 `caseXxx()` 方法注册针对不同实例元素（`ComponentInstance`、`ConnectionInstance`、`EndToEndFlowInstance` 等）的访问者逻辑。

---

## 2. 分析框架基础设施（org.osate.analysis.model）

**Bundle 路径：** `analyses/org.osate.analysis.model`

该 Bundle 极为精简，仅提供两个类：

- `org.osate.analysis.model.internal.Activator`：OSGi Bundle 激活器
- `org.osate.analysis.model.util.AnalysisModelTraversal`：分析模型遍历工具

`AnalysisModelTraversal` 提供 `preOrder()` 和 `postOrder()` 两个静态方法，接受一个 `BusloadSwitch`（或其他 EMF Switch）对分析模型进行前序/后序遍历。该类被 `NewBusLoadAnalysis` 的两阶段算法（先前序建立结果树，再后序计算预算）直接使用。

```java
// NewBusLoadAnalysis 中的两阶段遍历示例
AnalysisModelTraversal.preOrder(busLoadModel,
    new BandwidthAndResultPreOrderSwitch(results, somResult), monitor);
AnalysisModelTraversal.postOrder(busLoadModel,
    new CapacityAndBudgetPostOrderSwitch(results), monitor);
```

---

## 3. 流延迟分析（org.osate.analysis.flows）

### 3.1 整体结构

```
org.osate.analysis.flows/
├── FlowLatencyAnalysisSwitch.java      核心分析 Switch
├── FlowanalysisPlugin.java             OSGi Activator
├── dialogs/FlowLatencyDialog.java      参数配置对话框
├── handlers/CheckFlowLatency.java      Eclipse Handler（入口）
├── internal/utils/
│   ├── AnalysisUtils.java              时间范围缩放等工具方法
│   └── FlowLatencyUtil.java            分区调度、ARINC653 工具
├── model/
│   ├── LatencyContributor.java         延迟贡献者抽象基类
│   ├── LatencyContributorComponent.java  组件贡献者
│   ├── LatencyContributorConnection.java 连接贡献者
│   ├── LatencyReportEntry.java         单条 ETEF 的报告
│   ├── LatencyReport.java              完整报告集合
│   ├── LatencyCSVReport.java           CSV 格式输出
│   └── LatencyExcelReport.java         Excel 格式输出
├── preferences/
│   ├── Constants.java                  偏好常量定义
│   ├── Initializer.java                偏好初始化
│   └── Page.java                       偏好页面 UI
└── reporting/
    ├── exporters/CsvExport.java
    ├── exporters/ExcelExport.java
    └── model/{Line,Report,ReportSeverity,Section,ReportedCell}.java
```

### 3.2 FlowLatencyAnalysisSwitch 核心类

**类全名：** `org.osate.analysis.flows.FlowLatencyAnalysisSwitch`
**父类：** `AadlProcessingSwitchWithProgress`

该类实现了端到端流延迟分析的全部算法逻辑。关键字段和状态：

```java
public class FlowLatencyAnalysisSwitch extends AadlProcessingSwitchWithProgress {
    private LatencyReport report;

    // 记录非周期总线（无 Period 属性）的有序集合，用于后续计算排队延迟
    private final Set<ComponentInstance> asyncBuses = new TreeSet<>(...);

    // 映射 (总线实例, 连接实例) -> 该连接对应的延迟贡献者
    private final Map<Pair<ComponentInstance, ConnectionInstance>, LatencyContributor>
        connectionsToContributors = new HashMap<>();

    // 记录已计算的最大传输时间，排队延迟计算的基础
    private final Map<Pair<ComponentInstance, ConnectionInstance>, Double>
        computedMaxTransmissionLatencies = new HashMap<>();
}
```

**关键方法：**

| 方法 | 说明 |
|------|------|
| `initSwitches()` | 注册 `caseEndToEndFlowInstance(etef)` 访问者，逐一遍历模型中的所有 ETEF |
| `analyzeLatency(etef, som, async)` | 对单个 ETEF 进行延迟分析，返回 `LatencyReportEntry` |
| `mapComponentInstance(etef, fei, entry)` | 计算组件（线程/设备/分区等）的延迟贡献 |
| `mapConnectionInstance(etef, fei, entry)` | 计算连接的延迟贡献（含绑定总线/虚拟总线） |
| `processActualConnectionBindingsTransmission(...)` | 递归处理 `DeploymentProperties::Actual_Connection_Binding` 指定的传输延迟 |
| `processActualConnectionBindingsSampling(...)` | 处理绑定总线的采样延迟（含 `Required_Virtual_Bus_Class`） |
| `processSamplingAndQueuingTimes(...)` | 对异步总线加入排队延迟条目（延迟到 `fillInQueuingTimes()` 阶段计算） |
| `fillInQueuingTimes(system)` | Issue 1148 修复：在所有传输时间计算完成后，统一计算异步总线的排队等待时间 |
| `invoke(SystemInstance, som, ...)` | 主入口：遍历全部 SOM 或指定 SOM 并返回 `AnalysisResult` |
| `invokeAndSaveResult(...)` | 调用 `invoke()` 后自动保存 `.result` 和 `.csv` 文件 |
| `finalizeResults()` | 调用 `report.finalizeAllEntries()` 完成最终统计 |

**5 个分析参数（`Constants.java` 中定义）：**

```java
// 偏好 Key 常量（均定义在 org.osate.analysis.flows.preferences.Constants）
ASYNCHRONOUS_SYSTEM        = "org.osate.analysis.flows.asynchronous_system"
PARTITONING_POLICY         = "org.osate.analysis.flows.partitioning_policy"
WORST_CASE_DEADLINE        = "org.osate.analysis.flows.report_worstcasedeadline"
BESTCASE_EMPTY_QUEUE       = "org.osate.analysis.flows.bestcase_empty_queue"
DISABLE_QUEUING_LATENCY    = "org.osate.analysis.flows.disable_queuing_latency"
```

分区策略有两个枚举值：`PARTITIONEND`（分区窗口末尾）和 `MAJORFRAMEDELAYED`（主帧延迟）。

### 3.3 LatencyContributor 层次结构

```
LatencyContributor（抽象基类）
├── LatencyContributorComponent   用于组件、分区、总线等
└── LatencyContributorConnection  用于连接实例
```

**`LatencyContributorMethod` 枚举（13 个值）：**

```java
public enum LatencyContributorMethod {
    UNKNOWN,
    DEADLINE,
    RESPONSE_TIME,        // SEI::Response_Time
    PROCESSING_TIME,      // Timing_Properties::Compute_Execution_Time
    DELAYED,              // 延迟连接引起的帧延迟采样
    SAMPLED,              // 周期组件采样延迟
    FIRST_PERIODIC,       // ETEF 第一个周期组件的初始采样
    SPECIFIED,            // Communication_Properties::Latency 显式规约
    QUEUED,               // 队列延迟（端口队列或总线排队）
    TRANSMISSION_TIME,    // Communication_Properties::Transmission_Time
    PARTITION_FRAME,      // 分区主帧率
    PARTITION_SCHEDULE,   // 分区帧偏移
    PARTITION_OUTPUT,     // 分区 I/O 延迟到分区窗口末尾或主帧
    SAMPLED_PROTOCOL      // 周期总线/协议的采样
}
```

每个 `LatencyContributor` 携带以下数值字段：

| 字段 | 说明 |
|------|------|
| `minValue` / `maxValue` | 实际最小/最大延迟（ms） |
| `expectedMin` / `expectedMax` | Flow Specification 规约延迟（ms）|
| `samplingPeriod` | 周期采样/分区周期（ms）|
| `partitionOffset` | ARINC653 分区帧偏移（ms）|
| `partitionDuration` | 分区窗口长度（ms）|
| `immediateDeadline` | 用于 LAST_IMMEDIATE 检查的截止期（ms）|

`getTotalMinimum()` / `getTotalMaximum()` 方法通过对 `subContributors` 列表递归求和获得含子贡献者的完整延迟，这是层次化建模的核心。

### 3.4 端到端延迟计算算法

算法分为三个关键阶段：

**阶段一：遍历 ETEF 元素**

对每个 `FlowElementInstance`，根据其实际类型分派：
- `FlowSpecificationInstance` 或 `ComponentInstance` → `mapComponentInstance()`
- `ConnectionInstance` → `mapConnectionInstance()`
- `EndToEndFlowInstance`（嵌套） → 递归处理

**阶段二：最坏/最优情况值选取逻辑（mapComponentInstance 中）**

```
最坏情况优先级：
  1. SEI::Response_Time（最高优先）
  2. Timing_Properties::Compute_Execution_Time（若偏好为 WCET 而非 Deadline）
  3. Timing_Properties::Deadline（若显式赋值且偏好为 Deadline 模式）
  4. Communication_Properties::Latency（Flow Specification 规约值）
  5. Deadline = Period（默认值，附 Info 消息）
  6. UNKNOWN（未知）

最优情况优先级：
  1. SEI::Response_Time
  2. Timing_Properties::Compute_Execution_Time（最小值）
  3. Communication_Properties::Latency（Flow Specification 规约最小值）
```

**阶段三：异步总线排队时间补充（fillInQueuingTimes）**

针对无 `Timing_Properties::Period` 属性的异步总线，在所有传输时间计算完毕后：
1. 收集绑定到该总线的所有连接及其最大传输时间
2. 对每条连接，其最大排队等待时间 = 其他所有连接传输时间之和（最坏情况抢占）
3. 最优情况排队时间 = 0（总线空闲假设）

**传输时间计算（getTimeToTransferData）：**

```java
// 来自 Communication_Properties::Transmission_Time 属性
TransmissionTime tt = CommunicationProperties.getTransmissionTime(bus);
double min = tt.fixed.min + datasize * tt.perByte.min;
double max = tt.fixed.max + datasize * tt.perByte.max;
```

### 3.5 plugin.xml 扩展点注册

```xml
<!-- 标记类型：用于在实例文件上显示流延迟分析问题 -->
<extension id="FlowLatencyObjectMarker"
           point="org.eclipse.core.resources.markers">
  <super type="org.osate.aadl2.modelsupport.AadlObjectMarker"/>
  <persistent value="true"/>
</extension>

<!-- UI 命令 -->
<extension point="org.eclipse.ui.commands">
  <command id="org.osate.analysis.flows.checkFlowLatency"
           categoryId="org.osate.analysis.category"
           name="%CheckFlowLatency.label"/>
</extension>

<!-- Handler 绑定（仅在 .aaxl2 或 SystemInstance 被选中时启用） -->
<extension point="org.eclipse.ui.handlers">
  <handler class="org.osate.analysis.flows.handlers.CheckFlowLatency"
           commandId="org.osate.analysis.flows.checkFlowLatency">
    <enabledWhen>
      <reference definitionId="org.osate.ui.definition.isInstanceFileOrSystemInstanceSelected"/>
    </enabledWhen>
  </handler>
</extension>

<!-- 菜单位置：Timing 菜单和工具栏，以及 Navigator 右键菜单 -->
<extension point="org.eclipse.ui.menus">
  <menuContribution locationURI="menu:org.osate.ui.timingMenu?after=core">...</menuContribution>
  <menuContribution locationURI="toolbar:org.osate.ui.timingToolbar?after=core">...</menuContribution>
  <menuContribution locationURI="popup:org.osate.ui.timingNavigatorPopup?after=core">...</menuContribution>
</extension>

<!-- ALISA 集成注册：允许 ALISA 通过反射调用分析 -->
<extension point="org.osate.pluginsupport.registeredjavaclasses">
  <class path="org.osate.analysis.flows.FlowLatencyAnalysisSwitch"/>
</extension>
```

### 3.6 输出格式

`CheckFlowLatency.finalizeAnalysis()` 方法在分析结束后同步生成三种输出：

1. **`.result` 文件**（EMF 对象，可被 ALISA 引用）：`FlowLatencyUtil.saveAnalysisResult(latResult)`
2. **CSV 文件**（位于 `reports/` 目录）：`LatencyCSVReport.generateCSVReport(latResult)`
3. **Excel 文件**（`.xlsx`）：`LatencyExcelReport.generateExcelReport(latResult)`
4. **Eclipse Markers**（`.aaxl2` 文件上的问题标记）：调用 `errManager` 生成

---

## 4. 资源预算分析（org.osate.analysis.resource.budgets）

### 4.1 模块结构

```
org.osate.analysis.resource.budgets/
├── ResourceBudgetPlugin.java         OSGi Activator
├── busload/
│   ├── BusLoadModelBuilder.java      将系统实例转换为 BusLoadModel
│   └── NewBusLoadAnalysis.java       新版总线负载分析（@since 4.0）
├── handlers/
│   ├── BoundResourceAnalysisHandler.java      已绑定资源分析 Handler
│   ├── BusLoadAnalysisHandler.java            旧版总线负载 Handler
│   ├── NewBusLoadAnalysisHandler.java         新版总线负载 Handler
│   ├── NotBoundResourceAnalysisHandler.java   未绑定资源分析 Handler
│   └── PowerAnalysisHandler.java              电源分析 Handler
└── logic/
    ├── AbstractLoggingAnalysis.java   日志输出基类
    ├── AbstractResourceAnalysis.java  资源分析抽象基类（含预算汇总逻辑）
    ├── BoundResourceAnalysis.java     已绑定资源分析（MIPS + 内存）
    ├── BusLoadAnalysis.java           旧版总线负载分析
    ├── NotBoundResourceAnalysis.java  未绑定（预算汇总）分析
    └── PowerAnalysis.java             电源消耗分析
```

### 4.2 AbstractResourceAnalysis 关键方法

`AbstractResourceAnalysis` 是所有资源分析逻辑的基类，定义了三种资源类型枚举：

```java
protected enum ResourceKind {
    MIPS, RAM, ROM, Memory
}
```

核心方法 `sumBudgets(ci, rk, som, prefix)` 递归汇总组件树中的资源预算：

```java
protected double sumBudgets(ComponentInstance ci, ResourceKind rk,
        final SystemOperationMode som, String prefix) {
    // 跳过非活跃组件（不在当前 SOM 中）
    if (!ci.isActive(som)) return 0.0;
    // 对每个子组件递归汇总
    // 纯硬件组件（BUS/PROCESSOR/MEMORY）返回 -1 标记
    // 线程的 MIPS = 通过 getThreadExecutioninMIPS() 从执行时间和处理器速度换算
}
```

**MIPS 计算链（getThreadExecutioninMIPS）：**

```
1. 尝试读取 SEI::Instructions_Per_Dispatch（单位 MIPD）→ 除以 Period
2. 若无，则：执行时间(s) / 周期(s) × 参考处理器 MIPS 容量
   - 参考处理器来自 Timing_Properties::Reference_Processor
   - 或绑定的物理处理器（Deployment_Properties::Actual_Processor_Binding）
```

### 4.3 BoundResourceAnalysis：处理器与内存负载

`BoundResourceAnalysis.analysisBody()` 在每个 SOM 下顺序执行三项检查：

**处理器负载检查（checkProcessorLoad）：**

```java
// 读取处理器 MIPS 容量
double MIPScapacity = getMIPSCapacityInMIPS(curProcessor, 0.0);
// getMIPSCapacityInMIPS 优先读取 SEI::MIPSCapacity，
// 降级到 Timing_Properties::Processor_Capacity

// 获取绑定到该处理器的所有软件组件
EList<ComponentInstance> boundComponents =
    InstanceModelUtil.getBoundSWComponents(curProcessor);

// 对每个绑定组件汇总 MIPS 预算
double totalMIPS = 0.0;
for (ComponentInstance bci : boundComponents) {
    totalMIPS += sumBudgets(bci, ResourceKind.MIPS, som, "");
}

// 超出容量则 ERROR，无预算则 WARNING，正常则 INFO
```

**内存负载检查（checkMemoryLoad）：**

读取三种内存容量属性（优先级依次降低）：
- `SEI::RAMCapacity` / `SEI::ROMCapacity`（KB）
- `Memory_Properties::Memory_Size`（KB）

实际使用量通过 `getMemoryUseActual()` 计算，RAM 部分 = `Data_Size + Heap_Size + Stack_Size`，ROM 部分 = `Code_Size`（均支持 `Source_xxx` 回退属性）。

### 4.4 NewBusLoadAnalysis：新版总线负载分析

该类（`@since 4.0`）采用两阶段 EMF Switch 遍历：

**第一阶段（PreOrder）`BandwidthAndResultPreOrderSwitch`：**
- 建立 `Result` 树结构，与 `BusLoadModel` 的元素一一对应
- 计算并传播数据开销（`Memory_Properties::Data_Size` 指定的协议头开销）

**第二阶段（PostOrder）`CapacityAndBudgetPostOrderSwitch`：**
- `caseConnection()`：从 `SEI::BandwidthBudget` 读取预算，
  从 `Memory_Properties::Data_Size × 消息速率` 计算实际带宽
- `caseBusOrVirtualBus()`：从 `SEI::BandwidthCapacity` 读取容量，
  汇总子元素（虚拟总线 + 连接 + 广播源）的预算和实际用量
- 报告格式：`result.values = [capacity, budget, requiredBudget, actual, #vb, #conn, #broadcast, overhead_bytes]`

**消息速率解析优先级（getOutgoingMessageRatePerSecond）：**

```
1. SEI::Data_Rate（per second）
2. SEI::Message_Rate（per second）
3. Communication_Properties::Output_Rate 的 MaxDataRate 字段
   - 若单位为 PerDispatch：除以组件周期换算为 PerSecond
4. 包含组件的 Timing_Properties::Period 倒数（1/period）
```

### 4.5 读取的主要 AADL 属性汇总

| 属性 | 用途 |
|------|------|
| `SEI::MIPSCapacity` | 处理器 MIPS 容量 |
| `SEI::MIPSBudget` | 组件 MIPS 预算 |
| `SEI::RAMCapacity` / `SEI::ROMCapacity` | 内存容量 |
| `SEI::RAMBudget` / `SEI::ROMBudget` | 内存预算 |
| `SEI::RAMActual` / `SEI::ROMActual` | 实际内存使用 |
| `SEI::BandwidthCapacity` | 总线带宽容量（KB/s）|
| `SEI::BandwidthBudget` | 总线/连接带宽预算（KB/s）|
| `SEI::Instructions_Per_Dispatch` | 线程每次调度的指令数（MIPD）|
| `Deployment_Properties::Actual_Memory_Binding` | 内存绑定 |
| `Deployment_Properties::Actual_Processor_Binding` | 处理器绑定 |
| `Memory_Properties::Memory_Size` | 通用内存大小 |
| `Memory_Properties::Data_Size` | 数据大小（用于带宽计算） |
| `Memory_Properties::Heap_Size` / `Stack_Size` / `Code_Size` | 详细内存细分 |
| `Timing_Properties::Processor_Capacity` | 处理器速度（降级读取）|
| `Timing_Properties::Reference_Processor` | 参考处理器（MIPS 换算）|
| `Communication_Properties::Output_Rate` | 输出速率（记录类型）|

---

## 5. 架构分析（org.osate.analysis.architecture）

### 5.1 关键检查类

`org.osate.analysis.architecture` 包下包含以下独立检查类，每个类对应一类具体的架构约束验证：

**PortConnectionConsistency（端口连接一致性）**

```java
// 继承 AadlProcessingSwitchWithProgress，通过 InstanceSwitch 遍历所有 ConnectionInstance
public class PortConnectionConsistency extends AadlProcessingSwitchWithProgress {
    @Override
    public void initSwitches() {
        instanceSwitch = new InstanceSwitch<String>() {
            @Override
            public String caseConnectionInstance(ConnectionInstance conni) {
                checkPortConsistency(srcFI, dstFI, conni);
                return DONE;
            }
        };
    }
}
```

`checkPortConsistency()` 检查以下 6 项约束：
1. `Data_Model::Data_Size`：源端和目标端的数据大小（字节）必须一致
2. `Communication_Properties::Output_Rate` / `Input_Rate`：数据率单位必须相同
3. 最大输出速率不得超过最大输入速率（否则会丢包）
4. 最小输出速率不得低于最小输入速率
5. `SEI::Data_Rate`（消息速率）：源端和目标端必须一致
6. `Data_Model::Base_Type`（基类型）和 `Measurement_Unit`（度量单位）：两端必须匹配

**ConnectionBindingConsistency（连接绑定一致性）**

检查 `Deployment_Properties::Actual_Connection_Binding` 指定的绑定是否满足连接对通信协议的 `Required_Virtual_Bus_Class` 要求。

**PropertyTotals（属性汇总）**

计算并验证系统的重量属性：
- 读取 `SEI::GrossWeight`（毛重）、`SEI::NetWeight`（净重）、`SEI::WeightLimit`（重量限制）
- 递归汇总所有子组件的重量，以 `kg` 为单位
- 同时计算 `SEI::Price` 属性的价格汇总
- 适用的组件类别：`SYSTEM, PROCESSOR, MEMORY, BUS, DEVICE, ABSTRACT`
- 忽略的组件类别：`PROCESS, VIRTUAL_BUS, VIRTUAL_PROCESSOR, THREAD, THREAD_GROUP`

**CheckBindingConstraints（绑定约束检查）**

验证已设置的 Actual 绑定是否满足 Allowed 绑定约束（`Allowed_Processor_Binding` 等）。

**FindUnusedClassifiersAnalysis（查找未使用分类器）**

```java
// 使用 AadlFinder 扫描工作空间中所有 Classifier 声明与引用
public final class FindUnusedClassifiersAnalysis {
    public static final String MARKER_TYPE =
        "org.osate.analysis.architecture.UnusedClassifierMarker";

    // 算法三步骤：
    // 1. getAllObjectsOfTypeInCollection() 收集指定包中的所有 Classifier URI
    // 2. getAllReferencesToTypeInWorkspace() 收集工作空间中所有对 Classifier 的引用
    // 3. 对比两个集合，无引用者标记为 WARNING
}
```

支持快速修复（`UnusedClassifierMarkerResolution`）：选择"Remove unused classifier"可直接删除未使用的分类器声明。

**ARINC429ConnectionConsistency（ARINC-429 连接一致性）**

针对 ARINC-429 航空总线协议的特定属性检查（独立于通用端口一致性检查）。

### 5.2 plugin.xml 扩展点注册（architecture）

注册了以下 Eclipse 标记类型（Markers）：

```
ModelStatisticsObjectMarker
ConnectionBindingConsistencyObjectMarker
A429ConnectionConsistencyObjectMarker
PortConnectionConsistencyObjectMarker
WeightTotalObjectMarker
CheckBindingConstraintsObjectMarker
UnusedClassifierMarker
```

以及以下 UI 命令，分布在不同菜单位置：

| 命令 ID | 菜单位置 | Handler 类 |
|---------|---------|-----------|
| `checkConnectionBindingConsistency` | semanticChecksMenu | `CheckConnectionBindingConsistency` |
| `portConnectionConsistency` | semanticChecksMenu | `DoPortConnectionConsistency` |
| `checkA429PortConnectionConsistency` | semanticChecksMenu | `CheckA429PortConnectionConsistency` |
| `propertyTotals` | budgetMenu | `DoPropertyTotals` |
| `checkBindingConstraints` | semanticChecksMenu | `CheckBindingConstraints` |
| `findUnusedClassifiers` | modelCleanupMenu | `FindUnusedClassifiers` |

---

## 6. 模式分析（org.osate.analysis.modes）

### 6.1 核心算法：SOM 可达性分析

**ReachabilityAnalyzer（主分析类）**

```java
public final class ReachabilityAnalyzer {
    private ReachabilityConfiguration config;
    private SystemInstance root;
    private Resource graphs;   // 保存模式图的 EMF 资源（.modemodel 文件）

    public AnalysisResult analyzeModel(IProgressMonitor monitor) {
        ModeDomain.clearData();
        // 1. 填充模式域（fillModeDomains）
        //    读取 SEI::Mode_Domain 属性，识别独立的模式域边界
        // 2. 验证模式域（validateModeDomains）
        //    检查模态组件是否都包含在某个模式域中
        // 3. 对每个 ModeDomain 运行可达性分析（d.analyze()）
        // 4. markUnreachableSOMs()：标记不可达的 SOM
        // 5. 写出报告文件（HTML/DOT/SMV）
    }
}
```

**ModeDomain（模式域）**

代表系统中一个独立的模式空间（由 `SEI::Mode_Domain` 属性标记的组件子树）。每个 `ModeDomain` 维护一个模式迁移图（`graph`），通过分析模式迁移触发器（事件端口、内部特征）构建有向图，图的每一级（Level）代表一组可能的并发模式状态。

**不可达 SOM 检测算法：**

```java
private void markUnreachableSOMs() {
    // 从 root.getSystemOperationModes() 取出所有 SOM 的有序列表
    // 通过 generateNodes() 生成所有模式域组合的笛卡尔积
    // 对比 SOM 中 getCurrentModes() 与笛卡尔积中的节点
    // 不在任何合法节点组合中的 SOM 被标记为 INFO 诊断（"xxx is not reachable"）
}
```

**输出格式（由 ReachabilityConfiguration 控制）：**

| 导出器类 | 输出格式 | 说明 |
|---------|---------|------|
| `HtmlExporter` | `.html` | 可交互的模式迁移图，浏览器查看 |
| `DotExporter` | `.dot` | Graphviz 格式，可生成图片 |
| `SmvExporter` | `.smv` | NuSMV 模型检验输入格式 |

报告存放路径：`reports/som-reachability/<模型名>.modemodel`

### 6.2 `ModeDomainAnalyzer` 与 `CrossReferenceUtil`

`ModeDomainAnalyzer` 负责解析单个 `ModeDomain` 内的模式迁移规则，利用 `CrossReferenceUtil` 解析 AADL 模式迁移（`ModeTransition`）中的触发端口跨组件引用关系。

`DefaultedHashMap` 是一个带默认值的 `HashMap` 工具类，用于在构建图时避免频繁的 null 检查。

### 6.3 org.osate.analysis.modes.ui

提供 SOM 可达性分析的 UI 配置对话框，允许用户选择生成哪种格式的报告（HTML/DOT/SMV）以及是否启用各类诊断信息。

---

## 7. 模型统计（org.osate.modelstats）

### 7.1 统计维度

**ComponentCounter.countComponents(SystemInstance si)** 是唯一的统计入口：

```java
public static ElementsCounts countComponents(SystemInstance si) {
    HashMap<ComponentCategory, Integer> categoryCounter = new HashMap<>();
    int connectionCount = 0;
    int eteCount = 0;

    // 将顶层系统放入计数（eAllContents 不含自身）
    categoryCounter.put(si.getCategory(), 1);

    // 遍历实例树中的所有对象
    for (Iterator<EObject> it = si.eAllContents(); it.hasNext();) {
        EObject obj = it.next();
        if (obj instanceof ComponentInstance)
            categoryCounter.merge(((ComponentInstance) obj).getCategory(), 1, Integer::sum);
        else if (obj instanceof EndToEndFlowInstance)
            eteCount++;
        else if (obj instanceof ConnectionInstance)
            connectionCount++;
    }
    return new ElementsCounts(categoryCounter, connectionCount, eteCount);
}
```

**ElementsCounts 提供的统计字段：**

| 方法 | 说明 |
|------|------|
| `getTotalComponentCount()` | 所有组件实例的总数 |
| `getComponentCountMap()` | `Map<ComponentCategory, Integer>`，按类别分类的数量 |
| `getConnectionsCount()` | 连接实例总数 |
| `getEndToEndFlowsCount()` | 端到端流实例总数 |

统计的 `ComponentCategory` 包含全部 AADL 组件类别：`SYSTEM, PROCESS, THREAD, THREAD_GROUP, SUBPROGRAM, SUBPROGRAM_GROUP, DATA, ABSTRACT, BUS, VIRTUAL_BUS, PROCESSOR, VIRTUAL_PROCESSOR, DEVICE, MEMORY`。

### 7.2 UI 展示（org.osate.modelstats.ui）

提供一个 Eclipse 视图，以表格或图形方式展示上述统计结果，支持在 Navigator 中对选定的 `.aaxl2` 文件直接触发统计分析。

---

## 8. 装箱分析（org.osate.analysis.binpacking）

### 8.1 包结构

```
org.osate.analysis.binpacking/
├── EAnalysis/BinPacking/           核心装箱算法库（非 OSATE 包名）
│   ├── SoftwareNode.java           软件任务节点（带宽 + 亲和性约束）
│   ├── HardwareNode.java           硬件处理器节点（容量 + 可用容量）
│   ├── CompositeSoftNode.java      若干 SoftwareNode 的连通子图
│   ├── ProcessingLoad.java         处理负载接口
│   ├── Expansor.java               硬件扩展策略接口
│   ├── LowLevelBinPacker.java      底层装箱接口
│   ├── BaseLowLevelBinPacker.java  含 partition/getDisconnected 的基础实现
│   ├── BFCPBinPacker.java          Best-Fit Connected Partition（最优拟合连通分区）
│   ├── BFBPBinPacker.java          Best-Fit By Priority
│   ├── DFCPBinPacker.java          Decreasing-First Connected Partition
│   ├── DFBPBinPacker.java          Decreasing-First By Priority
│   ├── NFCHoBinPacker.java         Next-Fit w/Compaction and Horizontal Optimization
│   ├── RandomBinPacker.java        随机装箱（基准测试用）
│   ├── RMScheduler.java            Rate-Monotonic 调度器
│   ├── EDFScheduler.java           Earliest Deadline First 调度器
│   ├── DMScheduler.java            Deadline-Monotonic 调度器
│   ├── FixedPriorityPollingScheduler.java  固定优先级轮询调度
│   └── ...（30 余个辅助类）
└── org/osate/analysis/binpacking/
    ├── BinpackingPlugin.java       OSGi Activator
    └── rma/
        ├── RMA.java                Rate-Monotonic Analysis 可调度性测试
        ├── RMASchedulerNew.java    RMA 调度器新版实现
        └── PER_TaskObj.java        周期任务对象（周期、执行时间、优先级）
```

### 8.2 BFCPBinPacker 算法（Best-Fit Connected Partition）

这是最核心的装箱算法。`BFCPBinPacker.solve()` 方法实现流程：

```
1. 计算所有软件任务的总带宽（aggregateBandwidth）
2. 调用 expansor.createInitialHardware() 创建初始硬件节点集合
3. 调用 getDisconnectedComponents() 将任务图分解为连通子图（CompositeSoftNode）
4. 对每个连通子图（按 BandwidthComparator 排序）：
   a. 按 AffinityComparator 对处理器排序（亲和组件优先）
   b. 遍历处理器列表，调用 processor.canAddToFeasibility(composite) 检查约束
   c. 若找到合适处理器：
      - 调用 processor.addIfFeasible(composite) 部署
      - 处理与已部署邻居之间的消息（寻找/创建链路）
   d. 若无合适处理器：
      - 尝试 partition() 将子图分割为更小块（按 BY_BANDWIDTH 策略）
      - 若分割失败，调用 expansor.expandProcessorForModule() 扩展硬件
      - 扩展失败则分析失败（返回 false）
```

### 8.3 RMA 可调度性分析

`org.osate.analysis.binpacking.rma.RMA` 实现经典的响应时间分析（Response Time Analysis）用于 Rate-Monotonic 可调度性测试：

```
对每个任务 i（按优先级从高到低排序）：
  R_i = C_i + sum_j<i(ceil(R_i/T_j) * C_j)
  迭代直到收敛或超过截止期 D_i
```

`PER_TaskObj` 封装任务属性：周期（`T`）、执行时间（`C`）、截止期（`D`）、优先级、名称。

`PER_TaskComparatorByPriority` 用于按优先级排序任务，优先级数值越小优先级越高（符合 POSIX 约定）。

---

## 9. 代码生成检查器（org.osate.codegen.checker）

### 9.1 框架结构

```
org.osate.codegen.checker/
├── Activator.java                      OSGi Activator
├── checks/
│   ├── AbstractCheck.java              检查基类（含 addError/getErrors）
│   ├── DataCheck.java                  数据组件检查
│   ├── MemoryCheck.java                内存组件检查
│   ├── ProcessCheck.java               进程组件检查
│   ├── ProcessorCheck.java             处理器组件检查（含平台特化）
│   └── ThreadCheck.java                线程组件检查
├── handlers/CheckerHandler.java        Eclipse Handler 入口
├── report/ErrorReport.java             错误报告数据对象
└── utils/PokProperties.java            POK 专属属性读取工具
```

`AbstractCheck` 定义接口：

```java
public abstract class AbstractCheck {
    private List<ErrorReport> errors = new ArrayList<>();

    public abstract void perform(SystemInstance si);

    protected void addError(ErrorReport er) { errors.add(er); }
    public List<ErrorReport> getErrors() { return errors; }
}
```

### 9.2 各 Check 类的具体检查项目

**ThreadCheck（线程检查）**

| 检查项 | 属性/条件 | 严重性 |
|--------|----------|--------|
| 调度策略 | `Thread_Properties::Dispatch_Protocol` 必须为 `Periodic` 或 `Sporadic` | ERROR |
| 周期定义 | `Timing_Properties::Period` 不为零 | ERROR |
| 截止期定义 | `Timing_Properties::Deadline` 不为零 | ERROR |
| 子程序 Source_Name | 调用的每个子程序需设置 `Programming_Properties::Source_Name` | ERROR |
| 子程序 Source_Text | 调用的每个子程序需设置 `Programming_Properties::Source_Text`（非空列表） | ERROR |
| 子程序 Source_Language | 调用的每个子程序需设置 `Programming_Properties::Source_Language`（非空列表）| ERROR |

**ProcessCheck（进程检查）**

| 检查项 | 属性/条件 | 严重性 |
|--------|----------|--------|
| 至少一个线程 | 进程必须包含至少一个 `THREAD` 子组件 | ERROR |
| 绑定到虚拟处理器 | `Deployment_Properties::Actual_Processor_Binding` 目标必须为 `VIRTUAL_PROCESSOR` | ERROR |
| 绑定到内存 | `Deployment_Properties::Actual_Memory_Binding` 必须已设置 | ERROR |

**ProcessorCheck（处理器检查，含平台特化）**

该类通过 `vxworks()`、`deos()`、`pok()` 方法判断当前目标平台，执行平台特定检查：

| 平台 | 检查项 | 属性 |
|------|--------|------|
| VxWorks / Deos | 处理器调度表 | `ARINC653::Module_Schedule` 不为空 |
| VxWorks | 每个虚拟处理器的源名称 | `Programming_Properties::Source_Name` |
| Deos | 虚拟处理器执行时间 | `Timing_Properties::Execution_Time` |
| Deos | 虚拟处理器周期 | `Timing_Properties::Period` |
| POK | 槽分配一致性 | `POK::Slots_Allocation` 包含所有虚拟处理器 |
| POK | 槽数量一致性 | `POK::Slots_Allocation` 与 `POK::Slots` 的元素数相同 |

`PokProperties.getSlotsAllocation(cpu)` 读取 `POK::Slots_Allocation` 属性（类型为虚拟处理器引用列表）。

---

## 10. 模型导入（org.osate.importer + org.osate.importer.simulink）

### 10.1 通用导入框架（org.osate.importer）

**中间模型（org.osate.importer.model）**

```java
// 导入后的中间表示，与具体工具无关
public class Component implements Comparable<Object> {
    public enum ComponentType {
        EXTERNAL_INPORT, EXTERNAL_OUTPORT, BLOCK, REFERENCE, UNKNOWN
    }
    public enum PortType {
        BOOL, FLOAT, DOUBLE, INT, UNKNOWN
    }

    private String name;          // 组件名称（保留原始名称）
    private ComponentType type;   // 组件类型
    private int identifier;       // 工具内部 ID（如 Simulink SID）
    private List<Connection> connections;
    private List<Component> subEntities;
    private Component parent;
    private List<StateMachine> stateMachines;  // 关联的状态机
    private List<Component> inports;   // 有序入端口列表
    private List<Component> outports;  // 有序出端口列表
}

// 状态机模型
public class StateMachine {
    // 包含 State 列表 + Transition 列表
}

public class Model {
    // 顶层容器：持有所有 Component 和 Connection
}
```

**Utils.toAadl(String)** 将工具中的名称转换为合法的 AADL 标识符（如将空格、斜杠等替换为下划线）。

### 10.2 Simulink 导入器（org.osate.importer.simulink）

**导入流程**

`ImportModel.processComponents(node, model, currentComponent)` 解析 Simulink `.mdl`/`.slx` XML 文件的组件结构：

```java
// XML 节点映射规则（BlockType → ComponentType）
"Inport"    → ComponentType.EXTERNAL_INPORT
"From"      → ComponentType.EXTERNAL_INPORT
"Outport"   → ComponentType.EXTERNAL_OUTPORT
"SubSystem" → ComponentType.BLOCK（递归处理子系统）
"Reference" → ComponentType.REFERENCE（若 SourceFile 为 "simulink" 则跳过内置块）
```

`ImportModel.processNodeConnections(node, model)` 解析 `<Line>` 元素，提取 `<p Name="Src">` 和 `<p Name="Dst">` 的连接端点，并解析 `<Branch>` 分支（一对多连接）。

**连接端点解析（addConnection）：**

```java
// 从 "blockIndex/portIndex" 格式的字符串中提取连接信息
srcBlockIndex = Utils.getConnectionPointInformation(src, CONNECTION_FIELD_BLOCK_INDEX);
srcInterfaceIndex = Utils.getConnectionPointInformation(src, CONNECTION_FIELD_PORT_INDEX);
// 通过 model.findComponentById(blockIndex) 查找对应组件
// 通过 component.getOutPort(portIndex - 1) 访问有序端口
```

**AadlProjectCreator（AADL 代码生成）**

`org.osate.importer.simulink.generator.AadlProjectCreator` 将中间模型转换为 AADL 包文件：
- `BLOCK` 类型的组件 → AADL `system` 组件，内部端口生成为 `feature`
- `REFERENCE` 类型 → 在 AADL 包中引用外部已知的系统类型
- `Connection` → AADL 连接声明

**StateFlow 导入（ImportStateFlow）**

`org.osate.importer.simulink.ImportStateFlow` 解析 Simulink 的 StateFlow（状态图）部分，将其转换为 `StateMachine` 中间模型，进而可以生成 AADL 行为附件（Behavior Annex）。

`StateFlowInstance` 追踪 StateFlow 中组件的实例关系，用于层次化状态机的正确嵌套生成。

---

## 11. 扩展开发指南

### 11.1 添加自定义分析器

基于 OSATE2 分析框架添加新分析的标准步骤：

**步骤 1：创建 Eclipse Plugin 项目，在 `plugin.xml` 中注册命令和 Handler**

```xml
<extension point="org.eclipse.ui.commands">
  <command id="com.example.myanalysis.run"
           categoryId="org.osate.analysis.category"
           name="My Custom Analysis"/>
</extension>

<extension point="org.eclipse.ui.handlers">
  <handler class="com.example.myanalysis.handlers.MyAnalysisHandler"
           commandId="com.example.myanalysis.run">
    <enabledWhen>
      <reference definitionId="org.osate.ui.definition.isInstanceFileOrSystemInstanceSelected"/>
    </enabledWhen>
  </handler>
</extension>

<!-- 若需被 ALISA 调用 -->
<extension point="org.osate.pluginsupport.registeredjavaclasses">
  <class path="com.example.myanalysis.MyAnalysisSwitch"/>
</extension>
```

**步骤 2：实现 Handler**

```java
public class MyAnalysisHandler
        extends AbstractInstanceOrDeclarativeModelModifyHandler {

    @Override
    protected String getActionName() { return "My Analysis"; }

    @Override
    public String getMarkerType() {
        return "com.example.myanalysis.MyMarker";
    }

    @Override
    protected boolean canAnalyzeDeclarativeModels() { return false; }

    @Override
    protected void analyzeInstanceModel(IProgressMonitor monitor,
            AnalysisErrorReporterManager errManager,
            SystemInstance root, SystemOperationMode som) {
        MyAnalysisSwitch sw = new MyAnalysisSwitch(monitor, root);
        sw.processPostOrderComponentInstance(root);
        // 收集 sw.getResults() 并报告
    }
}
```

**步骤 3：实现 Switch（访问者模式）**

```java
public class MyAnalysisSwitch extends AadlProcessingSwitchWithProgress {

    public MyAnalysisSwitch(IProgressMonitor monitor, SystemInstance root) {
        super(monitor, PROCESS_PRE_ORDER_ALL);
    }

    @Override
    protected void initSwitches() {
        instanceSwitch = new InstanceSwitch<String>() {
            @Override
            public String caseComponentInstance(ComponentInstance ci) {
                // 读取 AADL 属性
                double period = PropertyUtils.getScaled(
                    TimingProperties::getPeriod, ci, TimeUnits.MS).orElse(0.0);
                // 执行检查逻辑
                if (someConditionFails(ci)) {
                    error(ci, "Violation message");
                }
                return DONE;
            }
        };
    }
}
```

**步骤 4：使用 org.osate.result 返回结构化结果**

```java
// 创建分析结果（用于 ALISA 集成和 .result 文件）
AnalysisResult ar = ResultUtil.createAnalysisResult("My Analysis", root);
Result r = ResultUtil.createResult("component check", ci, ResultType.SUCCESS);
ResultUtil.addRealValue(r, computedValue, "ms");
ResultUtil.addError(r, ci, "Actual exceeds budget");
ar.getResults().add(r);
```

### 11.2 读取 AADL 属性的推荐方式

OSATE2 推荐使用 `PropertyUtils` 工具类（位于 `org.osate.pluginsupport.properties`）读取属性：

```java
// 读取带单位的实数属性
double period = PropertyUtils.getScaled(
    TimingProperties::getPeriod, element, TimeUnits.MS).orElse(0.0);

// 读取带单位的区间属性（返回 RealRange）
RealRange execTime = PropertyUtils.getScaledRange(
    TimingProperties::getComputeExecutionTime, element, TimeUnits.MS)
    .orElse(RealRange.ZEROED);

// 读取枚举属性
EnumerationLiteral protocol = GetProperties.getDispatchProtocol(component);

// 读取列表属性（如绑定）
List<InstanceObject> bindings = DeploymentProperties
    .getActualConnectionBinding(connOrVB).orElse(Collections.emptyList());
```

### 11.3 分析结果的多模式（SOM）处理

对于需要在每个系统操作模式下独立分析的场景：

```java
// 使用 SOMIterator 遍历所有 SOM
final SOMIterator soms = new SOMIterator(root);
while (soms.hasNext()) {
    final SystemOperationMode som = soms.nextSOM();
    // 执行单 SOM 下的分析
    // 通过 ci.isActive(som) 判断组件是否在该 SOM 中活跃
}
```

### 11.4 关键依赖关系

| 依赖 Bundle | 用途 |
|-------------|------|
| `org.osate.aadl2` | AADL 元模型（Ecore） |
| `org.osate.aadl2.instantiation` | 实例模型创建 |
| `org.osate.result` | 分析结果数据结构 |
| `org.osate.pluginsupport` | `PropertyUtils`、`PropertyUtils` 工具类 |
| `org.osate.contribution.sei` | SEI 属性集（`SEI::MIPSCapacity` 等）|
| `org.osate.aadl2.contrib` | 标准 AADL 属性集贡献（`TimingProperties`、`DeploymentProperties` 等）|
| `org.osate.ui` | `AbstractAaxlHandler` 等 UI 基类 |
