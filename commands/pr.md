---
description: 代码审查通过后，提交 PR 并在评论中补齐 code-review 结果与发布物料清单
argument-hint: [目标分支,默认 main] [--draft] [rust|nextjs]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(cargo:*), Read, Grep, Glob
---

# /pr — 提交 PR 并补齐发布物料

把**已通过 /review 的变更**交付成可评审、可上线的 PR。只做三件事：

1. 提 PR，title 规范、body 写清改了什么和为什么
2. 把 /review 结果作为 PR comment 发出
3. 扫描变更，生成发布物料清单作为另一条 PR comment

## 参数解析

`$ARGUMENTS` 按空格切分：
- `--draft` → draft 模式
- `rust` / `nextjs` → 强制项目类型（影响物料扫描范围）
- 第一个非 flag、非类型的 token = 目标分支（默认 `main`）

---

## 前置检查（任一失败立即停下）

### A. Git 状态

1. 当前在 feature 分支（不能是 main/master/develop）
2. 工作区干净（`git status --porcelain` 为空）
3. 已推送到远端（有未推送的先 `git push -u origin <branch>`）
4. 检查是否已存在 PR（`gh pr view --json number,state`），已存在走更新流程

### B. Review 校验

```bash
BRANCH=$(git branch --show-current)
HEAD_SHA=$(git rev-parse --short=7 HEAD)
BRANCH_SAFE=$(echo "$BRANCH" | tr '/ ' '--' | tr -cd '[:alnum:]-_')
REVIEW_DIR=".claude/reviews/$BRANCH_SAFE"
REVIEW_MD="$REVIEW_DIR/$HEAD_SHA.md"
REVIEW_JSON="$REVIEW_DIR/$HEAD_SHA.json"
```

校验（按优先级）：
| 校验项 | 阻断？ | 不符提示 |
|---|---|---|
| 文件存在 | 阻断 | "未找到 commit `<SHA>` 的 review 记录，请先运行 /review" |
| JSON 可解析 | 阻断 | "review 记录损坏，请重跑 /review" |
| `judgment == "PASS"` | 阻断 | "review 判定 FAIL，请修复后重跑" |
| `head_sha` 匹配当前 HEAD | 阻断 | "review 后有新改动，请重跑 /review" |
| `branch` 匹配 | 仅告警 | 可能是 rename，代码未变可忽略 |
| `timestamp` >24h | 仅告警 | 提示可能需要重跑 |

---

## Step 1 — 检测项目类型

同 /review 的检测逻辑。影响 Step 4 的物料扫描范围。

## Step 2 — 关联 Issue

从分支名或 review JSON 提取 issue 编号。找到则读取 issue title/body 作为 PR 描述的上下文。

## Step 3 — 提交 / 更新 PR

PR title 格式：`<type>(<scope>): <描述>`（同 /spec 的标题规范）

PR body 结构：
- **背景**：关联 issue + 一句话说明为什么改
- **变更要点**：3-5 条核心改动
- **影响范围**：涉及的模块/页面
- **测试情况**：跑了什么验证

新建：`gh pr create --title "..." --body-file /tmp/pr-body.md [--draft]`
更新：`gh pr edit <num> --title "..." --body-file /tmp/pr-body.md`

## Step 4 — 贴 Review 结果

读取 `$REVIEW_MD`，作为 PR comment 发出。用 `<!-- review-bot:v1 -->` 标记幂等更新。

## Step 5 — 贴发布物料清单

扫描变更生成物料清单，用 `<!-- release-bot:v1 -->` 标记幂等更新。

**通用扫描项**：
- 环境变量：grep 代码里实际引用，不只看 `.env.example`
- SQL / Migration：读内容判断破坏性
- 第三方配置：找到代码里明确的 URL/endpoint 才列出

**Rust 项目额外**：
- `Cargo.toml` + `Cargo.lock` 依赖变更
- Docker / systemd 配置变更
- 新增 feature flags

**Next.js 项目额外**：
- `package.json` 依赖变更
- 构建配置变更

没有就写"无"，不凑条目。

## Step 6 — 总结

```
✅ PR 已<提交|更新>: <PR_URL>

Title: <PR_TITLE>
Review: PASS @ <head_sha> (<timestamp>)

已附加:
- 📝 PR 描述
- 🔍 Code Review 结果
- 🚀 发布物料清单

请 reviewer 关注:
- <≤3 条关键点>
```
