# RTK Wiki 索引 (中文版)

## 架构
- [系统架构](system-architecture.md) -- CLI 代理路由, Commands 枚举, 数据流图
- [核心基础设施](core-infrastructure.md) -- 配置, 追踪 (SQLite), tee, 工具函数, 过滤器, toml_filter, 运行器
- [回退与代理系统](fallback-system.md) -- 三层回退, 代理模式, 安全属性

## 模式
- [过滤器实现模式](filter-patterns.md) -- 10 种过滤策略 (A-J), 令牌节省, 模块清单
- [Rust 模式与规范](rust-patterns.md) -- 错误处理, 正则, 测试, 所有权, 依赖
- [TOML 过滤器 DSL](toml-filter-dsl.md) -- 8 阶段管道, 59 个内置过滤器, 内联测试 DSL

## 系统
- [钩子系统](hooks-system.md) -- AI 代理集成, 权限, 完整性, 信任, 安全
- [分析与报告](analytics-system.md) -- gain, 经济分析, discover, learn, 会话采用率

## 设计方案
- [rtk optimize 设计文档](optimize-command-design.md) -- 原始设计方案, TOML 自动生成, 配置调优, 模块集成
- [rtk optimize 实现文档](optimize-command.md) -- 个性化优化引擎, 使用场景, 架构, 四个分析器详解
