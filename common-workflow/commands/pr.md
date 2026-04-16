---
description: 代码审查通过后，提交 PR 并在评论中补齐 code-review 结果与发布物料清单
argument-hint: [目标分支,默认 main] [--draft]
allowed-tools: Bash(gh:*), Bash(git:*), Read, Grep, Glob
---

# /pr — 提交 PR 并补齐发布物料

你的任务是把**已经通过 /review 的变更**正式交付成一个可评审、可上线的 PR。
你**不做代码审查**(/review 已完成),你只做三件事:

1. 提 PR,body 写清**这次改了什么、为什么**
2. 把本次 /review 的结果作为 PR comment 发出来
3. 扫描变更,生成**发布物料清单**(SQL、环境变量、第三方配置)作为另一条 PR comment

## 参数

$ARGUMENTS

- 第一个非 flag 参数 = 目标分支(默认 `main`)
- `--draft` = 创建 draft PR

---

## 前置检查(任一失败立即停下)

### A. Git 状态检查

1. **当前在 feature 分支**:`git branch --show-current`,不能是 `main` / `master` / `develop`
2. **工作区干净**:`git status --porcelain` 必须为空
3. **已推送到远端**:`git log origin/<branch>..HEAD --oneline`,有未推送的先 `git push -u origin <branch>`
4. **没有已存在 PR**:`gh pr view --json number,state`,已存在则提示用户选择"更新"或退出,默认**更新**

### B. Code Review 校验(核心防呆)

5. **定位 review 记录**:

```bash
   BRANCH=$(git branch --show-current)
   HEAD_SHA=$(git rev-parse --short=7 HEAD)
   BRANCH_SAFE=$(echo "$BRANCH" | tr '/ ' '--' | tr -cd '[:alnum:]-_')

   REVIEW_DIR=".claude/reviews/$BRANCH_SAFE"
   REVIEW_MD="$REVIEW_DIR/$HEAD_SHA.md"
   REVIEW_JSON="$REVIEW_DIR/$HEAD_SHA.json"
```

6. **校验两个文件都存在**:

   - `REVIEW_JSON` 或 `REVIEW_MD` 不存在 → 停下:
     > 未找到当前 commit(`<HEAD_SHA>`)在分支 `<branch>` 的 code-review 记录。
     > 请先运行 `/review`。
     >
     > (如果刚跑过 review,可能是 review 后又提交了新代码,请重跑 review。)

7. **校验 JSON 内容**,任一不符停下:

   | 校验项 | 不符时的提示 |
   |---|---|
   | JSON 能解析 | "review 记录损坏,请重新运行 /review" |
   | `judgment == "PASS"` | "最近一次 /review 判定为 FAIL,请修复后重跑" |
   | `branch == $(git branch --show-current)` | "review 的分支与当前分支不符(review: X, 当前: Y)" |
   | `head_sha == $(git rev-parse --short=7 HEAD)` | "review 之后代码有新改动,请重新运行 /review" |

   > 注:按"分支 + commit"的目录结构,正常情况下 branch 和 head_sha 一定匹配。这些校验是兜底,防止文件被手动移动或 json 被篡改。

8. **时效性提示(不阻断)**:
   - `timestamp` 距今 > 24 小时 → 打印警告,但继续:
     > ⚠️ review 结果已超过 24 小时,如代码未变请忽略;如有疑虑请重跑 /review。

校验全过后,把 `REVIEW_MD` 路径、`judgment`、`timestamp`、`head_sha` 存起来,Step 4 用。

---

## 执行步骤

### Step 1 — 收集变更信息

```bash
TARGET=${1:-main}
git log $TARGET..HEAD --oneline
git diff $TARGET...HEAD --stat
git diff $TARGET...HEAD
```

读取所有修改文件(重点文件全读,其他扫一眼)。如果 commit message 或分支名里有 issue 编号(`feat/42-xxx` 或 `refs #42`),记下来。

### Step 2 — 生成 PR Body

**原则**:PR body 是"变更说明",不是 changelog,不是 review 报告。让 reviewer 在 30 秒内知道"这个 PR 想干嘛、改了哪些关键位置"。

模板:

```markdown
## 背景

<1-3 句,说清为什么做。有 spec 就链接过去: Closes #42 / Refs #42>

## 变更要点

- <关键改动 1,精确到模块/文件级别>
- <关键改动 2>
- <...>

## 关键实现决策

<只写非显然的决策,例如"选用 X 而非 Y 因为..."。没有就写"无特殊决策">

## 自测情况

- [ ] 本地构建通过
- [ ] 核心路径手测过: <列出具体路径>
- [x] /review 已通过(见评论)

## 相关链接

- Spec: #<num>(如有)
```

禁止:
- ❌ "本 PR 完整解决了 xxx 问题"这种空话
- ❌ 罗列所有文件改动(diff 里有)
- ❌ 在 body 里贴 review 结果(要放 comment)

### Step 3 — 创建或更新 PR

body 写入 `/tmp/pr-body-<ts>.md`,然后:

**新建**:
```bash
gh pr create \
  --base $TARGET \
  --title "<type>: <简短标题>" \
  --body-file /tmp/pr-body-<ts>.md \
  [--draft]
```

title 前缀:`feat` / `fix` / `refactor` / `perf` / `docs` / `chore`。

**更新**:`gh pr edit <num> --body-file /tmp/pr-body-<ts>.md`

拿到 PR URL 和 number。

### Step 4 — 发布 Code Review 结果(Comment 1)

直接读取前置检查中确定的 `REVIEW_MD` 文件内容。

写入 `/tmp/pr-review-<ts>.md`:

```markdown
## 🔍 Code Review 结果

> 由 /review 于 `<timestamp>` 产出,判定:**PASS**
> 审查 commit: `<head_sha>` · 审查文件数: `<files_count>`

<原样贴入 REVIEW_MD 的完整内容>

---

**⚠️ 作为 reviewer,以下是自动 review 无法覆盖、需要你人工确认的:**

1. **业务逻辑正确性**:review 判断代码写法,不判断"这是否符合业务预期"
2. **文案与 UX**:review 的 i18n 检查只能发现硬编码,看不出翻译是否合适
3. **性能的实际影响**:review 基于 schema 判断索引,真实数据量下的性能需人工判断
```

发布:`gh pr comment <num> --body-file /tmp/pr-review-<ts>.md`

### Step 5 — 发布物料清单(Comment 2)

**这是 /pr 最核心的产出**。扫描 diff,严格按以下三类生成清单。**只列真实存在的,不存在的类别明写"无"**。

#### 5.1 SQL 变动扫描

扫描:
- 新增/修改的 `.sql` 文件
- 新增/修改的 migration 文件(`migrations/`、`supabase/migrations/`、`db/migrations/` 等)
- 代码里内联的 `CREATE TABLE` / `ALTER TABLE` / `CREATE INDEX` / `DROP`

对每条判断:
- **是否破坏性**:DROP、ALTER COLUMN 改类型、NOT NULL 加约束、删列 → 破坏性
- **执行时机**:新增表/列被新代码读写 → **代码发布前**;删除旧列/表 → **代码发布后**
- **大表风险**:在 balance_transactions、api_keys 等大表上 `ALTER TABLE ADD COLUMN NOT NULL` 或 `CREATE INDEX`(非 CONCURRENTLY)→ 标红警告

#### 5.2 环境变量扫描

扫描:
- 代码中新出现的 `process.env.XXX`(相对 target 分支对比)
- `.env.example` / `.env.sample` 的变更
- Rust 的 `std::env::var` / `env!`

对每条判断:
- 端:服务端 / 客户端(客户端必须 `NEXT_PUBLIC_` 前缀)
- 必填 / 选填(没有会 crash?走降级?)
- 是否密钥(API key、secret、token)→ 提醒走密钥管理平台
- 目标环境:dev / staging / prod 都要配吗

#### 5.3 第三方配置扫描

识别:
- 新增/修改的 webhook / 回调 URL
- 新厂商接入(provider adapter 新文件)→ 厂商控制台配额度/白名单
- OAuth client 变更
- DNS / CDN / 存储桶策略变更
- 定时任务变更(cron)
- 外部 service(Redis / 队列)的 key prefix / channel 变更

#### 5.4 Comment 格式

写入 `/tmp/pr-release-<ts>.md`:

```markdown
## 🚀 发布物料清单

> 由 /pr 自动扫描本次变更生成。**请部署人按此清单逐项执行**。

### 1. SQL 变动

<无则写"无 SQL 变动。",否则逐条:>

#### 📄 <文件路径 或 "内联 SQL @ 文件:行">
- **执行时机**:代码发布前 / 代码发布后 / 任意时机
- **是否破坏性**:是 / 否
- **风险提示**:<大表 DDL、锁表风险等,无则"无">
- **SQL**:
  ```sql
  <原样贴>
  ```
- **回滚**:<回滚 SQL 或"不可回滚,需重建">

### 2. 环境变量

<无则写"无环境变量变动。">

| 变量名 | 端 | 必填 | 是否密钥 | 用途 | 默认值/降级 |
|---|---|---|---|---|---|
| `XXX` | 服务端 | ✅ | ✅ | ... | 无,未配置启动失败 |

**操作清单**:
- [ ] dev 环境已配置
- [ ] staging 环境已配置
- [ ] prod 环境已配置
- [ ] 密钥类已通过密钥管理平台写入,未出现在 git 历史

### 3. 第三方配置

<无则写"无第三方配置变动。",否则逐条:>

- [ ] **厂商 xAI**:在 xAI 控制台为 prod 域名添加 API Key,配额设为 xxx
- [ ] **Webhook**:在 <供应商> 后台更新回调地址为 `https://xxx.com/api/webhook/yyy`

### 4. 发布顺序

<基于上面扫描结果给出具体顺序,例如:>

1. prod 执行 SQL: `migrations/20260416_add_xxx.sql`
2. 配置 prod 环境变量 `XXX_API_KEY`
3. 在 xAI 控制台添加域名白名单
4. 合并 PR,触发部署
5. 部署完成后冒烟测试:<1-2 条路径>

### 5. 回滚预案

1. revert PR 并重新部署
2. <如有需配套回滚的 SQL,列出>
3. <不可逆部分写清:"xxx 不可完全回滚,需手动处理">
```

如果本次 PR **三类都无**,Comment 2 可以简化为一句话:
> 本次变更无 SQL、环境变量、第三方配置变动,直接合并部署即可。

发布:`gh pr comment <num> --body-file /tmp/pr-release-<ts>.md`

### Step 6 — 终局输出

对话里简短打印:

```
✅ PR 已提交: <PR URL>

Review: PASS @ <head_sha> (<timestamp>)

已附加:
- 📝 PR 描述(背景 + 变更要点)
- 🔍 Code Review 结果(Comment #1)
- 🚀 发布物料清单(Comment #2)

请 reviewer 关注:
- <3 条以内,本次需要人工确认的关键点>
```

---

## 扫描纪律(延续 /review 风格)

**看到什么写什么,不靠猜**。

- 环境变量:看代码里实际出现的 `process.env.XXX`,不只看 `.env.example`
- SQL:读内容判断破坏性,不只看文件名
- 第三方配置:找到代码里明确的 URL/endpoint 才列出
- 没有就写"无",不凑条目

## 禁止事项

- ❌ 不重新做代码审查
- ❌ 不合并 PR
- ❌ 不推送到保护分支
- ❌ 不自动跑 review 来绕过校验
- ❌ 不写"建议检查一下 xxx"这种模糊项
