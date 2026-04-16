---
name: review-rules
description: "Rust 后端项目的代码审查规则，供 /review command 加载。覆盖 unsafe、生命周期、并发、错误处理、性能等维度。"
user-invocable: false
---

# 后端审查规则（Rust）

以下规则在 `/review` 执行时自动加载，补充通用审查维度。

---

## 1. 安全（Rust 补充）

- `unsafe` 块是否有充分理由？是否有注释说明 safety invariant
- `unsafe` 块的范围是否最小化？不能把整个函数标 unsafe 来"省事"
- FFI 边界：传入/传出的指针是否校验了 null、对齐、生命周期
- 密钥/token 是否通过 `secrecy::Secret` 或类似方式包装，防止意外日志输出
- SQL 查询是否使用参数化（sqlx 的 `query!` / `query_as!`），禁止字符串拼接

判定：无 safety 注释的 unsafe、SQL 拼接、密钥明文日志 → FAIL

## 2. 错误处理

- 是否滥用 `.unwrap()` / `.expect()` 在非测试代码中？生产路径必须用 `?` 或 `match`
- 自定义错误类型是否实现了 `std::error::Error`，是否保留了 source chain
- `anyhow` vs `thiserror` 使用是否合理：库代码用 `thiserror`，应用代码用 `anyhow`
- 错误信息是否包含足够上下文（用 `.context()` / `.with_context()`）
- panic 是否可能在 async 任务中发生？async 任务 panic 会导致整个 runtime 行为不确定

判定：生产路径 unwrap 可能触发的 panic → FAIL

## 3. 并发与异步

- `tokio::spawn` 的任务是否正确处理了 JoinHandle（不能 fire-and-forget 后丢弃错误）
- 是否有跨 `.await` 持有 `Mutex` 锁？应使用 `tokio::sync::Mutex` 或重构为消息传递
- `Arc<Mutex<T>>` 是否必要？能否用 `DashMap`、channel、或 actor 模式替代
- 阻塞操作（文件 IO、CPU 密集计算）是否放在 `spawn_blocking` 中
- `select!` 分支是否处理了所有 cancel safety 问题

判定：跨 await 持有 std Mutex、阻塞操作在 async 上下文中直接调用 → FAIL

## 4. 生命周期与所有权

- 是否有不必要的 `.clone()` 来"解决"借用检查？如果数据量大，考虑 `Arc` 或重构
- 生命周期标注是否最小化？不需要的 `'a` 不要加
- `'static` 约束是否合理？是否因为 `tokio::spawn` 要求而过度 clone
- 返回引用的函数是否有悬垂引用风险

## 5. 类型系统与 API 设计

- 公开 API 的参数是否使用了合适的类型（`&str` vs `String`、`impl Into<T>` 等）
- 枚举是否 `#[non_exhaustive]`（如果是库的公开类型）
- `Option<Option<T>>` 或 `Result<Result<T, E1>, E2>` 等嵌套是否可以简化
- 新增的 public struct/enum 是否派生了合理的 trait（`Debug`、`Clone`、`Serialize` 等）

## 6. 性能

- 热路径是否有不必要的堆分配（`String` vs `&str`、`Vec` vs slice、`Box` vs stack）
- 集合类型选择是否合理：`HashMap` vs `BTreeMap`、`Vec` vs `VecDeque`
- 是否有 O(n²) 或更差的算法隐藏在看似简单的循环中
- 序列化/反序列化是否在热路径上？是否可以用 zero-copy（`serde_json::RawValue`、`bytes::Bytes`）
- 数据库连接池大小是否合理，是否有连接泄漏风险（未 drop 的 connection）

判定：热路径 O(n²) 且数据量不可控 → FAIL

## 7. 依赖与 Cargo

- 新增依赖是否必要？是否有更轻量的替代
- `Cargo.toml` 中依赖版本是否合理（不要用 `*`，patch 版本可以用 `~`）
- feature flags 是否最小化启用（不要 `features = ["full"]` 除非真的全用）
- `build.rs` 变更是否影响编译时间

## 8. 测试

- 新增的公开函数/方法是否有对应测试
- 测试是否覆盖了错误路径，不只是 happy path
- 集成测试是否正确隔离（数据库 fixture、临时文件清理）
- `#[ignore]` 的测试是否有注释说明原因

## 9. 日志与可观测性

- 关键操作是否有 `tracing` span / event
- 日志级别是否合理：error 只用于需要人工介入的情况，info 用于业务事件，debug 用于排查
- 结构化日志字段是否包含足够的关联信息（request_id、user_id 等）
- 是否有敏感信息泄露到日志（密码、token、PII）

判定：密钥/PII 出现在日志中 → FAIL
