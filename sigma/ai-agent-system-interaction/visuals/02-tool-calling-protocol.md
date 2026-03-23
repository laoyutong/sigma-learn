# 02. 工具协议的底层逻辑 (JSON Schema 到执行)

## 核心概念：文本到函数的映射

大模型本身并不执行代码，它只是一个高级的“文本生成器”。当我们说“大模型调用了工具”时，实际上发生的是：**大模型生成了一段符合特定格式的 JSON 文本，宿主机（Host）解析这段文本，并在本地执行对应的代码。**

## 真实世界的协议长什么样？

你猜测的结构：`type:"readFile"，payload:"~/.ssh/id_rsa"` 已经非常接近本质了！

在真实的 OpenAI API（以及目前业界的标准 Function Calling 协议中），大模型返回的 JSON 结构大致如下：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "readFile",
        "arguments": "{\"path\": \"~/.ssh/id_rsa\"}"
      }
    }
  ]
}
```

### 宿主机（Host）的执行逻辑

当你的 TypeScript 脚本（如 LangChain Agent）收到这个响应时，它会执行类似如下的逻辑：

```typescript
// 伪代码：Host 的执行引擎
const response = await llm.invoke(messages);

if (response.tool_calls) {
  for (const call of response.tool_calls) {
    const toolName = call.function.name; // "readFile"
    const args = JSON.parse(call.function.arguments); // { path: "~/.ssh/id_rsa" }
    
    // 找到本地对应的工具函数
    const tool = tools.find(t => t.name === toolName);
    
    // 真正执行本地代码！
    const result = await tool.invoke(args);
    
    // 将结果发回给大模型
    messages.push({ role: "tool", tool_call_id: call.id, content: result });
  }
}
```

## 关键思考：大模型怎么知道有这些工具？

大模型在云端，它不是神仙，不可能预知你的电脑上写了哪些 TypeScript 函数。
为了让大模型能够返回正确的 `name` 和 `arguments`，**宿主机必须在发送用户问题时，把本地工具的“说明书”（JSON Schema）一并发送给大模型。**

这个说明书包含了：
1. 工具的名称 (`name`)
2. 工具的描述 (`description` - 极其重要，大模型靠这个决定用哪个工具)
3. 参数的结构 (`parameters` - 告诉大模型需要传哪些字段)
