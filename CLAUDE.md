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
| 创建新主题脚手架 | `bun run create-theme` |

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

### 双 API 面

项目暴露两套 API 面：

1. **Hono 路由**（`/api/...`）：面向公众的 REST 端点，位于 `features/<name>/api/hono/`。聚合在 `src/lib/hono/routes.ts`，导出 `PublicApiType` 用于 RPC 客户端类型推断。使用 Hono 专属中间件（`src/lib/hono/middlewares.ts`）。

2. **TanStack Server Functions**：管理后台和需认证的操作，位于 `features/<name>/api/`，使用 `createServerFn()`。使用 TanStack 中间件链（`src/lib/middlewares.ts`）。

Hono 中间件（`cacheMiddleware`、`rateLimitMiddleware`、`shieldMiddleware`、`turnstileMiddleware`、`baseMiddleware`）与 TanStack 中间件（`dbMiddleware`、`sessionMiddleware`、`authMiddleware`、`adminMiddleware`）是**互相独立**的。

### 功能模块三层架构

每个功能位于 `src/features/<name>/`，遵循以下结构：

```
features/<name>/
  data/               # 数据层：纯 Drizzle 查询，无业务逻辑
  <name>.service.ts   # 服务层：业务逻辑 + 缓存编排
  <name>.schema.ts    # Zod schema + 缓存 key 工厂
  api/                # API 层：TanStack Server Functions（薄转发）
  api/hono/           # Hono 公开路由端点
  components/         # 功能专属 React 组件
  queries/            # TanStack Query hooks / queryOptions
  workflows/          # Cloudflare Workflows（持久化业务流程）
```

全部功能模块：`ai`、`auth`、`cache`、`comments`、`config`、`dashboard`、`email`、`friend-links`、`import-export`、`mcp`、`media`、`notification`、`oauth-clients`、`oauth-provider`、`pageview`、`posts`、`search`、`site-documents`、`tags`、`theme`、`version`、`webhook`。

- **数据层**（`data/`）：Repository 模式，如 `PostRepo.findPostById(db, id)`。
- **服务层**（`.service.ts`）：编排数据访问 + 缓存，接收 `context`（db、session、executionCtx）。
- **API 层**（`api/`）：`createServerFn()` 搭配中间件，委托给服务层。Server Function 命名加 `Fn` 后缀（`getPostsFn`）。

### 中间件链

**TanStack 中间件**（`src/lib/middlewares.ts`）：

```
dbMiddleware → sessionMiddleware → authMiddleware → adminMiddleware
```

- `adminMiddleware` 已包含前置中间件（db + session + auth）。
- `createRateLimitMiddleware()` 用于公开端点（基于 Durable Object `RateLimiter`）。
- `turnstileMiddleware` 用于 Cloudflare Turnstile 人机验证。

**Hono 中间件**（`src/lib/hono/middlewares.ts`）：

`baseMiddleware`、`cacheMiddleware`、`rateLimitMiddleware`、`shieldMiddleware`、`turnstileMiddleware` —— 与 TanStack 中间件独立。

### 错误处理（Result 模式）

四条核心规则（完整指南：`docs/error-handling-quickstart.md`）：

1. `Result` **仅用于业务错误**（如 `POST_NOT_FOUND`、`TAG_NAME_ALREADY_EXISTS`）。
2. **请求级错误**（鉴权、权限、限流、人机验证）由中间件 `throw` —— **不要**包装进 `Result`。
3. 无业务错误的服务直接返回 `T` —— **不要**包装为 `ok()`。
4. `error-handler.ts` 中的 `handleServerError` 必须对请求错误码做穷举 switch（`code satisfies never`）。

关键文件：
- `src/lib/errors/error.ts`：`Result<TData, TError>`，提供 `ok()`、`err()`、`unwrap()`。
- `src/lib/errors/request-errors.ts`：请求级错误构造函数 `createXxxError()`。
- `src/lib/errors/error-handler.ts`：客户端全局错误处理器。

### 双层缓存

| 层 | 技术 | 用途 |
|---|---|---|
| CDN | Cache-Control 响应头 | 边缘缓存，通过页面头或 Hono 路由设置 |
| KV | 版本化 key，由 `CacheService` 管理 | 服务端缓存，见 `src/features/cache/cache.service.ts` |

失效方式：`bumpVersion(context, "posts:list")` 批量失效，`deleteKey(context, key)` 精确失效。CDN 缓存清除通过 Cloudflare API（`src/lib/invalidate.ts`）。

### Cloudflare Workers 绑定

- **D1**（SQLite）：主数据库，通过 Drizzle ORM 访问，表定义在 `src/lib/db/schema/`
- **KV**：缓存层
- **R2**：媒体文件存储
- **Queues**：异步副作用（邮件、webhook、浏览量、快照）——处理器在 `src/lib/queue/queue.handler.ts`
- **Workflows**：持久化业务流程，实现位于各功能模块内（如 `features/posts/workflows/`）
- **Durable Objects**：`RateLimiter`、`PasswordHasher`，实现在 `src/lib/do/`
- **Workers AI**：评论审核

**Workflows vs Queues 区分**（ADR-0004）：Workflows 用于有序、可恢复的持久化业务流程（文章处理、定时发布、导入导出）。Queues 用于高吞吐副作用（邮件、webhook、浏览量、快照）。

### 主题系统

主题契约定义于 `src/features/theme/contract/`，规定了组件、配置、布局、页面的接口。构建时通过 `THEME` 环境变量选择主题，`@theme` 路径别名指向当前主题目录。现有两个主题：`default` 和 `fuwari`。

配置层次：
- `src/blog.config.ts`：站点标识、社交链接、图标等默认/回退值，类型为 `satisfies SiteConfig`
- 运行时 `SiteConfig`：存储在数据库 `system_config` 表，可从管理后台 Settings 覆盖
- `ThemeConfig`（编译时）：数据获取参数（如页面大小），定义在各主题的 `config.ts`，不可运行时修改

主题开发指南：`docs/theme-guide.md`（含 `bun run create-theme` 脚手架命令）。

### 路由

TanStack Router 文件路由，位于 `src/routes/`：
- `_public/` — 公开页面（首页、文章列表/详情、搜索）
- `_auth/` — 登录/注册
- `_user/` — 用户资料
- `admin/` — 管理后台
- `src/routeTree.gen.ts` 为**自动生成**文件 —— 请勿编辑。

### 国际化

Paraglide JS，支持 `zh` 和 `en` 两个语言。消息文件在 `messages/zh.json` 和 `messages/en.json`。编辑消息后需运行 `bun i18n:compile`。

## 关键目录

### `src/lib/`

| 路径 | 用途 |
|---|---|
| `auth/` | Better Auth 配置：`auth.config.ts`、`auth.server.ts`、`auth.client.ts` |
| `db/schema/` | Drizzle 表定义：`posts.table.ts`、`comments.table.ts`、`auth.table.ts` 等 |
| `do/` | Durable Object 实现：`rate-limiter.ts`、`password-hasher.ts` |
| `env/` | Zod 校验的环境变量：`server.env.ts`、`client.env.ts` |
| `errors/` | Result 类型和错误处理：`error.ts`、`request-errors.ts`、`error-handler.ts` |
| `hono/` | Hono 应用组装：`routes.ts`、`middlewares.ts` |
| `queue/` | Queue 处理器和 schema：`queue.handler.ts`、`queue.schema.ts` |
| `worker/` | Worker 入口：`app-handler.ts`、`root-handler.ts` |

### `src/components/`（共享组件）

与主题无关的共享组件：
- `admin/` — 管理后台组件
- `common/` — 通用组件（如 Turnstile）
- `ui/` — 基础 UI 原语
- `tiptap-editor/` — 富文本编辑器

主题专属组件在 `features/theme/themes/<theme>/components/`，**不同主题之间不应互相导入**。

### `src/hooks/`（共享 Hooks）

非功能模块专属的共享 hooks：`use-active-toc`、`use-debounce`、`use-delay-unmount`、`use-navigate-back`、`use-previous-location`。

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

## 提交前检查

- `bun check` 通过（类型检查 + Lint + 格式化）
- `bun run test` 通过
- 新功能有测试覆盖
- 错误处理遵循 Result 模式（请求错误不进 Result、业务错误不 throw、无业务错误不包 `ok()`）

## 领域术语

完整术语表见 `CONTEXT.md`。关键辨析：**Post**（非 "article"）、**Tag**（非 "category"）、**Media**（非 "asset"）、**Public Content Snapshot**、**Verifying Comment** 与 **Pending Comment**（两种不同状态）。

## Issue 追踪

Issue 和 PRD 以本地 Markdown 管理在 `.scratch/` 下：
- 每个功能一个目录：`.scratch/<feature-slug>/`
- PRD：`.scratch/<feature-slug>/PRD.md`
- Issue：`.scratch/<feature-slug>/issues/<NN>-<slug>.md`（从 `01` 编号）
- 分流状态通过 `Status:` 行标记：`needs-triage`、`needs-info`、`ready-for-agent`、`ready-for-human`、`wontfix`
- 评论追加在 `## Comments` 标题下

详见 `docs/agents/issue-tracker.md` 和 `docs/agents/triage-labels.md`。

## 架构决策记录（ADR）

位于 `docs/adr/`：

- **ADR-0001**：Cloudflare Workers 为唯一部署目标 —— **不要**添加 Node/Vercel 可移植性抽象
- **ADR-0002**：公开内容从独立快照读取，不直接读取 Post 编辑状态
- **ADR-0003**：主题契约分离公开页面展示与路由/数据逻辑
- **ADR-0004**：Workflows = 有序可恢复的持久化业务流程（文章处理、定时发布、导入导出）；Queues = 高吞吐批量副作用（邮件、webhook、浏览量、快照）
