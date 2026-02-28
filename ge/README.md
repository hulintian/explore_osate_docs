# GE - 图形编辑器

GE（Graphical Editor）是 OSATE2 的图形化建模组件，使用 JavaFX + e(fx)clipse 技术栈，通过 **Business Object Handler（BOH）模式** 实现 AADL 模型与图形视图的双向绑定。

源码位置：`osate2/ge/`，987 个 Java 文件，130,898 行（全部手写，无 Xtend），6 个 Eclipse 插件。

---

## 1. 插件组织

| 插件 | 职责 |
|------|------|
| `org.osate.ge` | 核心框架：BOH 注册表、图表数据模型、同步引擎、渲染管线 |
| `org.osate.ge.aadl2` | AADL 元素的 BOH 实现（组件、特征、连接、流、模式） |
| `org.osate.ge.errormodel` | EMV2 错误传播的 BOH 实现 |
| `org.osate.ge.ba` | 行为附件（BA）状态机的 BOH 实现 |
| `org.osate.ge.swt` | SWT 辅助工具（嵌入 JavaFX 到 SWT） |
| `org.osate.ge.tests` | 单元/集成测试 |

---

## 2. 四层架构

```
Business Object Layer     — 领域模型（AADL / EMV2 / BA 元素）
        ↓
Diagram Runtime (AgeDiagram)  — 内存中图表表示（DiagramElement 树）
        ↓
Diagram Rendering (GefAgeDiagram) — JavaFX 场景图生成
        ↓
UI Layer (AgeEditor)      — Eclipse EditorPart 集成
```

---

## 3. Business Object Handler（BOH）模式

核心设计模式——图中所有可见元素都是"业务对象"（Business Object，BO），每种 BO 由一个 Handler 管理。

### 3.1 Handler 接口

```java
BusinessObjectHandler {
    isApplicable(ctx)  → boolean          // 是否管理此类 BO
    getCanonicalReference(ctx)            // 永久标识（从模型根的路径）
    getRelativeReference(ctx)             // 父级相对标识（用于 XMI 持久化）
    getGraphicalConfiguration(ctx)        // 视觉表示：形状/颜色/标签/连接线型
    getName(ctx)       → String           // 大纲视图显示名
    getNameForDiagram(ctx) → String       // 图中标签文字
    canRename() / validateName()          // 重命名能力与验证
    canDelete() / canCopy()               // 编辑能力声明
    getIconId(ctx)                        // 大纲视图图标 ID
}
```

### 3.2 注册与发现

- Handler 通过扩展点注册：`org.osate.ge.businessObjectHandlers`
- **BusinessObjectProvider** 为给定上下文贡献子 BO，注册点：`org.osate.ge.businessObjectProviders`
- 多个 Handler 可竞争同一 BO，第一个 `isApplicable()` 返回 `true` 者胜出

### 3.3 各域 Handler 覆盖范围

| Handler 包 | 覆盖范围 |
|-----------|---------|
| `org.osate.ge.aadl2` | SystemType/Impl、ProcessType/Impl、ThreadType/Impl 等所有 AADL 组件，Feature、Connection、FlowSpec、Mode |
| `org.osate.ge.errormodel` | ErrorPropagation、ErrorBehaviorState、ErrorFlow、CompositeState |
| `org.osate.ge.ba` | BehaviorAnnex、BehaviorState、BehaviorTransition |

---

## 4. 图表数据模型（diagram.ecore）

定义在 `ge/org.osate.ge/model/diagram.ecore`，当前格式版本：7。

```
Diagram
├── formatVersion: int
├── config: DiagramConfiguration
│   ├── type: String              — 图表类型 ID（Package / Structure / Custom）
│   ├── context: CanonicalBusinessObjectReference  — 根 BO（确定图表范围）
│   ├── enabledAadlProperties: String[]            — 显示哪些 AADL 属性
│   └── connectionPrimaryLabelsVisible: Boolean
└── element[*]: DiagramElement    — 树结构（递归嵌套）

DiagramElement
├── uuid: String                  — 图内唯一标识
├── bo: RelativeBusinessObjectReference  — 引用的业务对象
├── position: Point (x, y)
├── size: Dimension (width, height)
├── dockArea: String              — 停靠区域（TOP/BOTTOM/LEFT/RIGHT）
├── bendpoints: Point[]           — 连接弯折点
├── primaryLabelPosition: Point
├── background / outline / fontColor: Color
├── fontSize / lineWidth: Double
├── primaryLabelVisible: Boolean
├── image: String                 — 自定义图片引用
└── showAsImage: Boolean
```

**序列化**：XMI 格式，UUID 标识元素，相对引用（RelativeBusinessObjectReference）保证跨机器可移植性。

---

## 5. 双向同步机制

### 5.1 AADL 模型 → 图表（正向同步）

```
AADL 模型变更（ResourceChangeEvent）
    ↓ ModelChangeNotifier
BusinessObjectTreeUpdater → DiagramUpdater → AgeDiagram → 场景图
        ↓                           ↓
BusinessObjectProvider          为新 BO 创建 DiagramElement
（发现新子 BO）                  为删除的 BO 创建"Ghost Element"（保留布局）
```

**更新流水线**：
1. `DiagramToBusinessObjectTreeConverter`：从当前图表构建 BO 树快照
2. `BusinessObjectTreeUpdater`：基于 Handler/Provider 展开树，识别新增/删除/变更
3. `DiagramUpdater`：协调树与 DiagramElement 列表，维护位置/大小

### 5.2 图表编辑 → AADL 模型（反向同步）

```
用户在图表中编辑（创建/删除/重命名）
    ↓ Handler.canDelete() / Handler.deleter()
    ↓ Handler.canRename() / Handler.renamer()
AADL EMF 模型写操作（在 WorkspaceJob 中执行）
    ↓ 触发正向同步（见上）
图表更新
```

### 5.3 关键服务

| 服务 | 说明 |
|------|------|
| `ReferenceResolutionService` | 解析相对引用到运行时 BO 实例 |
| `ReferenceBuilderService` | 从 BO 重建规范引用 |
| **Ghost Elements** | 隐藏元素保留布局信息，便于 BO 重新出现时恢复位置 |
| **Future Elements** | 为正在创建的 BO 预分配位置 |

---

## 6. 渲染管线

```
AgeDiagram 变更事件
    ↓ DiagramModificationListener
场景节点创建/修改
    ├── GefDiagramElement（元数据容器）
    ├── sceneNode（JavaFX Node）
    └── primaryLabel（LabelNode）
    ↓
StyleCalculator（图形配置 → 渲染器无关 Style 对象）
    ↓
StyleToFx（Style → JavaFX FxStyle）
    ↓
FxStyleApplier（FxStyle → JavaFX 节点属性）
    ↓
场景图更新（FXCanvas 呈现到 SWT）
```

### 6.1 场景节点类型

| 节点类型 | 用途 |
|---------|------|
| `DiagramRootNode` | 所有形状和连接的容器 |
| `ContainerShape` | 组件的复合形状（可嵌套） |
| `DockedShape` | 停靠到容器边缘的特征形状 |
| `ConnectionNode` | 带弯折点的连接线 |
| `LabelNode` | 带样式的文本标签 |
| `FeatureGroupNode` | 特征组的圆形标记 |
| `FlowIndicatorNode` | 流规范指示器 |
| `ImageNode` | 自定义图片元素 |

### 6.2 布局系统

| 机制 | 说明 |
|------|------|
| **Position** | 绝对坐标或相对于父级 |
| **DockArea** | 停靠侧（TOP/BOTTOM/LEFT/RIGHT） |
| **Bendpoints** | 连接路由点列表 |
| **ELK 集成** | 自动布局引擎（`AgeLayoutOptions`），算法可选 |
| **手动定位** | 用户拖动后持久化到 diagram.ecore |

---

## 7. Eclipse 编辑器集成

```
AgeEditor（Eclipse EditorPart）
├── FXCanvas            — 将 JavaFX 场景嵌入 SWT
├── Scene Graph         — JavaFX Group 包含 DiagramRootNode
├── Interaction Layer   — 鼠标/键盘事件处理
├── Tool System         — 调色板工具（选择/形状创建/连接绘制）
├── ModelChangeNotifier — 监听外部 AADL 模型变更
├── AgeContentOutlinePage — 图表元素树形大纲视图
├── TabbedPropertySheetPage — 属性标签页（SWT 嵌套）
└── IOperationHistory   — 撤销/重做（Eclipse 标准命令框架）
```

**图表类型**：
- **Package Diagram**：显示 AADL 包中的分类器
- **Structure Diagram**：显示组件实现的内部结构
- **Custom Diagram**：用户自由组合任意元素

---

## 8. 扩展点

| 扩展点 | 用途 |
|--------|------|
| `org.osate.ge.businessObjectHandlers` | 注册自定义 BO Handler |
| `org.osate.ge.businessObjectProviders` | 贡献子业务对象（决定图中显示哪些子元素） |
| `org.osate.ge.diagramTypes` | 注册新图表类型 |
| `org.osate.ge.images` | 注册图片资源 |
| `org.osate.ge.tools` | 注册自定义调色板工具 |
| `org.osate.ge.contentFilters` | 过滤图中显示的 BO |

---

## 9. 与其他模块的集成

```
AADL 文本（core）
    ↓ EMF ResourceSet
业务对象（AADL 元素）
    ↓ ge.aadl2 Handler
图表 DiagramElement（struct/package diagram）

annex error_model {** ... **}（emv2）
    ↓ EMV2AnnexInstantiator → EMV2AnnexInstance
    ↓ ge.errormodel Handler
错误传播可视化（独立 diagram）

annex behavior_specification {** ... **}（ba）
    ↓ ge.ba Handler
行为状态机可视化
```

---

## 10. 依赖关系

```
core（AADL 元模型、EMF、Xtext） ──→ ge（读取 AADL 元素作为 BO）
emv2（EMV2AnnexInstance）       ──→ ge（ge.errormodel Handler）
ba（BehaviorAnnex）             ──→ ge（ge.ba Handler）
ge ────────────────────────────────→ 无下游依赖（纯展示层）
```
