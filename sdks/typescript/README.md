# Agent User Interaction Protocol TypeScript SDK

[Agent User Interaction Protocol](https://ag-ui.com) 的 TypeScript SDK。

更多信息请访问[官方文档](https://docs.ag-ui.com/)。

## 多模态用户消息

```ts
import { UserMessageSchema } from "@ag-ui/core/schemas";

const message = UserMessageSchema.parse({
  id: "user-123",
  role: "user" as const,
  content: [
    { type: "text", text: "Please describe this image" },
    {
      type: "image",
      source: {
        type: "url",
        value: "https://example.com/cat.png",
        mimeType: "image/png",
      },
    },
  ],
});

console.log(message);
// { id: "user-123", role: "user", content: [...] }
```