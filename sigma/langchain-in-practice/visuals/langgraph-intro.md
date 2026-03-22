# 迈向 LangGraph：AgentExecutor 的局限与图结构的优势

## AgentExecutor 的本质

AgentExecutor 内部是一个**固定形态的 while 循环**：

```
while (true) {
  output = agent.invoke(state)        // 调用 LLM
  if (output.returnValues) return     // 直接回答 → 退出
  executeTools(output.toolCalls)      // 有工具调用 → 执行后继续
}
```

**只有一种执行路径**：调工具 → 调工具 → ... → 回答。

## AgentExecutor 的局限

| 需求 | AgentExecutor | 原因 |
|------|--------------|------|
| 线性工具链（搜索→摘要→回答） | ✓ 能做 | 符合固定循环结构 |
| 条件分支（小额自动通过，大额人审） | ✗ 不能 | 无法插入条件跳转节点 |
| 人工介入节点（Human-in-the-loop） | ✗ 不能 | 循环无法暂停等待外部输入 |
| 多 Agent 协作（路由 → 专家 Agent） | ✗ 难做 | 无法定义节点间的有向关系 |
| 循环回路（生成 → 审查 → 不通过重新生成） | ✗ 不能 | 无法定义带条件的回边 |

## LangGraph 的解法：有向图

LangGraph 把工作流建模成**有向图**：

- **节点（Node）**：任意函数（调 LLM、执行工具、人工确认...）
- **边（Edge）**：固定跳转或条件跳转
- **状态（State）**：在节点间流动的共享数据

```
          ┌─────────────────────────────────┐
          ↓                                 │（审查不通过）
    [路由节点] ──写代码──→ [代码生成] → [代码审查]
         │                                  │（审查通过）
         └──普通问题──→ [通用回答]           ↓
                            │             [END]
                            └──────────────↑
```

## LangChain 代码对比

**AgentExecutor（线性，无法定制）**：

```typescript
const executor = new AgentExecutor({ agent, tools })
// 内部结构固定，无法插入自定义节点
```

**LangGraph（图结构，完全可定制）**：

```typescript
import { StateGraph, END } from "@langchain/langgraph"

const graph = new StateGraph({ channels: stateSchema })
  .addNode("router",     routerFn)        // 路由节点
  .addNode("codeGen",    codeGenFn)       // 代码生成节点
  .addNode("codeReview", codeReviewFn)    // 代码审查节点
  .addNode("general",    generalFn)       // 通用回答节点
  .addConditionalEdges("router", (state) => {
    return state.isCodeTask ? "codeGen" : "general"  // 条件分支
  })
  .addConditionalEdges("codeReview", (state) => {
    return state.approved ? END : "codeGen"  // 审查不通过 → 重新生成
  })
  .addEdge("codeGen", "codeReview")
  .addEdge("general", END)

const app = graph.compile()
await app.invoke({ input: "帮我写一个 TypeScript 排序函数" })
```

## 选择指南

| 场景特征 | 选择 |
|---------|------|
| 工具调用是线性的（A → B → C → 回答） | `AgentExecutor` |
| 需要条件分支（根据状态走不同路径） | `LangGraph` |
| 需要循环回路（失败重试、迭代优化） | `LangGraph` |
| 需要 Human-in-the-loop（等待人工确认） | `LangGraph` |
| 多个专门化 Agent 协作 | `LangGraph` |

## 一句话总结

**AgentExecutor = while 循环（固定形状）**
**LangGraph = 有向图（任意形状）**

当你的工作流超出"调工具直到完成"这个简单模式时，就是切换到 LangGraph 的时机。
