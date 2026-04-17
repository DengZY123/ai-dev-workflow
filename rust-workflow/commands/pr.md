---
description: Rust 项目代码审查通过后，提交 PR 并在评论中补齐 code-review 结果与发布物料清单
argument-hint: [目标分支,默认 main] [--draft]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(cargo:*), Read, Grep, Glob
---

# /pr — Rust 项目提交 PR 并补齐发布物料

你的任务是把**已经通过 /review 的 Rust 变更**正式交付成一个可评审、可上线的 PR。
你**不做代码审查**（/review 已完成），你只做三件事：

1. 提 PR，title 规范、body 写清**这次改了什么、为什么**
2. 把本次 /review 的结果作为 PR comment 发出来
3. 扫描变更，生成**Rust 项目发布物料清单**（Cargo.toml / migration / env / Docker / systemd / 第三方配置）作为另一条 PR comment

## 参数解析

`$ARGUMENTS` 是一整个字符串，按空格切分并分类：

- 以 `--` 开头的 token 是 flag，目前只识别 `--draft`
- 第一个非 flag token = 目标分支（默认 `main`）
- 其他 token 忽略并提示用户

示例：
- `$ARGUMENTS = "develop --draft"` → 目标分支 `develop`，draft 模式
- `$ARGUMENTS = "--draft main"` → 目标分支 `main`，draft 模式
- `$ARGUMENTS = ""` → 目标分支 `main`，非 draft

---

## 前置检查（任一失败立即停下）

### A. Git 状态检查

1. **当前在 feature 分支**：`git branch --show-current`，不能是 `main` / `master` / `develop`
2. **工作区干净**：`git status --porcelain` 必须为空
3. **已推送到远端**：`git log origin/<branch>..HEAD --oneline`，有未推送的先 `git push -u origin <branch>`
4. **是否已存在 PR**：`gh pr view --json number,state`，已存在则默认走**更新**流程

### B. Code Review 校验（核心防呆）

5. **定位 review 记录**（与 /review 使用的路径生成方式必须一致）：

```bash
BRANCH=$(git branch --show-current)
HEAD_SHA=$(git rev-parse --short=7 HEAD)
# 分支名消毒：/、空格、特殊字符都换成 -
BRANCH_SAFE=$(echo "$BRANCH" | tr '/ ' '--' | tr -cd '[:alnum:]-_')

REVIEW_DIR=".claude/reviews/$BRANCH_SAFE"
REVIEW_MD="$REVIEW_DIR/$HEAD_SHA.md"
REVIEW_JSON="$REVIEW_DIR/$HEAD_SHA.json"
```

> ⚠️ 若 `/review` 的路径算法调整，请同步此处保持一致。

6. **校验两个文件都存在**：

   - `REVIEW_JSON` 或 `REVIEW_MD` 不存在 → 停下：
     > 未找到当前 commit（`<HEAD_SHA>`）的 code-review 记录。
     > 请先运行 `/review`。
     >
     > （如果刚跑过 review，可能是 review 后又提交了新代码，请重跑。）

7. **校验 JSON 内容**，按以下优先级，任一不符停下：

   | 校验项 | 是否阻断 | 不符时的提示 |
   |---|---|---|
   | JSON 能解析 | ✅ 阻断 | "review 记录损坏，请重新运行 /review" |
   | `judgment == "PASS"` | ✅ 阻断 | "最近一次 /review 判定为 FAIL，请修复后重跑" |
   | `head_sha == $(git rev-parse --short=7 HEAD)` | ✅ 阻断 | "review 之后代码有新改动，请重新运行 /review" |
   | `branch == $(git branch --show-current)` | ⚠️ 仅告警 | "review 时分支名为 X，当前为 Y（可能是 rename），如代码未变可忽略" |

   > head_sha 是真正的防呆锚点；分支名校验只告警，允许 rename。

8. **时效性提示（不阻断）**：
   - `timestamp` 距今 > 24 小时 → 打印警告：
     > ⚠️ review 结果已超过 24 小时，如代码未变请忽略；如有疑虑请重跑 /review。

校验全过后，把 `REVIEW_MD` 路径、`judgment`、`timestamp`、`head_sha`、`files_count`、`project_type`、`workspace_members`、`checks_run` 存起来，后续用。

### C. Rust 项目基本健康检查（不阻断，仅告警）

在 review 产物校验通过后，做一次轻量确认：

```bash
# 确认格式没问题（review 不一定查 fmt）
cargo fmt --all -- --check
```

- 不通过 → 打印警告但继续：
  > ⚠️ `cargo fmt --check` 不通过，建议在合并前跑 `cargo fmt --all`。PR 照常提交。

> 不跑 `cargo check`/`cargo clippy`，这些是 /review 的职责；此处只查格式。

---

## 执行步骤

### Step 1 — 收集变更信息

```bash
TARGET=<解析得到的目标分支>
git log $TARGET..HEAD --oneline
git diff $TARGET...HEAD --stat
git diff $TARGET...HEAD
```

读取所有修改文件（重点文件全读，其他扫一眼）。

**Rust 项目重点读取**：
- `Cargo.toml`（根 + 所有 workspace member）
- `Cargo.lock` 变更（判断依赖实际升级情况）
- `rust-toolchain.toml` / `rust-toolchain`（工具链变更）
- 新增/修改的 migration 文件（`migrations/`、`sqlx/migrations/`、`diesel/migrations/` 等）
- `Dockerfile` / `docker-compose.yml` / systemd unit / k8s manifest
- `.env.example` / `config/*.toml` / `config/*.yaml`

**提取 issue 编号**：扫描所有 commit message 和当前分支名，用正则匹配：
- `#(\d+)`（如 `refs #42`、`closes #42`）
- `^(feat|fix|refactor|perf|docs|chore)/(\d+)[-_]` 开头的分支名

第一个匹配到的编号存为 `ISSUE_NUM`。没有设为空。

### Step 2 — 生成 PR Title

Title 是 reviewer 在列表里唯一看到的东西，必须能一眼看懂。

**格式**：`<type>(<scope>): <简短描述>`

**规则**：

1. **type 必须是以下之一**，基于本次改动的主要性质：
   - `feat` — 新功能
   - `fix` — 修 bug
   - `refactor` — 不改行为的重构
   - `perf` — 性能优化
   - `docs` — 只改文档/注释
   - `chore` — 构建、依赖、配置、脚手架
   - `test` — 只改测试
   - `build` — 专指 `Cargo.toml` / build.rs / 工具链升级
   - `ci` — CI 配置变更

2. **scope（Rust 项目特殊约定）**：
   - workspace 项目 → **优先用 crate 名**：`feat(auth): ...`、`perf(core): ...`
   - 单 crate 项目 → 用模块名：`fix(handler): ...`
   - 跨多个 crate → 省略：`refactor: 统一 error 类型`
   - **不堆叠多 scope**：不写 `feat(auth,db): ...`
   - 特殊 scope 约定：
     - `deps` — 依赖升级（`chore(deps): 升级 tokio 到 1.40`）
     - `build` — 构建脚本 / `Cargo.toml` 元数据

3. **描述部分**：
   - 用**祈使句**，动词开头："添加"、"修复"、"重构"、"升级"，不用"已添加"、"完成了"
   - **长度**：整个 title 不超过 72 字符，中文按 2 字符计，理想在 50 字符内
   - 不加句号、不加感叹号
   - **不要**把 issue 编号塞进 title（`#42` 放 body 的 Closes 行）
   - **不要**前缀 "PR:" / "【xxx】" / emoji

4. **type 判定优先级**（多类改动时，按影响选主要的）：
   `feat` > `fix` > `perf` > `refactor` > `build` > `docs` > `test` > `ci` > `chore`

**好例子**：
- `feat(auth): 支持 OAuth 登录`
- `fix(billing): 修复余额计算整数溢出`
- `refactor(core): 用 thiserror 替换 anyhow`
- `perf(db): 为 transactions.user_id 添加 btree 索引`
- `chore(deps): 升级 tokio 到 1.40`
- `build: 将 MSRV 从 1.72 提升到 1.75`

**坏例子**：
- ❌ `Feat: Added OAuth.`（大写 + 完成时 + 句号）
- ❌ `feat: 完整支持 OAuth，修复若干 bug，并重构 auth crate`（多件事混在一起）
- ❌ `feat(auth,db,api): xxx`（多 scope）
- ❌ `feat: 支持 OAuth #42`（issue 号不进 title）

把最终 title 存为 `PR_TITLE`。

### Step 3 — 生成 PR Body

**原则**：PR body 是"变更说明"，不是 changelog，不是 review 报告。让 reviewer 30 秒内知道"这个 PR 想干嘛、改了哪几个 crate"。

模板：

```markdown
## 背景

<1-3 句，说清为什么做>

<如果 Step 1 提取到 ISSUE_NUM，追加一行: Closes #<ISSUE_NUM>>

## 变更要点

- <关键改动 1，精确到 crate::module 级别>
- <关键改动 2>
- <...>

## 涉及 crate

<workspace 项目必填，基于 git diff --stat 聚合到 crate 粒度>
- `auth` — 新增 OAuth handler
- `core` — Error 枚举新增 variant

<单 crate 项目可省略此节>

## 关键实现决策

<只写非显然的决策，例如 "选用 thiserror 而非 anyhow，因为 ..."。没有就写"无特殊决策">

## 兼容性说明

<涉及公开 API 变更必填：>
- **破坏兼容**：是 / 否
- **SemVer 影响**：patch / minor / major
- **MSRV 变化**：无 / 从 X 升到 Y

<纯内部改动可省略此节>

## 自测情况

> 由 /pr 自动填入。本地构建/手测项需作者在 PR 创建后自行勾选。

- [ ] `cargo fmt --all -- --check` 通过
- [ ] `cargo check --all-targets --all-features` 通过
- [ ] `cargo clippy --all-targets --all-features -- -D warnings` 通过
- [ ] `cargo test` 通过
- [ ] 核心路径手测过：<列出具体路径，作者请替换>
- [x] /review 已通过（见评论）

## 相关链接

<如有 spec/设计文档链接写这里，没有就删掉本节>
```

**禁止**：
- ❌ "本 PR 完整解决了 xxx 问题"这种空话
- ❌ 罗列所有文件改动（diff 里有）
- ❌ 在 body 里贴 review 结果（要放 comment）
- ❌ 虚构 issue 编号 — ISSUE_NUM 为空就不写 Closes 行

### Step 4 — 创建或更新 PR

body 写入 `/tmp/pr-body-${HEAD_SHA}.md`（用 commit sha 做文件名，避免并发冲突），然后：

**新建**：
```bash
gh pr create \
  --base $TARGET \
  --title "$PR_TITLE" \
  --body-file /tmp/pr-body-${HEAD_SHA}.md \
  [--draft]
```

**更新已存在的 PR**：
```bash
gh pr edit <num> \
  --title "$PR_TITLE" \
  --body-file /tmp/pr-body-${HEAD_SHA}.md
```

拿到 PR URL 和 number，存为 `PR_NUM` 和 `PR_URL`。

### Step 5 — 发布 Code Review 结果（Comment 1）

**处理旧 comment**（仅更新场景）：

```bash
gh pr view $PR_NUM --json comments --jq '.comments[] | {id, body: .body[0:80]}'
```

找到以 `## 🔍 Code Review 结果` 开头的旧 comment，用 `gh api` 删除：
```bash
gh api -X DELETE /repos/:owner/:repo/issues/comments/<comment_id>
```

> 如果删除失败，退化为在旧 comment 顶部追加 `> ⚠️ 已被更新版本替代`。

读取 `REVIEW_MD` 内容，写入 `/tmp/pr-review-${HEAD_SHA}.md`：

```markdown
## 🔍 Code Review 结果

> 由 /review 于 `<timestamp>` 产出，判定：**PASS**
> 审查 commit: `<head_sha>` · 审查文件数: `<files_count>`
> 项目类型: `<project_type>` · Workspace members: `<workspace_members 或 N/A>`
> 已执行检查: cargo_check=<✓/✗> · cargo_clippy=<✓/✗> · cargo_test=<✓/✗> · db_schema_verified=<✓/✗>

<原样贴入 REVIEW_MD 的完整内容>

---

**⚠️ 以下事项自动 review 无法覆盖，请 reviewer 重点人工确认：**

1. **业务逻辑正确性**：review 判断代码写法，不判断"这是否符合业务预期"
2. **并发语义**：`cargo check` 拦不住逻辑级死锁、cancellation 丢数据、spawn 孤儿 task
3. **unsafe 正确性**：review 只验证 SAFETY 注释格式，invariant 真实性需人工核
4. **性能的实际影响**：review 基于代码模式判断，真实负载下的性能需 benchmark
5. **公开 API 稳定性承诺**：是否符合本项目的 SemVer 策略
```

发布：`gh pr comment $PR_NUM --body-file /tmp/pr-review-${HEAD_SHA}.md`

### Step 6 — 发布物料清单（Comment 2）

**处理旧 comment**：同 Step 5，查找以 `## 🚀 发布物料清单` 开头的旧 comment，删除或标记过期。

**这是 /pr 最核心的产出**。扫描 diff，按以下六类生成清单。**只列真实存在的，不存在的类别明写"无"**。

#### 6.1 Cargo 依赖与工具链变更

扫描：
- `Cargo.toml` 的 `[dependencies]` / `[dev-dependencies]` / `[build-dependencies]` 变更
- workspace 根 `Cargo.toml` 的 `[workspace.dependencies]` 变更
- `Cargo.lock` 实际版本跳动（新 crate 加入、版本升级、版本降级）
- `rust-toolchain.toml` / `rust-toolchain` 变更
- `rust-version` 字段（MSRV）变更

对每条判断：
- **变更类型**：新增 / 升级 / 降级 / 移除
- **是否破坏性**：major 版本跳动（如 0.x → 0.y 或 1.x → 2.x）→ 破坏性
- **feature flag 变更**：启用/禁用的 feature 列表
- **MSRV 影响**：新依赖是否要求更高 rustc 版本
- **编译时间/体积影响**：heavy 依赖（`reqwest` 默认、`tonic`、`polars` 等）标注"显著增加编译时间"
- **审计建议**：新增三方 crate（非 tokio-rs / serde-rs / rust-lang 官方）建议跑 `cargo audit`

#### 6.2 数据库 Migration 扫描

扫描：
- 新增/修改的 migration 文件：`migrations/`、`sqlx/migrations/`、`diesel/migrations/`、`sea-orm/migration/`
- `sqlx::query!` / `sqlx::query_as!` 宏中内联的 SQL（仅数据读写判断用，不视为 migration）
- 代码中直接执行的 `CREATE TABLE` / `ALTER TABLE` / `CREATE INDEX` / `DROP`

对每条判断：
- **是否破坏性**：DROP、ALTER COLUMN 改类型、NOT NULL 加约束、删列 → 破坏性
- **执行时机**：
  - 新增表/列被新代码读写 → **代码发布前**
  - 删除旧列/表 → **代码发布后**
  - 加索引 → 任意（但要看锁表风险）
- **大表风险**：在项目中被频繁读写或体量较大的表上执行 `ALTER TABLE ADD COLUMN NOT NULL`、非 `CONCURRENTLY` 的 `CREATE INDEX`、锁表 DDL → 标红警告
- **sqlx 离线查询数据**：是否需要在 migration 后重跑 `cargo sqlx prepare` 更新 `.sqlx/` 目录？

> 判断大表的依据优先级：
> 1. 项目根目录 `CLAUDE.md` / `.claude/project.md` 中声明的大表清单
> 2. 从 schema/migration 历史判断的事务类、鉴权类、日志类表
> 3. 保守标注"⚠️ 请人工确认此表体量"

#### 6.3 配置与环境变量扫描

扫描：
- 代码中新出现的环境变量引用（相对 target 分支对比新增的）：
  - `std::env::var("XXX")` / `std::env::var_os("XXX")`
  - `env!("XXX")` / `option_env!("XXX")`（编译期，特别危险）
  - `dotenv` / `dotenvy` 相关
  - `config` / `figment` / `envy` crate 读取的字段
- `.env.example` / `.env.sample` / `.env.template` 的变更
- `config/*.toml` / `config/*.yaml` / `settings.toml` 的变更

对每条判断：
- **读取时机**：
  - `env!()` → **编译期**，必须在构建环境配置
  - `std::env::var()` / config crate → 运行时，启动时读取
- **必填 / 选填**（没有会 panic？走降级？）
- **是否密钥**（API key、secret、token、database password、private key）→ 提醒走密钥管理平台，**警告不要提交到 git**
- **目标环境**：dev / staging / prod 都要配吗
- **是否有对应的 struct 字段/默认值**：config crate 定义的 struct 是否有 `#[serde(default)]`

#### 6.4 部署资源扫描

扫描：
- `Dockerfile` / `.dockerignore` 变更
- `docker-compose.yml` / `docker-compose.*.yml` 变更
- k8s manifest（`k8s/`、`deploy/`、`helm/charts/`）变更
- systemd unit 文件（`*.service`）
- 构建脚本 `build.rs` 变更

对每条判断：
- **是否需要重新构建镜像**
- **基础镜像变更**（`FROM rust:1.72` → `FROM rust:1.75`）
- **运行时依赖**（新增 apt 包、动态链接库）
- **端口/卷/权限变更**
- **systemd `Restart=` / `LimitNOFILE=` 等关键参数变更**
- **资源限制变更**（CPU/内存 limit/request）

#### 6.5 第三方配置扫描

识别：
- 新增/修改的 webhook / 回调 URL（代码里硬编码或 config 里声明的）
- 新厂商接入（新 HTTP client、新 SDK 引入）→ 厂商控制台配额度/白名单
- OAuth client / App 变更
- DNS / CDN / 存储桶策略变更
- 定时任务变更（`tokio-cron-scheduler`、外部 cron）
- 外部 service（Redis / Kafka / NATS / RabbitMQ）的 key prefix / channel / topic 变更
- 监控 / 可观测性接入（Prometheus metric 端点、OpenTelemetry exporter）

#### 6.6 Comment 格式

写入 `/tmp/pr-release-${HEAD_SHA}.md`：

```markdown
## 🚀 发布物料清单

> 由 /pr 基于 commit `<head_sha>` 扫描本次 Rust 项目变更生成。**请部署人按此清单逐项执行**。

### 1. Cargo 依赖与工具链

<无则写"无依赖/工具链变更。"，否则：>

| 项 | 类型 | 变更 | 破坏性 | 备注 |
|---|---|---|---|---|
| `tokio` | 升级 | 1.35 → 1.40 | 否 | minor 版本，无 API 变化 |
| `uuid` | 新增 | v1.8.0 | - | 启用 feature `v7, serde` |
| rust-toolchain | 升级 | 1.72 → 1.75 | **MSRV 变化** | CI 镜像需同步升级 |

**操作清单**：
- [ ] CI 构建镜像的 Rust 版本已对齐
- [ ] 新增三方 crate 已跑 `cargo audit`
- [ ] heavy 依赖变更已评估编译时间影响

### 2. 数据库 Migration

<无则写"无 migration 变动。"，否则逐条：>

#### 📄 `migrations/20260417120000_add_oauth_tokens.sql`
- **执行时机**：代码发布前
- **是否破坏性**：否
- **风险提示**：在新建表上操作，无锁表风险
- **sqlx 离线数据**：需重跑 `cargo sqlx prepare` 并提交 `.sqlx/` 变更
- **SQL**：
  ```sql
  <原样贴>
  ```
- **回滚**：`DROP TABLE oauth_tokens;`

### 3. 配置与环境变量

<无则写"无配置变动。"，否则：>

| 变量 / 字段 | 读取时机 | 必填 | 密钥 | 用途 | 默认值 |
|---|---|---|---|---|---|
| `DATABASE_URL` | 启动时 | ✅ | ✅ | 主库连接串 | 无，未配置 panic |
| `OAUTH_CLIENT_ID` | 启动时 | ✅ | ❌ | OAuth 应用标识 | 无 |
| `OAUTH_CLIENT_SECRET` | 启动时 | ✅ | ✅ | OAuth 应用密钥 | 无 |

**操作清单**：
- [ ] dev 环境已配置
- [ ] staging 环境已配置
- [ ] prod 环境已配置
- [ ] 密钥已通过密钥管理平台写入，未出现在 git 历史
- [ ] `env!()` 编译期变量已在 CI 构建环境配置

### 4. 部署资源

<无则写"无部署资源变更。"，否则逐条：>

- [ ] **Dockerfile**：基础镜像从 `rust:1.72-bookworm` 升级到 `rust:1.75-bookworm`，重新构建镜像
- [ ] **k8s**：`deployment.yaml` 内存 limit 从 512Mi 调整到 1Gi
- [ ] **systemd**：`service` 文件 `LimitNOFILE` 调整为 65535

### 5. 第三方配置

<无则写"无第三方配置变动。"，否则逐条：>

- [ ] **厂商 <name>**：在 <控制台> 为 prod 域名添加 OAuth redirect URI
- [ ] **Webhook**：在 <供应商> 后台更新回调地址为 `https://api.example.com/webhook/oauth`

### 6. 发布顺序

<基于上面扫描结果给出具体顺序，例如：>

1. 配置 prod 环境变量 `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET`
2. 在 <OAuth Provider> 控制台添加 prod redirect URI
3. prod 执行 migration：`migrations/20260417120000_add_oauth_tokens.sql`
4. 重新构建镜像（基础镜像升级）
5. 合并 PR，触发部署
6. 部署完成后冒烟测试：POST `/auth/oauth/login` → 跳转 → 回调 → 查 `oauth_tokens` 表有新记录

### 7. 回滚预案

1. revert PR 并重新部署（镜像回退到上一版）
2. migration 回滚：`DROP TABLE oauth_tokens;`
3. 环境变量可保留（下次使用），或从配置中移除
4. <不可逆部分写清，如 "数据已写入 oauth_tokens 表，回滚会丢失，需备份">
```

如果本次 PR **六类都无**，Comment 2 可以简化为一句话：
> 本次变更无依赖/migration/环境变量/部署/第三方配置变动，直接合并部署即可。

发布：`gh pr comment $PR_NUM --body-file /tmp/pr-release-${HEAD_SHA}.md`

### Step 7 — 终局输出

对话里简短打印：

```
✅ PR 已<提交|更新>: <PR_URL>

Title: <PR_TITLE>
Review: PASS @ <head_sha> (<timestamp>)
Rust 项目类型: <project_type> · 涉及 crate: <workspace_members 或 "单 crate">

已附加:
- 📝 PR 描述（背景 + 变更要点 + 兼容性说明）
- 🔍 Code Review 结果（Comment #1）
- 🚀 发布物料清单（Comment #2）

本次扫描到的关键物料:
- 依赖变更: <N 条或"无">
- Migration: <N 条或"无">
- 环境变量: <N 条或"无">
- 部署资源: <N 条或"无">
- 第三方配置: <N 条或"无">

请 reviewer 重点关注:
- <3 条以内，基于变更特征给出>
```

---

## 扫描纪律（延续 /review 风格）

**看到什么写什么，不靠猜**。

- 依赖变更：读 `Cargo.toml` + `Cargo.lock` 的 diff，不只看 `Cargo.toml` 声明
- 环境变量：grep 代码里实际出现的 `std::env::var` / `env!`，不只看 `.env.example`
- Migration：读文件内容判断破坏性，不只看文件名
- 第三方配置：找到代码里明确的 URL/endpoint 才列出
- 部署资源：只看真实变更的文件，不假设
- 没有就写"无"，不凑条目

## 禁止事项

- ❌ 不重新做代码审查
- ❌ 不合并 PR
- ❌ 不推送到保护分支
- ❌ 不自动跑 /review 来绕过校验
- ❌ 不跑 `cargo build` / `cargo clippy`（review 职责）
- ❌ 不写"建议检查一下 xxx"这种模糊项
- ❌ 不虚构 issue 编号、不猜测 scope
- ❌ 不省略 `Cargo.lock` 的分析（Cargo.toml 只声明，Lock 才是实际生效版本）
