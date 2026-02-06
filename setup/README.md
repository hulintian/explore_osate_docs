# Setup - 开发环境配置

## 模块概述

Setup模块包含了OSATE2开发环境的自动化配置文件，主要使用Eclipse Oomph技术来自动化开发环境的安装和配置过程。这些配置文件使得新开发者能够快速搭建一致的开发环境。

## 主要内容

### Oomph Setup文件
**核心配置文件**：
- **osate2.setup** - 最新版OSATE2开发环境配置
- **OSATEConfiguration.setup** - OSATE配置模块

**历史版本配置**：
按照Eclipse版本组织的历史配置文件：
- osate2_2025-06.setup - Eclipse 2025-06
- osate2_2025-03.setup - Eclipse 2025-03
- osate2_2024-03.setup - Eclipse 2024-03
- osate2_2023-12.setup - Eclipse 2023-12
- osate2_2023-03.setup - Eclipse 2023-03
- osate2_2022-06.setup - Eclipse 2022-06
- osate2_2022-03.setup - Eclipse 2022-03
- osate2_2021-03.setup - Eclipse 2021-03
- osate2_2020-06.setup - Eclipse 2020-06
- osate2_2020-03.setup - Eclipse 2020-03
- osate2_2019-12.setup - Eclipse 2019-12
- osate2_2019-09.setup - Eclipse 2019-09
- osate2_2019-03.setup - Eclipse 2019-03
- osate2_2018-12.setup - Eclipse 2018-12
- osate2_2018-09.setup - Eclipse 2018-09

### 代码质量配置
- **osate2-checkstyle.xml** - Checkstyle代码风格检查配置

## Eclipse Oomph技术

### 什么是Oomph？
Eclipse Oomph是一个Eclipse项目，提供了：
- **自动化安装** - 自动下载和安装Eclipse及插件
- **环境配置** - 自动配置工作空间和项目设置
- **一致性保证** - 确保所有开发者使用相同的配置
- **简化流程** - 一键式开发环境设置

### Setup文件结构
Setup文件是XML格式的配置文件，包含：
- Eclipse版本和产品定义
- 必需的Eclipse特性和插件
- P2仓库位置
- Git仓库配置
- 项目导入配置
- 工作空间首选项
- JDK配置
- Maven/Tycho配置

## 配置内容

### 1. Eclipse基础平台
**包含内容**：
- Eclipse Platform版本
- Eclipse SDK
- Eclipse Modeling Framework (EMF)
- Xtext框架
- Maven/Tycho支持

### 2. 开发工具
**IDE插件**：
- Java开发工具 (JDT)
- 插件开发工具 (PDE)
- Git集成 (EGit)
- Maven集成 (M2E)

**建模工具**：
- EMF编辑器
- Xtext/Xtend
- Sirius（图形建模）

**质量工具**：
- Checkstyle
- SpotBugs
- JaCoCo（代码覆盖率）

### 3. 源代码管理
**Git配置**：
- 仓库URL
- 分支选择
- 克隆位置
- 子模块处理

### 4. 项目配置
**自动导入**：
- 所有OSATE2子项目
- 项目依赖关系
- 构建路径配置
- 启动配置

### 5. 工作空间首选项
**代码格式**：
- Java代码格式化规则
- 代码模板
- 保存操作（自动格式化等）

**编辑器设置**：
- 字体和颜色
- 文本编码（UTF-8）
- 行结束符
- Tab vs 空格

**构建设置**：
- 自动构建
- 清理策略
- 编译器警告级别

### 6. JRE/JDK配置
**Java环境**：
- 推荐的JDK版本
- JRE路径配置
- 编译器合规性级别

## 使用方法

### 使用Oomph安装OSATE2开发环境

#### 第一步：安装Eclipse Installer
1. 下载Eclipse Installer
2. 启动Installer
3. 切换到高级模式（Advanced Mode）

#### 第二步：添加Setup文件
1. 在Product页面选择Eclipse产品
2. 点击"+" 添加用户项目
3. 选择或输入OSATE2 setup文件URL
4. 选择要安装的版本配置

#### 第三步：配置变量
设置必要的变量：
- **安装目录** - Eclipse安装位置
- **工作空间位置** - 工作空间路径
- **Git克隆位置** - 源代码克隆位置
- **JDK位置** - Java开发工具包路径

#### 第四步：执行安装
1. 确认配置
2. 点击Finish开始安装
3. 等待自动下载和配置
4. 首次启动Eclipse进行额外配置

### 手动使用Setup文件
如果已有Eclipse安装：
1. Help → Perform Setup Tasks
2. 导入setup文件
3. 执行配置任务

## Setup文件版本管理

### 版本策略
**主配置文件**：
- osate2.setup - 始终指向最新稳定配置
- 用于新开发者和常规开发

**版本特定文件**：
- 针对特定Eclipse版本优化
- 用于维护旧版本
- 用于兼容性测试

### 选择合适的版本
**最新开发**：
- 使用osate2.setup或最新版本文件
- 获得最新特性和改进

**维护分支**：
- 使用对应Eclipse版本的setup文件
- 确保环境一致性

## Checkstyle配置

### osate2-checkstyle.xml
**目的**：统一代码风格

**检查规则**：
- 命名约定
- 代码格式
- 注释要求
- 复杂度限制
- 最佳实践

**集成方式**：
- Eclipse Checkstyle插件
- Maven Checkstyle插件
- CI/CD流水线检查

### 代码风格标准
**Java编码规范**：
- 缩进：Tab或空格
- 行长度限制
- 括号风格
- 导入顺序

**注释规范**：
- JavaDoc要求
- 类和方法注释
- 版权声明

## 自动化配置任务

### Oomph执行的任务
1. **下载任务** - 下载Eclipse和插件
2. **安装任务** - 安装必需组件
3. **Git克隆** - 克隆源代码仓库
4. **项目导入** - 导入所有项目
5. **首选项设置** - 配置工作空间
6. **目标平台** - 设置目标平台定义
7. **工作集** - 创建项目工作集

### 启动后任务
首次启动Eclipse时：
- 索引工作空间
- 构建项目
- 解析依赖
- 应用首选项

## 环境一致性

### 好处
**团队协作**：
- 统一的开发环境
- 减少"在我机器上能运行"问题
- 简化问题重现

**新人上手**：
- 快速环境搭建
- 降低配置难度
- 减少配置错误

**持续集成**：
- 本地环境与CI环境一致
- 可重现的构建
- 依赖版本锁定

## 故障排除

### 常见问题

#### 安装失败
**原因**：
- 网络问题
- P2仓库不可用
- 磁盘空间不足

**解决**：
- 检查网络连接
- 使用镜像仓库
- 清理磁盘空间
- 重试安装

#### Git克隆失败
**原因**：
- 认证问题
- 网络超时
- Git配置错误

**解决**：
- 配置Git凭证
- 使用SSH密钥
- 调整超时设置

#### 项目构建错误
**原因**：
- JDK版本不匹配
- 依赖未解析
- 目标平台问题

**解决**：
- 验证JDK版本
- 刷新依赖
- 重新加载目标平台

## 自定义Setup文件

### 创建自定义配置
**步骤**：
1. 复制现有setup文件
2. 使用Oomph模型编辑器编辑
3. 修改仓库URL、分支等
4. 添加/删除任务
5. 测试配置

### 扩展配置
**可添加内容**：
- 额外的Eclipse插件
- 自定义项目模板
- 公司特定配置
- 额外的工具集成

## 维护和更新

### Setup文件维护
**定期更新**：
- 跟随Eclipse新版本
- 更新插件版本
- 调整配置参数

**测试**：
- 在干净环境测试
- 验证所有任务成功
- 检查项目构建

### 版本控制
- Setup文件在Git中版本控制
- 变更需要审查
- 保留历史版本

## 最佳实践

### 使用建议
1. **使用最新配置** - 除非有特殊需求
2. **完整安装** - 让Oomph完成所有配置
3. **定期更新** - 同步最新的setup文件
4. **报告问题** - 发现配置问题及时反馈

### 开发者建议
1. **遵循配置** - 使用setup配置的设置
2. **不要手动修改** - 避免手动改变自动配置的内容
3. **保持同步** - 定期执行setup更新任务

## 相关资源

### 文档
- [Eclipse Oomph官方文档](https://projects.eclipse.org/projects/tools.oomph)
- OSATE开发者指南
- Eclipse开发环境设置指南

### 工具
- Eclipse Oomph Installer
- Setup Editor
- Setup Archiver

## 下一步分析

- [ ] 深入研究setup文件的具体配置内容
- [ ] 分析不同版本配置的差异
- [ ] 研究Oomph模型和任务机制
- [ ] 探索自定义配置方法
- [ ] 分析Checkstyle规则详情
- [ ] 研究与CI/CD的集成
- [ ] 探索自动化测试环境配置
- [ ] 分析跨平台配置的最佳实践
