# 模块 4 Brief：三层缓存——为什么你的博客这么快

## Teaching Arc

- **隐喻：图书馆预约系统** —— 想象你去图书馆借一本书。第一种方式：直接去书架找（CDN 边缘缓存，最快）。第二种方式：问前台管理员，她查一下预约系统告诉你有没有（Cache API）。第三种方式：管理员打电话给仓库确认（KV 缓存）。最后才需要从总馆调书（数据库查询）。每一层都比下一层快得多。
- **开场钩子：** "一个博客文章页面可能需要查询数据库、生成目录、格式化内容。但为什么你的博客加载只需要几百毫秒？因为绝大多数请求根本不需要碰数据库。"
- **核心洞察：** 缓存不是一层，而是三层。CDN 在全球边缘节点缓存 HTML 页面，Cache API 在 Worker 内部缓存响应，KV 在应用层缓存数据。最重要的是：缓存失效比缓存写入难 100 倍——这个项目用「版本号」优雅地解决了这个问题。
- **Why should I care?** 缓存是最常见也最容易搞错的性能优化。当你说"AI 帮我加个缓存"，你需要知道加在哪一层。如果 AI 只加了一层内存缓存，你需要知道真正的生产环境需要三层。

## Code Snippets (pre-extracted)

### Snippet 1: 缓存服务的核心 —— 读穿缓存 + 版本失效
File: `src/features/cache/cache.service.ts` (lines 13-52, get method)
```ts
export async function get<T extends z.ZodTypeAny>(
  context: BaseContext & { executionCtx: ExecutionContext },
  key: CacheKey,
  schema: T,
  fetcher: () => Promise<z.infer<T>>,
  options: { ttl?: Duration } = {},
): Promise<z.infer<T>> {
  const { ttl = "1h" } = options;
  const { env } = context;
  const serializedKey = serializeKey(key);

  const kvData = await env.KV.get(serializedKey, "json").catch(/* ... */);

  if (kvData !== null && kvData !== undefined) {
    const result = schema.safeParse(kvData);
    if (result.success) {
      return result.data;
    }
  }

  const data = await fetcher();

  if (data === null || data === undefined) return data;

  context.executionCtx.waitUntil(
    set(context, key, JSON.stringify(data), { ttl }),
  );

  return data;
}
```

### Snippet 2: 版本号失效 —— 一键清空整个命名空间
File: `src/features/cache/cache.service.ts` (lines 130-190, getVersion + bumpVersion)
```ts
export async function getVersion(context: BaseContext, namespace: CacheNamespace) {
  const key = `ver:${namespace}`;
  const v = await context.env.KV.get(key).catch(/* ... */);
  if (v && !Number.isNaN(Number.parseInt(v))) {
    return `v${v}`;
  }
  return "v1";
}

export async function bumpVersion(context: BaseContext, namespace: CacheNamespace) {
  const key = `ver:${namespace}`;
  const current = await context.env.KV.get(key).catch(/* ... */);

  let next = 1;
  if (current) {
    const parsed = Number.parseInt(current);
    if (!Number.isNaN(parsed)) {
      next = parsed + 1;
    }
  }

  await context.env.KV.put(key, next.toString()).catch(/* ... */);
}
```

### Snippet 3: Hono 缓存中间件 —— Worker 级缓存
File: `src/lib/hono/middlewares.ts` (cacheMiddleware 概念代码，基于实际代码简化)
```ts
// 请求进入时：检查 Cache API 是否有缓存的响应
const cache = caches.default;
const cachedResponse = await cache.match(request);
if (cachedResponse) return cachedResponse;

// 没有缓存：继续处理请求
const response = await next();

// 响应返回后：如果是可缓存的响应，存入 Cache API
if (response.status === 200 && !hasSetCookie(response)) {
  c.executionCtx.waitUntil(cache.put(request, response.clone()));
}

return response;
```

### Snippet 4: 服务层使用缓存
File: `src/features/posts/services/posts.service.ts` (lines 48-82, 使用 CacheService.get)
```ts
export async function findPostBySlug(context: Context, slug: string) {
  // 1. 获取当前缓存版本号
  const version = await CacheService.getVersion(context, "posts:detail");
  // 2. 将版本号拼入缓存 key
  const cacheKey = [version, "post", slug] as const;

  // 3. 读穿缓存：先查 KV，miss 时回源数据库
  const post = await CacheService.get(
    context,
    cacheKey,
    PostWithTocSchema,
    async () => {
      const post = await PostRepo.findPostBySlug(context.db, slug);
      if (!post) return null;
      // ... 处理高亮、目录生成 ...
      return processedPost;
    },
  );
  return post;
}
```

## Interactive Elements

- [x] **代码↔中文翻译** — Snippet 1 (CacheService.get 读穿缓存)
- [x] **层级切换演示** — 三层缓存切换：CDN 层（全球边缘节点，缓存 HTML）、Cache API 层（Worker 内部，缓存响应）、KV 层（应用层，缓存数据）。每层显示其位置、存储内容、失效方式
- [x] **测验** — 3 个问题，风格：架构决策/调试
  - Q1: "管理员更新了一篇文章，但用户仍然看到旧版本。三层缓存中，哪一层最可能没有被正确失效？" (答案：CDN 层——KV 版本号 bump 和 Cache API 失效可能是正确的，但 CDN purge 可能失败)
  - Q2: "版本号失效相比逐个删除缓存 key，最大的优势是什么？" (答案：一次 bump 就能让整个命名空间的所有旧 key 失效，不需要知道有哪些 key)
  - Q3: "如果 KV 服务暂时不可用，findPostBySlug 还能工作吗？" (答案：能——CacheService.get 的 KV.get 有 catch 处理，会 fallthrough 到 fetcher 回源数据库)
- [x] **数据流动画** — 角色：Browser、CDN、Cache API、KV、D1。步骤：请求到达 → CDN 检查（命中/未命中）→ Cache API 检查（命中/未命中）→ KV 检查（命中/未命中）→ D1 查询 → 数据返回并逐层写入缓存
- [x] **Callout box** — "💡 关键洞察：`executionCtx.waitUntil()` 是一个神奇的 API——它告诉 Cloudflare '这个操作很重要，请在后台完成它，但不用等它结束再返回响应'。这就是为什么缓存写入不会拖慢用户看到的响应速度。"
- [x] **模式卡片** — 展示三种缓存失效策略：版本号 bump（批量失效）、精确 key 删除（单条失效）、CDN Purge API（全球清除）

## Reference Files to Read

- `references/interactive-elements.md` → "Layer Toggle Demo", "Message Flow / Data Flow Animation", "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Callout Boxes", "Pattern/Feature Cards"
- `references/content-philosophy.md` → 全部
- `references/gotchas.md` → 全部

## Connections

- **上一个模块：** "请求的一生" —— 提到了 cacheMiddleware 和 CacheService，本模块深入讲解它们的工作原理
- **下一个模块：** "身份验证与安全" —— 讲解认证中间件链、密码哈希、限流
- **风格/语气说明：** 图书馆预约系统的比喻贯穿始终。层级切换演示是本模块的核心交互元素。
