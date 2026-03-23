# 错误处理与 Sub-Agent 容错 (Fault Tolerance)

在多 Agent 系统中，错误处理必须是分层的。底层 Worker 的崩溃绝不能导致整个 Orchestrator 进程挂掉。

## 容错的三个层级

### 1. Worker 内部容错 (Self-Healing)
这是最基础的防线，也就是你说的"捕获 search_hotel 的错误"。
Worker 自己的 ReAct 循环应该能处理普通的工具调用失败。

```typescript
// Worker 内部的重试逻辑
const searchHotelTool = tool(async ({ city }) => {
  try {
    return await api.search(city);
  } catch (e) {
    // 不要直接抛出异常，而是返回友好的错误信息给大模型
    // 让大模型自己决定是换个关键词，还是放弃
    return `搜索失败: ${e.message}。请尝试更换搜索条件或使用备用工具。`;
  }
});
```

### 2. Orchestrator 级容错：降级与重路由 (Fallback & Rerouting)
如果 Worker 彻底崩溃（比如 LLM 接口超时、输出格式严重破坏导致 JSON 解析失败），异常会抛给 Orchestrator。
这就是你说的"尝试换一个 Agent 来执行"。

```typescript
async function orchestrator(task) {
  try {
    // 尝试调用主 Worker
    return await venueAgent.invoke(task);
  } catch (e) {
    console.log("VenueAgent 崩溃，启动容错策略...");
    
    // 策略 A: 换个模型重试 (比如从快模型切到聪明模型)
    return await venueAgentSmart.invoke(task); 
    
    // 策略 B: 换个 Worker (比如场地找不到，找线上会议 Agent)
    return await onlineMeetingAgent.invoke(task);
  }
}
```

### 3. Orchestrator 级容错：人类介入 (Human-in-the-loop)
当所有自动化策略都失败时，系统不应该直接报错退出，而是应该**挂起当前状态**，请求人类介入。

```typescript
// LangGraph 中的中断模式 (Interrupt)
async function orchestratorNode(state) {
  if (state.venue_status === "FAILED_ALL_RETRIES") {
    // 挂起执行，等待人类提供一个场地
    const humanInput = await interrupt("场地Agent已崩溃，请人工指定一个场地：");
    return { venue_result: humanInput };
  }
}
```

## 核心原则：Fail-Safe vs Fail-Fast

在单 Agent 时代，我们倾向于 **Fail-Fast**（早报错早结束），因为只有一个执行流。
在 Sub-Agent 时代，我们必须采用 **Fail-Safe**（安全降级）。

**架构师的直觉**：把 Worker 当作不可靠的第三方微服务来对待。调用 Worker 时，必须包裹 `try-catch`，必须设置 `timeout`，必须准备 `fallback` 方案。
