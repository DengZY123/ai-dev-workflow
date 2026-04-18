---
name: spec-rules-nextjs
description: "Next.js / React 前端项目的技术方案补充规则，供 /spec command 加载。关注 RSC 决策、组件设计、客户端/服务端边界等。"
user-invocable: false
---

# 前端方案补充规则（Next.js / React）

`/spec` 检测到前端项目时自动加载，补充以下关注点到方案设计中。

---

## 方案设计补充章节

### 组件设计

每个新增/修改的组件说明：
- Server Component 还是 Client Component，为什么
- Props 接口定义（TypeScript 类型）
- 数据来源：Server Component 直接 fetch / props 传入 / 客户端 hook
- 是否需要 Suspense boundary 和 loading 状态

### 路由与页面结构

- 新增页面的路由路径和文件位置
- 是否需要 `layout.tsx`、`loading.tsx`、`error.tsx`
- 动态路由参数和 `generateStaticParams` 策略
- Metadata / SEO 配置

### 数据获取策略

- 服务端获取 vs 客户端获取的决策理由
- 缓存策略：`force-cache` / `no-store` / `revalidate` 的选择
- Server Action 的使用场景和错误处理

### 状态管理

- 新增的全局状态放在哪里（Zustand store / Context）
- 表单状态管理方式

### 国际化

- 新增的 i18n key 清单
- 是否涉及日期/数字/货币格式化

---

## 风险补充关注点

- Bundle size 影响：新增依赖的大小，是否需要动态导入
- Hydration 风险：是否有服务端/客户端渲染不一致的可能
- 移动端适配：核心交互在小屏上是否可用
