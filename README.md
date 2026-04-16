# Dev Workflow Marketplace

个人开发流程 Claude Code plugin marketplace，集中管理 Next.js 前端、Rust 后端和通用工作流的 skills 与 commands。

## Plugins

| Plugin | 说明 |
|--------|------|
| `nextjs-workflow` | Next.js 组件创建、测试策略、部署流程 |
| `rust-workflow` | Cargo 管理、Rust 测试、Clippy lint 修复 |
| `common-workflow` | 研发流程核心：/spec → /review → /pr |

## 研发流程

```
需求/Issue → /spec 生成方案 → 人工评审 → 开发 → /review 代码审查 → /pr 提交PR → 人工审批 merge
```

## 安装

```bash
# 添加 marketplace
/plugin marketplace add DengZY123/ai-dev-workflow

# 从 marketplace 中选装需要的 plugin
/plugin install nextjs-workflow
/plugin install rust-workflow
/plugin install common-workflow
```

## 更新

```bash
# 拉取最新版本
/plugin marketplace update
```

## 本地开发调试

```bash
claude --plugin-dir ./nextjs-workflow
claude --plugin-dir ./rust-workflow
claude --plugin-dir ./common-workflow
```

修改 SKILL.md / command .md 后使用 `/reload-plugins` 热重载，无需重启 Claude Code。

## 目录结构

```
./
├── .claude-plugin/
│   └── marketplace.json
├── nextjs-workflow/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── skills/
│   │   ├── nextjs-component/SKILL.md
│   │   ├── nextjs-testing/SKILL.md
│   │   └── nextjs-deploy/SKILL.md
│   └── README.md
├── rust-workflow/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── skills/
│   │   ├── rust-cargo/SKILL.md
│   │   ├── rust-testing/SKILL.md
│   │   └── rust-clippy/SKILL.md
│   └── README.md
├── common-workflow/
│   ├── .claude-plugin/
│   │   └── plugin.json
│   ├── commands/
│   │   ├── spec.md          # /spec — 需求 → 技术方案
│   │   ├── review.md        # /review — 代码审查
│   │   └── pr.md            # /pr — 提交 PR + 发布物料
│   ├── skills/
│   │   └── git-commit/SKILL.md
│   └── README.md
├── .gitignore
└── README.md
```

## 注意事项

- skill scripts 中引用插件内文件时，使用 `${CLAUDE_PLUGIN_ROOT}` 环境变量，不要硬编码路径
- 所有文件和目录名使用 kebab-case
- `skills/` 和 `commands/` 目录位于 plugin 根目录下，不在 `.claude-plugin/` 内
