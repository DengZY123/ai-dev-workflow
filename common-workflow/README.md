# common-workflow

通用开发流程 plugin，封装从需求到上线的核心研发流程。

## Commands

| Command | 触发场景 |
|---------|---------|
| `/spec` | 基于需求或 issue 生成技术方案，写入 GitHub Issue |
| `/review` | 审查当前代码变更，输出 PASS/FAIL 报告并持久化 |
| `/pr` | 提交 PR，附加 review 结果和发布物料清单 |

## Skills

| Skill | 触发场景 |
|-------|---------|
| `git-commit` | 提交代码、生成 conventional commit 信息、commit scope/type 格式化 |

## 流程串联

```
需求 → /spec → 人工评审方案 → 开发 → /review → /pr → 人工审批 merge
```

- `/spec` 产出技术方案到 issue，等人确认后再动手写代码
- `/review` 产出 PASS/FAIL 报告并落盘到 `.claude/reviews/`
- `/pr` 读取 review 产物，校验 PASS 后才允许提 PR，同时生成发布物料清单

## 注意

- `/spec` 的 Step 4 依赖 `.claude/skills/spec-writing/SKILL.md` 定义方案模板，需在目标项目中配置
- `/pr` 强依赖 `/review` 的产物（`.claude/reviews/<branch>/<sha>.json`），未通过 review 无法提 PR
