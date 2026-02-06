# Core 核心模块

## 模块概述

Core模块是OSATE2的核心基础模块，提供了AADL (Architecture Analysis & Design Language) 的核心功能和基础设施。这是整个OSATE2项目的基础，其他所有模块都依赖于它。

## 主要组件

### AADL核心组件
- **org.osate.aadl2** - AADL2元模型的核心实现
- **org.osate.aadl2.edit** - AADL2模型的编辑支持
- **org.osate.aadl2.contrib** - AADL属性集贡献
- **org.osate.aadl2.modelsupport** - 模型支持工具

### 实例化支持
- **org.osate.aadl2.instantiation** - AADL模型实例化引擎
- **org.osate.aadl2.instance.textual** - 实例模型的文本表示
- **org.osate.aadl2.instance.textual.ui** - 实例模型文本界面
- **org.osate.aadl2.instance.ui** - 实例模型用户界面

### 编辑器和UI
- **org.osate.aadl2.model.editor** - 模型编辑器
- **org.osate.ui** - 用户界面框架
- **org.osate.workspace** - 工作空间管理

### Xtext支持
- **org.osate.xtext.aadl2** - 基于Xtext的AADL2语法支持
- **org.osate.xtext.aadl2.properties** - AADL属性语法支持
- **org.osate.xtext.aadl2.ui** - Xtext编辑器UI

### 插件和扩展
- **org.osate.pluginsupport** - 插件支持框架
- **org.osate.annexsupport** - 附件（Annex）支持框架

### 第三方库
- **org.jgrapht** - 图论算法库

## 技术架构

### 元模型层
Core模块实现了完整的AADL2元模型，基于EMF (Eclipse Modeling Framework)：
- 组件（Component）
- 连接（Connection）
- 特性（Feature）
- 属性（Property）
- 包（Package）

### 解析器层
使用Xtext框架实现的AADL语言解析器：
- 词法分析
- 语法分析
- 语义验证
- 代码补全

### 实例化引擎
将声明式AADL模型转换为可分析的实例模型：
- 组件实例化
- 连接解析
- 模式展开
- 属性值计算

## 关键功能

1. **AADL语言支持**
   - 完整的AADL2.3语法支持
   - 语法高亮和代码补全
   - 实时语法检查和错误提示

2. **模型验证**
   - 语义一致性检查
   - 类型检查
   - 约束验证

3. **模型实例化**
   - 自动创建系统实例
   - 连接绑定解析
   - 模式展开

4. **属性管理**
   - 预定义属性集
   - 自定义属性支持
   - 属性值继承和覆盖

## 依赖关系

### 外部依赖
- Eclipse Platform
- EMF (Eclipse Modeling Framework)
- Xtext
- JGraphT

### 被依赖
几乎所有其他OSATE2模块都依赖Core模块，包括：
- analyses (分析模块)
- emv2 (错误模型)
- alisa (验证框架)
- ge (图形编辑器)
- ba (行为附件)

## 开发指南

### 构建系统
- Maven/Tycho构建
- 版本: 2.18.0-SNAPSHOT

### 关键包结构
```
org.osate.aadl2/
├── model/           # EMF元模型定义
├── src/             # Java源代码
│   └── org/osate/aadl2/  # 核心API
└── META-INF/        # 插件元数据
```

### 扩展点
Core模块提供了多个扩展点供其他模块使用：
- Annex解析器注册
- 属性集贡献
- 验证器扩展
- UI贡献

## 相关文档

- [AADL标准规范](http://www.aadl.info/)
- [Eclipse Modeling Framework](https://www.eclipse.org/modeling/emf/)
- [Xtext文档](https://www.eclipse.org/Xtext/documentation/)

## 下一步分析

- [ ] 深入分析AADL2元模型结构
- [ ] 研究实例化引擎实现机制
- [ ] 分析Xtext语法定义
- [ ] 研究扩展点机制
- [ ] 分析核心API设计模式
