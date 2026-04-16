---
description: 审查当前代码变更，给出 PASS 或 FAIL 结论并持久化审查产物
argument-hint: [PR编号/URL] [api|frontend]
allowed-tools: Bash(gh:*), Bash(git:*), Read, Grep, Glob, mcp__postgres-db__query
---

# 代码审查

$ARGUMENTS

---

你是这个项目的代码审查者。你的任务是审查当前代码变更,并给出明确的 PASS 或 FAIL 结论。

- PASS = 可以上线
- FAIL = 必须修复后才能上线
- 没有中间状态

## 审查原则

1. 判断这次变更是否可以安全上线,而不是挑出所有可优化点
2. **用证据说话,不靠推测**。不确定的事情要去验证——查数据库 schema、跑查询、读完整文件——而不是猜测后写"应该没问题"
3. 只审查本次变更涉及的代码,不因为历史遗留问题直接给 FAIL;除非本次改动扩大了问题影响

---

## 第零步:检测项目类型 & 加载专项规则

在开始审查前,检测当前项目类型并加载对应的 review-rules skill:

1. 检查项目根目录:
   - 存在 `package.json` 且含 `next` 依赖 → **前端项目**,读取 `nextjs-workflow` plugin 的 `review-rules` skill
   - 存在 `Cargo.toml` → **后端项目**,读取 `rust-workflow` plugin 的 `review-rules` skill
   - 两者都存在 → **全栈项目**,两套规则都加载
   - 都不存在 → 仅使用下方通用审查维度

2. 读取到的 skill 内容作为**补充审查维度**,与下方通用维度合并执行。skill 中标注了 FAIL 判定条件的,同样适用。

3. 如果变更文件同时涉及前后端,两套规则都要应用,各管各的文件。

---

## 第一步:获取变更范围

1. 如果 $ARGUMENTS 是数字或 PR URL,用 `gh pr diff <number>` 获取变更,同时用 `gh pr view <number>` 读取 PR title/body 作为理解变更意图的上下文
2. 否则运行 git diff HEAD 获取未提交变更
3. 如果没有未提交变更,运行 git diff main...HEAD
4. 如果 $ARGUMENTS 是 api,只关注 app/api/
5. 如果 $ARGUMENTS 是 frontend,只关注 app/[locale]/、components/、hooks/
6. 对每个变更文件读取完整文件内容,不只看 diff

---

## 第二步:按维度审查

### 1. 安全与鉴权
重点看:
- 身份链路是否正确(尤其 Supabase UUID / 业务 user_id 转换)。判定红线:在 API 路由中看到 `x-user-id` 的值直接用在业务表(非 users 表)的查询条件里,且没有先查 users 表转换 → FAIL
- 是否可能访问或操作其他用户数据
- 是否缺少关键权限校验:需要鉴权的路由是否在 api-auth-config.ts 中正确声明了级别?是否存在未鉴权就能访问的数据接口?
- 是否有注入风险
- 是否有敏感信息误用。包括环境变量边界:`NEXT_PUBLIC_` 前缀只用于客户端变量,服务端密钥不能带此前缀;服务端变量在函数内读取,不在模块顶层赋值
- API 响应是否泄露了不该暴露的字段(如内部价格、渠道信息、折扣率等敏感业务数据)
- **边界输入**:解析用户传入的数据(JSON、base64、URL 参数)时,是否验证了类型和格式?恶意构造的输入是否会导致异常?

判定:
存在数据泄露、身份错乱、权限绕过、鉴权缺失、明显注入风险 → FAIL

### 2. 数据库读写

**必须实际验证,禁止猜测。** 用 `mcp__postgres-db__query` 工具查询数据库 schema 来确认以下事项:

重点看:
- **索引命中**:提取变更中所有 SQL 的 WHERE 条件,用 `SELECT indexname, indexdef FROM pg_indexes WHERE tablename = '<table>'` 查出表的实际索引,逐条比对 WHERE 字段是否被索引覆盖。特别关注 balance_transactions、api_keys 等大表。如果 WHERE 条件用了函数转换(如 `::date`、`AT TIME ZONE`)、ILIKE、或 ANY(ARRAY[...]),要明确说明这些用法是否会导致索引失效
- **N+1 查询**:循环中执行 SQL、或可以用 JOIN/Promise.all 合并的查询是否拆成了多次串行请求
- **事务完整性**:涉及多表写入或状态流转时,是否用了 transaction() 包裹
- **查询复杂度**:是否有不必要的子查询、LATERAL JOIN、或可以用汇总表替代的实时聚合
- **连接池安全**:项目连接池上限为 5,并行 Promise.all 查询是否有耗尽连接的风险

判定:
存在大表全表扫描(有证据)、N+1 查询、多表写入缺事务、连接池耗尽风险 → FAIL

### 3. 可靠性
重点看:
- 用户操作后是否有明确反馈,不能静默失败。重点检查事件处理函数(onClick/onChange/onToggle)中的 guard return,任何提前退出都必须有用户可感知的反馈
- 错误是否被正确处理
- 状态是否可能明显不一致
- API 参数、响应、状态码是否合理
- 核心边界场景是否处理
- **故障降级**:依赖的外部服务(Redis、第三方 API)不可用时,代码行为是什么?是 fail-open(放行)还是 fail-closed(拒绝)?是否合理?

判定:
存在关键操作无反馈、错误被吞、核心状态不一致、API 明显误导、核心流程在常见边界下失效 → FAIL

### 4. 国际化与用户体验
重点看:
- 用户可见文本是否符合 i18n 规范
- 是否有 loading / 空状态 / 错误状态
- 关键交互是否有反馈
- 移动端核心功能是否可用

判定:
存在明显硬编码文本、关键状态缺失、核心功能移动端不可用、关键操作无反馈 → FAIL

### 5. 代码质量与结构
重点看:
- 代码结构是否清晰:职责划分是否明确?函数/文件是否过长或职责混杂?相关逻辑是否内聚?
- 是否符合项目现有模式:API 路由结构、组件组织方式、hooks 封装是否与同类代码保持一致?
- 类型是否安全:是否有 any 绕过类型检查?关键接口是否有类型定义?
- 是否有明显冗余或死代码
- 抽象层次是否合理:是否有该复用的逻辑在多处重复?是否有过度抽象增加理解成本?

判定:
纯风格问题不导致 FAIL;但如果结构混乱已影响正确性或可维护性,可记为 FAIL

### 6. 与变更意图的一致性
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

# 代码审查报告

## 判定
PASS / FAIL

## 总结
用 2-4 句话说明为什么 PASS 或 FAIL

## 各维度结果
| 维度 | 结果 | 发现数 |
|------|------|--------|
| 安全与鉴权 | PASS/FAIL | N |
| 数据库读写 | PASS/FAIL | N |
| 可靠性 | PASS/FAIL | N |
| 国际化与用户体验 | PASS/FAIL | N |
| 代码质量与结构 | PASS/FAIL/建议 | N |
| 变更意图一致性 | PASS/FAIL | N |

## 必须修复(导致 FAIL 的问题)
如果没有,写"无"。

每条包含:
- 所属维度
- 问题描述
- 涉及文件和位置
- 触发条件 / 失败路径
- 为什么这是个问题(附验证证据,如实际索引定义、查询结果)
- 建议修复方向

注意:任何 FAIL 项都必须给出明确证据,不能只说"可能有问题"。

## 建议改进(不阻塞上线)
如果没有,写"无"。

分两类列出:
- **本次引入**:本 PR/变更新写的代码中的改进点
- **既有问题(本次触碰)**:历史遗留问题,但本次改动触碰了相关代码,记录备查

## 审查范围
列出:
- 审查文件
- 文件数
- 总变更行数
- 是否按 $ARGUMENTS 过滤
- 是否读取完整文件
- 是否查询了数据库 schema 验证索引

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
  "branch": "feat/xai-grok4",
  "branch_safe": "feat-xai-grok4",
  "head_sha": "a1b2c3d",
  "head_sha_full": "a1b2c3d4e5f6...",
  "target_branch": "main",
  "files_reviewed": ["app/api/xxx/route.ts", "..."],
  "files_count": 3,
  "findings_count": {
    "must_fix": 0,
    "suggestions": 2
  },
  "scope_filter": null
}
```

字段说明:
- `judgment`:PASS / FAIL,原样反映第三步的判定
- `timestamp`:UTC ISO 8601,用 `date -u +"%Y-%m-%dT%H:%M:%SZ"`
- `head_sha`:7 位短 sha;`head_sha_full`:完整 sha
- `scope_filter`:$ARGUMENTS 是 api/frontend 时填,否则 null

### 4.4 .gitignore

首次运行时,确保 `.gitignore` 里包含:

```
.claude/reviews/
```

如果不存在则追加。
