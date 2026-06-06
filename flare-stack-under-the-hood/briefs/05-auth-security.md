# 模块 5 Brief：门卫系统——身份验证与安全

## Teaching Arc

- **隐喻：机场安检** —— 从进入机场到登上飞机，你要经过多道检查：值机柜台验身份证、安检门扫描行李、登机口核对登机牌。一个博客的安全系统也是如此——每个敏感操作都要经过多层验证，每层检查不同的东西。
- **开场钩子：** "当有人试图登录你的博客管理后台，系统怎么知道他是不是真的你？如果有人用机器人疯狂注册账号，系统又怎么拦住他？"
- **核心洞察：** 安全不是一道门，而是层层递进的检查链：身份验证（你是谁）→ 权限检查（你能做什么）→ 限流（你做了多少次）→ 人机验证（你是不是真人）。每一层解决不同的威胁。
- **Why should I care?** 当你让 AI 帮你加一个"只有管理员能用"的功能，你需要知道怎么正确地保护它。如果 AI 只检查了 cookie 但没有检查角色权限，你的管理后台就可能被普通用户访问。

## Code Snippets (pre-extracted)

### Snippet 1: 中间件链 —— 层层递进的安检
File: `src/lib/middlewares.ts` (lines 46-106)
```ts
// 第一层：建立数据库连接
export const dbMiddleware = createMiddleware({ type: "function" }).server(
  async ({ next, context }) => {
    const db = getDb(context.env);
    return next({ context: { db } });
  },
);

// 第二层：识别用户身份（通过 cookie）
export const sessionMiddleware = createMiddleware({ type: "function" })
  .middleware([dbMiddleware])
  .server(async ({ next, context }) => {
    const auth = getAuth({ db: context.db, env: context.env });
    const session = await auth.api.getSession({ headers: getRequestHeaders() });
    return next({ context: { auth, session } });
  });

// 第三层：要求必须登录
export const authMiddleware = createMiddleware({ type: "function" })
  .middleware([sessionMiddleware])
  .server(async ({ next, context }) => {
    if (!context.session) throw createAuthError();
    return next({ context: { session: context.session } });
  });

// 第四层：要求管理员角色
export const adminMiddleware = createMiddleware({ type: "function" })
  .middleware([authMiddleware])
  .server(async ({ context, next }) => {
    if (context.session.user.role !== "admin") throw createPermissionError();
    return next({ context: { session: context.session } });
  });
```

### Snippet 2: 认证路由的多重防护
File: `src/lib/hono/routes.ts` (lines 97-121, protected auth paths)
```ts
const protectedAuthPaths = [
  "/api/auth/sign-in/email",
  "/api/auth/sign-up/email",
  "/api/auth/request-password-reset",
  "/api/auth/send-verification-email",
] as const;

protectedAuthPaths.forEach((path) => {
  app.post(
    path,
    baseMiddleware,                              // 注入数据库和认证
    turnstileMiddleware,                         // Cloudflare 人机验证
    rateLimitMiddleware({
      capacity: 5, interval: "1m",               // 每分钟最多 5 次
      identifier: createRateLimiterIdentifier,
    }),
    rateLimitMiddleware({
      capacity: 10, interval: "1h",               // 每小时最多 10 次
      identifier: (c) => `hourly:${createRateLimiterIdentifier(c)}`,
    }),
    forwardAuthRequest,
  );
});
```

### Snippet 3: 限流中间件 —— 使用 Durable Object
File: `src/lib/middlewares.ts` (lines 109-133, createRateLimitMiddleware)
```ts
export const createRateLimitMiddleware = (options) => {
  return createMiddleware({ type: "function" })
    .middleware([sessionMiddleware])
    .server(async ({ next, context }) => {
      const identifier =
        getRequestHeader("cf-connecting-ip") || session?.user.id || "unknown";
      const uniqueIdentifier = `${identifier}:${scope}`;

      // 用 Durable Object 实现分布式限流
      const id = context.env.RATE_LIMITER.idFromName(uniqueIdentifier);
      const rateLimiter = context.env.RATE_LIMITER.get(id);

      const result = await rateLimiter.checkLimit(options);

      if (!result.allowed) {
        throw createRateLimitError(result.retryAfterMs);
      }

      return next();
    });
};
```

### Snippet 4: Better Auth 配置
File: `src/lib/auth/auth.config.ts` (key config excerpt)
```ts
export const authConfig = {
  advanced: {
    generateId: false,
    crossSubDomainCookies: { enabled: false },
    cookies: { session_token: { attributes: { maxAge: 60 * 60 * 24 * 30 } } },
  },
  emailAndPassword: { enabled: true, requireEmailVerification: true },
  socialProviders: {
    github: { clientId: env.GITHUB_CLIENT_ID, clientSecret: env.GITHUB_CLIENT_SECRET },
  },
  databaseHooks: {
    user: {
      create: {
        before: async (user) => {
          if (user.email === env.ADMIN_EMAIL) {
            return { data: { ...user, role: "admin" } };
          }
          return { data: user };
        },
      },
    },
  },
};
```

## Interactive Elements

- [x] **代码↔中文翻译** — Snippet 1 (中间件链) 展示层层递进的安全检查
- [x] **群聊动画** — 角色：Browser（浏览器）、Auth（认证服务）、Session（会话管理器）、RateLimiter（限流器）、Turnstile（人机验证）。消息流：
  - Browser: "有人要登录！"
  - Auth: "先过人机验证" → Turnstile: "是真人，放行"
  - RateLimiter: "这个 IP 1 分钟内已经试了 4 次了，还剩 1 次机会"
  - Session: "验证密码... 通过！给你一个会话 cookie"
  - Auth: "检查邮箱是否是管理员... 是！角色设为 admin"
- [x] **测验** — 3 个问题，风格：场景/安全
  - Q1: "你想添加一个'只有登录用户才能发表评论'的功能。应该在 server function 上使用哪个中间件？" (答案：authMiddleware，它确保用户已登录)
  - Q2: "有人用脚本每秒发 100 次登录请求。系统能在哪一层拦住他？" (答案：RateLimiter Durable Object 会限制每分钟 5 次，Turnstile 也会挡住机器人)
  - Q3: "Better Auth 的 databaseHooks 做了什么？为什么新用户注册时需要检查邮箱？" (答案：检查邮箱是否是管理员邮箱，如果是则自动设置 admin 角色)
- [x] **编号步骤卡片** — 登录流程的 4 个检查步骤
- [x] **Callout box** — "💡 关键洞察：Durable Object 是 Cloudflare Workers 的'有状态组件'——普通 Worker 是无状态的（每次请求都是全新开始），但 Durable Object 可以记住状态（比如这个 IP 已经请求了多少次）。限流器就是用它来追踪每个 IP 的请求次数。"

## Reference Files to Read

- `references/interactive-elements.md` → "Group Chat Animation", "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Callout Boxes", "Numbered Step Cards"
- `references/content-philosophy.md` → 全部
- `references/gotchas.md` → 全部

## Connections

- **上一个模块：** "三层缓存" —— 本模块转向安全层面，中间件和缓存中间件是同一套系统
- **下一个模块：** "异步任务" —— 讲解 Queue 和 Workflow 如何处理不需要立即返回的后台工作
- **风格/语气说明：** 机场安检的比喻贯穿始终。群聊动画是本模块的核心交互元素。
