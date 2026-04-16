---
description: 代码审查通过后，提交 PR 并在评论中补齐 code-review 结果与发布物料清单
argument-hint: [目标分支,默认 main] [--draft]
allowed-tools: Bash(gh:*), Bash(git:*), Read, Grep, Glob
---

# /pr — 提交 PR 并补齐发布物料

你的任务是把**已经通过 /review 的变更**正式交付成一个可评审、可上线的 PR。
你**不做代码审查**(/review 已完成),你只做三件事:

1. 提 PR,title 规范、body 写清**这次改了什么、为什么**
2. 把本次 /review 的结果作为 PR comment 发出来
3. 扫描变更,生成**发布物料清单**(SQL、环境变量、第三方配置)作为另一条 PR comment

## 参数解析

`$ARGUMENTS` 是一整个字符串,你要自己按空格切分并分类:

- 以 `--` 开头的 token 是 flag,目前只识别 `--draft`
- 第一个非 flag token = 目标分支(默认 `main`)
- 其他 token 忽略并提示用户

示例:
- `$ARGUMENTS = "develop --draft"` → 目标分支 `develop`,draft 模式
- `$ARGUMENTS = "--draft main"` → 目标分支 `main`,draft 模式
- `$ARGUMENTS = ""` → 目标分支 `main`,非 draft

---

## 前置检查(任一失败立即停下)

### A. Git 状态检查

1. **当前在 feature 分支**:`git branch --show-current`,不能是 `main` / `master` / `develop`
2. **工作区干净**:`git status --porcelain` 必须为空
3. **已推送到远端**:`git log origin/<branch>..HEAD --oneline`,有未推送的先 `git push -u origin <branch>`
4. **是否已存在 PR**:`gh pr view --json number,state` 判断,已存在则默认走**更新**流程(Step 3 会区分)

### B. Code Review 校验(核心防呆)

5. **定位 review 记录**(与 /review 使用的路径生成方式必须一致):

```bash
   BRANCH=$(git branch --show-current)
   HEAD_SHA=$(git rev-parse --short=7 HEAD)
   # 分支名消毒:/、空格、特殊字符都换成 -
   BRANCH_SAFE=$(echo "$BRANCH" | tr '/ ' '--' | tr -cd '[:alnum:]-_')

   REVIEW_DIR=".claude/reviews/$BRANCH_SAFE"
   REVIEW_MD="$REVIEW_DIR/$HEAD_SHA.md"
   REVIEW_JSON="$REVIEW_DIR/$HEAD_SHA.json"
```

> ⚠️ 若 `/review` 的路径算法未来调整,请同步修改此处,保持两侧一致。

6. **校验两个文件都存在**:

   - `REVIEW_JSON` 或 `REVIEW_MD` 不存在 → 停下:
     > 未找到当前 commit(`<HEAD_SHA>`)的 code-review 记录。
     > 请先运行 `/review`。
     >
     > (如果刚跑过 review,可能是 review 后又提交了新代码,请重跑 review。)

7. **校验 JSON 内容**,按以下优先级,任一不符停下:

   | 校验项 | 是否阻断 | 不符时的提示 |
   |---|---|---|
   | JSON 能解析 | ✅ 阻断 | "review 记录损坏,请重新运行 /review" |
   | `judgment == "PASS"` | ✅ 阻断 | "最近一次 /review 判定为 FAIL,请修复后重跑" |
   | `head_sha == $(git rev-parse --short=7 HEAD)` | ✅ 阻断 | "review 之后代码有新改动,请重新运行 /review" |
   | `branch == $(git branch --show-current)` | ⚠️ 仅告警 | "review 时分支名为 X,当前为 Y(可能是 rename),如代码未变可忽略" |

   > head_sha 是真正的防呆锚点;分支名校验只做告警,允许用户 rename 分支。

8. **时效性提示(不阻断)**:
   - `timestamp` 距今 > 24 小时 → 打印警告,但继续:
     > ⚠️ review 结果已超过 24 小时,如代码未变请忽略;如有疑虑请重跑 /review。

校验全过后,把 `REVIEW_MD` 路径、`judgment`、`timestamp`、`head_sha`、`files_count` 存起来,Step 4 用。

---

## 执行步骤

### Step 1 — 收集变更信息

```bash
TARGET=<解析得到的目标分支>
git log $TARGET..HEAD --oneline
git diff $TARGET...HEAD --stat
git diff $TARGET...HEAD
```

读取所有修改文件(重点文件全读,其他扫一眼)。

**提取 issue 编号**:扫描所有 commit message 和当前分支名,用以下正则匹配:
- `#(\d+)`(如 `refs #42`、`closes #42`)
- `^(feat|fix|refactor|perf|docs|chore)/(\d+)[-_]` 开头的分支名(如 `feat/42-xxx`)

把**第一个**匹配到的编号存为 `ISSUE_NUM`,Step 2 的 body 和 Step 3 的 title 可能用到。没有就设为空。

### Step 2 — 生成 PR Title(规范)

Title 是 reviewer 在列表里唯一看到的东西,必须能一眼看懂。

**格式**:`<type>(<scope>): <简短描述>`

**规则**(逐条强制):

1. **type 必须是以下之一**,基于本次改动的主要性质:
   - `feat` — 新功能
   - `fix` — 修 bug
   - `refactor` — 不改行为的重构
   - `perf` — 性能优化
   - `docs` — 只改文档/注释
   - `chore` — 构建、依赖、配置、脚手架
   - `test` — 只改测试
   - `style` — 格式化、空格、分号(不涉及逻辑)

2. **scope 可选**,用小写英文,写最主要的改动范围:
   - 有明确模块/目录 → 填,如 `feat(auth): 支持 OAuth 登录`
   - 跨多个模块或根目录级改动 → 省略,如 `chore: 升级 Node 到 20`
   - scope 不要堆叠,最多一个,不写 `feat(auth,payment): xxx`

3. **描述部分**:
   - 用**祈使句**,动词开头:"添加"、"修复"、"重构"、"升级",**不**用"已添加"、"完成了"
   - **长度**:整个 title(含 prefix)不超过 **72 字符**,中文按 2 字符计,理想在 50 字符内
   - **不加句号**,不加感叹号
   - **不要**把 issue 编号塞进 title(`#42` 放 body 的 Closes 行)
   - **不要**前缀 "PR:" / "【xxx】" / emoji

4. **type 判定优先级**(多类改动时,按影响选主要的):
   `feat` > `fix` > `perf` > `refactor` > `docs` > `test` > `style` > `chore`
   例:同时新增功能 + 改了构建脚本,用 `feat` 不用 `chore`。

**好例子**:
- `feat(auth): 支持 OAuth 登录`
- `fix(billing): 修复余额为负时的结算异常`
- `refactor(api): 拆分 provider adapter 基类`
- `perf(search): 为 name 字段添加 trigram 索引`
- `chore: 升级 pnpm 到 9.x`

**坏例子**:
- ❌ `Feat: Added new login feature.`(大写前缀 + 完成时 + 句号)
- ❌ `feat: 完整支持 OAuth 登录,修复若干 bug,并重构了 auth 模块,详见 body`(太长,多件事混在一起)
- ❌ `feat(auth,ui,api): xxx`(多 scope)
- ❌ `feat: 支持 OAuth 登录 #42`(issue 号不进 title)

把最终决定的 title 存为 `PR_TITLE`。

### Step 3 — 生成 PR Body

**原则**:PR body 是"变更说明",不是 changelog,不是 review 报告。让 reviewer 在 30 秒内知道"这个 PR 想干嘛、改了哪些关键位置"。

模板:

```markdown
## 背景

<1-3 句,说清为什么做>

<如果 Step 1 提取到 ISSUE_NUM,追加一行: Closes #<ISSUE_NUM>>

## 变更要点

- <关键改动 1,精确到模块/文件级别>
- <关键改动 2>
- <...>

## 关键实现决策

<只写非显然的决策,例如"选用 X 而非 Y 因为..."。没有就写"无特殊决策">

## 自测情况

> 由 /pr 自动填入。本地构建/手测项需作者在 PR 创建后自行勾选。

- [ ] 本地构建通过
- [ ] 核心路径手测过:<列出具体路径,作者请替换>
- [x] /review 已通过(见评论)

## 相关链接

<如有 spec/设计文档链接写这里,没有就删掉本节>
```

**禁止**:
- ❌ "本 PR 完整解决了 xxx 问题"这种空话
- ❌ 罗列所有文件改动(diff 里有)
- ❌ 在 body 里贴 review 结果(要放 comment)
- ❌ 虚构 issue 编号 — ISSUE_NUM 为空就不写 Closes 行

### Step 4 — 创建或更新 PR

body 写入 `/tmp/pr-body-${HEAD_SHA}.md`(用 commit sha 做文件名,避免并发冲突),然后:

**新建**:
```bash
gh pr create \
  --base $TARGET \
  --title "$PR_TITLE" \
  --body-file /tmp/pr-body-${HEAD_SHA}.md \
  [--draft]
```

**更新已存在的 PR**:
```bash
gh pr edit <num> \
  --title "$PR_TITLE" \
  --body-file /tmp/pr-body-${HEAD_SHA}.md
```

拿到 PR URL 和 number,存为 `PR_NUM` 和 `PR_URL`。

### Step 5 — 发布 Code Review 结果(Comment 1)

**处理旧 comment**(仅更新场景):

先查 PR 上的 comment 列表:
```bash
gh pr view $PR_NUM --json comments --jq '.comments[] | {id, body: .body[0:80]}'
```

找到以 `## 🔍 Code Review 结果` 开头的旧 comment,用 `gh api` 删除:
```bash
gh api -X DELETE /repos/:owner/:repo/issues/comments/<comment_id>
```

> 如果删除失败(比如权限),退化为在旧 comment 顶部追加 `> ⚠️ 已被更新版本替代` 的说明。

读取前置检查中确定的 `REVIEW_MD` 文件内容,写入 `/tmp/pr-review-${HEAD_SHA}.md`:

```markdown
## 🔍 Code Review 结果

> 由 /review 于 `<timestamp>` 产出,判定:**PASS**
> 审查 commit: `<head_sha>` · 审查文件数: `<files_count>`

<原样贴入 REVIEW_MD 的完整内容>

---

**⚠️ 以下事项自动 review 无法覆盖,请 reviewer 重点人工确认:**

1. **业务逻辑正确性**:review 判断代码写法,不判断"这是否符合业务预期"
2. **文案与 UX**:review 的 i18n 检查只能发现硬编码,看不出翻译是否合适
3. **性能的实际影响**:review 基于 schema 判断索引,真实数据量下的性能需人工判断
```

发布:`gh pr comment $PR_NUM --body-file /tmp/pr-review-${HEAD_SHA}.md`

### Step 6 — 发布物料清单(Comment 2)

**处理旧 comment**:同 Step 5,查找以 `## 🚀 发布物料清单` 开头的旧 comment,删除或标记过期。

**这是 /pr 最核心的产出**。扫描 diff,严格按以下三类生成清单。**只列真实存在的,不存在的类别明写"无"**。

#### 6.1 SQL 变动扫描

扫描:
- 新增/修改的 `.sql` 文件
- 新增/修改的 migration 文件(`migrations/`、`supabase/migrations/`、`db/migrations/`、`prisma/migrations/` 等)
- 代码里内联的 `CREATE TABLE` / `ALTER TABLE` / `CREATE INDEX` / `DROP`

对每条判断:
- **是否破坏性**:DROP、ALTER COLUMN 改类型、NOT NULL 加约束、删列 → 破坏性
- **执行时机**:新增表/列被新代码读写 → **代码发布前**;删除旧列/表 → **代码发布后**
- **大表风险**:在项目中**被频繁读写或体量较大的表**上执行 `ALTER TABLE ADD COLUMN NOT NULL`、非 CONCURRENTLY 的 `CREATE INDEX`、锁表 DDL → 标红警告

> 判断大表的依据优先级:
> 1. 项目根目录 `CLAUDE.md` 或 `.claude/project.md` 中声明的大表清单
> 2. 从 schema/migration 历史判断的事务类、鉴权类、日志类表
> 3. 保守标注"⚠️ 请人工确认此表体量"

#### 6.2 环境变量扫描

扫描:
- 代码中新出现的环境变量引用(相对 target 分支对比新增的)
  - Node/TS: `process.env.XXX`
  - Rust: `std::env::var("XXX")` / `env!("XXX")`
  - Python: `os.environ["XXX"]` / `os.getenv("XXX")`
  - Go: `os.Getenv("XXX")`
- `.env.example` / `.env.sample` / `.env.template` 的变更

对每条判断:
- **端**:服务端 / 客户端
  - 客户端变量需符合当前框架的公开前缀约定(Next.js=`NEXT_PUBLIC_`,Vite=`VITE_`,Nuxt=`NUXT_PUBLIC_`,CRA=`REACT_APP_`);从 `package.json` 依赖推断框架
- **必填 / 选填**(没有会 crash?走降级?)
- **是否密钥**(API key、secret、token、private key)→ 提醒走密钥管理平台,警告不要提交到 git
- **目标环境**:dev / staging / prod 都要配吗

#### 6.3 第三方配置扫描

识别:
- 新增/修改的 webhook / 回调 URL
- 新厂商接入(provider adapter 新文件、SDK 新依赖)→ 厂商控制台配额度/白名单
- OAuth client / App 变更
- DNS / CDN / 存储桶策略变更
- 定时任务变更(cron、scheduled job)
- 外部 service(Redis / MQ / 队列)的 key prefix / channel / topic 变更

#### 6.4 Comment 格式

写入 `/tmp/pr-release-${HEAD_SHA}.md`:

```markdown
## 🚀 发布物料清单

> 由 /pr 基于 commit `<head_sha>` 扫描本次变更生成。**请部署人按此清单逐项执行**。

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

- [ ] **厂商 <name>**:在 <控制台> 为 prod 域名添加 API Key,配额设为 xxx
- [ ] **Webhook**:在 <供应商> 后台更新回调地址为 `https://xxx.com/api/webhook/yyy`

### 4. 发布顺序

<基于上面扫描结果给出具体顺序,例如:>

1. prod 执行 SQL: `migrations/20260416_add_xxx.sql`
2. 配置 prod 环境变量 `XXX_API_KEY`
3. 在 <厂商> 控制台添加域名白名单
4. 合并 PR,触发部署
5. 部署完成后冒烟测试:<1-2 条路径>

### 5. 回滚预案

1. revert PR 并重新部署
2. <如有需配套回滚的 SQL,列出>
3. <不可逆部分写清:"xxx 不可完全回滚,需手动处理">
```

如果本次 PR **三类都无**,Comment 2 可以简化为一句话:
> 本次变更无 SQL、环境变量、第三方配置变动,直接合并部署即可。

发布:`gh pr comment $PR_NUM --body-file /tmp/pr-release-${HEAD_SHA}.md`

### Step 7 — 终局输出

对话里简短打印:

```
✅ PR 已<提交|更新>: <PR_URL>

Title: <PR_TITLE>
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

- 环境变量:看代码里实际出现的引用,不只看 `.env.example`
- SQL:读内容判断破坏性,不只看文件名
- 第三方配置:找到代码里明确的 URL/endpoint 才列出
- 没有就写"无",不凑条目

## 禁止事项

- ❌ 不重新做代码审查
- ❌ 不合并 PR
- ❌ 不推送到保护分支
- ❌ 不自动跑 review 来绕过校验
- ❌ 不写"建议检查一下 xxx"这种模糊项
- ❌ 不虚构 issue 编号、不猜测 scope