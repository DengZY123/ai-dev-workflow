# Dev Workflow Marketplace

个人开发流程 Claude Code plugin marketplace。每个插件自包含完整研发流程（/spec → /review → /pr），并内置对应技术栈的专项规则。

## Plugins

| Plugin | 说明 |
|--------|------|
| `nextjs-workflow` | Next.js 全流程：/spec → /review → /pr + 前端专项规则 |
| `rust-workflow` | Rust 全流程：/spec → /review → /pr + 后端专项规则 |

## 研发流程

```
需求/Issue → /spec 生成方案 → 人工评审 → 开发 → /review 代码审查 → /pr 提交PR → 人工审批 merge
```

## 安装

```bash
# 添加 marketplace
/plugin marketplace add DengZY123/ai-dev-workflow

# 前端项目装这个
/plugin install nextjs-workflow

# 后端项目装这个
/plugin install rust-workflow
```

## 更新

```bash
/plugin marketplace update
```

## 本地开发调试

```bash
claude --plugin-dir ./nextjs-workflow
claude --plugin-dir ./rust-workflow
```

修改 SKILL.md / command .md 后使用 `/reload-plugins` 热重载，无需重启 Claude Code。

## 注意事项

- 两个插件各自包含完整的 commands 和 skills，装一个就够用
- common-workflow 作为内部共享源保留，不再单独发布
- skill scripts 中引用插件内文件时，使用 `${CLAUDE_PLUGIN_ROOT}` 环境变量
