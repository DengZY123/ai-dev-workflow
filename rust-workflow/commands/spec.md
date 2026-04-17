---
description: 基于需求生成 Rust 项目的技术方案，产出到 GitHub Issue，供人评审
argument-hint: <需求描述 或 issue URL/编号>
allowed-tools: Bash(gh:*), Bash(cargo:*), Read, Grep, Glob, WebFetch, mcp__postgres-db__query
---

# /spec — 生成 Rust 代码方案

你的任务是把一个需求转成可评审、可执行的技术方案，并写入 GitHub Issue。这是研发流水线的第一道闸门，产出质量决定后续所有代码质量。

针对 Rust 项目特有的关注点（crate 结构、错误建模、并发模型、unsafe、API 稳定性、编译时间），产出不能是通用模板的简单套用。

## 输入

$ARGUMENTS

输入可能是：

- 一段自然语言需求（例："在 auth crate 加 OAuth 登录"）
- 一个已有 issue 的编号或 URL（例：`#42` 或 `https://github.com/.../issues/42`）—— 此时读取 issue 正文作为需求

## 核心约定（贯穿全流程）

- **技术方案只进 issue 的 comment，不进 issue body**。issue body 是需求，comment 是方案，职责分离。
- **方案 comment 必须幂等**：每次运行 `/spec` 都用首行标记 `<!-- spec-bot:v1 -->` 识别；已存在则**更新**同一条 comment，不新增。
- **issue 标题遵循 Conventional Commits 风格固定格式**（见 Step 5）；不符合格式的视为草稿，会触发更新流程（需用户确认）。
- **issue body 永远不动**。

## 执行步骤

### Step 0 — 检测 Rust 项目形态 & 加载规则

1. 读 `Cargo.toml`，记录四个 boolean 字段（后续章节取舍依此展开）：
   - `is_workspace`：是否有 `[workspace]` 段
   - `has_web`：是否含 `actix-web` / `axum` / `rocket` / `warp` / `tonic`
   - `has_db`：是否含 `sqlx` / `sea-orm` / `diesel`
   - `is_async`：是否含 `tokio` / `async-std`

2. **必读**：`rust-workflow` plugin 的 `spec-writing` skill 和 `spec-rules` skill。
   - `spec-writing`：方案结构模板和写作规范
   - `spec-rules`：Rust 专项补充章节

3. 记录上述字段为 `PROJECT_SHAPE`，Step 4 选择章节时用。

### Step 1 — 理解需求 & 扫描代码库

1. 判断输入类型：
   - 纯数字或 `#数字` → issue 编号
   - `https://github.com/.../issues/N` → issue URL
   - 其他 → 自然语言需求
2. 如果是 issue 编号/URL，用 `gh issue view <num> --json number,title,body,comments,url,labels` 读取内容。
3. 用 Grep/Glob + Read 扫描仓库，重点搞清：
   - **crate 拓扑**：workspace member 列表、各 crate 职责、依赖图（`cargo metadata --format-version 1` 可输出 JSON，必要时跑一次）
   - **错误类型体系**：项目有没有统一的 `Error` enum？用 `thiserror` 还是 `anyhow`？API 边界的错误怎么转换
   - **并发/异步形态**：runtime（tokio 多线程 vs current_thread）、共享状态套路（`Arc<Mutex>` / `DashMap` / actor）、任务生命周期管理方式
   - **数据库访问层**：DAO/Repository 模式？`sqlx::query!` 还是 ORM？连接池配置
   - **鉴权/中间件栈**：axum `Router::route_layer` / actix `wrap`，现有鉴权 middleware 的挂载位置
   - **配置加载**：`config` / `figment` / `envy` / 直接 `std::env`
   - **相关模块的现有实现**（特别是新 provider 接入、新 crate 添加类需求）
4. 记录扫描到的**相关 crate 和路径**，Step 3.5 估算规模时用。

### Step 2 — 判断是否需要澄清

以下情况**必须先反问**，禁止直接出方案：

- 用户场景不清晰（谁用、什么时候用、解决什么问题）
- 边界模糊（是否包含 X、是否兼容旧行为）
- **非功能需求缺失**：
  - **性能**：QPS 量级？P99 延迟要求？
  - **并发**：单实例并发量？有无背压要求？
  - **资源**：内存上限？是否需要 `no_std` 或 embedded？
  - **可用性**：SLA 要求？是否允许重启？
- **Rust 特有澄清点**：
  - 是否对 **MSRV**（最低支持 Rust 版本）有要求？
  - 是否是库 crate？若是，需明确**公开 API 的兼容承诺**（SemVer 级别）
- 涉及外部系统但未说明对接方式（厂商 API、鉴权、计费、rate limit）

反问时遵守：

- 一次问完所有关键问题，用编号列出，不挤牙膏
- 每个问题给出默认假设，用户可以只回答"默认即可"
- 问题不超过 5 个，抓主要矛盾

需求已经足够清晰则跳过此步。

### Step 3 — 判断方案粒度

根据复杂度自适应：

- **简单**（单 crate、<3 个文件改动、无架构决策）→ 1 个方案 + 关键实现点
- **中等**（跨 crate、有架构选型、有明显 trade-off）→ 2-3 个方案对比 + 推荐项
- **复杂**（workspace 级改造、新核心抽象、破坏公开 API 兼容、涉及 unsafe/FFI 新增）→ 2-3 个方案对比 + 推荐项 + 风险专章

**Rust 项目典型的"必须多方案对比"场景**：
- 引入新第三方 crate vs 自研（对编译时间/二进制体积影响大时）
- 破坏公开 API 的改动（需对比兼容改法 vs 破坏改法）

### Step 3.5 — 估算改动规模

基于 Step 1 的代码库扫描结果估算改动规模。输出**放在方案 comment 最前面**（紧跟 `<!-- spec-bot:v1 -->` 标记）。

**Size（T-shirt size，必填）**：

| Size | 典型特征 | 参考周期 |
|---|---|---|
| `XS` | 单文件、配置/文案级、无逻辑、不改公开 API | <0.5 人日 |
| `S` | 单 crate 内、<3 文件、无架构决策 | 0.5–1 人日 |
| `M` | 跨 crate、少量抽象调整、无破坏兼容 | 2–4 人日 |
| `L` | 新 crate / 新抽象、多处改造、需设计评审 | 1–2 周 |
| `XL` | 跨 workspace、架构级、破坏公开 API、新增 unsafe 边界 | >2 周，**建议先拆分** |

**Rust 专属辅助字段**（必填）：

- **预计改动文件数**：给区间，如 `~5–8 个`
- **涉及 crate**：列出具体 crate 名（基于 Step 1 扫描结果）
- **破坏公开 API**：是 / 否（`pub` 项增删/签名变更、需 SemVer 主版本升级、MSRV 提升 → 都算"是"）
- **数据迁移**：是 / 否
- **新增 unsafe 块**：是 / 否（若是，后续方案必须单独章节论证）
- **新增依赖**：是 / 否（若是，列出 crate + 选择理由 + 启用的 feature；heavy 依赖如 `reqwest` 全家桶、`tonic` 编译链会拉长，需标注）
- **需要架构评审**：是 / 否（Size = L/XL，或任一 risk 字段为"是"，默认需要）

**评估纪律：**

- 基于实际代码扫描结果给判断，禁止凭感觉定档
- Size 给出后，附一句理由（哪几项决定了这个档位）
- Size = XL 时，先在对话里提示用户"建议先拆分"并列出拆分建议，**不直接出完整方案**，等用户确认是否继续

**规模估算 block 末尾必须带免责声明**：

> 本估算用于排期参考和风险识别，不作为工作量计量或绩效指标。

### Step 4 — 撰写方案

严格按 `spec-writing` skill 的模板输出 markdown，并**叠加** `spec-rules` skill 里定义的 Rust 专项章节。最终方案结构：

```
<!-- spec-bot:v1 -->

## 改动规模估算
（Step 3.5 的输出，含免责声明）

## 方案

# <方案标题>

## 背景与目标
...

## 现状分析
（含 crate 拓扑和受影响 crate 列表）

## 方案设计

### 方案概述

### 详细设计

#### Crate 与模块结构
（新增代码放哪个 crate / 是否新建 crate / pub 暴露面）

#### 错误处理策略
（错误类型定义、传播链路、重试策略）

#### 并发/异步模型
（任务生命周期、共享状态、是否需要 spawn_blocking）

#### 数据模型变更
（**含数据库时，必须用 mcp__postgres-db__query 实查 schema**，完整 DDL + 回滚）

#### API 变更
（公开 API 增删改、rust 函数签名、HTTP/RPC 接口）

#### 核心流程图
（Mermaid 流程图，覆盖主要参与者、分支、错误路径）

#### 核心逻辑
（伪代码或状态机描述）

#### 文件变更清单

### 关键决策
（格式与收录规则严格按 `spec-writing` skill：问题开头，只列需评审拍板或有明显 trade-off 的决策，3–5 条。禁止把实现细节、命名、行业通识当决策列出来。）

## 风险与边界

### 已知风险
（包含 unsafe invariant、编译时间、二进制体积、MSRV 影响）

### 边界场景

### 兼容性
（公开 API SemVer 影响、数据迁移路径、feature flag 组合）

## 验收标准
（严格按 `spec-writing` skill 的两段式：**业务场景验收** 用"给定 / 操作 / 期望"三段式覆盖 happy path + 边界 + 兼容降级 + 容错 + 副作用；**工程质量基线** 一段清单覆盖 fmt/clippy/test、公开 API 文档、`tracing` 日志、`unsafe` invariant。
**禁止**：五层分类（行为/测试/工具链/可观测性/文档）、英文 snake_case 测试函数名作场景名、把"覆盖核心分支 + 1 个错误路径"这种测试策略写进验收。）

## 工作量估算
```

**最终 comment 结构固定为**：

```
<!-- spec-bot:v1 -->

## 改动规模估算
...

## 方案
...
```

### Step 4.5 — 方案写完后的硬性自检

**在落 comment 之前**，对照下面 checklist 逐项核对。任一项不过就**就地重写对应章节**，不允许带着不合格的方案进入 Step 5/6。

**方案概述**：

- [ ] 第一段是"改动前的请求流程"（完整伪步骤 + "本次要动第 X 步"），不是 1/2/3 式的改动清单
- [ ] 没有出现 "X 扩字段"、"Y 签名加参"、"Z 扩展 SCAN" 这种文件/符号堆砌的起手式
- [ ] 核心分支（三态 / 降级）用了表格或小标题对比
- [ ] 没有用 `Option<{ channels: Option<HashMap<String, Vec<Decimal>>> }>` 这种类型签名冒充说明

**关键决策**：

- [ ] 每条决策的标题**以问号结尾**（否则删除或重写）
- [ ] 3–5 条；超 5 条剔除实现细节，少于 2 条复核 Size
- [ ] 没有把"实现位置选择"（短路放哪、过滤用 SQL 还是 Rust）当决策
- [ ] 需外部确认的决策，独立写了 **"需 PM 在上线前确认：XX"** 行，不藏在括号/小字里
- [ ] 没有单独的"待确认"章节（待确认项都在对应决策的"需 ... 确认"行里）

**验收标准**：

- [ ] **没有任何 `- [ ]` checkbox**
- [ ] **没有出现 "行为级"、"测试级"、"工具链级"、"可观测性级"、"文档级" 字样**
- [ ] 每个业务场景有独立的 `### 场景 N：<中文短句>` 标题
- [ ] 每个场景都有 "给定 / 操作 / 期望" 三段
- [ ] 场景名是中文短句，不是英文 snake_case 测试函数名
- [ ] 至少覆盖 happy path + 边界 + 兼容/降级 三类（Size ≥ S）
- [ ] 工程质量基线单独一段清单，不拆到场景里

自检不过的项，**直接修改方案 markdown**，改完再过一遍。不要把 checklist 本身写进 comment。

### Step 5 — 判断并处理 issue 标题

**标准格式**：

```
<type>(<scope>)?: <动词开头的描述>
```

**type**（必选）：`feat` / `fix` / `refactor` / `perf` / `docs` / `test` / `chore` / `build` / `ci`

**scope**（可选，小写，**Rust 项目优先用 crate 名**）：

- workspace 项目：crate 名（如 `feat(auth): 支持 OAuth`、`perf(core): 减少 clone`）
- 单 crate 项目：模块名（如 `fix(handler): 修复 panic`）
- 跨 crate / 根目录级 → 省略（如 `chore: 升级 tokio 到 1.40`）
- 不写多 scope：不用 `feat(auth,api): ...`

**描述部分**：

- 中文为主，专有名词保留原文（tokio、sqlx、axum、Rust）
- 动词开头（接入 / 支持 / 修复 / 重构 / 优化 / 移除）
- 不加句号、不加感叹号
- 控制在 50 字以内，超了拆 issue

**正则校验**：

```
^(feat|fix|refactor|perf|docs|test|chore|build|ci)(\([a-z0-9/_-]+\))?: .+
```

**处理流程**：

- **输入是自然语言需求**：按上述格式生成标题，Step 6 新建 issue 时用。
- **输入是已有 issue**：
  1. 正则匹配原标题：匹配则跳过；不匹配进入第 2 步
  2. 基于方案内容生成新标题：
     - type：新增能力 → `feat`；修 bug → `fix`；不改外部行为的内部改造 → `refactor`；提升性能 → `perf`；只动测试/构建/CI → 对应 type；无法判断默认 `feat`
     - scope：基于 Step 1 扫描到的主要 crate；跨多个 crate 选最主要的一个或省略
  3. 对话里展示 `原标题 → 新标题`，征求用户确认。用户未明确否决即执行：
     ```bash
     gh issue edit <num> --title "<新标题>"
     ```

### Step 6 — 幂等写入方案 Comment 和 Label

**确定目标 issue 编号**：

- 输入是已有 issue → 即该 issue
- 输入是自然语言需求 → 用 `gh issue create --title "<Step 5 生成的标题>" --body "<一句话需求摘要>" --label spec` 新建，拿到编号

**写入流程**：

1. 把最终 comment 内容写入临时文件 `/tmp/spec-<timestamp>.md`，避免 shell 转义。

2. **查找已有 spec comment**：
   ```bash
   gh api repos/:owner/:repo/issues/<num>/comments \
     --jq '.[] | select(.body | startswith("<!-- spec-bot:v1 -->")) | .id'
   ```

3. **分支处理**：
   - **未找到** → 新增：`gh issue comment <num> --body-file /tmp/spec-<timestamp>.md`
   - **找到一条** → 更新：
     ```bash
     gh api -X PATCH repos/:owner/:repo/issues/comments/<comment_id> \
       -F body=@/tmp/spec-<timestamp>.md
     ```
   - **找到多条**（历史遗留）→ 更新最早那条，其余删除。删除失败不中断，在总结里提示手动清理。

4. **根据规模估算打 label**：
   ```bash
   gh issue edit <num> --add-label size/<SIZE>
   # 按 risk 字段追加：
   # --add-label risk/breaking        破坏公开 API / SemVer 主版本 / MSRV 提升
   # --add-label risk/migration       需数据迁移
   # --add-label risk/arch-review     需架构评审
   # --add-label risk/unsafe          新增 unsafe 块
   ```

   label 不存在先创建：`gh label create size/<SIZE> --color <color>`

   建议颜色：
   - `size/XS`: `c2e0c6` / `size/S`: `bfd4f2` / `size/M`: `fbca04` / `size/L`: `f9a03f` / `size/XL`: `e11d21`
   - `risk/*`: `e11d21`（`risk/unsafe` 用 `b60205` 区分严重度）
   - `spec`: `5319e7`

   创建或添加失败跳过，在总结里提示。

5. 拿到 comment URL 和 issue URL，在对话里打印。

6. **任何 `gh` 命令失败**：打印完整错误、保留临时文件路径、告知用户可手动执行的命令，不静默失败。

### Step 7 — 输出总结

对话里简短告诉用户：

- Issue URL 和 spec comment URL
- 本次是**新增**还是**更新**了 spec comment
- 标题变更情况（`原标题 → 新标题`，或"未变更"）
- **改动规模**：Size + 简要理由
- **新增的 label**：列出
- **Rust 专项关注**（若有）：
  - 是否破坏公开 API（含 SemVer 主版本 / MSRV 提升）
  - 是否新增 unsafe（含必要性判断）
  - 是否引入 heavy 依赖（影响编译时间/二进制体积）
- 方案粒度（1 个 / N 个对比）
- 如有推荐项，明确指出
- 需要用户评审的关键决策点（3 条以内）

---

## 禁止事项

- ❌ 不写任何代码文件（这是 `/spec`，不是 `/build`）
- ❌ 不运行 `cargo build` 等编译命令（`cargo metadata` / `cargo tree` 可接受，用于分析）
- ❌ 不修改 issue body（原需求正文）
- ❌ 不在未告知用户的情况下修改 issue 标题
- ❌ 不新增重复的 spec comment；同一 issue 同一版本的方案只能有一条
- ❌ 不塞无关"最佳实践"大段文字
- ❌ 不用模糊词掩盖未决策项；有取舍就写明取舍，真未决列入"待确认"
- ❌ 不用精确数字（LoC / 工时）做规模估算，只用 T-shirt size + 辅助字段
- ❌ 不省略规模估算末尾的免责声明
- ❌ 涉及 unsafe / FFI / 破坏公开 API 的方案，不得遗漏对应章节

## 完成标志

用户看到 issue URL 和 comment URL，打开后能立刻判断"方案对不对"，不追问细节；重复运行 `/spec` 不产生 comment 堆积；标题和 label 符合约定，方便筛选和排期；Rust 专项风险（公开 API、unsafe、MSRV、编译时间）都已显式标注。
