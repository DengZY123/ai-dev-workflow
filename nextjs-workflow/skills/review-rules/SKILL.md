---
name: review-rules
description: "Next.js / React 前端项目的代码审查规则，供 /review command 加载。覆盖 RSC/RCC 边界、hydration、i18n、bundle size、客户端安全等维度。"
---

# 前端审查规则（Next.js / React）

以下规则在 `/review` 执行时自动加载，补充通用审查维度。

---

## 1. 安全与鉴权（前端补充）

- `NEXT_PUBLIC_` 环境变量不能包含密钥、token、内部 URL
- 客户端组件（`"use client"`）中不能直接调用数据库或读取服务端环境变量
- Server Action 是否校验了用户身份？未鉴权的 Server Action 等同于公开 API
- `dangerouslySetInnerHTML` 的输入是否经过 sanitize
- 第三方脚本（`<Script>`）是否限定了 `strategy` 和来源

## 2. RSC / RCC 边界

- `"use client"` 是否加在了正确的层级？能下沉就下沉，避免整棵子树变客户端
- Server Component 中是否误用了 `useState`、`useEffect`、`onClick` 等客户端 API
- Client Component 是否误用了 `cookies()`、`headers()` 等服务端 API
- 跨边界传递的 props 是否可序列化（不能传函数、class 实例、Date 对象）

判定：RSC/RCC 边界错误会导致构建失败或运行时报错 → FAIL

## 3. Hydration 与渲染一致性

- 服务端和客户端渲染结果是否一致？常见问题：
  - 使用 `Date.now()`、`Math.random()` 等非确定性值
  - 依赖 `window`、`localStorage` 等浏览器 API 做条件渲染
  - `suppressHydrationWarning` 是否合理使用（只用于时间戳等已知差异）
- 动态导入（`next/dynamic`）的 `ssr: false` 是否必要

判定：明确的 hydration mismatch → FAIL

## 4. 数据获取模式

- 页面级数据获取是否在 Server Component / `generateMetadata` / Route Handler 中完成
- 是否有不必要的客户端 `useEffect` + `fetch`（应该用 Server Component 或 Server Action）
- `fetch` 的缓存策略是否合理：`force-cache` / `no-store` / `revalidate` 选择是否匹配业务场景
- 并行数据获取是否用了 `Promise.all` 而非瀑布式串行

## 5. 国际化（i18n）

- 用户可见文本是否通过 i18n 函数（如 `t('key')`）而非硬编码字符串
- 新增的 i18n key 是否在所有语言文件中都有对应翻译
- 日期、数字、货币格式是否使用 `Intl` API 或 i18n 库格式化
- RTL 布局是否考虑（如果项目支持阿拉伯语等）

## 6. 性能与 Bundle

- 大型依赖（lodash、moment、chart 库）是否 tree-shake 或动态导入
- 图片是否使用 `next/image`，是否设置了合理的 `sizes` 和 `priority`
- 是否有不必要的客户端 JS（能在服务端完成的逻辑搬到了客户端）
- `metadata` / `generateMetadata` 是否正确设置（SEO 影响）

判定：引入 >100KB 未 tree-shake 的依赖且有替代方案 → 建议（不 FAIL）

## 7. 状态管理与副作用

- `useEffect` 依赖数组是否完整，是否有遗漏导致的无限循环或过期闭包
- 表单状态是否用了 `useActionState` / `useFormStatus`（Next.js 推荐模式）
- 全局状态（Zustand / Context）的更新粒度是否合理，是否会导致不必要的重渲染
- 组件卸载时是否清理了订阅、定时器、AbortController

## 8. 样式与响应式

- 移动端核心功能是否可用（不只是"能看"，要"能用"）
- 是否有固定宽度导致小屏溢出
- 暗色模式是否适配（如果项目支持）
- CSS 类名是否遵循项目约定（Tailwind / CSS Modules / styled-components）

## 9. 可访问性（a11y）

- 交互元素是否有合适的语义标签（`<button>` 而非 `<div onClick>`）
- 图片是否有 `alt` 文本
- 表单控件是否关联 `<label>`
- 焦点管理：模态框、抽屉打开后焦点是否正确转移

判定：关键交互元素缺少语义标签或键盘不可达 → FAIL
