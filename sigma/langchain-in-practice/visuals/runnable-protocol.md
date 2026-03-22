# Runnable 协议：三种调用方式

> invoke / stream / batch — 同一条 chain，三种不同的执行模式

## .invoke() — 等待完整结果

```typescript
const result = await chain.invoke({ question: "什么是泛型？" })
// result: string（完整答案，等全部生成后才返回）
```

适合：后端 API 处理单个请求，不需要实时展示的场景。

## .stream() — 流式逐块输出

```typescript
const stream = await chain.stream({ question: "什么是泛型？" })
for await (const chunk of stream) {
  process.stdout.write(chunk) // chunk: string 片段，逐字输出
}
// 效果：泛 → 型 → 是 → 一 → 种...（像 ChatGPT 那样逐字显示）
```

适合：聊天界面，需要打字机效果实时展示。

## .batch() — 并发批量处理

```typescript
const results = await chain.batch([
  { question: "什么是泛型？" },
  { question: "什么是装饰器？" },
  { question: "什么是类型守卫？" },
])
// results: string[]（3 个请求并发执行，不是串行排队）
```

适合：批量处理数据，多条记录独立生成内容。

## stream() 的 chunk 类型：取决于链末尾的组件

| 链末尾组件 | invoke() 返回 | stream() 的 chunk |
|-----------|--------------|------------------|
| `StringOutputParser` | `string` | `string` 片段（"泛" "型" "是"…） |
| `ChatModel`（无 parser） | `AIMessage` | `AIMessageChunk` 对象 |
| `JsonOutputParser` | `object` | 部分 JSON 对象（逐步填充） |

> **规律**：chunk 类型 = 该组件 invoke() 返回类型的"片段版"。链末尾用 `StringOutputParser` 是最常见的做法，chunk 就是干净的 string 片段。

## 选择指南

| 场景 | 用哪个 |
|------|--------|
| 后端 API，处理单个请求，不需要实时 | `.invoke()` |
| 聊天界面，需要打字机效果逐字显示 | `.stream()` |
| 批量处理数据，多条记录独立生成内容 | `.batch()` |
