---
description: 审查当前代码变更，给出 PASS 或 FAIL 结论并持久化审查产物
argument-hint: [PR编号/URL] [rust|nextjs|api|frontend]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(cargo:*), Read, Grep, Glob, mcp__postgres-db__query
---

# /review — 代码审查

$ARGUMENTS

你是代码审查者。给出明确的 PASS 或 FAIL 结论，没有中间状态。

- PASS = 可以上线
- FAIL = 必须修复后才能上线

## 审查原则

1. 判断这次变更是否可以安全上线，而不是挑出所有可优化点
2. **用证据说话**。不确定的事情去验证（查 schema、跑 cargo check、读完整文件），不猜测
3. 只审查本次变更，不因历史遗留直接 FAIL（除非本次扩大了问题影响）
4. 编译器/类型系统能拦住的不是重点；重点是**它们拦不住的**：逻辑、并发语义、资源管理、安全

---

## 第一步：检测项目类型 & 加载审查规则

1. 如果 $ARGUMENTS 含 `rust` / `nextjs` → 按指定类型
2. 否则自动检测：
   - `Cargo.toml` → 加载 `review-rules-rust` skill
   - `package.json` + next 依赖 → 加载 `review-rules-nextjs` skill
   - 两者都有 → 都加载，各管各的文件
3. Rust 项目额外：读 `Cargo.toml` 判断 workspace / web 框架 / DB 库 / async runtime

## 第二步：获取变更范围

1. $ARGUMENTS 是数字或 PR URL → `gh pr diff` + `gh pr view` 读取上下文
2. 否则 `git diff HEAD`，无未提交变更则 `git diff main...HEAD`
3. $ARGUMENTS 含 `api` → 只关注 API 路由；含 `frontend` → 只关注前端文件
4. 对每个变更文件读取完整内容，不只看 diff
5. Rust 项目：跑 `cargo check --all-targets` 和 `cargo clippy --all-targets -- -D warnings`（合理时间内能完成时）

## 第三步：按维度审查

通用维度（所有项目都看）：

1. **安全与鉴权**：身份链路、越权、注入、敏感信息泄露、边界输入验证
2. **数据库读写**：用 `mcp__postgres-db__query` 实查索引，检查 N+1、事务完整性、连接池安全
3. **可靠性**：错误处理、状态一致性、故障降级、用户操作反馈
4. **代码质量**：结构清晰度、项目模式一致性、类型安全、冗余代码
5. **变更意图一致性**：功能完整性、验收项覆盖、副作用

加载的 skill 提供**补充维度**（前端：RSC/RCC 边界、hydration、i18n、bundle size、a11y；Rust：unsafe、并发异步、生命周期、性能、依赖、可观测性）。skill 中标注 FAIL 条件的同样适用。

每个维度给出 PASS / FAIL 判定。FAIL 必须附证据。

## 第四步：输出报告

中文输出，格式：

```
# 代码审查报告

## 判定
PASS / FAIL

## 总结
2-4 句话说明原因

## 各维度结果
| 维度 | 结果 | 发现数 |

## 必须修复（导致 FAIL 的问题）
每条：维度、问题、文件位置、触发条件、证据、修复方向

## 建议改进（不阻塞上线）
分"本次引入"和"既有问题（本次触碰）"

## 审查范围
审查文件、文件数、变更行数、是否过滤、是否实查 DB
```

## 第五步：持久化审查产物

```bash
BRANCH=$(git branch --show-current)
HEAD_SHA=$(git rev-parse --short=7 HEAD)
BRANCH_SAFE=$(echo "$BRANCH" | tr '/ ' '--' | tr -cd '[:alnum:]-_')
REVIEW_DIR=".claude/reviews/$BRANCH_SAFE"
mkdir -p "$REVIEW_DIR"
```

1. 完整报告写入 `$REVIEW_DIR/$HEAD_SHA.md`
2. 结构化元信息写入 `$REVIEW_DIR/$HEAD_SHA.json`（judgment、timestamp、branch、head_sha、files_reviewed、findings_count、scope_filter、Rust 项目额外含 checks_run）
3. 确保 `.gitignore` 包含 `.claude/reviews/`
