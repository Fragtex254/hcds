# 模块 2 Brief：代码库的组织方式

## Teaching Arc

- **隐喻：图书馆系统** —— 图书馆不是一个大箱子把所有书扔进去。它有分区（文学、科学、历史）、每个分区有管理员、每个管理员有借阅台。代码库也一样——每个「功能」就像图书馆的一个分区，有自己的数据区、服务区和前台。
- **开场钩子：** "如果你让 AI 帮你改博客的评论功能，它应该改哪个文件？如果你说'改评论那个文件夹里的代码'，你已经比大多数人强了。但具体是哪个文件呢？"
- **核心洞察：** 每个功能模块都遵循三层架构：数据层（data/）负责读写数据库、服务层（.service.ts）负责业务逻辑、API 层（api/）负责接收请求。知道这个结构，你就能精准地告诉 AI "把这段逻辑放在服务层"。
- **Why should I care?** 理解代码组织方式 = 知道代码住在哪里。当你跟 AI 说"在评论功能里加一个点赞功能"，AI 不会把代码乱放，你会知道它应该在 `features/comments/` 下面创建新文件。

## Code Snippets (pre-extracted)

### Snippet 1: 功能模块的目录结构
File: `src/features/posts/` directory structure
```
features/posts/
  data/posts.data.ts          # 数据层：纯数据库查询
  services/posts.service.ts   # 服务层：业务逻辑 + 缓存
  schema/posts.schema.ts      # Schema：Zod 验证 + 缓存 key
  api/                        # API 层：Server Functions
    posts.queries.ts          #   查询（读操作）
    posts.mutations.ts        #   变更（写操作）
  api/hono/                   # Hono 公开路由
    posts.list.route.ts
    posts.detail.route.ts
  components/                 # React 组件
  queries/                    # TanStack Query hooks
  workflows/                  # Cloudflare Workflows
```

### Snippet 2: 数据层 —— 纯粹的数据库查询
File: `src/features/posts/data/posts.data.ts` (lines 21-52, findPostBySlug)
```ts
export async function findPostBySlug(db: Database, slug: string) {
  const post = await db.query.posts.findFirst({
    where: and(
      eq(postsTable.slug, slug),
      eq(postsTable.status, "published"),
      lte(postsTable.publishedAt, new Date().toISOString()),
    ),
    columns: {
      id: true,
      title: true,
      summary: true,
      readTimeInMinutes: true,
      slug: true,
      publicContentJson: true,
      pinnedAt: true,
      publishedAt: true,
    },
    with: {
      postTags: {
        columns: { postId: false, tagId: false },
        with: { tag: { columns: { id: true, name: true } } },
      },
    },
  });

  if (!post) return undefined;

  return {
    ...post,
    tags: post.postTags.map((pt) => pt.tag),
    postTags: undefined,
  };
}
```

### Snippet 3: 服务层 —— 业务逻辑 + 缓存编排
File: `src/features/posts/services/posts.service.ts` (lines 48-82, findPostBySlug)
```ts
export async function findPostBySlug(
  context: Context,
  slug: string,
) {
  const version = await CacheService.getVersion(context, "posts:detail");
  const cacheKey = [version, "post", slug] as const;

  const post = await CacheService.get(
    context,
    cacheKey,
    PostWithTocSchema,
    async () => {
      const post = await PostRepo.findPostBySlug(context.db, slug);
      if (!post) return null;
      if (post.publicContentJson) {
        const { publicContentJson, ...rest } = post;
        const content = contentSchema.parse(publicContentJson);
        return { ...rest, content, toc: generateTocFromContent(content) };
      }
      const content = await highlightCodeBlocks(post.contentJson);
      const { contentJson, ...rest } = post;
      context.executionCtx.waitUntil(
        PostRepo.updatePublicContentSnapshot(context.db, post.id, content),
      );
      return { ...rest, content, toc: generateTocFromContent(content) };
    },
  );
  return post;
}
```

### Snippet 4: API 层 —— 薄薄的转发层
File: `src/features/posts/api/posts.queries.ts` (lines 46-60, postBySlugQuery)
```ts
export const postBySlugQuery = createServerFn({ method: "GET" })
  .validator((slug: string) => slug)
  .handler(async ({ data: slug, context }) => {
    const { db, env, executionCtx } = context;
    const post = await PostService.findPostBySlug(
      { db, env, executionCtx },
      slug,
    );
    return post;
  });
```

## Interactive Elements

- [x] **代码↔中文翻译** — Snippet 3 (服务层 findPostBySlug) 展示缓存编排
- [x] **测验** — 3 个问题，风格：架构决策
  - Q1: "你想给博客添加'文章点赞'功能。以下哪个做法最符合项目架构？" (答案：在 features/posts/data/ 添加数据库查询，在 service 添加业务逻辑，在 api 添加 server function)
  - Q2: "数据层（data/）和服务层（service/）的区别是什么？" (答案：数据层只做数据库查询，不做缓存或业务判断；服务层编排缓存和业务逻辑)
  - Q3: "AI 帮你写了一段代码，把数据库查询和缓存逻辑写在了同一个函数里。根据项目规范，这有什么问题？" (答案：违反了分层原则，应该把数据库查询放在 data/ 层，缓存编排放在 service 层)
- [x] **拖拽匹配** — 匹配代码职责到正确的层：
  - "查询数据库获取文章列表" → data/
  - "检查缓存是否过期，决定是否回源" → service
  - "接收 HTTP 请求，验证参数" → API
  - "生成 AI 摘要" → service
- [x] **视觉文件树** — 展示 features/posts/ 的目录结构，每个目录带一行说明
- [x] **Callout box** — "💡 关键洞察：这种分层叫'关注点分离'（Separation of Concerns）。就像餐厅里厨师不负责端菜、服务员不负责炒菜一样——每层只做自己的事，改动时不会互相影响。"
- [x] **Icon-label rows** — 展示三层架构：Data Layer（🗄️ 数据库查询）、Service Layer（⚙️ 业务逻辑 + 缓存）、API Layer（🔌 请求入口）

## Reference Files to Read

- `references/interactive-elements.md` → "Drag-and-Drop Matching", "Visual File Tree", "Icon-Label Rows", "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Callout Boxes"
- `references/content-philosophy.md` → 全部
- `references/gotchas.md` → 全部

## Connections

- **上一个模块：** "点击发布后发生了什么" —— 引用了 PostService 和 PostRepo，但没有解释它们的关系。本模块深入讲解。
- **下一个模块：** "请求的一生" —— 从浏览器发出请求到返回响应的完整路径，包括 Hono 路由和 TanStack SSR
- **风格/语气说明：** 演员命名延续模块 1。文件树和目录结构使用视觉文件树组件。
