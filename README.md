> 这是 v0.1 初始版本。内容会随实际使用迭代，欢迎在每次使用后调整。
> 看到"为什么这里要这样"想不起答案的规则，就删掉它。

# dev-workflow

团队研发流程工具库，作为 Claude Code plugin 使用。封装从需求到上线的核心流程：`/spec → /review → /pr`。

## 定位

- 这是一个**交接资产**，不只是当前工具——未来新人接手时能直接用
- 内容遵循"最小可行版"原则，随实际使用生长，不提前堆规则
- 支持 Next.js 前端 + Rust 后端混合栈，自动检测项目类型

## Commands

| Command | 用途 |
|---------|------|
| `/spec` | 需求 → 技术方案，写入 GitHub Issue |
| `/review` | 代码审查，PASS/FAIL 判定，产物落盘 |
| `/pr` | 提交 PR + review 结果 + 发布物料清单 |
| `/discuss` | 帮你梳理业务逻辑或代码逻辑 |

所有 command 支持参数 `rust` / `nextjs` 强制指定项目类型，默认自动检测。

## Skills

| Skill | 用途 |
|-------|------|
| `spec-writing` | 技术方案结构模板和写作规范 |
| `spec-rules-rust` | Rust 方案补充规则 |
| `spec-rules-nextjs` | Next.js 方案补充规则 |
| `review-rules-rust` | Rust 审查规则 |
| `review-rules-nextjs` | Next.js 审查规则 |
| `git-commit` | Conventional commit 规范（待填充） |
| `pr-review` | PR 自查清单（待填充） |

## 研发流程

```
需求/Issue → /spec 生成方案 → 人工评审 → 开发 → /review 代码审查 → /pr 提交PR → 人工审批 merge
```

## 安装

```bash
claude --plugin-dir /path/to/dev-workflow
```

## 目录结构

```
dev-workflow/
  commands/         ← 4 个命令入口
  skills/           ← 7 个知识模块
  README.md
```
