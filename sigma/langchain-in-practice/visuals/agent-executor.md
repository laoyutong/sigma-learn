# 构建 Agent：createToolCallingAgent + AgentExecutor

## 完整代码结构

```typescript
import { createToolCallingAgent, AgentExecutor } from "langchain/agents";
import { ChatPromptTemplate, MessagesPlaceholder } from "@langchain/core/prompts";

// Prompt 必须包含 agent_scratchpad 插槽
const prompt = ChatPromptTemplate.fromMessages([
  ["system", "你是一个助手。"],
  new MessagesPlaceholder("chat_history"),
  ["human", "{input}"],
  new MessagesPlaceholder("agent_scratchpad"),  // ← 必须有
]);

// agent = 大脑（LLM + tools + prompt）
const agent = createToolCallingAgent({ llm: model, tools, prompt });

// executor = 躯体（跑 while 循环）
const executor = new AgentExecutor({ agent, tools });

const result = await executor.invoke({
  input: "北京今天天气怎么样？",
  chat_history: [],
});
```

## 职责分工

| 对象 | 职责 |
|------|------|
| `agent` | 调用 LLM，决定下一步：调工具 or 直接回答 |
| `executor` | 跑 while 循环，执行工具，管理 scratchpad，直到 agent 给出最终回答 |

## agent_scratchpad 是什么

当次运行的**中间过程**，存在当前 messages 里，不持久化：

```
第1轮后：[AIMessage(tool_call), ToolMessage(结果)]
第2轮后：[...第1轮, AIMessage(tool_call), ToolMessage(结果)]
最终：agent 返回纯文本，循环退出，scratchpad 丢弃
```

## while 循环轮数

**轮数 = 工具调用次数 + 1**（最后一轮 agent 直接回答，无 tool_call）
