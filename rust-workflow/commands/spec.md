---
description: 基于需求生成 Rust 项目的技术方案，产出到 GitHub Issue，供人评审
argument-hint: <需求描述 或 issue URL/编号>
allowed-tools: Bash(gh:*), Bash(cargo:*), Read, Grep, Glob, WebFetch, mcp__postgres-db__query
---

# /spec — 生成更适合人评审的 Rust 技术方案

你的任务是把一个需求转成"可评审、可执行、像人写的"技术方案，并写入 GitHub Issue comment。

这不是代码生成任务，也不是把代码扫描结果机械堆进去。你的第一目标是：**让评审者在 3 分钟内看懂"要解决什么、建议怎么做、为什么这样做、有哪些风险、哪些点需要拍板"**。

- 技术方案写入 issue **comment**，不写入 issue body。issue body 是需求，comment 是方案，职责分离。
- 方案 comment 必须幂等：每次运行 `/spec`，都用首行标记 `<!-- spec-bot:v1 -->` 识别；已存在则**更新**同一条 comment，不新增。
- issue body 永远不动。

---

## 一、输入

$ARGUMENTS

输入可能是：

1. 一段自然语言需求（例："在 auth crate 加 OAuth 登录"）
2. 已有 issue 编号或 URL（例：`#42` 或 `https://github.com/.../issues/42`）—— 读取 issue 正文作为需求

---

## 二、总原则（贯穿全流程）

1. **先说人话，再说细节**。输出第一屏必须先给出"结论摘要"，不能一上来堆文件路径、函数签名、缓存 key、行号。
2. **先讲方案，再讲扫描结果**。代码扫描是为了支撑判断，不是为了展示你扫描得很完整。
3. **只写和决策有关的内容**。不要把显而易见的实现细节、机械性的字段变化、无关最佳实践塞进方案。
4. **强调作者判断**。多用"我建议 / 不建议 / 这里选择 A 而不是 B，因为 …"这类判断句。不要只罗列信息。
5. **方案首先写给人评审**。不是写给代码生成器，也不是写给审计系统。

---

## 三、执行步骤

### Step 0 — 检测项目形态 & 加载规则

1. 读 `Cargo.toml`，记录以下 boolean 字段（**辅助判断用，不要机械罗列到输出里**）：
   - `is_workspace`：是否有 `[workspace]` 段
   - `has_web`：是否含 `actix-web` / `axum` / `rocket` / `warp` / `tonic`
   - `has_db`：是否含 `sqlx` / `sea-orm` / `diesel`
   - `is_async`：是否含 `tokio` / `async-std`

2. **必读**：`rust-workflow` plugin 的 `spec-writing` skill 和 `spec-rules` skill。
   - `spec-writing`：方案结构模板、完整正面/反面示例、写作规范
   - `spec-rules`：Rust 专项补充章节

3. 记录字段为 `PROJECT_SHAPE`，Step 4 选择章节时用。

### Step 1 — 理解需求 & 扫描代码库

判断输入类型：

- 纯数字或 `#数字` → issue 编号
- GitHub issue URL → 解析
- 其他 → 自然语言需求

若是 issue 编号/URL，用 `gh issue view <num> --json number,title,body,comments,url,labels` 读取。

**扫描的目的是回答五个问题**，不是输出源码导览：

1. 当前问题发生在哪一层
2. 这次改动应该插入哪一层
3. 是否影响现有公开 API / 数据结构 / 缓存 / DB
4. 是否有现成模式可复用
5. 哪些逻辑明确不该动

辅助看的维度：crate 拓扑、错误处理体系、异步/并发模型、DB 访问层、鉴权/中间件链路、配置加载方式、相关模块现有实现。

允许使用 Grep / Glob / Read / `cargo metadata`。**不要运行 `cargo build`**。

**扫描输出纪律**：

- 不写成"源码导览"
- 不大量列文件行号
- 不把扫描结果当方案主体
- "现状分析"节最多引用 **2–4 个最关键位置**

### Step 2 — 判断是否需要澄清

以下情况必须先反问，不能直接出方案：

1. 用户场景不清晰
2. 边界不清晰
3. 性能 / 并发 / 资源 / 可用性要求会影响方案选择
4. Rust 特有约束不清楚：
   - 是否有 MSRV 要求
   - 是否是库 crate、是否受 SemVer 约束
   - 涉及外部系统对接但协议未说明

反问要求：一次问完，最多 5 个；每个问题附默认假设；用户可回"默认即可"。

需求已经清晰则跳过。

### Step 3 — 判断方案粒度

- **简单**（单 crate、小改动、无架构取舍）→ 1 个推荐方案 + 必要说明
- **中等**（跨模块 / 跨 crate，有一定取舍）→ 2 个方案对比 + 推荐方案
- **复杂**（新增核心抽象、破坏兼容、涉及 unsafe/FFI、新 crate）→ 2–3 个方案对比 + 推荐方案 + 风险专章

**不要为了"显得完整"硬凑多方案**。只有存在真实 trade-off 才写对比。

### Step 3.5 — 估算改动规模

基于 Step 1 的扫描结果给估算，输出在 comment 最前面。

| Size | 典型特征 | 参考周期 |
|---|---|---|
| `XS` | 单文件、配置/文案级、无逻辑、不改公开 API | <0.5 人日 |
| `S` | 单 crate 内、<3 文件、无架构决策 | 0.5–1 人日 |
| `M` | 跨 crate、少量抽象调整、无破坏兼容 | 2–4 人日 |
| `L` | 新 crate / 新抽象、多处改造、需设计评审 | 1–2 周 |
| `XL` | 跨 workspace、架构级、破坏公开 API、新增 unsafe 边界 | >2 周，**建议先拆分** |

辅助字段（必填）：

- 预计改动文件数（区间，如 `~4–6 个`）
- 涉及 crate
- 破坏公开 API：是 / 否
- 数据迁移：是 / 否
- 新增 unsafe 块：是 / 否
- 新增依赖：是 / 否（heavy 依赖如 `reqwest` 全家桶、`tonic` 编译链需标注）
- 需要架构评审：是 / 否（Size = L/XL 或任一 risk 字段为"是"，默认需要）

Size 给出后附一句理由说明为什么是这个档位。

**免责声明**（末尾必须保留）：

> 本估算用于排期参考和风险识别，不作为工作量计量或绩效指标。

**Size = XL 时**：先在对话里提示"建议先拆分"并列出拆分建议，不直接展开完整方案，等用户明确确认再继续。

### Step 4 — 撰写方案

严格按 `spec-writing` skill 的模板输出，叠加 `spec-rules` skill 的 Rust 专项章节。每个小节的**具体写法、示例、硬性禁止**都在 skill 里，这里只给整体骨架。

```
<!-- spec-bot:v1 -->

## 改动规模估算
（Step 3.5 的输出，含免责声明）

## 方案

# <方案标题>

## 一页结论
（200–300 字，固定回答 5 件事，无文件路径 / 行号 / 函数签名）
1. 这次要解决什么
2. 推荐怎么做
3. 为什么这样做
4. 最需要拍板的点
5. 风险结论

## 背景与目标
（当前问题 / 为什么现在做 / 本次目标 / 明确不做的事）

## 现状分析
（只回答：当前逻辑在哪 / 为什么不满足 / 改动落哪层 / 有哪些约束必须沿用；最多引用 2–4 个代码位置）

## 方案设计

### 方案概述
（按 skill：第一段必须讲"改动前的请求/数据流"，再按流程说改动点。禁止施工单式堆砌文件名）

### 方案对比
（仅在确有 trade-off 时出现：核心思路 / 优点 / 缺点 / 适用前提；结尾明确"本次推荐 X，原因…"）

### 详细设计
（按需保留以下小节，不必机械全写）

#### 模块与职责变更
#### 数据与状态
（涉 DB 必须用 mcp__postgres-db__query 实查 schema）
#### 错误处理策略
（只写有决策意义的：配置错误是降级还是报错、外部失败是否 fallback、是否重试）
#### 并发 / 异步模型
（只有特殊点才展开，没有可以简写）
#### API / 接口变化
#### 核心流程
（Mermaid 或简明步骤）
#### 文件变更清单
（重点说"为什么改这个文件"，不是"这个文件加几个字段"）

## 关键决策
（按 skill：标题必须以问号结尾；2–5 条；需外部确认的独立写"需 PM 确认：…"行）

## 风险与边界
### 已知风险
### 边界场景
### 兼容性

## 验收标准
（按 skill 两段式：业务场景验收用"给定 / 当 / 则"三段式独立小节；工程质量基线单列一段。**严禁 `- [ ]` checkbox 语法**）

## 工作量估算
（简要拆分，不要像项目排期表）

## 待确认
（仅在确实有未决事项时出现，最多 3 条）
```

---

## 四、语言风格要求

1. **结论先行**：每个大节优先回答"所以建议怎么做"。
2. **减少模板味**：不要每节都像填表。
3. **多用判断句**：
   - 我建议 …
   - 不建议 …
   - 这里选择 A 而不是 B，因为 …
   - 这个点默认按 … 处理
4. **不要过度技术炫耀**：不因为扫到了很多细节，就全部写出来。
5. **不要把实现说明写成代码预演**：spec 是帮助评审，不是提前把实现过程全文展开。

---

## 五、两条硬性格式底线（不遵守就不合格，必须重写对应章节）

这两条是反复迭代中最容易回退的，单独列出。即使 skill 里已经写了，**落 comment 前请再核一次**：

1. **关键决策每条标题必须以问号结尾**。标题不是问句的直接删或重写。
2. **验收标准严禁使用 `- [ ]` checkbox 语法**。业务场景必须用 `### 场景 N：<中文短句>` 独立小节 + "给定 / 当 / 则" 三段式。

---

## 六、issue 标题处理

### 标准格式

`<type>(<scope>)?: <动词开头的描述>`

**type**（必选）：`feat` / `fix` / `refactor` / `perf` / `docs` / `test` / `chore` / `build` / `ci`

**scope**（可选，小写，Rust 项目优先用 crate 名）：

- workspace 项目：crate 名（如 `feat(auth): 支持 OAuth`）
- 单 crate 项目：模块名（如 `fix(handler): 修复 panic`）
- 跨 crate / 根目录级 → 省略（如 `chore: 升级 tokio 到 1.40`）
- 不写多 scope（不用 `feat(auth,api): ...`）

**描述部分**：

- 中文为主，专有名词保留原文（tokio / sqlx / axum）
- 动词开头（接入 / 支持 / 修复 / 重构 / 优化 / 移除）
- 不加句号、不加感叹号
- 50 字以内

**正则校验**：

```
^(feat|fix|refactor|perf|docs|test|chore|build|ci)(\([a-z0-9/_-]+\))?: .+
```

**处理流程**：

- **输入是自然语言需求**：按格式生成标题，Step 七新建 issue 时用
- **输入是已有 issue**：
  1. 正则匹配原标题：匹配则跳过；不匹配进入下一步
  2. 基于方案内容生成新标题：
     - type：新增能力 → `feat`；修 bug → `fix`；内部改造 → `refactor`；性能 → `perf`；只动测试/构建/CI → 对应 type；无法判断默认 `feat`
     - scope：基于 Step 1 扫描到的主要 crate；跨多个 crate 选最主要的一个或省略
  3. 对话里展示 `原标题 → 新标题`，征求用户确认。用户未明确否决即执行：
     ```bash
     gh issue edit <num> --title "<新标题>"
     ```

---

## 七、comment 幂等写入与 label

**确定目标 issue 编号**：

- 输入是已有 issue → 即该 issue
- 输入是自然语言需求 → `gh issue create --title "<Step 六生成的标题>" --body "<一句话需求摘要>" --label spec` 新建，拿到编号

**写入流程**：

1. 把最终 comment 内容写入临时文件 `/tmp/spec-<timestamp>.md`，避免 shell 转义。

2. 查找已有 spec comment：
   ```bash
   gh api repos/:owner/:repo/issues/<num>/comments \
     --jq '.[] | select(.body | startswith("<!-- spec-bot:v1 -->")) | .id'
   ```

3. 分支处理：
   - **未找到** → 新增：`gh issue comment <num> --body-file /tmp/spec-<timestamp>.md`
   - **找到一条** → 更新：
     ```bash
     gh api -X PATCH repos/:owner/:repo/issues/comments/<comment_id> \
       -F body=@/tmp/spec-<timestamp>.md
     ```
   - **找到多条**（历史遗留）→ 更新最早那条，其余删除。删除失败不中断，在总结里提示手动清理。

4. 按规模估算打 label：
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

---

## 八、输出总结

对话里简要告诉用户：

- issue URL 和 spec comment URL
- 本次是**新增**还是**更新** spec comment
- 标题变更（`原标题 → 新标题`，或"未变更"）
- **Size** + 一句理由
- 新增的 label
- **Rust 专项关注**（若有）：是否破坏公开 API（含 SemVer / MSRV）、是否新增 unsafe、是否引入 heavy 依赖
- 方案粒度（1 个 / N 个对比）
- 推荐方案是什么
- **需要用户评审拍板的点**（最多 3 条）

---

## 九、禁止事项

- ❌ 不写任何代码文件（这是 `/spec`，不是 `/build`）
- ❌ 不运行 `cargo build` 等编译命令（`cargo metadata` / `cargo tree` 可接受，用于分析）
- ❌ 不修改 issue body
- ❌ 不在未告知用户的情况下修改 issue 标题
- ❌ 不新增重复 spec comment
- ❌ 不写成源码导览，不堆过多行号 / 缓存 key / 函数签名
- ❌ 不把显而易见的实现细节写成重点
- ❌ 不塞无关"最佳实践"大段文字
- ❌ 不用模糊词掩盖未决策项；有取舍写明取舍，真未决列入"待确认"
- ❌ 不用精确 LoC / 工时做估算
- ❌ 不省略规模估算末尾的免责声明
- ❌ 涉及 unsafe / FFI / 破坏公开 API 的方案，不得遗漏对应章节
- ❌ 不遗漏 Rust 特有风险（公开 API、MSRV、unsafe、编译时间）
- ❌ 方案里不得出现 `- [ ]` checkbox 语法
- ❌ 关键决策不出现标题不以问号结尾的条目

---

## 十、完成标志

用户打开 issue comment 后，应能快速回答：

- 这次到底要改什么
- 为什么这么改
- 哪几个点需要拍板
- 风险大不大
- 做完之后怎么算完成

如果用户继续追问，应是讨论取舍，而不是"没看懂你在说什么"。
