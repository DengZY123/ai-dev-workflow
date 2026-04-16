---
description: 基于需求生成技术方案，产出到 GitHub Issue，供人评审
argument-hint: <需求描述 或 issue URL/编号>
allowed-tools: Bash(gh:*), Read, Grep, Glob, WebFetch
---

# /spec — 生成代码方案

你的任务是把一个需求转成可评审、可执行的技术方案，并写入 GitHub Issue。这是研发流水线的第一道闸门，产出质量决定后续所有代码质量。

## 输入

$ARGUMENTS

输入可能是：

- 一段自然语言需求（例：“接入 xAI Grok 4 模型”）
- 一个已有 issue 的编号或 URL（例：`#42` 或 `https://github.com/.../issues/42`）—— 此时读取 issue 正文作为需求

## 核心约定（贯穿全流程）

- **技术方案只进 issue 的 comment，不进 issue body**。issue body 是需求，comment 是方案，职责分离。
- **方案 comment 必须幂等**：每次运行 `/spec` 都用首行标记 `<!-- spec-bot:v1 -->` 识别；已存在则**更新**同一条 comment，不新增。
- **issue 标题遵循 Conventional Commits 风格固定格式**（见 Step 5）；不符合格式的视为草稿，会触发更新流程（需用户确认）。
- **issue body 永远不动**。

## 执行步骤

### Step 0 — 检测项目类型 & 加载规则

1. 检查项目根目录：
   - 存在 `package.json` 且含 `next` 依赖 → 前端项目，加载 `nextjs-workflow` plugin 的 `spec-rules` skill
   - 存在 `Cargo.toml` → 后端项目，加载 `rust-workflow` plugin 的 `spec-rules` skill
   - 两者都存在 → 全栈项目，两套规则都加载
   - 都不存在 → 仅使用通用模板
2. **必读**：`common-workflow` plugin 的 `spec-writing` skill，严格遵守其中的方案结构模板。
3. 加载到的 `spec-rules` 作为补充章节要求，合并到方案模板中输出。

> 若 skill 查找失败，打印缺失项并继续（退化为通用模板），不要中断。

### Step 1 — 理解需求 & 扫描代码库

1. 判断输入类型：
   - 纯数字或 `#数字` → issue 编号
   - `https://github.com/.../issues/N` → issue URL
   - 其他 → 自然语言需求
2. 如果是 issue 编号/URL，用 `gh issue view <num> --json number,title,body,comments,url,labels` 读取内容，保存 `title`、`body`、`comments`、`labels` 备用。
3. 用 Grep/Glob 快速扫描仓库，理解：
   - 项目结构（前端 Next.js / 后端 Rust 的组织方式）
   - 相关模块的现有实现（尤其是 provider adapter 相关代码，如果需求涉及）
   - 已有的约定和模式
4. 记录扫描到的**相关模块路径**，后续 Step 3.5 估算规模时会用到。

### Step 2 — 判断是否需要澄清

评估需求的清晰度，以下情况必须先反问，禁止直接出方案：

- 用户场景不清晰（谁用、什么时候用、解决什么问题）
- 边界模糊（是否包含 X、是否兼容旧行为）
- 非功能需求缺失（性能、并发、成本、安全）
- 涉及外部系统但未说明对接方式（厂商 API、鉴权、计费）

反问时遵守：

- 一次问完所有关键问题，用编号列出，不要挤牙膏
- 每个问题给出你的默认假设，用户可以只回答“默认即可”
- 问题不超过 5 个，抓主要矛盾

如果需求已经足够清晰，跳过此步，直接进入 Step 3。

### Step 3 — 判断方案粒度

根据复杂度自适应：

- **简单**（单模块、<3 个文件改动、无架构决策）→ 1 个方案 + 关键实现点
- **中等**（跨模块、有架构选型、有明显 trade-off）→ 2-3 个方案对比 + 推荐项
- **复杂**（涉及新厂商对接、新核心抽象、破坏兼容）→ 2-3 个方案对比 + 推荐项 + 风险专章

### Step 3.5 — 估算改动规模

基于 Step 1 的代码库扫描结果估算改动规模。输出**放在方案 comment 最前面**（紧跟 `<!-- spec-bot:v1 -->` 标记）。

**Size（T-shirt size，必填）**，从以下选一个：

| Size | 典型特征 | 参考周期 |
|---|---|---|
| `XS` | 单文件、配置/文案级改动、无逻辑 | <0.5 人日 |
| `S` | 单模块、<3 文件、无架构决策 | 0.5–1 人日 |
| `M` | 跨模块、少量抽象调整、无破坏兼容 | 2–4 人日 |
| `L` | 新模块/新抽象、多处改造、需要设计评审 | 1–2 周 |
| `XL` | 跨系统、架构级、破坏兼容 | >2 周，**建议先拆分** |

**辅助字段**：

- **预计改动文件数**：给区间，如 `~5–8 个`
- **涉及模块**：列出具体路径（基于 Step 1 的扫描结果）
- **破坏兼容**：是 / 否
- **数据迁移**：是 / 否
- **新增依赖**：是 / 否（若是，列出）
- **需要架构评审**：是 / 否（Size = L 或 XL，或任一 risk 字段为"是"时，默认需要）

**评估时必须遵守：**

- 基于实际代码扫描结果给判断，禁止凭感觉定档
- Size 给出后，附一句理由（哪几项决定了这个档位）
- Size = XL 时，先在对话里提示用户"建议先拆分"并列出拆分建议，**不直接出完整方案**，等用户确认是否继续

**规模估算 block 的末尾必须带免责声明**：

> 本估算用于排期参考和风险识别，不作为工作量计量或绩效指标。

### Step 4 — 撰写方案

严格按 `spec-writing` skill 的模板输出 markdown，放在变量里准备写入 issue comment。

**最终 comment 的结构**：

```
<!-- spec-bot:v1 -->

## 改动规模估算
（Step 3.5 的输出，含免责声明）

## 方案
（Step 4 的正文）
```

### Step 5 — 判断并处理 issue 标题

**标准格式**：

```
<type>(<scope>)?: <动词开头的描述>
```

**type**（必选，封闭集合）：

| type | 用途 |
|---|---|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `refactor` | 重构，不改变外部行为 |
| `perf` | 性能优化 |
| `docs` | 文档 |
| `test` | 测试 |
| `chore` | 杂项（依赖升级等） |
| `build` | 构建系统、打包 |
| `ci` | CI/CD 配置 |

**scope**（可选，小写）：

- 前端常用：`web` / `ui` / `api` / `auth`
- 后端常用：`server` / `core` / `provider` / `db`
- 全栈可嵌套，如 `provider/xai`
- 不确定时可省略

**描述部分规则**：

- 中文为主，专有名词保留原文（xAI、Grok、Next.js）
- 动词开头（接入 / 支持 / 修复 / 重构 / 优化 / 移除）
- 不加句号，不用省略号
- 控制在 50 字以内，超了就拆 issue

**正则校验**：

```
^(feat|fix|refactor|perf|docs|test|chore|build|ci)(\([a-z0-9/_-]+\))?: .+
```

**处理流程**：

- **输入是自然语言需求**：按上述格式生成标题，Step 6 新建 issue 时直接使用。
- **输入是已有 issue**：
  1. 用正则匹配原标题：
     - **匹配** → 标题不动，跳过。
     - **不匹配** → 视为草稿，进入第 2 步。
  2. 基于方案内容生成新标题：
     - type 选择：新增能力 → `feat`；修 bug → `fix`；不改行为的内部改造 → `refactor`；提升性能 → `perf`；只动文档/测试/构建 → 对应 type；无法判断默认 `feat`
     - scope 选择：基于 Step 1 扫描到的主要模块；跨多个模块时选最主要的一个或省略
  3. 在对话里展示 `原标题 → 新标题`，征求用户确认。用户未明确否决即执行：
     ```bash
     gh issue edit <num> --title "<新标题>"
     ```

### Step 6 — 幂等写入方案 Comment 和 Label

**确定目标 issue 编号**：

- 输入是已有 issue → 即该 issue
- 输入是自然语言需求 → 用 `gh issue create --title "<Step 5 生成的标题>" --body "<一句话需求摘要>" --label spec` 新建，拿到编号

**写入流程**：

1. 把最终 comment 内容（`<!-- spec-bot:v1 -->` + 规模估算 + 方案）写入临时文件 `/tmp/spec-<timestamp>.md`，避免 shell 转义问题。

2. **查找已有 spec comment**：
   ```bash
   gh api repos/:owner/:repo/issues/<num>/comments \
     --jq '.[] | select(.body | startswith("<!-- spec-bot:v1 -->")) | .id'
   ```

3. **分支处理**：
   - **未找到** → 新增：
     ```bash
     gh issue comment <num> --body-file /tmp/spec-<timestamp>.md
     ```
   - **找到一条** → 更新：
     ```bash
     gh api -X PATCH repos/:owner/:repo/issues/comments/<comment_id> \
       -F body=@/tmp/spec-<timestamp>.md
     ```
   - **找到多条**（历史遗留）→ 更新最早那条，其余删除：
     ```bash
     gh api -X DELETE repos/:owner/:repo/issues/comments/<id>
     ```
     删除失败（权限不足）不中断，在总结里提示用户手动清理。

4. **根据规模估算打 label**：
   ```bash
   gh issue edit <num> --add-label size/<SIZE>
   # 任一 risk 字段为"是"时，追加对应 label：
   # --add-label risk/breaking
   # --add-label risk/migration
   # --add-label risk/arch-review
   ```
   若 label 不存在，先创建：
   ```bash
   gh label create size/<SIZE> --color <color>
   ```
   建议颜色：
   - `size/XS`: `c2e0c6` / `size/S`: `bfd4f2` / `size/M`: `fbca04` / `size/L`: `f9a03f` / `size/XL`: `e11d21`
   - `risk/*`: `e11d21`
   - `spec`: `5319e7`

   label 创建或添加失败（权限不足）跳过，在总结里提示。

5. 拿到 comment URL 和 issue URL，在对话里打印。

6. **任何 `gh` 命令失败**：打印完整错误、保留临时文件路径、告知用户可手动执行的命令，不要静默失败。

### Step 7 — 输出总结

在对话里简短告诉用户：

- Issue URL 和 spec comment URL
- 本次是**新增**还是**更新**了 spec comment
- 标题变更情况（`原标题 → 新标题`，或"未变更"）
- **改动规模**：Size + 简要理由
- **新增的 label**：列出
- 方案粒度（1 个 / N 个对比）
- 如果有推荐项，明确指出
- 需要用户评审的关键决策点（3 条以内）

## 禁止事项

- ❌ 不写任何代码文件（这是 `/spec`，不是 `/build`）
- ❌ 不修改 issue body（原需求正文）
- ❌ 不在未告知用户的情况下修改 issue 标题
- ❌ 不新增重复的 spec comment；同一 issue 同一版本的方案只能有一条
- ❌ 不在方案里塞无关的"最佳实践"大段文字，每句话都要服务于本次需求
- ❌ 不用模糊词掩盖未决策项；有取舍就写明取舍，真未决就明确列入"待确认"
- ❌ 不用精确数字（LoC / 工时）做规模估算，只用 T-shirt size + 辅助字段
- ❌ 不省略规模估算末尾的免责声明

## 完成标志

用户看到 issue URL 和 comment URL，打开后能立刻判断"方案对不对"，不需要追问细节；重复运行 `/spec` 不会产生 comment 堆积；标题和 label 都符合约定，方便后续筛选和排期。