---
name: review-rules-nextjs
description: "Next.js / React 前端项目的代码审查规则，供 /review command 加载。覆盖 RSC/RCC 边界、hydration、i18n、bundle size、客户端安全等维度。"
user-invocable: false
---

# 前端审查规则（Next.js / React）

`/review` 检测到前端项目时自动加载，补充通用审查维度。

---

## 1. 安全（前端补充）

- `NEXT_PUBLIC_` 环境变量不能包含密钥、token、内部 URL
- 客户端组件中不能直接调用数据库或读取服务端环境变量
- Server Action 校验用户身份（未鉴权的 Server Action 等同公开 API）
- `dangerouslySetInnerHTML` 的输入经过 sanitize

## 2. RSC / RCC 边界

- `"use client"` 加在正确层级，能下沉就下沉
- Server Component 不误用 `useState`、`useEffect`、`onClick`
- Client Component 不误用 `cookies()`、`headers()`
- 跨边界 props 可序列化（不传函数、class 实例、Date）

RSC/RCC 边界错误导致构建失败或运行时报错 → FAIL

## 3. Hydration

- 服务端/客户端渲染结果一致（注意 `Date.now()`、`Math.random()`、`window` 依赖）
- `suppressHydrationWarning` 仅用于已知差异（时间戳等）

明确的 hydration mismatch → FAIL

## 4. 数据获取

- 页面级数据获取在 Server Component / Route Handler 中完成
- 避免不必要的客户端 `useEffect` + `fetch`
- `fetch` 缓存策略匹配业务场景
- 并行获取用 `Promise.all`

## 5. 国际化

- 用户可见文本通过 i18n 函数
- 新增 key 在所有语言文件中有对应翻译
- 日期/数字/货币使用 `Intl` API 格式化

## 6. 性能与 Bundle

- 大型依赖 tree-shake 或动态导入
- 图片使用 `next/image`
- 能在服务端完成的逻辑不搬到客户端

## 7. 状态管理与副作用

- `useEffect` 依赖数组完整
- 组件卸载时清理订阅、定时器、AbortController

## 8. 可访问性

- 交互元素使用语义标签（`<button>` 而非 `<div onClick>`）
- 图片有 `alt`，表单控件关联 `<label>`
- 模态框焦点正确转移

关键交互元素缺少语义标签或键盘不可达 → FAIL
