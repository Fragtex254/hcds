# 模块 3 Brief：请求的一生

## Teaching Arc

- **隐喻：邮局分拣中心** —— 一封信从投递到送达，要经过分拣、分类、路由、投递等多个站点。一个 HTTP 请求也是如此——从浏览器发出，经过 CDN、Worker、路由层、中间件、SSR 渲染，最终变成用户看到的网页。
- **开场钩子：** "你在浏览器地址栏输入博客网址，按回车。0.5 秒后，完整的网页出现了。这 0.5 秒里，你的请求穿越了多少个'关卡'？"
- **核心洞察：** 请求经过的每一层都有自己的职责——CDN 负责缓存命中、Hono 负责路由分发、中间件负责安全检查、TanStack 负责数据加载和渲染。理解这个链条，你就能在出问题时知道该排查哪一层。
- **Why should I care?** 当页面加载慢或者返回 404，你需要知道问题出在哪个环节。是 CDN 没缓存？是路由写错了？还是数据库查询太慢？理解请求的路径 = 获得调试的起点。

## Code Snippets (pre-extracted)

### Snippet 1: Worker 入口 —— 请求的第一站
File: `src/lib/worker/app-handler.ts` (full file)
```ts
import handler from "@tanstack/react-start/server-entry";
import { Hono } from "hono";
import { paraglideMiddleware } from "@/paraglide/server.js";
import { app } from "@/lib/hono/routes";

const serve = new Hono();

serve.all("*", async (c) => {
  return paraglideMiddleware(c.req.raw, () => {
    return app.fetch(c.req.raw, c.env, c.executionCtx);
  });
});

export { serve as appWorkerHandler };
```

### Snippet 2: Hono 路由表 —— 请求的分拣规则
File: `src/lib/hono/routes.ts` (lines 23-147, key routing)
```ts
export const app = new Hono<{ Bindings: Env }>();

// 1. 所有 GET 请求先检查缓存
app.get("*", cacheMiddleware);

// 2. 公开 API 路由
const publicApi = new Hono<{ Bindings: Env }>()
  .route("/posts", postsListRoute)
  .route("/post", postsDetailRoute)
  .route("/post", postsRelatedRoute)
  .route("/tags", tagsRoute)
  .route("/search", searchRoute);
app.route("/api", publicApi);

// 3. 站点文档（RSS、sitemap 等）
app.route("/", siteDocumentsRoute);

// 4. 图片服务
app.get("/images/:key{.+}", async (c) => { /* R2 图片 */ });

// 5. 认证路由（带限流和人机验证）
app.get("/api/auth/*", baseMiddleware, forwardAuthRequest);
protectedAuthPaths.forEach((path) => {
  app.post(path,
    baseMiddleware,
    turnstileMiddleware,
    rateLimitMiddleware({ capacity: 5, interval: "1m", identifier: createRateLimiterIdentifier }),
    rateLimitMiddleware({ capacity: 10, interval: "1h", identifier: (c) => `hourly:${createRateLimiterIdentifier(c)}` }),
    forwardAuthRequest,
  );
});

// 6. 防护层：未知路径返回缓存的 404
app.all("*", shieldMiddleware);

// 7. 兜底：交给 TanStack Start 渲染
app.all("*", (c) => {
  return handler.fetch(c.req.raw, {
    context: { env: c.env, executionCtx: getExecutionContext(c) },
  });
});
```

### Snippet 3: 中间件链 —— 安全检查站
File: `src/lib/middlewares.ts` (lines 46-106, middleware chain)
```ts
// 注入数据库连接
export const dbMiddleware = createMiddleware({ type: "function" }).server(
  async ({ next, context }) => {
    const db = getDb(context.env);
    return next({ context: { db } });
  },
);

// 解析用户会话
export const sessionMiddleware = createMiddleware({ type: "function" })
  .middleware([dbMiddleware])
  .server(async ({ next, context }) => {
    const auth = getAuth({ db: context.db, env: context.env });
    const session = await auth.api.getSession({ headers: getRequestHeaders() });
    return next({ context: { auth, session } });
  });

// 要求登录
export const authMiddleware = createMiddleware({ type: "function" })
  .middleware([sessionMiddleware])
  .server(async ({ next, context }) => {
    if (!context.session) throw createAuthError();
    return next({ context: { session: context.session } });
  });

// 要求管理员权限
export const adminMiddleware = createMiddleware({ type: "function" })
  .middleware([authMiddleware])
  .server(async ({ context, next }) => {
    if (context.session.user.role !== "admin") throw createPermissionError();
    return next({ context: { session: context.session } });
  });
```

## Interactive Elements

- [x] **数据流动画** — 角色：Browser、CDN、Worker、Hono、Middleware、TanStack SSR、D1 数据库。步骤：浏览器发出请求 → CDN 检查缓存 → Worker 接收 → Hono 路由匹配 → cacheMiddleware 检查 Cache API → shieldMiddleware 验证路径 → TanStack SSR 加载数据 → D1 返回数据 → HTML 渲染 → 响应返回浏览器
- [x] **代码↔中文翻译** — Snippet 2 (Hono 路由表) 展示路由分发逻辑
- [x] **测验** — 3 个问题，风格：调试/追踪
  - Q1: "用户访问 /api/posts 返回 404，但 /post/my-slug 正常。问题最可能在请求路径的哪个环节？" (答案：Hono 路由匹配——/api/posts 是列表路由，检查 postsListRoute 是否正确挂载)
  - Q2: "请求 /admin/settings 时，中间件链的执行顺序是什么？" (答案：dbMiddleware → sessionMiddleware → authMiddleware → adminMiddleware)
  - Q3: "如果 CDN 缓存了一个错误页面（500），用户多久后能看到正确页面？" (答案：notFound/serverError 策略只缓存 10 秒，之后 CDN 会回源重新获取)
- [x] **Callout box** — "💡 关键洞察：中间件就像机场安检——每一层检查不同的东西。数据库连接、用户身份、管理权限，层层递进。如果中间任何一层检查不通过，请求就会被拒绝，不会到达后面的代码。"
- [x] **模式卡片** — 展示请求经过的每一层：CDN Cache、Cache API、Hono Router、Middleware Chain、TanStack SSR、D1 Database

## Reference Files to Read

- `references/interactive-elements.md` → "Message Flow / Data Flow Animation", "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Callout Boxes", "Pattern/Feature Cards"
- `references/content-philosophy.md` → 全部
- `references/gotchas.md` → 全部

## Connections

- **上一个模块：** "代码库的组织方式" —— 介绍了功能模块的三层架构，本模块展示请求如何到达这些模块
- **下一个模块：** "三层缓存" —— 深入讲解模块 3 中提到的 cacheMiddleware 和 CacheService 如何协同工作
- **风格/语气说明：** 使用邮局/分拣中心的比喻贯穿整个模块。数据流动画是本模块的核心视觉元素。
