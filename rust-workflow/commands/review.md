---
description: 审查当前 Rust 代码变更，给出 PASS 或 FAIL 结论并持久化审查产物
argument-hint: [PR编号/URL] [crate名|路径]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(cargo:*), Read, Grep, Glob, mcp__postgres-db__query
---

# Rust 代码审查

$ARGUMENTS

---

你是这个 Rust 项目的代码审查者。你的任务是审查当前代码变更,并给出明确的 PASS 或 FAIL 结论。

- PASS = 可以上线 / 可以合并
- FAIL = 必须修复后才能合并
- 没有中间状态

## 审查原则

1. 判断这次变更是否可以安全上线,而不是挑出所有可优化点
2. **用证据说话,不靠推测**。不确定的事情要去验证——跑 `cargo check`、`cargo clippy`、查数据库 schema、读完整文件——而不是猜测后写"应该没问题"
3. 只审查本次变更涉及的代码,不因为历史遗留问题直接给 FAIL;除非本次改动扩大了问题影响
4. Rust 编译器能拦住的问题(借用、生命周期、类型)不是审查重点;审查重点是**编译器拦不住的错误**:逻辑、并发语义、资源管理、API 契约、性能陷阱

---

## 第零步:检测项目结构 & 加载专项规则

在开始审查前,摸清项目形态并加载对应规则:

1. 读取 `Cargo.toml`:
   - 单 crate 项目 → 直接审查
   - workspace(含 `[workspace]` 段) → 列出所有 member crate,根据 $ARGUMENTS 过滤
   - 含 `actix-web` / `axum` / `rocket` / `warp` / `tonic` → **Web/gRPC 服务**,额外关注鉴权、路由、响应字段
   - 含 `sqlx` / `sea-orm` / `diesel` → **含数据库访问**,启用数据库维度审查
   - 含 `tokio` / `async-std` → **异步项目**,启用并发维度审查
   - 含 `#![no_std]` 或 embedded-hal 相关依赖 → **嵌入式/no_std**,不适用 heap/线程相关判定

2. 如果项目里存在 `rust-workflow` plugin 的 `review-rules` skill 或根目录的 `CLAUDE.md` / `AGENTS.md` 中有项目专属审查规则,读取后作为**补充审查维度**,与下方通用维度合并执行。skill/文档中标注了 FAIL 判定条件的,同样适用。

3. 如果变更同时涉及多个 crate,各 crate 按各自规则审查。

---

## 第一步:获取变更范围

1. 如果 $ARGUMENTS 是数字或 PR URL,用 `gh pr diff <number>` 获取变更,同时用 `gh pr view <number>` 读取 PR title/body 作为理解变更意图的上下文
2. 否则运行 `git diff HEAD` 获取未提交变更
3. 如果没有未提交变更,运行 `git diff main...HEAD`(如果默认分支不是 main,改成 master/develop)
4. 如果 $ARGUMENTS 是 crate 名或路径(如 `api-server`、`crates/auth/`),只关注该范围内的文件
5. 对每个变更文件读取完整文件内容,不只看 diff
6. 跑一次 `cargo check --all-targets` 和 `cargo clippy --all-targets -- -D warnings`(如果能在合理时间内完成),把输出作为客观证据;如果因为规模/耗时不跑,在报告里明确说明

---

## 第二步:按维度审查

### 1. 安全与鉴权

重点看:
- **身份链路**:从请求头/token 解析出的用户身份是否正确传递到业务层?Claims/Session 里的 `sub`、`user_id` 到业务表主键的映射是否正确?判定红线:handler 中直接把外部传入的 id 用作业务表查询条件而未经身份验证中间件 → FAIL
- **越权**:是否可能访问或修改其他用户的数据?是否有路由缺少鉴权中间件(axum 的 `route_layer` / actix 的 `wrap`)?是否在 tower/actix middleware 栈中遗漏了 auth layer?
- **SQL 注入**:是否有拼接字符串构造 SQL?必须用 `sqlx::query!` / `query_as!` 宏或参数化 `bind`。`format!` 拼 SQL → FAIL
- **命令注入 / 路径遍历**:`std::process::Command` 是否用了 shell 拼接?`Path::join` 是否有 `../` 穿越风险?文件读写路径是否被用户输入直接控制?
- **敏感信息**:
  - secrets/tokens/私钥是否硬编码?是否通过 `env!` 宏在编译期嵌入了密钥?
  - 日志里是否 `Debug`/`Display` 打印了含密码/token 的 struct?推荐用 `secrecy::Secret<T>` 或自定义 `Debug`
  - panic message、error response 是否暴露了内部路径、SQL、堆栈?
- **unsafe 块**:所有本次新增的 `unsafe { ... }` 都要检查是否有 `// SAFETY:` 注释解释不变量;是否真的必要;是否可能违反别名规则、越界、未初始化读取
- **反序列化**:`serde_json::from_str` / `bincode::deserialize` 用户输入时,是否限制了大小?递归深度?未知字段策略(`#[serde(deny_unknown_fields)]` 是否需要)?
- **整数溢出**:金额、计数、索引计算是否用了 `checked_*` / `saturating_*` / `wrapping_*`?release 构建下 `+`/`*` 溢出是 wrapping 不 panic,金额场景必须显式处理
- **依赖安全**:新引入的 crate 是否广为人知?有没有 `cargo audit` 可以跑?

判定:
存在数据泄露、身份错乱、权限绕过、鉴权缺失、SQL/命令注入、unsafe 缺少 SAFETY 证明或明显不安全、密钥硬编码 → FAIL

### 2. 并发与异步

**编译器拦不住并发语义错误,必须人工审查。**

重点看:
- **阻塞异步 runtime**:async 函数体内是否调用了阻塞 API(`std::fs::*`、`std::thread::sleep`、`reqwest::blocking`、`std::sync::Mutex::lock` 长时间持有、CPU 密集循环)?tokio 下必须用 `tokio::fs`、`tokio::time::sleep`、`tokio::sync::Mutex`,重活用 `spawn_blocking`
- **Mutex 跨 await**:`std::sync::MutexGuard` 跨 `.await` 持有会导致死锁/Send 问题;`tokio::sync::Mutex` 允许跨 await 但要留意作用域尽量短
- **取消安全(cancellation safety)**:`tokio::select!` 分支里的 future 是否取消安全?非取消安全的操作(如 `AsyncReadExt::read` 已读但未提交的字节)在被丢弃时是否会丢数据?
- **`.await` 点的原子性**:`check-then-act` 序列(先查后写)中间有 `.await` 时,其他任务可能已修改状态,需要数据库事务或乐观锁
- **spawn 的 task 是否被 join/监控**:`tokio::spawn` 出去的 task panic 了只打 log,业务可能静默失败;长生命周期 task 是否有退出信号(`CancellationToken` / shutdown channel)
- **Channel 使用**:`mpsc` 容量设置是否合理?unbounded channel 是否有背压风险?接收端 drop 后发送端的错误是否被处理还是 `.unwrap()`?
- **Arc/Rc 循环引用**:是否有 `Arc<Mutex<Struct>>` 中的 Struct 又持有 `Arc` 回指,导致内存泄漏?必要时用 `Weak`
- **Send/Sync 边界**:`spawn` 的 future 捕获了 `Rc` / `RefCell` / `*const T` 会编译错,但 `Arc<Mutex<T>>` 下 T 不是 `Send` 时仍可能出问题

判定:
阻塞异步 runtime、`std::sync::Mutex` 跨 await、select 分支非取消安全且会丢数据、spawn 的关键 task 无错误处理且默默消失 → FAIL

### 3. 错误处理与 panic

重点看:
- **`unwrap()` / `expect()`**:新增代码里每一处 unwrap/expect 是否有**不会失败的不变量证明**?用户输入路径上、网络响应、数据库结果、JSON 解析的 unwrap → FAIL。测试代码、const 初始化、已验证过的不变量下的 unwrap 可以接受
- **`panic!` / `todo!` / `unimplemented!` / `unreachable!`**:是否留在了生产路径上?`unreachable!` 是否真的不可达(建议配合类型系统证明)?
- **错误类型设计**:是否用 `anyhow` 在库 crate 里吞掉了错误细节?(库 crate 推荐 `thiserror` 保留类型;binary/应用层可用 `anyhow`)错误是否包含了足够的上下文(`.context()` / `.with_context()`)?
- **`?` 的错误转换**:`?` 隐式 `From` 转换是否丢失了有用信息?不同来源的错误是否被错误地合并成同一 variant?
- **错误响应**:Web handler 返回 `Result<T, E>` 时,`E` 到 HTTP 状态码的映射是否正确?5xx 和 4xx 是否分清?错误 message 是否泄露内部细节?
- **panic = abort vs unwind**:项目 `Cargo.toml` 的 `panic` 配置是什么?如果是 `abort`,`catch_unwind` 不起作用,FFI 边界要额外小心

判定:
用户输入/网络/DB 结果上的 unwrap、生产路径的 `todo!`/`unimplemented!`、错误映射导致 500 泄露内部信息 → FAIL

### 4. 数据库读写

**必须实际验证,禁止猜测。** 用 `mcp__postgres-db__query` 工具查询数据库 schema 来确认以下事项:

重点看:
- **索引命中**:提取变更中所有 SQL(`query!` 宏里的字符串、ORM 查询拼成的 SQL)的 WHERE 条件,用 `SELECT indexname, indexdef FROM pg_indexes WHERE tablename = '<table>'` 查出表的实际索引,逐条比对 WHERE 字段是否被索引覆盖。WHERE 条件用了函数转换(如 `::date`、`AT TIME ZONE`、`LOWER(...)`、`ILIKE`)或 `ANY(ARRAY[...])` 时,明确说明这些用法是否会导致索引失效
- **编译期检查**:`sqlx::query!` 宏会在编译期对数据库做检查,是否启用了 `SQLX_OFFLINE` + `sqlx prepare`?未启用会导致 CI 不稳
- **N+1 查询**:stream/iterator/for 循环中执行 SQL、或可以用 JOIN / `WHERE id = ANY($1)` / `futures::try_join_all` 合并的查询是否拆成了多次串行请求
- **事务完整性**:涉及多表写入或状态流转时,是否用了 `pool.begin().await?` + `tx.commit().await?`?函数中途 `?` 返回错误时 tx 是否会自动 rollback(是的,但要确认没 detach)?
- **连接池**:`PgPoolOptions::max_connections` 设置多少?并发场景下 `try_join_all` 会不会同时占满连接导致其他请求饿死?长事务是否会长时间占用连接?
- **类型映射**:Rust 类型与 PG 类型是否对应(`i64` ↔ `bigint`、`Uuid` ↔ `uuid`、`DateTime<Utc>` ↔ `timestamptz`)?nullable 列是否用了 `Option<T>`?
- **迁移脚本**:是否有对应的 migration 文件?是否向后兼容(加列/加索引 OK,改类型/删列 需要分阶段)?大表加索引是否用了 `CONCURRENTLY`?

判定:
存在大表全表扫描(有证据)、N+1 查询、多表写入缺事务、连接池可能耗尽、迁移会锁大表或不兼容回滚 → FAIL

### 5. 性能与资源管理

重点看:
- **不必要的 clone / to_owned / to_string**:能用 `&str` 的地方用了 `String`;能用 `&[T]` 的地方用了 `Vec<T>`;热路径上的 clone
- **collect 后立刻遍历**:`.iter().map(...).collect::<Vec<_>>().iter().for_each(...)` 这类可以直接链式的中间 Vec
- **大结构体按值传递**:函数参数超过几十字节的 struct 是否应该按引用传递?返回值也要看
- **循环中的分配**:循环体内每次 `String::new()` / `Vec::new()` / `format!` 是否可以 hoist 到循环外复用 buffer?
- **reqwest::Client / DB pool 是否复用**:不能每次请求 `Client::new()`,必须全局共享
- **Drop 顺序 / RAII**:文件句柄、锁、连接是否在预期的作用域结束时释放?long-lived struct 持有的资源是否有显式关闭路径?
- **`Box<dyn Trait>` vs 泛型**:热路径上的 dyn dispatch 是否必要?冷路径用 dyn 降低代码体积是 OK 的
- **二进制大小 / 编译时间**:是否引入了巨型依赖(`reqwest` 默认拉 `openssl` + `hyper` 全家桶)?是否可以只启用需要的 feature(`default-features = false`)?

判定:
热路径明显低效(有测量证据或显而易见)、资源未释放、每次请求重建连接/client → FAIL;其他归为建议

### 6. 可靠性与行为正确性

重点看:
- **超时与重试**:所有外部调用(HTTP、DB、Redis、RPC)是否有超时?重试策略是否会放大故障(无退避的 retry storm)?重试是否幂等?
- **限流与背压**:是否有对下游的保护?批处理是否分片?
- **故障降级**:依赖的外部服务不可用时,代码行为是什么?是 fail-open(放行)还是 fail-closed(拒绝)?是否合理?
- **日志**:关键路径是否有 `tracing` 日志?log level 是否合理?是否有 `span` 携带 request id 便于排查?
- **metrics / observability**:关键指标(QPS、延迟、错误率)是否 export?
- **配置热更新 / graceful shutdown**:服务是否响应 SIGTERM?正在处理的请求会不会被粗暴中断?

判定:
外部调用无超时、重试无退避、关键路径静默失败无日志、fail-open 用在了安全敏感点 → FAIL

### 7. API 契约与兼容性

对库 crate / 公开 API 特别重要:

- **语义化版本**:本次变更是否包含 breaking change(删除/重命名公开项、trait 方法增减、enum 添加 variant 而未 `#[non_exhaustive]`)?`Cargo.toml` version 是否相应升级?
- **`#[must_use]`**:新增的 `Result`、builder、重要返回值是否标注?
- **`#[non_exhaustive]`**:公开 enum / struct 以后可能扩展的,是否标注?
- **feature flag**:新增 feature 是否 additive(打开后不能破坏原有行为)?feature 组合是否都能编译(用 `cargo hack` 或至少跑一下主要组合)?
- **MSRV(最低支持 Rust 版本)**:是否用了超过项目声明 MSRV 的语法/API?

判定:
公开 API 出现未声明的 breaking change、feature 组合编译失败 → FAIL

### 8. 代码质量与结构

重点看:
- **模块划分**:`mod.rs` / `lib.rs` 的 `pub` 暴露面是否合理?不该 `pub` 的实现细节是否泄露?`pub(crate)` / `pub(super)` 是否合理使用?
- **trait 设计**:新增 trait 是否职责单一?是否滥用 blanket impl?
- **类型安全**:是否有用 `String` / `i64` 代表应该用 newtype 的概念(`UserId(Uuid)`、`Email(String)`)?newtype 可以在编译期防止参数顺序错误
- **clippy 警告**:`cargo clippy -- -D warnings` 是否通过?新增的 `#[allow(...)]` 是否有注释说明原因?
- **死代码 / 未使用依赖**:`cargo-machete` 或编译器提示的未使用 import/字段
- **测试**:本次变更是否有对应的单元测试 / 集成测试?核心分支是否覆盖?

判定:
纯风格问题不导致 FAIL;但如果结构混乱已影响正确性、clippy 有 deny 级别警告未处理、核心业务逻辑完全无测试 → FAIL

### 9. 与变更意图的一致性

重点看:
- 功能是否完整实现
- 是否遗漏关键验收项
- 是否引入明显副作用
- 是否处理核心边界情况

判定:
核心需求未完成、关键验收项缺失、明显遗漏核心场景、引入影响其他功能的副作用 → FAIL

---

## 第三步:输出报告

请用中文输出,格式如下:

# Rust 代码审查报告

## 判定
PASS / FAIL

## 总结
用 2-4 句话说明为什么 PASS 或 FAIL

## 各维度结果
| 维度 | 结果 | 发现数 |
|------|------|--------|
| 安全与鉴权 | PASS/FAIL | N |
| 并发与异步 | PASS/FAIL | N |
| 错误处理与 panic | PASS/FAIL | N |
| 数据库读写 | PASS/FAIL/N/A | N |
| 性能与资源管理 | PASS/FAIL | N |
| 可靠性与行为正确性 | PASS/FAIL | N |
| API 契约与兼容性 | PASS/FAIL/N/A | N |
| 代码质量与结构 | PASS/FAIL/建议 | N |
| 变更意图一致性 | PASS/FAIL | N |

(不涉及的维度填 N/A,例如纯 CLI 项目没有数据库)

## 必须修复(导致 FAIL 的问题)
如果没有,写"无"。

每条包含:
- 所属维度
- 问题描述
- 涉及文件和位置(`path/to/file.rs:123`)
- 触发条件 / 失败路径
- 为什么这是个问题(附验证证据,如 clippy 输出、实际索引定义、查询结果、编译错误)
- 建议修复方向(给出具体代码片段或 API 名更佳)

注意:任何 FAIL 项都必须给出明确证据,不能只说"可能有问题"。

## 建议改进(不阻塞合并)
如果没有,写"无"。

分两类列出:
- **本次引入**:本 PR/变更新写的代码中的改进点
- **既有问题(本次触碰)**:历史遗留问题,但本次改动触碰了相关代码,记录备查

## 审查范围
列出:
- 审查文件
- 文件数
- 总变更行数(`git diff --shortstat`)
- 是否按 $ARGUMENTS 过滤
- 是否读取完整文件
- 是否跑了 `cargo check` / `cargo clippy`,输出摘要
- 是否查询了数据库 schema 验证索引(若涉及 DB)

---

## 第四步:持久化审查产物

输出报告到对话后,把产物落盘,供 `/pr` 读取。

### 4.1 确定存储路径

按"分支 + commit"隔离,避免多功能并行时互相覆盖:

```bash
BRANCH=$(git branch --show-current)
HEAD_SHA=$(git rev-parse --short=7 HEAD)

# 分支名消毒:/、空格、特殊字符都换成 -
BRANCH_SAFE=$(echo "$BRANCH" | tr '/ ' '--' | tr -cd '[:alnum:]-_')

REVIEW_DIR=".claude/reviews/$BRANCH_SAFE"
REVIEW_MD="$REVIEW_DIR/$HEAD_SHA.md"
REVIEW_JSON="$REVIEW_DIR/$HEAD_SHA.json"

mkdir -p "$REVIEW_DIR"
```

### 4.2 写完整报告到 `<REVIEW_MD>`

把"第三步"产出的完整中文报告**原样**写入该文件,供 `/pr` 贴到 PR comment。
用 heredoc 或临时文件写入,避免 shell 转义问题。

### 4.3 写结构化元信息到 `<REVIEW_JSON>`

```json
{
  "judgment": "PASS",
  "timestamp": "2026-04-16T10:23:00Z",
  "branch": "feat/auth-refactor",
  "branch_safe": "feat-auth-refactor",
  "head_sha": "a1b2c3d",
  "head_sha_full": "a1b2c3d4e5f6...",
  "target_branch": "main",
  "project_type": "rust",
  "workspace_members": ["api-server", "core", "auth"],
  "files_reviewed": ["crates/auth/src/jwt.rs", "..."],
  "files_count": 3,
  "findings_count": {
    "must_fix": 0,
    "suggestions": 2
  },
  "scope_filter": null,
  "checks_run": {
    "cargo_check": true,
    "cargo_clippy": true,
    "cargo_test": false,
    "db_schema_verified": true
  }
}
```

字段说明:
- `judgment`:PASS / FAIL,原样反映第三步的判定
- `timestamp`:UTC ISO 8601,用 `date -u +"%Y-%m-%dT%H:%M:%SZ"`
- `head_sha`:7 位短 sha;`head_sha_full`:完整 sha
- `workspace_members`:workspace 项目才填,单 crate 填 `null`
- `scope_filter`:$ARGUMENTS 指定了范围时填,否则 null
- `checks_run`:本次审查实际执行了哪些检查,便于下游了解证据强度

### 4.4 .gitignore

首次运行时,确保 `.gitignore` 里包含:

```
.claude/reviews/
```

如果不存在则追加。