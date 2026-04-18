---
name: review-rules-rust
description: "Rust 后端项目的代码审查规则，供 /review command 加载。覆盖 unsafe、错误处理、并发、性能、可观测性等维度。"
user-invocable: false
---

# 后端审查规则（Rust）

`/review` 检测到 Rust 项目时自动加载，补充通用审查维度。

---

## 1. 安全（Rust 补充）

- `unsafe` 块：是否有 `// SAFETY:` 注释、范围是否最小化、是否真的必要
- FFI 边界：指针是否校验 null、对齐、生命周期
- 密钥/token 通过 `secrecy::Secret` 或自定义 `Debug` 防止日志泄露
- SQL 使用参数化（`query!` / `query_as!` / `bind`），字符串拼接 SQL → FAIL
- 整数溢出：金额/计数场景使用 `checked_*` / `saturating_*`
- 反序列化用户输入时限制大小和递归深度

## 2. 错误处理

- 生产路径使用 `?` 或 `match`，`.unwrap()` 仅限测试代码。生产路径 unwrap 可能 panic → FAIL
- 错误信息包含足够上下文（`.context()` / `.with_context()`）
- async 任务中的 panic 会导致 runtime 行为不确定，需特别关注

## 3. 并发与异步

- `tokio::spawn` 的 JoinHandle 正确处理（不丢弃错误）
- 跨 `.await` 持有 `std::Mutex` → FAIL（应使用 `tokio::sync::Mutex` 或消息传递）
- 阻塞操作（文件 IO、CPU 密集）放在 `spawn_blocking` 中
- `select!` 分支处理 cancel safety

## 4. 生命周期与所有权

- 不必要的 `.clone()` 是否可以用 `Arc` 或重构替代
- `'static` 约束是否因 `tokio::spawn` 而过度 clone

## 5. 类型系统与 API 设计

- 公开 API 参数类型合适（`&str` vs `String`、`impl Into<T>`）
- 新增 public struct/enum 派生合理的 trait（`Debug`、`Clone`、`Serialize`）

## 6. 性能

- 热路径避免不必要的堆分配
- 集合类型选择合理
- 隐藏的 O(n²) 且数据量不可控 → FAIL
- 数据库连接池无泄漏风险

## 7. 依赖与 Cargo

- 新增依赖是否必要，feature flags 最小化启用
- `Cargo.toml` 版本合理（不用 `*`）

## 8. 测试

- 新增公开函数有对应测试，覆盖错误路径
- 集成测试正确隔离

## 9. 日志与可观测性

- 关键操作有 `tracing` span/event，级别合理
- 结构化日志含关联信息（request_id、user_id）
- 密钥/PII 出现在日志中 → FAIL
