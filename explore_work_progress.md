# OSATE2 分析工作进度

> 本文档记录对 osate2 项目的分析进度，作为后续持续分析的大纲。
> 最后更新：2026-02-28（第二次更新）

---

## 一、已完成的分析文档

### 1.1 总览文档（高层次概览）

| 文档 | 行数 | 深度评估 |
|------|-----:|---------|
| [01_项目概述.md](01_项目概述.md) | 222 | ✅ 完整：OSATE/AADL 历史、定位、应用领域、目标用户 |
| [02_代码结构分析.md](02_代码结构分析.md) | 322 | ✅ 完整：目录结构、模块划分、依赖关系 |
| [03_核心功能模块.md](03_核心功能模块.md) | 404 | ✅ 完整：文本编辑、图形编辑、分析、验证功能概述 |
| [04_技术架构.md](04_技术架构.md) | 474 | ✅ 完整：技术栈、四层架构、OSGi/扩展点、构建系统 |
| [05_开发指南.md](05_开发指南.md) | 565 | ✅ 完整：环境搭建、构建命令、扩展开发入门 |

### 1.2 源码级设计文档

| 文档 | 行数 | 深度评估 |
|------|-----:|---------|
| [06_AADL2元模型设计.md](06_AADL2元模型设计.md) | 282 | ✅ 中等：Ecore 类层次、Type-Implementation 分离、Feature/Connection/Flow/Property 建模 |
| [07_实例化引擎设计.md](07_实例化引擎设计.md) | 254 | ✅ 中等：InstantiateModel 八阶段流程、连接多段追踪、SOM 枚举 |
| [08_Xtext语言集成设计.md](08_Xtext语言集成设计.md) | 300 | ✅ 中等：非标准 Xtext 架构、双语法层次、多层作用域、Annex 解析集成 |
| [09_分析框架设计.md](09_分析框架设计.md) | 315 | ✅ 较深：Handler 层次、触发流程、Result 元模型、流延迟算法、Marker 生成 |
| [10_EMV2错误模型与图形编辑器设计.md](10_EMV2错误模型与图形编辑器设计.md) | 474 | ✅ 较深：EMV2 语法/类型系统/状态机、故障树生成、GE BOH 模式、双向同步 |
| [11_ALISA验证框架与Annex支持设计.md](11_ALISA验证框架与Annex支持设计.md) | 799 | ✅ 深入：ReqSpec/Verify/Alisa/Assure 四层流水线、Annex 注册表体系 |
| [12_行为附件设计.md](12_行为附件设计.md) | 334 | ✅ 中等：ANTLR 4.4 语法、BehaviorAnnex 元模型、状态机验证、GE 集成 |
| [13_UI框架与导航设计.md](13_UI框架与导航设计.md) | 315 | ✅ 中等：AADL 导航器、属性视图、向导框架、命令/Handler 体系 |
| [14_构建系统与测试设计.md](14_构建系统与测试设计.md) | 619 | ✅ 较深：Maven/Tycho 层次、目标平台、测试基础设施、CI/CD、产品打包 |

### 1.3 子项目 README（源码级）

| 子项目 | 行数 | 深度评估 |
|--------|-----:|---------|
| [core/README.md](core/README.md) | 1303 | ✅ 深入：AADL2 元模型 API、实例模型、属性系统、Result 元模型、Xtext 组件 |
| [analyses/README.md](analyses/README.md) | 1032 | ✅ 深入：Handler 层次、流延迟算法、资源预算、安全分析框架 |
| [alisa/README.md](alisa/README.md) | 1204 | ✅ 深入：六层 DSL 架构、AssureProcessor 执行流程、EMF 结果模型、reqtrace 状态 |
| [emv2/README.md](emv2/README.md) | 302 | ✅ 较深：Xtext 语法/类型系统/状态机/条件表达式、实例化流水线、传播图、故障树算法 *(本次更新)* |
| [ge/README.md](ge/README.md) | 251 | ✅ 较深：四层架构、BOH 模式完整实现、图表数据模型、双向同步、渲染管线、扩展点 *(本次更新)* |

### 1.4 新增源码级专题文档

| 文档 | 行数 | 深度评估 |
|------|-----:|---------|
| [15_属性系统深入分析.md](15_属性系统深入分析.md) | 530 | ✅ 深入：PropertyType/Expression 层次、求值流程、PropertyUtils API、模态值处理、自定义属性集开发 *(本次新建)* |

---

## 二、浅层完成——有"下一步分析"待项

这些子项目 README 已有骨架，但停留在概述层面，未深入源码实现。

### 2.1 emv2/README.md ✅ 已完成整合（302 行，较深）

本次已整合 10 号文档中的全部 EMV2 源码级内容，覆盖：
- [x] ErrorModel.xtext 语法结构（顶层/类型系统/传播声明/状态机/条件表达式）
- [x] 错误模型实例化流水线（EMV2AnnexInstantiator 12 步流程）
- [x] 传播图（PropagationGraph）Ecore 模型与构建方式
- [x] 故障树生成算法（反向遍历/门优化/概率计算）
- [x] 正向错误影响分析（PropagateErrorSources）
- [x] 完整分析链与依赖关系

**仍可深入（可选）：**
- [ ] 概率计算引擎的具体数学实现（OR/AND 门的概率聚合公式）
- [ ] 与安全标准工具链的集成（PRISM/NuSMV）

### 2.2 ge/README.md ✅ 已完成整合（251 行，较深）

本次已整合 10 号文档中的全部 GE 源码级内容，覆盖：
- [x] 四层架构（BO Layer → AgeDiagram → GefAgeDiagram → AgeEditor）
- [x] BOH 模式：Handler 接口、注册/发现机制、各域 Handler 覆盖范围
- [x] 图表数据模型（diagram.ecore）结构与 XMI 序列化
- [x] 双向同步：正向（AADL→图）和反向（图→AADL）机制
- [x] 渲染管线：StyleCalculator→StyleToFx→FxStyleApplier→场景图
- [x] 场景节点类型、布局系统（ELK 集成）
- [x] Eclipse 编辑器集成与扩展点

**仍可深入（可选）：**
- [ ] 实验性分支（`osate-ge.experimental.*`）的功能与状态
- [ ] 自定义 BOH 开发的完整实战示例

### 2.3 ba/README.md（356 行，偏浅）

已有内容：语法概述、状态机结构（12 号文档已有源码级内容，需整合）

**待深入的源码分析：**
- [ ] ANTLR 语法文件与解析器实现细节
- [ ] 代码生成目标（生成到哪种目标语言/中间格式）
- [ ] 与形式化验证工具（BLESS、SPIN）的集成点
- [ ] 调度语义实现（Dispatch Condition 解析）
- [ ] BA 在 GE 中的图形可视化实现

### 2.4 releng/README.md（405 行，中等）

已有内容：构建结构，与 14 号文档大部分重合

**待深入分析：**
- [ ] P2 更新站点发布流程（category.xml 结构、版本策略）
- [ ] GitHub Actions CI 配置文件（`.github/workflows/`）的实际内容
- [ ] 多平台产品打包产物结构
- [ ] SonarCloud 集成配置

### 2.5 examples/README.md（347 行，偏浅）

已有内容：示例分类概述

**待深入分析：**
- [ ] 逐一列出所有 75 个 AADL 示例文件的功能分类
- [ ] 识别覆盖各子系统功能点的代表性示例
- [ ] 梳理适合入门学习的最小示例集

### 2.6 setup/README.md（346 行，偏浅）

已有内容：Oomph 机制概述

**待深入分析：**
- [ ] `osate2.setup` 文件的具体任务链与变量定义
- [ ] Checkstyle 规则详情（哪些规则、严格程度）
- [ ] 历史版本配置文件的变迁

### 2.7 tools/README.md（403 行，中等）

已有内容：属性代码生成器概述

**待深入的源码分析：**
- [ ] `PropertiesCodeGen`（Xtend 模板）的完整生成逻辑
- [ ] 支持的属性类型完整映射表（AADL 类型 → Java 类型）
- [ ] Eclipse 集成：触发时机、增量构建支持

---

## 三、尚未开始的分析专题

这些主题在现有文档中仅被零散提及，尚无专门分析。

### 3.1 属性系统深入分析 ✅ 已完成（530 行）

**文档**：[15_属性系统深入分析.md](15_属性系统深入分析.md)

- [x] PropertyType / PropertyExpression 完整层次结构
- [x] Property / PropertyAssociation / ModalPropertyValue 详细结构
- [x] 完整属性求值流程（acceptsProperty → PropertyAcc → NamedElementImpl）
- [x] 循环引用检测机制（ThreadLocal 栈）
- [x] 模态属性值（ModalPropertyValue）按 SOM 的选择算法
- [x] PropertyUtils 全 API（标量/带单位数值/范围/列表/异常类型）
- [x] GetProperties 全 API（定义查找/标准属性集强类型 getter）
- [x] 自定义属性集开发 + tools 代码生成器集成指南
- [x] 完整使用示例（整数/带单位实数/列表/枚举/模态属性）

### 3.2 Resolute 约束验证语言（优先级：高）

**建议新建文档**：`16_Resolute验证语言.md`

- [ ] `org.osate.resolute` 插件的语言设计（规则/函数/计算/证明）
- [ ] 与 ALISA Verify DSL 的集成方式（`VerificationMethod.resolute`）
- [ ] 内置函数库（`member`、`sum`、`forall`、`exists` 等）
- [ ] 用户自定义规则的编写与注册

### 3.3 模型切片与 ARINC653（优先级：中）

**建议新建文档**：`17_模型切片与ARINC653.md`

- [ ] `org.osate.slicer` 切片算法的实现原理
- [ ] 切片的输入（组件/连接/流）与输出（子图 AADL 模型）
- [ ] `org.osate.codegen.checker` 的验证规则列表
- [ ] ARINC653 属性集定义（`ARINC653.aadl`）与航电建模模式

### 3.4 扩展开发实战指南（优先级：中）

**建议新建文档**：`18_扩展开发实战指南.md`

- [ ] 端到端：新建自定义分析插件的完整步骤（plugin.xml + Handler + Marker）
- [ ] 端到端：新建 Xtext Annex 子语言的完整步骤
- [ ] 扩展 GE 以支持新的图形元素（自定义 BOH）
- [ ] 注册自定义属性集并与代码生成器集成

### 3.5 性能与 EASE 脚本（优先级：低）

**建议新建文档**：`19_性能与EASE脚本.md`

- [ ] 实例化引擎对大型模型（1000+ 组件）的性能分析
- [ ] 属性缓存机制（`CachePropertyAssociationsSwitch`）的工作原理
- [ ] WorkspaceJob 锁粒度对并发分析的影响
- [ ] `ease-scripts/` 目录中脚本的功能与触发方式
- [ ] EASE 与 OSATE 集成点（可自动化的工作流）

---

## 四、分析路线图

```
阶段一：整合已有源码级内容到子项目 README ✅ 部分完成
  ├── emv2/README.md  ✅ 已完成（302 行，较深）
  ├── ge/README.md    ✅ 已完成（251 行，较深）
  └── ba/README.md    ⬜ 待整合（整合 12 号文档中 BA 部分）

阶段二：新开源码级专题分析 ⬜ 进行中
  ├── 15_属性系统深入分析.md      ✅ 已完成（530 行，深入）
  ├── 16_Resolute验证语言.md      ⬜ 待分析（高优先级）
  └── 17_模型切片与ARINC653.md   ⬜ 待分析（中优先级）

阶段三：实践与性能（长期）
  ├── 18_扩展开发实战指南.md      ⬜ 待编写
  └── 19_性能与EASE脚本.md        ⬜ 待分析
```

**下次继续建议从以下任一项开始**：
1. `ba/README.md` 深化——整合 12 号文档中 BA 的 ANTLR 语法、状态机验证、代码生成目标内容
2. `16_Resolute验证语言.md`——读取 `core/org.osate.resolute` 源码，分析规则/函数/证明体系
3. `releng/README.md` 完善——补充 CI 配置文件实际内容

---

## 五、文档深度说明

| 等级 | 标准 |
|------|------|
| **深入** | 覆盖核心类/方法/算法的源码实现，有代码引用和调用链分析，1000 行以上 |
| **较深** | 覆盖主要流程和数据结构，有部分源码示例，400~800 行 |
| **中等** | 覆盖架构设计和关键组件，以文字描述为主，250~400 行 |
| **偏浅** | 仅有功能概述和组件列表，缺乏源码级内容，<300 行 |
