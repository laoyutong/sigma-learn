# 对话记忆：RunnableWithMessageHistory

## 解决的问题

手动维护 `chat_history` 数组很繁琐。`RunnableWithMessageHistory` 自动：
- 调用前：从 store 读取历史，注入 `chat_history` 插槽
- 调用后：把本轮的 HumanMessage + AIMessage append 回 store

## 完整代码

```typescript
import { RunnableWithMessageHistory } from "@langchain/core/runnables";
import { ChatMessageHistory } from "langchain/memory";

const store: Record<string, ChatMessageHistory> = {};

const withHistory = new RunnableWithMessageHistory({
  runnable: executor,
  getMessageHistory: (sessionId) => {
    if (!store[sessionId]) {
      store[sessionId] = new ChatMessageHistory();
    }
    return store[sessionId];
  },
  inputMessagesKey: "input",
  historyMessagesKey: "chat_history",
});

// 调用时传 sessionId
await withHistory.invoke(
  { input: "北京天气怎么样？" },
  { configurable: { sessionId: "user_123" } }
);
```

## 每轮调用的背后

```
invoke() 被调用
  ↓
读 store[sessionId] → 注入 chat_history
  ↓
executor 运行（含工具调用）
  ↓
把 HumanMessage + AIMessage append 进 store[sessionId]
  ↓
返回结果
```

## sessionId 隔离多用户

- `store["user_A"]` 和 `store["user_B"]` 完全独立
- 同一个 `runnable` 实例可以服务无数用户，互不干扰
