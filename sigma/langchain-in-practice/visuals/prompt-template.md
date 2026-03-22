# ChatModel + PromptTemplate 深入

> fromTemplate vs fromMessages，以及 MessagesPlaceholder 的关键作用

## 两种创建方式对比

**fromTemplate — 单条消息**

```typescript
const p = ChatPromptTemplate.fromTemplate("解释一下 {concept}")
// 生成：[HumanMessage("解释一下 {concept}")]
// 适合：简单问答，无需角色定义
```

**fromMessages — 多角色列表**

```typescript
const p = ChatPromptTemplate.fromMessages([
  ["system", "你是 {role}"],
  ["human",  "{input}"],
])
// 生成：[SystemMessage(...), HumanMessage(...)]
// 适合：Agent、对话机器人、需要角色设定
```

## invoke() 做了什么

```
📥 输入：{ language: "TypeScript", code: "const x = 1; x = 2;" }
   ↓ 填入模板变量
🔧 PromptTemplate 处理：把 {language} {code} 替换为实际值
   ↓ 返回格式化消息（还没调用模型！）
📤 输出：ChatPromptValue
   ├── SystemMessage: "你是一个专业的 TypeScript 代码审查员，风格简洁犀利。"
   └── HumanMessage:  "请审查这段代码：\n\nconst x = 1; x = 2;"
   ↓ .pipe(model) 之后才调用模型
🤖 ChatModel 输出：AIMessage
   └── "错误：`const` 声明的变量不可重新赋值。第 1 行应改为 `let x = 1;`。"
```

> **注意**：`prompt.invoke()` 只是格式化消息，**不调用模型**。要调用模型必须 `.pipe(model)`。

## MessagesPlaceholder — 注入动态历史（重要）

Agent 需要把整个对话历史塞进每次请求。`MessagesPlaceholder` 就是模板里的"插槽"，在 invoke 时动态注入任意数量的历史消息。

```typescript
import { MessagesPlaceholder } from "@langchain/core/prompts"

const prompt = ChatPromptTemplate.fromMessages([
  ["system", "你是一个助手。"],
  new MessagesPlaceholder("chat_history"), // ← 动态插槽
  ["human", "{input}"],
])

// 第一轮：历史为空
await chain.invoke({ input: "你好", chat_history: [] })

// 第二轮：注入上一轮记录
await chain.invoke({
  input: "刚才说了什么？",
  chat_history: [
    new HumanMessage("你好"),
    new AIMessage("你好！有什么我可以帮你的吗？"),
  ]
})
```

第二轮模型实际收到的消息顺序：
```
[system]         你是一个助手。
[human]（历史）  你好
[ai]  （历史）   你好！有什么我可以帮你的吗？
[human]（当前）  刚才说了什么？
```

> **关键理解**：模型是无状态的，每次都要把完整历史发过去。`MessagesPlaceholder` 是"把历史塞进 prompt"的标准做法，和手动 `messages.push(...)` 原理相同，只是换成了声明式写法。

## 选择指南

| 场景 | 用哪个 |
|------|--------|
| 简单单轮问答（无角色） | `fromTemplate` |
| 需要 system prompt 定义角色 | `fromMessages` |
| Agent / 多轮对话（需历史） | `fromMessages` + `MessagesPlaceholder` |
