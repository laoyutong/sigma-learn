# Sub-Agent 状态管理与上下文传递 (State & Context)

## 为什么不能传完整的历史记录？

1. **Token 爆炸**：如果 Orchestrator 有 20 轮对话（约 10k tokens），调 5 个 Worker，每个都传全量历史，就是 50k tokens 的消耗，成本和延迟都会剧增。
2. **注意力分散（Lost in the Middle）**：Worker 只需要找场地，但上下文里塞满了"不要用红色"（这是给物料 Agent 看的）、"我喜欢幽默"（给文案 Agent 看的）。Worker 容易被无关信息干扰，产生幻觉。

## 最佳工程实践：全局状态对象 (Global State Object) + 状态提炼 (State Distillation)

你提到的"对用户的偏好内容作总结后传入"非常准确。在工程实现上，这被称为**显式状态管理**。我们不再传递原始的 `Message[]` 数组，而是维护一个结构化的 `State` 对象。

### 架构演进

#### 错误做法：透传 Message History
```typescript
// ❌ 灾难：把所有聊天记录塞给底层 Worker
async function callVenueAgent(history: Message[]) {
  return venueAgent.invoke({ messages: history }); 
}
```

#### 正确做法：LangGraph 的 State 模式
在 LangGraph 等现代框架中，我们会定义一个强类型的全局状态对象。Orchestrator 的职责之一，就是**把非结构化的聊天记录，提炼（Distill）成结构化的 State**。

```typescript
// 1. 定义全局状态 (Global State)
interface ProjectState {
  chat_history: Message[];        // 完整的原始记录（只留给 Orchestrator 看）
  
  // 下面是提炼出的结构化上下文（给 Worker 看的）
  extracted_constraints: {
    total_budget: number;
    preferred_style: string;
    forbidden_colors: string[];
    location: string;
  };
  
  // 任务执行状态
  venue_result: VenueInfo | null;
  guest_list: string[] | null;
}

// 2. Orchestrator 的提炼步骤 (Distillation)
async function orchestratorNode(state: ProjectState) {
  // 每次用户说话，Orchestrator 先更新结构化约束
  const newConstraints = await llm.invoke(`
    基于最新对话: ${state.chat_history.at(-1)}
    更新当前的约束条件: ${JSON.stringify(state.extracted_constraints)}
  `);
  
  return { extracted_constraints: newConstraints };
}

// 3. Worker 节点：只接收自己需要的切片 (State Slicing)
async function venueAgentNode(state: ProjectState) {
  // ✅ 最佳实践：按需传递 (Context Slicing)
  // VenueAgent 根本看不到 chat_history，也看不到 forbidden_colors
  const prompt = `
    你的任务是找场地。
    地点：${state.extracted_constraints.location}
    预算上限：${state.extracted_constraints.total_budget * 0.4} // Orchestrator 分配的比例
  `;
  
  const result = await runVenueAgent(prompt);
  
  // Worker 执行完，把结果写回全局 State
  return { venue_result: result }; 
}
```

## 核心原则：Context Slicing（上下文切片）

在多 Agent 系统中，信息传递必须遵循 **"最小权限/最小知识原则" (Principle of Least Knowledge)**：
1. **Orchestrator 拥有上帝视角**：它能看到完整的 `Message[]` 和全局 `State`。
2. **Worker 只有管中窥豹**：Orchestrator 必须把信息切片，只把 Worker 绝对需要的变量（如地点、它的子预算）通过 Prompt 传给它。

这不仅是为了省 Token，更是为了**提高底层 Agent 的指令遵循能力**（Prompt 越干净，Agent 执行越准）。
