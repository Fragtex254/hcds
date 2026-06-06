# 模块 1 Brief：点击「发布」后发生了什么？

## Teaching Arc

- **隐喻：杂志印刷厂** —— 文章被批准后，经过排版、印刷、分发，最终到达读者手中。你点击「发布」的那一刻，就像杂志主编签发了一期新刊——之后有一整条自动化流水线在运转。
- **开场钩子：** "你写完一篇博客，点了「发布」按钮。几秒钟后，全世界的人都能看到它了。但在这几秒钟里，到底发生了什么？"
- **核心洞察：** 发布不是一步操作，而是一条由 Workflow 驱动的自动化流水线——AI 生成摘要、代码高亮、搜索索引、缓存失效，每一步都有重试机制保障。
- **Why should I care?** 当你的 AI 工具帮你写了一个「发布功能」，你需要知道真正的发布流程应该包含哪些步骤。如果它只写了个数据库更新，你就该知道缺了什么。

## Code Snippets (pre-extracted)

### Snippet 1: 入口文件 —— 一切从这里开始
File: `src/server.ts` (full file, 31 lines)
```ts
import { handleQueueBatch } from "@/lib/queue/queue.handler";

export { CommentModerationWorkflow } from "@/features/comments/workflows/comment-moderation";
export { ExportWorkflow } from "@/features/import-export/workflows/export.workflow";
export { ImportWorkflow } from "@/features/import-export/workflows/import.workflow";
export { PostAutoSnapshotWorkflow } from "@/features/posts/workflows/post-auto-snapshot";
export { PostProcessWorkflow } from "@/features/posts/workflows/post-process";
export { ScheduledPublishWorkflow } from "@/features/posts/workflows/scheduled-publish";
export { PasswordHasher } from "@/lib/do/password-hasher";
export { RateLimiter } from "@/lib/do/rate-limiter";

export default {
  async fetch(request, env, ctx) {
    const { handleRootRequest } = await import("@/lib/worker/root-handler");
    return handleRootRequest(request, env, ctx);
  },
  async queue(batch, env, ctx) {
    await handleQueueBatch(batch, env, ctx);
  },
} satisfies ExportedHandler<Env>;
```

### Snippet 2: 发布工作流 —— 自动化流水线的核心
File: `src/features/posts/workflows/post-process.ts` (lines 24-98, key steps)
```ts
export class PostProcessWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    const { postId, isPublished } = event.payload;
    if (isPublished) {
      await this.handlePublish(event, step, postId);
    } else {
      await this.handleUnpublish(step, postId);
    }
  }

  private async handlePublish(event, step, postId) {
    // 1. 检查内容是否有变化
    const { post: initialPost, shouldSkip } = await step.do(
      "check sync status",
      async () => { /* 计算内容哈希，对比旧哈希 */ }
    );
    if (shouldSkip || !initialPost) return;

    // 2. AI 生成摘要
    const updatedPost = await step.do(
      `generate summary for post ${postId}`,
      { retries: { limit: 3, delay: "5 seconds", backoff: "exponential" } },
      async () => { /* 调用 Workers AI 生成摘要 */ }
    );

    // 3. 构建高亮后的公开内容快照
    await step.do("build public content", async () => {
      const publicContentJson = post.contentJson
        ? await highlightCodeBlocks(post.contentJson)
        : null;
      await PostRepo.updatePublicContentSnapshot(db, postId, publicContentJson);
    });

    // 4. 更新搜索索引
    await step.do("update search index", async () => { /* 更新 Orama 索引 */ });

    // 5. 失效缓存
    await step.do("invalidate caches", async () => { /* KV + CDN 缓存失效 */ });

    // 6. 更新同步哈希
    await step.do("update sync hash", async () => { /* 防止重复处理 */ });
  }
}
```

### Snippet 3: 发布文章的服务层代码
File: `src/features/posts/services/posts.service.ts` (lines 84-127, publishPost method)
```ts
export async function publishPost(
  context: Context,
  input: { postId: number; publishedAt?: string | null },
) {
  const post = await PostRepo.findPostById(context.db, input.postId);
  if (!post) return err("POST_NOT_FOUND");

  const slug = await ensureUniqueSlug(context.db, post.title, post.id);

  let publishedAt = post.publishedAt;
  if (post.status === "draft") {
    publishedAt = input.publishedAt
      ? new Date(input.publishedAt).toISOString()
      : new Date().toISOString();
  }

  await PostRepo.updatePost(context.db, post.id, {
    publishedAt,
    status: "published",
    slug,
    updatedAt: new Date().toISOString(),
  });

  // 创建 Workflow 实例处理发布后任务
  const instance = await context.env.POST_PROCESS_WORKFLOW.create({
    params: {
      postId: post.id,
      isPublished: true,
      publishedAt,
    },
  });
  console.log(JSON.stringify({ message: "Post process workflow created", postId: post.id, workflowId: instance.id }));

  return ok({ postId: post.id, status: "published" as const, slug });
}
```

## Interactive Elements

- [x] **代码↔中文翻译** — Snippet 1 (server.ts 入口) 和 Snippet 3 (publishPost 服务)
- [x] **测验** — 3 个问题，风格：场景/架构决策
  - Q1: "发布一篇文章后，如果 AI 摘要生成失败了，文章还能发布吗？" (答案：能，Workflow 有重试机制，即使失败也不影响文章本身)
  - Q2: "为什么不直接在 publishPost 函数里做完所有事情（高亮、搜索索引、缓存失效），而要用 Workflow？" (答案：因为这些步骤可能失败需要重试，且不应阻塞用户操作)
  - Q3: "如果有人发布了文章，然后立刻修改了内容再发布，会发生什么？" (答案：sync hash 机制会检测到变化，重新处理)
- [x] **群聊动画** — 角色：PostService（发布者）、Workflow（流水线调度员）、AI（摘要生成器）、Search（搜索索引）、Cache（缓存管理器）。消息流：PostService 告诉 Workflow "新文章发布了！" → Workflow 问 AI "帮我写个摘要" → AI 回复摘要 → Workflow 告诉 Search "更新索引" → Workflow 告诉 Cache "清除旧缓存" → Workflow 回复 PostService "全部搞定"
- [x] **数据流动画** — 角色：用户、PostService、Workflow、AI、Search、Cache、CDN。步骤：用户点击发布 → PostService 保存到数据库 → 创建 Workflow 实例 → Workflow 检查同步状态 → AI 生成摘要 → 构建代码高亮快照 → 更新搜索索引 → 失效 KV 缓存 → 清除 CDN 缓存
- [x] **Callout box** — "💡 关键洞察：Workflow vs 函数调用——Workflow 是"可以暂停和重试的函数"。如果 AI 服务暂时不可用，Workflow 会自动重试 3 次（每次间隔更长），而不是直接报错给用户。"
- [x] **步骤卡片** — 发布流水线的 6 个步骤（带编号）

## Reference Files to Read

- `references/interactive-elements.md` → "Group Chat Animation", "Message Flow / Data Flow Animation", "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Callout Boxes", "Numbered Step Cards"
- `references/content-philosophy.md` → 全部（内容规则）
- `references/gotchas.md` → 全部（检查清单）
- `references/design-system.md` → 颜色变量（actor-1 到 actor-5）、间距、排版

## Connections

- **上一个模块：** 无（这是第一个模块）
- **下一个模块：** "代码库的组织方式" —— 介绍功能模块三层架构（data → service → API），解释为什么代码被组织成这样
- **风格/语气说明：**
  - 演员命名：PostService（发布者）、Workflow（调度员）、AI（摘要师）、Search（索引员）、Cache（管家）
  - 使用 teal 色调作为课程主题色
  - 每个技术术语首次出现时使用 glossary tooltip
