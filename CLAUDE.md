# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在本仓库中工作提供指引。

## 项目概述

Flare Stack Blog 是一个专为 Cloudflare Workers 构建的全栈博客 CMS。技术栈：TypeScript 严格模式、React 19 前端、Hono 网关、TanStack Start（SSR / Server Functions）、Drizzle ORM（D1 SQLite）、Better Auth + GitHub OAuth。包管理器为 **Bun**（>= 1.3）。

## 常用命令

| 用途 | 命令 |
|---|---|
| 开发服务器（端口 3000） | `bun dev` |
| 构建 | `bun run build` |
| 代码检查 | `bun lint` |
| 自动修复 | `bun lint:fix` |
| 类型检查 | `bun typecheck` |
| 提交前完整检查 | `bun check`（biome check --write && typecheck） |
| 运行全部测试 | `bun run test` |
| 仅 Node 单元测试 | `bun run test:node` |
| 运行单个测试文件 | `bun run test src/features/posts/posts.service.test.ts` |
| 按名称过滤测试 | `bun run test posts` |
| 生成数据库迁移 | `bun db:generate` |
| 本地数据库迁移 | `bun db:local` |
| 编译 i18n | `bun i18n:compile` |

Git 钩子：pre-commit 执行 `bun lint`，pre-push 执行 `bun typecheck`。

## 架构

### 请求流程

```
请求 → Cloudflare CDN（边缘缓存）
  → server.ts（Hono 入口）
    /api/auth/* → Better Auth
    /images/*   → R2 媒体服务
    其余        → TanStack Start
      → 中间件注入（db、auth、session）
      → 路由匹配 + Loader → KV 缓存 ↔ Service 层 ↔ D1 数据库
      → SSR 渲染（附带缓存头）
```

### 功能模块三层架构

每个功能位于 `src/features/<name>/`，遵循以下结构：

```
features/<name>/
  data/               # 数据层：纯 Drizzle 查询，无业务逻辑
  <name>.service.ts   # 服务层：业务逻辑 + 缓存编排
  <name>.schema.ts    # Zod schema + 缓存 key 工厂
  api/                # API 层：TanStack Server Functions（薄转发）
  components/         # 功能专属 React 组件
  queries/            # TanStack Query hooks / queryOptions
```

- **数据层**（`data/`）：Repository 模式，如 `PostRepo.findPostById(db, id)`。
- **服务层**（`.service.ts`）：编排数据访问 + 缓存，接收 `context`（db、session、executionCtx）。
- **API 层**（`api/`）：`createServerFn()` 搭配中间件，委托给服务层。Server Function 命名加 `Fn` 后缀（`getPostsFn`）。

### 中间件链

定义于 `src/lib/middlewares.ts`：

```
dbMiddleware → sessionMiddleware → authMiddleware → adminMiddleware
```

- `adminMiddleware` 已包含前置中间件（db + session + auth）。
- `createRateLimitMiddleware()` 用于公开端点（基于 Durable Object `RateLimiter`）。
- `turnstileMiddleware` 用于 Cloudflare Turnstile 人机验证。

### 错误处理（Result 模式）

- `src/lib/errors/error.ts`：`Result<TData, TError>`，提供 `ok()`、`err()`、`unwrap()`。
- **业务错误**（如 `POST_NOT_FOUND`、`TAG_NAME_ALREADY_EXISTS`）：服务层返回 `Result`。
- **请求级错误**（鉴权、权限、限流）：由中间件抛出，使用 `src/lib/errors/request-errors.ts` 中的 `createXxxError()`。
- 无业务错误的服务直接返回 `T` ——**不要** 包装为 `ok()`。
- 客户端错误处理在 `src/lib/errors/error-handler.ts`，对错误码做穷举 switch。
- 完整指南：`docs/error-handling-quickstart.md`。

### 双层缓存

| 层 | 技术 | 用途 |
|---|---|---|
| CDN | Cache-Control 响应头 | 边缘缓存，通过页面头或 Hono 路由设置 |
| KV | 版本化 key，由 `CacheService` 管理 | 服务端缓存，见 `src/features/cache/cache.service.ts` |

失效方式：`bumpVersion(context, "posts:list")` 批量失效，`deleteKey(context, key)` 精确失效。CDN 缓存清除通过 Cloudflare API（`src/lib/invalidate.ts`）。

### Cloudflare Workers 绑定

- **D1**（SQLite）：主数据库，通过 Drizzle ORM 访问
- **KV**：缓存层
- **R2**：媒体文件存储
- **Queues**：异步副作用（邮件、webhook、浏览量、快照）——处理器在 `src/lib/queue/queue.handler.ts`
- **Workflows**：持久化业务流程（文章处理、快照、评论审核、定时发布、导入导出）
- **Durable Objects**：`RateLimiter`、`PasswordHasher`
- **Workers AI**：评论审核

### 主题系统

主题契约定义于 `src/features/theme/contract/`，规定了组件、配置、布局、页面的接口。构建时通过 `THEME` 环境变量选择主题，`@theme` 路径别名指向当前主题目录。现有两个主题：`default` 和 `fuwari`。

### 路由

TanStack Router 文件路由，位于 `src/routes/`：
- `_public/` — 公开页面（首页、文章列表/详情、搜索）
- `_auth/` — 登录/注册
- `_user/` — 用户资料
- `admin/` — 管理后台
- `src/routeTree.gen.ts` 为**自动生成**文件 —— 请勿编辑。

### 国际化

Paraglide JS，支持 `zh` 和 `en` 两个语言。消息文件在 `messages/zh.json` 和 `messages/en.json`。编辑消息后需运行 `bun i18n:compile`。

## 代码规范

| 类型 | 规范 | 示例 |
|---|---|---|
| 组件文件 | kebab-case | `post-item.tsx` |
| 服务文件 | `<name>.service.ts` | `posts.service.ts` |
| 数据文件 | `<name>.data.ts` | `posts.data.ts` |
| Server Functions | camelCase + `Fn` 后缀 | `getPostsFn` |
| React 组件 | PascalCase | `PostItem` |
| 类型/接口 | PascalCase | `PostItemProps` |
| 常量 | SCREAMING_SNAKE_CASE | `CACHE_CONTROL` |

- **格式化**：Biome — 2 空格缩进、双引号、必须分号、尾随逗号。`noExplicitAny` 为错误级别。
- **提交信息**：Conventional Commits（`feat:`、`fix:`、`docs:`、`refactor:`）。
- **日志**：生产环境使用结构化 JSON —— `console.log(JSON.stringify({ message, key }))`。
- **导入路径**：`@/*` 映射 `src/*`，`@theme` 映射当前主题目录。

## 测试

两套测试配置：
- `src/**/*.integration.test.ts` — Cloudflare Workers 集成测试（`@cloudflare/vitest-pool-workers`）
- `src/**/*.test.ts`（排除 integration）— Node 单元测试，通过 `vitest.node.config.ts`

测试工具在 `tests/test-utils.ts`：`createTestContext`、`createAdminTestContext`、`seedUser`、`waitForBackgroundTasks`、`testRequest`。

## 领域术语

完整术语表见 `CONTEXT.md`。关键辨析：**Post**（非 "article"）、**Tag**（非 "category"）、**Media**（非 "asset"）、**Public Content Snapshot**、**Verifying Comment** 与 **Pending Comment**（两种不同状态）。Issue 和 PRD 以本地 Markdown 存放在 `.scratch/` 下，详见 `docs/agents/issue-tracker.md`。

## 架构决策记录（ADR）

位于 `docs/adr/`：
- 0001：Cloudflare Workers 为唯一部署目标 —— 不做可移植性抽象
- 0002：分离公开内容快照
- 0003：主题契约分离公开页面展示
- 0004：Workflows 处理持久化流程，Queues 处理高吞吐副作用
