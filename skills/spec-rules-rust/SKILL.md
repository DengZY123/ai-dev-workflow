---
name: spec-rules-rust
description: "Rust 后端项目的技术方案补充规则，供 /spec command 加载。关注 crate 结构、错误处理、并发模型、数据库设计等。"
user-invocable: false
---

# 后端方案补充规则（Rust）

`/spec` 检测到 Rust 项目时自动加载，补充以下关注点到方案设计中。

---

## 方案设计补充章节

### Crate 与模块结构

- 新增代码放在哪个 crate / module，为什么
- 是否需要新建 crate（workspace 成员）
- 公开 API（`pub`）的边界设计

### 错误处理策略

- 新增的错误类型定义（`thiserror` 枚举）
- 错误传播链路：从底层到 API 响应的映射
- 哪些错误需要重试，哪些直接返回

### 并发模型

- 异步任务的生命周期管理（`tokio::spawn` / `JoinSet`）
- 共享状态的并发访问方式（`Arc<Mutex>` / channel / actor）
- 是否有阻塞操作需要 `spawn_blocking`

### 数据库设计

- 新增/修改的表结构（完整 DDL）
- 索引设计：查询模式 → 索引选择的推导过程
- 迁移策略：是否需要分步迁移（先加列 → 部署 → 再加约束）
- 大表操作的锁表风险评估

### 外部依赖

- 新增 crate 依赖：名称、版本、启用的 feature flags、选择理由
- 外部服务对接：API 协议、鉴权方式、超时/重试策略、熔断降级

---

## 风险补充关注点

- `unsafe` 使用：是否必要，safety invariant 是什么
- 编译时间影响：新增依赖对增量编译的影响
- 内存使用：大数据量场景下的内存峰值估算
- 向后兼容：API 变更是否需要版本控制
