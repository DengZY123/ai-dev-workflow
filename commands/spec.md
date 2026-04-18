---
description: 基于需求生成技术方案，产出到 GitHub Issue，供人评审
argument-hint: <需求描述 或 issue URL/编号> [rust|nextjs]
allowed-tools: Bash(gh:*), Bash(cargo:*), Read, Grep, Glob, WebFetch, mcp__postgres-db__query
---

# /spec — 生成技术方案

把需求写成一份可评审、可执行的技术方案，存到 GitHub Issue 的 comment（不碰 body）。

- 方案只进 **comment**，issue body 永远不动
- 每次运行用首行标记 `<!-- spec-bot:v1 -->` **幂等更新**同一条 comment

## 输入

$ARGUMENTS

可能是自然语言需求、`#42`、或 issue URL。末尾可带 `rust` 或 `nextjs` 强制指定项目类型。

---

## Step 0 — 检测项目类型 & 加载规则

1. 如果 $ARGUMENTS 末尾有 `rust` / `nextjs`，按指定类型加载
2. 否则自动检测：
   - 存在 `Cargo.toml` → 加载 `spec-rules-rust` skill
   - 存在 `package.json` 且含 `next` 依赖 → 加载 `spec-rules-nextjs` skill
   - 两者都存在 → 两套都加载
3. **必读**：`spec-writing` skill（方案模板和写作规范）

skill 查找失败时打印缺失项并继续，不中断。

## Step 1 — 理解需求 & 扫描代码

- `#N` 或 URL → `gh issue view <num> --json number,title,body,comments,url,labels`
- 自然语言 → 直接作为需求

扫描代码库回答：改动落哪层 / 有无可复用模式 / 哪些不该动。用 Grep/Glob/Read，Rust 项目可用 `cargo metadata`。

## Step 2 — 判断是否澄清

以下情况先反问（一次问完 ≤5 条，每条附默认假设）：

- 用户场景 / 边界不清
- 非功能需求会影响方案（性能、并发、成本）
- 涉及外部系统但未说明对接方式

需求清晰就跳过。

## Step 3 — 方案粒度

- 单模块小改动 → 1 个推荐方案
- 跨模块有真实 trade-off → 2 个对比 + 推荐
- 新抽象 / 破坏兼容 → 2-3 个对比 + 推荐 + 风险专章

没有真实 trade-off 不硬凑对比。

## Step 4 — 估算规模

| Size | 典型特征 | 参考周期 |
|---|---|---|
| XS | 单文件、配置/文案 | <0.5 人日 |
| S | 单模块、<3 文件、无架构决策 | 0.5–1 人日 |
| M | 跨模块、少量抽象调整 | 2–4 人日 |
| L | 新模块/新抽象、需设计评审 | 1–2 周 |
| XL | 跨系统、架构级、破坏兼容 | >2 周，**先拆分** |

Size 后附一句理由。XL 先提示拆分，等用户确认再继续。

辅助字段：预计改动文件数（区间）、涉及模块、破坏兼容、数据迁移、新增依赖、需要架构评审。

> 本估算用于排期参考和风险识别，不作为工作量计量或绩效指标。

## Step 5 — 写方案

按 `spec-writing` skill 模板产出，叠加对应 `spec-rules-*` skill 的补充章节。章节有信息增量才写，无内容整节省略。

## Step 6 — 处理 issue 标题

格式：`<type>(<scope>)?: <动词开头的中文描述>`

- type：`feat` / `fix` / `refactor` / `perf` / `docs` / `test` / `chore` / `build` / `ci`
- scope：可选，小写，优先模块名/crate 名
- 描述 ≤50 字，动词开头，专有名词保留原文

已有 issue 标题匹配格式则跳过；不匹配则生成新标题，展示 `原 → 新`，用户未否决即执行。

## Step 7 — 幂等写入 comment & label

1. 方案写入临时文件 `/tmp/spec-<timestamp>.md`
2. 查已有 spec comment：`gh api repos/:owner/:repo/issues/<num>/comments --jq '.[] | select(.body | startswith("<!-- spec-bot:v1 -->")) | .id'`
3. 无 → 新增；有一条 → 更新；多条 → 更新最早，其余删除
4. 按 Size 打 label（`size/XS` ~ `size/XL`），risk 字段为真追加 `risk/*` label
5. 任一 `gh` 命令失败：打印完整错误 + 临时文件路径 + 可手动执行的命令

## Step 8 — 总结

简要告知：issue URL + comment URL、新增/更新、标题变更、Size + 理由、新增 label、推荐项、需要拍板的点（≤3 条）。
