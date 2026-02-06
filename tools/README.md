# Tools - 开发工具集

## 模块概述

Tools模块包含了OSATE2开发过程中使用的辅助工具，主要是属性代码生成器（Properties Code Generator），用于从AADL属性定义自动生成Java代码，简化属性访问和操作。

## 主要组件

### 属性代码生成器
- **org.osate.propertiescodegen** - 代码生成核心
  - 属性定义解析
  - Java代码生成
  - API生成

- **org.osate.propertiescodegen.ui** - 用户界面
  - Eclipse插件集成
  - 菜单和工具栏
  - 配置界面

- **org.osate.propertiescodegen.tests** - 单元测试
  - 生成器测试
  - API测试
  - 回归测试

## 属性代码生成器

### 功能概述

**目的**：
自动生成类型安全的Java API来访问AADL属性，避免手工编写重复的属性访问代码。

**优势**：
- **类型安全** - 编译时类型检查
- **代码复用** - 避免重复代码
- **维护性** - 属性定义变更时自动重新生成
- **一致性** - 统一的API风格
- **减少错误** - 避免手工编码错误

### 工作原理

#### 输入
AADL属性集定义文件（.aadl）：
```aadl
property set MyProperties is
  MaxLatency: Time applies to (thread);
  BufferSize: Size applies to (data);
  Priority: aadlinteger 1 .. 255 applies to (thread);
end MyProperties;
```

#### 处理过程
1. **解析** - 解析AADL属性定义
2. **分析** - 提取属性元信息
3. **生成** - 生成Java类和方法
4. **格式化** - 代码格式化和优化

#### 输出
Java源代码文件：
```java
public class MyProperties {
    public static Optional<Double> getMaxLatency(InstanceObject io) {
        // Generated code
    }

    public static void setMaxLatency(NamedElement ne, double value) {
        // Generated code
    }

    // More methods...
}
```

### 生成的API

#### Getter方法
**功能**：读取属性值

**方法签名**：
```java
public static Optional<Type> getPropertyName(NamedElement element)
```

**特点**：
- 使用Optional处理属性不存在的情况
- 自动类型转换
- 处理继承的属性值

#### Setter方法
**功能**：设置属性值

**方法签名**：
```java
public static void setPropertyName(NamedElement element, Type value)
```

**特点**：
- 类型检查
- 值范围验证
- 属性关联创建

#### 辅助方法
- **hasProperty** - 检查属性是否存在
- **removeProperty** - 删除属性
- **getPropertyDefinition** - 获取属性定义
- **getAppliesTo** - 获取适用范围

### 支持的属性类型

#### 基本类型
- **aadlboolean** → Boolean
- **aadlstring** → String
- **aadlinteger** → Long
- **aadlreal** → Double

#### 单位类型
- **Time** → Double (converted to milliseconds)
- **Size** → Long (converted to bytes)
- **Data_Rate** → Double (converted to bits per second)

#### 枚举类型
```aadl
Priority_Type: type enumeration (Low, Medium, High);
```
生成为Java枚举：
```java
public enum PriorityType {
    LOW, MEDIUM, HIGH
}
```

#### 范围类型
```aadl
aadlinteger 1 .. 10
```
生成带范围检查的方法。

#### 引用类型
- **Classifier Reference**
- **Component Reference**
- **Connection Reference**

#### 列表类型
```aadl
list of aadlinteger
```
生成List<Long>的访问方法。

#### 记录类型
```aadl
record (
  field1: aadlinteger;
  field2: aadlstring;
)
```
生成包含记录字段的Java类。

### 使用场景

#### 1. 属性访问简化
**手工方式**：
```java
try {
    Property prop = GetProperties.lookupProperty("MyProperties::MaxLatency");
    PropertyExpression expr = element.getPropertyValue(prop);
    // Complex type conversion and error handling...
} catch (Exception e) {
    // Handle errors
}
```

**生成的API**：
```java
Optional<Double> latency = MyProperties.getMaxLatency(element);
latency.ifPresent(value -> {
    // Use value
});
```

#### 2. 分析插件开发
在开发OSATE分析插件时：
- 快速访问所需属性
- 类型安全保证
- 减少模板代码

#### 3. 自定义属性集
为项目特定的属性集生成API：
- 领域特定属性
- 公司标准属性
- 分析工具属性

### 代码生成配置

#### 生成选项
- **输出目录** - 生成代码的目标位置
- **包名** - Java包名称
- **类名** - 生成的类名
- **访问级别** - public/package级别

#### 模板定制
- 代码生成模板
- 注释格式
- 命名约定

### Eclipse集成

#### 菜单和命令
- **右键菜单** - 在.aadl文件上右键选择"Generate Properties Code"
- **工具栏按钮** - 快速访问生成功能
- **快捷键** - 键盘快捷方式

#### 自动化
- **构建集成** - 在构建过程中自动生成
- **文件监听** - 属性文件变更时自动重新生成
- **增量生成** - 只重新生成变更的部分

#### 问题报告
- **错误标记** - 生成错误在问题视图显示
- **警告** - 潜在问题提示
- **快速修复** - 某些问题的自动修复建议

### 生成代码质量

#### 代码风格
- 遵循Java编码规范
- 符合Checkstyle要求
- 适当的注释

#### JavaDoc
自动生成的JavaDoc包含：
- 方法说明
- 参数描述
- 返回值说明
- 属性定义引用

#### 性能优化
- 缓存属性定义
- 延迟加载
- 避免重复查找

### 测试

#### 生成器测试
- 解析测试
- 生成逻辑测试
- 边界情况测试

#### 生成代码测试
- API功能测试
- 类型转换测试
- 错误处理测试

#### 集成测试
- 在实际AADL模型上测试
- 与OSATE核心集成测试

### 扩展和定制

#### 自定义生成器
**扩展点**：
- 自定义类型映射
- 自定义代码模板
- 附加方法生成

**应用**：
- 特定领域需求
- 公司编码标准
- 工具链集成

#### 生成器插件
开发新的代码生成器：
- 生成其他语言代码（C, Python）
- 生成验证代码
- 生成文档

### 最佳实践

#### 属性定义
1. **清晰命名** - 使用描述性名称
2. **适当类型** - 选择合适的属性类型
3. **文档化** - 添加属性描述
4. **组织** - 合理组织属性集

#### 代码生成
1. **版本控制** - 生成的代码是否入库
   - **建议：不入库** - 从源头生成
   - **替代：入库** - 便于审查和调试
2. **定期重新生成** - 保持与定义同步
3. **审查生成代码** - 理解生成的内容
4. **测试** - 测试生成的API

#### 使用生成的API
1. **类型安全** - 利用编译时检查
2. **Optional处理** - 正确处理属性不存在
3. **异常处理** - 处理可能的异常
4. **性能** - 注意频繁访问的性能

### 技术实现

#### 解析技术
- **Xtext解析器** - 重用AADL解析器
- **EMF模型** - 操作AADL元模型
- **访问者模式** - 遍历属性定义

#### 代码生成技术
- **模板引擎** - Xtend或Java模板
- **代码模型** - 构建Java代码模型
- **格式化** - JDT代码格式化器

#### 集成技术
- **Eclipse构建器** - 自动构建集成
- **资源监听** - 文件变更检测
- **作业调度** - 后台任务执行

### 已知限制

#### 不支持的特性
- 某些复杂的属性类型
- 高度嵌套的记录类型
- 某些特殊的属性约束

#### 性能考虑
- 大型属性集生成时间
- 内存使用

### 未来改进

#### 计划功能
- 更多类型支持
- 更好的错误报告
- 增量生成优化
- 其他语言支持

#### 社区贡献
- 开源扩展
- 模板共享
- 最佳实践分享

### 相关工具

#### 类似工具
- EMF代码生成
- Xtext生成器
- JAXB代码生成

#### 集成工具
- Maven插件（可能）
- Gradle插件（未来）
- 命令行工具

## 其他工具

### 潜在的工具
Tools模块可以扩展包含：
- 模型验证工具
- 转换工具
- 导入导出工具
- 文档生成工具

## 依赖关系

### 依赖模块
- **core** - AADL核心模型和解析
- Eclipse JDT - Java代码生成
- Xtext - 语言工具

### 被依赖
- 其他OSATE模块可能使用生成的属性API
- 自定义插件使用代码生成器

## 配置和使用

### 首选项设置
- 生成选项
- 输出位置
- 命名约定

### 项目配置
- 启用/禁用自动生成
- 指定属性集
- 配置构建器

## 文档资源

### 用户指南
- 如何使用代码生成器
- 生成的API参考
- 常见问题解答

### 开发者指南
- 扩展代码生成器
- 贡献新功能
- 测试指南

## 下一步分析

- [ ] 深入研究代码生成器实现机制
- [ ] 分析模板引擎的使用
- [ ] 研究支持的属性类型完整列表
- [ ] 探索自定义代码生成器开发
- [ ] 分析生成代码的最佳实践
- [ ] 研究与构建系统的集成
- [ ] 探索性能优化策略
- [ ] 分析实际使用案例
