# 02. Human-in-the-loop 的异步中断与恢复

## 核心痛点：打破同步的执行流

在普通的 Agent 框架（如 LangChain）中，工具的执行通常是一个连续的 `async/await` 流程：
`LLM 返回 JSON -> Host 解析 -> await tool.invoke() -> 将结果发回 LLM`

但是，如果这是一个高危操作（比如 `execute_shell("npm publish")`），我们需要**暂停**这个流程，等待用户在前端 UI 上点击“允许”，然后再**恢复**执行。
由于用户的点击可能在几秒后，也可能在几小时后，我们不能用 `while(true)` 去阻塞 Node.js 的主线程。

## 实现方案：基于 Promise 控制反转 (Deferred Promise)

在 TypeScript 中，实现这种“暂停-等待外部事件-恢复”的经典模式是提取 Promise 的 `resolve` 和 `reject` 函数，将其存储在外部状态机中。

### 核心代码实现

```typescript
// 1. 定义一个全局或会话级别的状态管理器
class ToolExecutionManager {
  // 存储挂起的执行请求
  private pendingApprovals = new Map<string, {
    resolve: (value: boolean) => void;
    reject: (reason: any) => void;
  }>();

  /**
   * 2. 拦截器函数：在工具执行前调用
   * 它返回一个 Promise，但不会立刻 resolve。
   * 只有当用户在 UI 上点击后，这个 Promise 才会被 resolve。
   */
  async requestUserApproval(toolName: string, args: any): Promise<boolean> {
    const executionId = crypto.randomUUID();
    
    // 触发 UI 事件（比如通过 WebSocket 发送给前端，或者在终端打印）
    console.log(`[UI Event] Agent wants to run ${toolName} with args:`, args);
    console.log(`[UI Event] Please approve execution ID: ${executionId}`);

    // 返回一个被“控制反转”的 Promise
    return new Promise((resolve, reject) => {
      this.pendingApprovals.set(executionId, { resolve, reject });
    });
  }

  /**
   * 3. 恢复函数：由前端 UI 的回调或 API 路由调用
   */
  handleUserResponse(executionId: string, isApproved: boolean) {
    const pending = this.pendingApprovals.get(executionId);
    if (!pending) throw new Error("Execution ID not found or already handled");

    // 恢复被挂起的 Promise
    pending.resolve(isApproved);
    this.pendingApprovals.delete(executionId);
  }
}

// ==========================================
// 实际使用场景
// ==========================================
const manager = new ToolExecutionManager();

async function executeToolWithConsent(toolName: string, args: any) {
  const isHighRisk = ["execute_shell", "delete_file"].includes(toolName);

  if (isHighRisk) {
    // 代码在这里会“暂停”（await），直到用户调用 handleUserResponse
    const approved = await manager.requestUserApproval(toolName, args);
    
    if (!approved) {
      return "User denied the operation.";
    }
  }

  // 用户同意，或者不是高危操作，继续执行
  console.log(`Executing ${toolName}...`);
  return "Success";
}

// 模拟前端用户的异步操作
setTimeout(() => {
  // 假设用户看到了 UI 弹窗，并在 3 秒后点击了“允许”
  // 前端调用后端 API，路由最终调用了这个方法：
  // manager.handleUserResponse("the-uuid", true);
}, 3000);
```

## 架构扩展：持久化状态机
如果你的 Agent 运行在 Serverless 环境（如 AWS Lambda，每次请求完进程就销毁了），你不能把 `resolve` 存在内存的 Map 里。
这时候，你需要：
1. 将当前 Agent 的上下文（Messages 数组）存入数据库（如 Redis/PostgreSQL）。
2. 将状态标记为 `WAITING_FOR_USER`，然后直接结束当前 HTTP 请求。
3. 当用户点击“允许”后，触发一个新的 HTTP 请求，从数据库恢复上下文，并继续执行工具。
