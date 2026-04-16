# rust-workflow

Rust 后端开发流程 plugin。

## Skills

| Skill | 用途 |
|-------|------|
| `review-rules` | 后端代码审查规则，供 `/review` command 自动加载 |
| `spec-rules` | 后端技术方案补充规则，供 `/spec` command 自动加载 |

## 与 common-workflow 的关系

`common-workflow` 提供 `/spec`、`/review`、`/pr` 三个 command 骨架。本 plugin 的 `review-rules` 和 `spec-rules` 在检测到 Rust 项目时自动加载，补充后端专项审查维度和方案设计要求。
