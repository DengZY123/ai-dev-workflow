---
description: 基于需求生成 Rust 项目的技术方案，产出到 GitHub Issue，供人评审
argument-hint: <需求描述 或 issue URL/编号>
allowed-tools: Bash(gh:*), Bash(cargo:*), Read, Grep, Glob, WebFetch, mcp__postgres-db__query
---

# /spec — 生成 Rust 技术方案

把需求写成一份"像同事写给评审看"的技术方案，存到 GitHub Issue 的 comment（不碰 body），让评审者 3 分钟内看懂：要改什么、怎么改、为什么、风险、要拍板什么。

- 方案只进 **comment**，issue body 永远不动
- 每次运行都用首行标记 `<!-- spec-bot:v1 -->` **幂等更新**同一条 comment，不新增

## 输入

$ARGUMENTS

可能是自然语言需求、`#42`、或 issue URL。

---

## Step 0 — 加载规则

**必读** `rust-workflow` plugin 的 `spec-writing` skill（方案模板、各节写法、正反例）和 `spec-rules` skill（Rust 专项关注点：公开 API、MSRV、unsafe、编译时间、数据库实查等）。

**本 command 不重复 skill 的内容**。skill 管"怎么写"，本 command 管"流程与 GitHub 操作"。

读 `Cargo.toml` 记录 `is_workspace` / `has_web` / `has_db` / `is_async` 四个 flag，辅助取舍，不输出到方案里。

## Step 1 — 理解需求 & 扫描代码

- `#N` 或 URL → `gh issue view <num> --json number,title,body,comments,url,labels`
- 自然语言 → 直接作为需求

扫描只为回答：问题在哪层 / 改动落哪层 / 是否影响公开 API·数据·缓存·DB / 有无可复用模式 / 哪些明确不该动。允许 `Grep` / `Glob` / `Read` / `cargo metadata`，**禁止 `cargo build`**。扫描服务判断，不写成源码导览。

## Step 2 — 判断是否澄清

以下情况先反问（一次问完 ≤5 条，每条附默认假设，用户可回"默认即可"）：

- 用户场景 / 边界不清
- 性能 / 并发 / 资源 / 可用性要求会影响方案
- Rust 特有：MSRV、是否库 crate 受 SemVer 约束、外部系统对接协议

需求清晰就跳过。

## Step 3 — 方案粒度

- 单 crate 小改动 → 1 个推荐方案
- 跨模块有真实 trade-off → 2 个对比 + 推荐
- 新抽象 / 破坏兼容 / unsafe·FFI / 新 crate → 2–3 个对比 + 推荐 + 风险专章

**没有真实 trade-off 不写对比**，不要为"显得完整"硬凑。

## Step 4 — 估算规模

| Size | 典型特征 | 参考周期 |
|---|---|---|
| XS | 单文件、配置/文案 | <0.5 人日 |
| S | 单 crate、<3 文件、无架构决策 | 0.5–1 人日 |
| M | 跨 crate、少量抽象调整 | 2–4 人日 |
| L | 新 crate / 新抽象、需设计评审 | 1–2 周 |
| XL | 跨 workspace / 架构级 / 破坏公开 API / 新增 unsafe 边界 | >2 周，**先拆分** |

辅助字段（必填）：预计改动文件数（区间）、涉及 crate、破坏公开 API、数据迁移、新增 unsafe、新增依赖（heavy 依赖标注）、需要架构评审。

Size 后附一句理由。末尾必须保留：

> 本估算用于排期参考和风险识别，不作为工作量计量或绩效指标。

**Size = XL** 先提示拆分，不直出完整方案，等用户确认再继续。

## Step 5 — 写方案

按 `spec-writing` skill 的模板产出，按 `spec-rules` skill 叠加 Rust 专项。

唯一需要本 command 再次强调的纪律：

- **章节有信息增量才写**。skill 模板里的小节是上限不是下限，无内容就整节省略或一句"无"带过
- **数据库改动必须用 `mcp__postgres-db__query` 实查 schema**，不凭记忆

## Step 6 — 处理 issue 标题

格式：`<type>(<scope>)?: <动词开头的中文描述>`

- **type**：`feat` / `fix` / `refactor` / `perf` / `docs` / `test` / `chore` / `build` / `ci`
- **scope**：优先 crate 名；单 crate 用模块名；跨 crate / 根目录级省略；不写多 scope
- 描述 ≤50 字，动词开头，不加句号感叹号，专有名词保留原文

正则：`^(feat|fix|refactor|perf|docs|test|chore|build|ci)(\([a-z0-9/_-]+\))?: .+`

流程：

- 自然语言需求 → 按格式生成，Step 7 建 issue 时用
- 已有 issue 标题匹配 → 跳过；不匹配 → 生成新标题，对话展示 `原 → 新`，用户未否决即执行：
  ```bash
  gh issue edit <num> --title "<新标题>"
  ```

## Step 7 — 幂等写入 comment & label

目标 issue：

- 已有 issue → 即该 issue
- 自然语言需求 → `gh issue create --title "<Step 6 标题>" --body "<一句话需求摘要>" --label spec` 拿 `<num>`

写入流程：

1. 方案写入临时文件 `/tmp/spec-<timestamp>.md`（避免 shell 转义）

2. 查已有 spec comment：
   ```bash
   gh api repos/:owner/:repo/issues/<num>/comments \
     --jq '.[] | select(.body | startswith("<!-- spec-bot:v1 -->")) | .id'
   ```

3. 分支：
   - **无** → `gh issue comment <num> --body-file /tmp/spec-<timestamp>.md`
   - **一条** → `gh api -X PATCH repos/:owner/:repo/issues/comments/<id> -F body=@/tmp/spec-<timestamp>.md`
   - **多条**（历史遗留）→ 更新最早那条，其余删除；删除失败不中断，在总结里提示手动清理

4. 按规模估算打 label：
   ```bash
   gh issue edit <num> --add-label size/<SIZE>
   # risk 字段为真则追加：
   # risk/breaking    破坏公开 API / SemVer / MSRV 提升
   # risk/migration   数据迁移
   # risk/arch-review 架构评审
   # risk/unsafe      新增 unsafe
   ```
   label 不存在先建：`gh label create size/<SIZE> --color <color>`
   颜色：`size/XS c2e0c6` / `S bfd4f2` / `M fbca04` / `L f9a03f` / `XL e11d21` / `risk/* e11d21` / `risk/unsafe b60205` / `spec 5319e7`
   创建或添加失败跳过，在总结里提示。

5. 打印 comment URL + issue URL

6. **任一 `gh` 命令失败**：打印完整错误 + 临时文件路径 + 可手动执行的命令，不静默

## Step 8 — 总结

对话里简要告知：

- issue URL + comment URL
- 新增 / 更新
- 标题：`原 → 新` 或"未变更"
- Size + 一句理由
- 新增的 label
- Rust 专项（破坏公开 API / 新增 unsafe / heavy 依赖，任一为真就讲）
- 方案粒度 + 推荐项
- 需要用户拍板的点（≤3 条）

---

## 纪律（仅流程级；格式级规则都在 skill）

- ❌ 不写代码文件，不跑 `cargo build`（`cargo metadata` / `cargo tree` 可用于分析）
- ❌ 不改 issue body；未告知就不改 issue 标题
- ❌ 不新增重复 spec comment
- ❌ 不用精确 LoC / 工时做估算
- ❌ Rust 专项风险（公开 API / MSRV / unsafe / 编译时间）有就写，没有就明确写"无"

**落 comment 前再核两条格式硬底线**（详细规则在 skill）：

1. 关键决策每条标题必须以问号结尾
2. 验收标准严禁 `- [ ]` checkbox 语法

## 完成标志

用户打开 comment 能立刻回答：改什么 / 为什么 / 要拍板什么 / 风险 / 怎么算完成。追问是讨论取舍，不是"没看懂"。
