# LangChain 架构与 LCEL 概览

> 从"为什么需要框架"出发，理解 Runnable 协议和管道哲学

## 第一步：框架解决了什么问题

**没有框架（写死的代码）**

```typescript
// 换模型？改这里
const res = await openai.chat.completions.create({...})

// 解析响应？改这里
res.choices[0].message

// 工具分发？写 switch
switch(name) {
  case "search": ...
  case "calc": ...
}
```

**有框架（可替换的积木）**

```typescript
// 换模型？只换这一行
const model = new ChatOpenAI()
// new ChatAnthropic()
// new ChatGoogleGenerativeAI()

// 其余代码完全不用改
const chain = prompt.pipe(model).pipe(parser)
```

> **核心思想**：把"会变化的部分"（用哪个模型、用哪些工具）和"不变的部分"（管道结构、数据流向）分离开来。

## 第二步：三大核心积木

| 组件 | 封装了什么 | 输入 → 输出 |
|------|-----------|------------|
| `PromptTemplate` | 把变量填入模板，生成 messages 数组 | `{topic: string}` → `BaseMessage[]` |
| `ChatModel` | 调用大模型 API，屏蔽 OpenAI / Claude / Gemini 格式差异 | `BaseMessage[]` → `AIMessage` |
| `OutputParser` | 把 AIMessage 解析成目标格式（字符串 / JSON / 对象） | `AIMessage` → `string` |

## 第三步：用 .pipe() 串联

```
PromptTemplate         ChatModel           OutputParser
IN: {topic}      →    IN: BaseMessage[]  →  IN: AIMessage
OUT: BaseMessage[]     OUT: AIMessage        OUT: string
        .pipe()                 .pipe()
```

**.pipe() 的唯一规则**：前一个组件的 Output 类型必须等于后一个组件的 Input 类型。TypeScript 在编译期帮你检查这件事。

```typescript
const chain = promptTemplate
  .pipe(chatModel)    // prompt OUT: BaseMessage[] = model IN ✓
  .pipe(outputParser) // model OUT: AIMessage = parser IN ✓

// chain 的完整类型：Runnable<{topic: string}, string>
const result = await chain.invoke({ topic: "量子计算" })
// result 类型：string ✓
```

## 第四步：Runnable 协议——让 .pipe() 成为可能

```typescript
interface Runnable<Input, Output> {
  // 最常用：同步调用，等待完整结果
  invoke(input: Input): Promise<Output>

  // 流式：一边生成一边输出（用于实时显示文字）
  stream(input: Input): AsyncIterable<Output>

  // 批量：并行处理多个输入
  batch(inputs: Input[]): Promise<Output[]>

  // 关键：连接下一个组件，返回新的 Runnable
  pipe<NewOutput>(
    next: Runnable<Output, NewOutput>
  ): Runnable<Input, NewOutput>
}
```

> **所有 LangChain 组件都实现了这个接口。** 无论是 PromptTemplate、ChatModel、OutputParser，还是你自定义的函数，只要实现了 Runnable，就能接入任何管道。

## 一句话总结

**LangChain = 三大积木（PromptTemplate / ChatModel / OutputParser）+ 一个协议（Runnable）+ 一个连接器（.pipe()）**

LCEL 的本质：让你用**声明式**的方式描述数据流向，而不是用命令式代码手动搬运数据。
