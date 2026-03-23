# 工程实现：LangGraph / TypeScript 实战 (Supervisor 模式)

## 核心架构：星型拓扑与状态机

你选择的 B（指向 Supervisor）完全正确。在 LangGraph 的 Supervisor 模式中，图的边（Edge）必须严格遵循星型拓扑：
1. **Supervisor 决定去向**：它的下一步可能是任何 Worker，也可能是结束（END）。
2. **Worker 只能汇报**：任何 Worker 执行完毕后，必须无条件返回给 Supervisor。

```
       ┌──────────────────┐
       │                  │
   ┌───▼───┐          ┌───┴───┐
   │       │ 路由选择 │       │
   │ Coder ├─────────►│ Super-│───► END
   │       │          │ visor │
   └───▲───┘          └───┬───┘
       │                  │
       │                  │
       └──────────────────┘
            无条件返回
```

## TypeScript 完整实战代码

这段代码展示了如何用 LangGraph.js 实现一个真正的 Sub-Agent 系统。

### 1. 定义全局状态 (State)
```typescript
import { StateGraph, END, Annotation } from "@langchain/langgraph";
import { BaseMessage } from "@langchain/core/messages";

// 核心：所有节点共享这个状态对象
const AgentState = Annotation.Root({
  messages: Annotation<BaseMessage[]>({
    reducer: (x, y) => x.concat(y), // 消息追加
    default: () => [],
  }),
  // 关键：Supervisor 通过这个字段决定下一步调谁
  next: Annotation<string>({
    reducer: (x, y) => y ?? x,
    default: () => "supervisor",
  }),
});
```

### 2. 定义 Worker 节点 (Researcher & Coder)
```typescript
// 这是一个包装函数，把普通的 Agent 包装成图里的一个 Node
async function researcherNode(state: typeof AgentState.State) {
  // 1. 从全局状态中提取它需要的信息（Context Slicing）
  const result = await researcherAgent.invoke({
    messages: state.messages,
  });
  
  // 2. 将结果写回全局状态，并把控制权交还给 Supervisor
  return {
    messages: [result], // 追加它的回答
    // 注意：这里没有设置 next，因为它的边是硬编码指回 Supervisor 的
  };
}

async function coderNode(state: typeof AgentState.State) {
  const result = await coderAgent.invoke({ messages: state.messages });
  return { messages: [result] };
}
```

### 3. 定义 Supervisor 节点 (核心路由逻辑)
```typescript
import { z } from "zod";

async function supervisorNode(state: typeof AgentState.State) {
  // 1. Supervisor 的 System Prompt 明确了它的路由职责
  const prompt = `你是一个主管。你的任务是管理对话，并在 Researcher 和 Coder 之间路由。
  如果任务完成，请输出 FINISH。
  否则，请输出下一个要调用的 Worker 名字。`;

  // 2. 强制 Supervisor 输出结构化的路由决策 (Output Contract)
  const routingSchema = z.object({
    next: z.enum(["Researcher", "Coder", "FINISH"]),
  });

  const decision = await llm.withStructuredOutput(routingSchema).invoke([
    { role: "system", content: prompt },
    ...state.messages, // 传入历史记录让它判断当前进度
  ]);

  // 3. 更新全局状态的 next 字段
  return { next: decision.next };
}
```

### 4. 组装并编译图 (Graph Compilation)
```typescript
// 1. 初始化图
const workflow = new StateGraph(AgentState)
  .addNode("supervisor", supervisorNode)
  .addNode("Researcher", researcherNode)
  .addNode("Coder", coderNode);

// 2. 定义边：Worker 永远无条件返回 Supervisor
workflow.addEdge("Researcher", "supervisor");
workflow.addEdge("Coder", "supervisor");

// 3. 定义条件边：Supervisor 根据 state.next 决定去向
workflow.addConditionalEdges(
  "supervisor",
  // 路由函数：读取 state.next
  (state) => state.next === "FINISH" ? END : state.next
);

// 4. 设置入口点
workflow.setEntryPoint("supervisor");

// 5. 编译成可执行的 App
const app = workflow.compile();

// 运行！
const finalState = await app.invoke({
  messages: [{ role: "user", content: "帮我查一下 LangGraph 的最新文档，然后写一个 Hello World 例子。" }]
});
```

## 架构复盘

在这段代码中，你今天学到的所有概念都落地了：
1. **Orchestrator-Worker 模型**：`supervisorNode` 只做路由，`researcherNode` 做脏活。
2. **Output Contract**：Supervisor 强制输出 `z.enum(["Researcher", "Coder", "FINISH"])`。
3. **状态管理**：`AgentState` 是全局唯一的真理来源。
4. **信任边界**：`Coder` 节点可以拥有写文件的 Tool，而 `Supervisor` 节点没有任何底层 Tool。

这就是工业级 Sub-Agent 系统的终极形态。
