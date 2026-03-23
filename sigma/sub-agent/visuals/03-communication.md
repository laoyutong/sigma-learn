# Agent 间通信机制：Tool Call vs Message Passing

## 核心机制：Tool Call 模式

在代码层面，Orchestrator 怎么"派发任务"？**它其实就是把 Worker 包装成了一个 Tool。**

对于 Orchestrator 来说，它根本不知道背后是一个复杂的 AI Agent，它只知道：
> "我有一个工具叫 `analyze_company`，参数是 `company_name`，返回值是一个包含财务数据的 JSON。"

### 代码推演（TypeScript）

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// 1. 定义 Worker 的能力（对 Orchestrator 来说，这就是一个普通的 Tool）
const analyzeCompanyTool = tool(
  async ({ company_name }) => {
    // 这里的内部实现，其实是启动了另一个完整的 Agent 循环！
    console.log(`[Worker] 开始分析: ${company_name}`);
    
    // Worker 自己的系统提示词
    const workerPrompt = `你是一个财务分析师。分析 ${company_name}。
    你必须使用 search_tool 查找最新财报。
    返回 JSON 格式：{ revenue: number, growth: number }`;
    
    // 运行 Worker Agent
    const workerResult = await runAgent(workerPrompt, [search_tool]);
    
    return workerResult; // 返回给 Orchestrator
  },
  {
    name: "analyze_company",
    description: "分析指定公司的财务状况。当你需要分析一家公司时调用此工具。",
    schema: z.object({
      company_name: z.string().describe("要分析的公司名称，如 Tesla")
    })
  }
);

// 2. Orchestrator 的配置
const orchestratorPrompt = `你是一个投资研究主管。
你需要分析用户指定的所有公司，并汇总成一份报告。
你可以并行调用 analyze_company 工具来获取各家公司的数据。`;

// Orchestrator 只需要把 Worker 当作工具传入
const orchestratorAgent = createAgent({
  tools: [analyzeCompanyTool], // Worker 变成了 Tool！
  systemPrompt: orchestratorPrompt
});
```

## 为什么这种设计很精妙？

1. **统一抽象**：对大模型来说，调用一个计算器（普通函数）和调用一个 Worker Agent（另一个大模型循环）**在接口上没有任何区别**。都是输出 JSON 参数，接收文本结果。
2. **无限嵌套**：既然 Worker 可以被包装成 Tool，那么 Worker 内部还可以再调其他 Worker（变成 Sub-Sub-Agent），形成树状的层级结构（Hierarchical）。
3. **隔离性**：Orchestrator 的上下文窗口里，只会看到 `Tool Call: analyze_company("Tesla")` 和 `Tool Result: {"revenue": 97.3...}`。Worker 内部长达 5000 token 的搜索记录、思考过程（ReAct），**全部被封装在 Worker 内部，不会污染 Orchestrator 的上下文**。

## 另一种模式：Message Passing（消息传递）

除了 Tool Call，还有一种模式是基于图（Graph）或 Actor 模型的**消息传递**（如 LangGraph 或 AutoGen 常用的方式）。

```
Orchestrator 节点 ---发送消息(Message)---> Worker 节点
                                             |
Orchestrator 节点 <---返回消息(Message)-------+
```

| 特性 | Tool Call 模式 (LangChain Agent) | Message Passing 模式 (LangGraph/AutoGen) |
|------|-----------------------------------|------------------------------------------|
| 触发方式 | LLM 主动决定调用工具 | 框架根据图的边（Edge）或路由规则转发消息 |
| 状态管理 | 隐式（在 Tool 函数的闭包里） | 显式（有一个全局的 State 对象在节点间流转） |
| 适用场景 | 动态决定子任务（比如不知道要查几家公司） | 流程固定或需要严格状态机的场景 |
| 调试难度 | 较难（嵌套的 Agent 循环不好追踪） | 较易（每一步状态流转都很清晰） |

## 总结

**Sub-Agent 架构的基石，就是把复杂的 Agent 逻辑封装在简单的接口（Tool 或 Message）之后。**
