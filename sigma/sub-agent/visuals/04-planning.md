# 任务拆解与委派策略 (Planning & Delegation)

## 为什么需要显式的 Planning 阶段？

当面对复杂、模糊、带有全局约束（如预算、时间线）的任务时，Orchestrator 不能直接开始调用 Worker（直接行动）。

如果直接行动，会导致：
1. **局部最优，全局崩溃**：比如场地 Agent 花光了所有预算。
2. **依赖倒置**：比如物料 Agent 先去印海报了，但场地 Agent 还没确定具体地址。
3. **遗漏任务**：想起来什么干什么，没有全盘视角。

因此，Orchestrator 必须被强制要求先进行 **Planning（计划）**。

## 架构实现：Plan-and-Solve 模式

在 Sub-Agent 架构中，处理复杂任务的标准范式是把 Orchestrator 拆分成两个阶段（或两个专门的 Agent）：

### 阶段 1：Planner (规划者)
**职责**：只动脑，不动手。输出一份执行蓝图。

**System Prompt 示例**：
```text
你是一个峰会项目总规划师。
你的任务不是执行，而是将大目标拆解为具体的子任务计划。
你必须考虑全局约束（如总预算 50 万）。

你有以下 Worker 可用：
- VenueAgent (场地)
- GuestAgent (嘉宾)
- MaterialAgent (物料)

请输出一个 JSON 格式的执行计划，包含：
1. 任务列表（必须有先后顺序）
2. 每个任务分配的 Worker
3. 传递给该 Worker 的具体约束（如分配的子预算）
```

**Planner 输出结果**：
```json
{
  "plan": [
    {
      "step": 1,
      "worker": "VenueAgent",
      "instruction": "寻找北京可容纳100人的场地",
      "constraints": { "budget_limit": 200000 } // 显式拆分预算！
    },
    {
      "step": 2,
      "worker": "GuestAgent",
      "instruction": "拟定互联网行业嘉宾邀请名单",
      "constraints": { "budget_limit": 150000 },
      "depends_on": [1] // 必须等场地定了再发邀请
    },
    {
      "step": 3,
      "worker": "MaterialAgent",
      "instruction": "设计并制作峰会物料",
      "constraints": { "budget_limit": 50000 },
      "depends_on": [1, 2] // 必须等场地和嘉宾名单确认
    }
  ],
  "reserve_budget": 100000 // 预留机动资金
}
```

### 阶段 2：Executor / Router (执行者/路由)
**职责**：严格按照 Planner 输出的蓝图，依次（或并行）调用对应的 Worker Tool。

**代码逻辑推演 (TypeScript)**：
```typescript
async function orchestrator(userRequest: string) {
  // 1. 强制规划阶段
  const plan = await plannerAgent.generatePlan(userRequest);
  
  const results = {};
  
  // 2. 执行阶段（根据依赖关系，决定是串行还是并行）
  for (const task of plan.tasks) {
    // 检查依赖是否完成
    if (task.depends_on.every(dep => results[dep])) {
      
      // 组装发给 Worker 的上下文：包含目标 + 约束 + 前置任务的结果
      const workerContext = `
        你的任务: ${task.instruction}
        你的硬性约束: ${JSON.stringify(task.constraints)}
        前置任务信息: ${getDependencyResults(task.depends_on, results)}
      `;
      
      // 动态路由到对应的 Worker Tool
      const workerTool = getToolByName(task.worker);
      results[task.step] = await workerTool.invoke(workerContext);
    }
  }
  
  return synthesize(results);
}
```

## 核心认知升级

在单 Agent 时代，ReAct 循环是：`Thought -> Action -> Observation`。
在 Sub-Agent 时代，面对复杂任务，循环变成了：`Plan -> Route -> Delegate -> Aggregate`。

**"预算分配"的本质，就是把「全局约束」降维打击成「局部约束」，然后硬编码到下发给 Worker 的 Prompt 里。** 这就是 Orchestrator 核心价值的体现。
