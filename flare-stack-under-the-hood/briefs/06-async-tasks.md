# 模块 6 Brief：幕后工厂——异步任务系统

## Teaching Arc

- **隐喻：工厂流水线** —— 汽车工厂里，组装一辆车不是一个人从头做到尾。焊车身的、喷漆的、装引擎的，各干各的，而且很多事情是并行进行的。用户点了「发布」后不需要等 AI 写完摘要才能看到结果——后台的「工厂」会自动把这些任务分配到不同的「生产线」上。
- **开场钩子：** "有人在你的博客上留了一条评论。几秒后，你收到一封邮件通知。但评论者根本没等邮件发完才看到'评论成功'——邮件是在后台悄悄发出去的。这是怎么做到的？"
- **核心洞察：** 项目使用两种异步工具：Queue 处理快速的批量任务（邮件、浏览量统计、webhook），Workflow 处理需要多步骤、可恢复的长任务（文章发布流程、AI 评论审核、导入导出）。选错工具会导致任务丢失或重复执行。
- **Why should I care?** 当你让 AI 帮你加一个"发邮件通知"功能，你需要知道这应该走 Queue 而不是在请求里直接发。如果 AI 在 API handler 里同步发邮件，用户体验会很差（等邮件发完才返回），而且如果邮件服务挂了，整个请求就失败了。

## Code Snippets (pre-extracted)

### Snippet 1: Queue 消息处理 —— 邮件分拣中心
File: `src/lib/queue/queue.handler.ts` (full file)
```ts
export async function handleQueueBatch(batch: MessageBatch, env: Env, ctx: ExecutionContext) {
  const pageviewBatch = [];

  for (const message of batch.messages) {
    const parsed = queueMessageSchema.safeParse(message.body);
    if (!parsed.success) {
      message.ack();
      continue;
    }

    try {
      const event = parsed.data;
      switch (event.type) {
        case "EMAIL":
          await handleEmailMessage({ env, executionCtx: ctx }, {
            ...event.data,
            idempotencyKey: message.id,
          });
          break;
        case "WEBHOOK":
          await handleWebhookMessage({ env }, event.data, message.id);
          break;
        case "POST_AUTO_SNAPSHOT":
          await handlePostAutoSnapshotMessage({ env }, event.data);
          break;
        case "PAGEVIEW":
          pageviewBatch.push({ data: event.data, message });
          continue;  // 不立即 ack，等批量处理
        default:
          event satisfies never;
          throw new Error("Unknown queue message type");
      }
      message.ack();
    } catch (error) {
      message.retry();  // 失败时自动重试
    }
  }

  // 浏览量批量处理——合并多条消息减少数据库写入
  if (pageviewBatch.length > 0) {
    try {
      await handlePageviewMessages({ env }, pageviewBatch.map((item) => item.data));
      for (const item of pageviewBatch) item.message.ack();
    } catch (error) {
      for (const item of pageviewBatch) item.message.retry();
    }
  }
}
```

### Snippet 2: AI 评论审核 Workflow —— 多步骤可恢复的后台任务
File: `src/features/comments/workflows/comment-moderation.ts` (key steps)
```ts
export class CommentModerationWorkflow extends WorkflowEntrypoint<Env, Params> {
  async run(event: WorkflowEvent<Params>, step: WorkflowStep) {
    const { commentId } = event.payload;

    // 1. 获取评论
    const comment = await step.do("fetch comment", async () => {
      return await CommentRepo.findCommentById(db, commentId);
    });

    // 2. 获取帖子上下文
    const post = await step.do("fetch post and thread", async () => {
      return await PostRepo.findPostById(db, comment.postId);
    });

    // 3. AI 审核（可能失败，有重试）
    const moderation = await step.do(
      "moderate comment",
      { retries: { limit: 3, delay: "3 seconds", backoff: "exponential" } },
      async () => {
        const result = await moderateComment({
          comment: comment.content,
          post: { title: post.title, summary: post.summary },
          replyToComment,
        });
        return result;
      },
    );

    // 4. 根据 AI 判断更新状态
    await step.do("update comment status", async () => {
      if (moderation.shouldPublish) {
        await CommentRepo.updateCommentStatus(db, commentId, "published");
      } else {
        await CommentRepo.updateCommentStatus(db, commentId, "pending");
      }
    });

    // 5. 发送通知
    await step.do("send notification", async () => {
      // 如果被标记为待审核，通知管理员
      // 如果通过审核且是回复，通知被回复者
    });
  }
}
```

### Snippet 3: Queue 消息类型定义
File: `src/lib/queue/queue.schema.ts` (key types)
```ts
// 四种消息类型
const emailMessageSchema = z.object({
  type: z.literal("EMAIL"),
  data: emailDataSchema,
});

const webhookMessageSchema = z.object({
  type: z.literal("WEBHOOK"),
  data: webhookDataSchema,
});

const postAutoSnapshotMessageSchema = z.object({
  type: z.literal("POST_AUTO_SNAPSHOT"),
  data: postAutoSnapshotDataSchema,
});

const pageviewMessageSchema = z.object({
  type: z.literal("PAGEVIEW"),
  data: pageviewDataSchema,
});

// 联合类型：消息必须是这四种之一
export const queueMessageSchema = z.discriminatedUnion("type", [
  emailMessageSchema,
  webhookMessageSchema,
  postAutoSnapshotMessageSchema,
  pageviewMessageSchema,
]);
```

## Interactive Elements

- [x] **代码↔中文翻译** — Snippet 1 (Queue handler) 展示消息分发和批量处理
- [x] **群聊动画** — 角色：Comment（评论）、Workflow（审核流水线）、AI（审核员）、Database（数据库）、Email（通知员）。消息流：
  - Comment: "新评论！有人回复了帖子"
  - Workflow: "好的，让我来处理。AI，帮我看看这条评论有没有问题"
  - AI: "看起来是正常评论，可以发布"
  - Workflow: "Database，把状态改成已发布"
  - Database: "搞定"
  - Workflow: "Email，通知被回复的人"
  - Email: "已经把通知邮件放进队列了，稍后发送"
- [x] **测验** — 3 个问题，风格：架构决策
  - Q1: "用户发表了评论后，应该用 Queue 还是 Workflow 来处理后续任务？" (答案：Workflow——因为评论审核是多步骤、需要重试的持久化流程)
  - Q2: "为什么要单独处理 PAGEVIEW 消息的批量合并，而不是像 EMAIL 一样逐条处理？" (答案：浏览量非常高频，逐条写数据库会产生大量写入。合并后一次写入更高效)
  - Q3: "如果 AI 审核服务暂时不可用，评论会丢失吗？" (答案：不会——Workflow 的 step.do 有重试机制（3 次，指数退避），AI 服务恢复后会自动重试)
- [x] **拖拽匹配** — 匹配任务类型到正确的异步工具：
  - "发送评论回复邮件通知" → Queue
  - "发布文章后生成 AI 摘要 + 更新搜索索引" → Workflow
  - "记录用户浏览量" → Queue
  - "AI 审核评论内容" → Workflow
  - "定时发布预设文章" → Workflow
  - "触发 Webhook 通知外部服务" → Queue
- [x] **Callout box** — "💡 关键洞察：Queue 和 Workflow 的区别就像'快递'和'审批流程'。快递（Queue）是把包裹扔进系统，它会到的，但你不需要知道中间经过了哪些站点。审批流程（Workflow）是需要按步骤执行、每步可重试、可以中途暂停恢复的复杂流程。选错会导致任务丢失或重复执行。"
- [x] **模式卡片** — 展示两种异步工具的特点对比：Queue（高吞吐、火后即忘、批量处理）vs Workflow（多步骤、可恢复、有状态）

## Reference Files to Read

- `references/interactive-elements.md` → "Group Chat Animation", "Drag-and-Drop Matching", "Code ↔ English Translation Blocks", "Multiple-Choice Quizzes", "Callout Boxes", "Pattern/Feature Cards"
- `references/content-philosophy.md` → 全部
- `references/gotchas.md` → 全部

## Connections

- **上一个模块：** "身份验证与安全" —— 本模块转向异步处理，解释了为什么某些操作不在请求中同步完成
- **下一个模块：** 无（这是最后一个模块）。本模块作为课程的收官，展示了一个完整的生产级博客系统如何处理后台任务。
- **风格/语气说明：** 工厂流水线的比喻贯穿始终。群聊动画（评论审核流程）是本模块的核心交互元素。拖拽匹配练习帮助区分 Queue 和 Workflow 的使用场景。
