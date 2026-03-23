# 03. MCP 架构深度解析 (Client-Server 模式)

## 痛点：N × M 的集成地狱

在没有 MCP 之前，如果我们要开发一个像 Cursor 这样的 AI 助手（Host），并且希望它能访问：
1. 本地文件系统
2. GitHub 的 Pull Requests
3. Slack 的聊天记录

那么 Cursor 的开发团队就必须在 Cursor 的源码里，用 TypeScript 分别写三套代码去对接这三个系统的 API，然后把它们的 JSON Schema 硬编码到 Cursor 里，发给大模型。

如果明天又出了一个新工具（比如 Notion），Cursor 团队就得加班改源码，发布新版本。
同时，如果另一个团队用 Python 写了一个叫 "AutoGPT" 的 Agent，他们也得把这四套工具用 Python 重新写一遍。

这就是经典的 **N 个 Agent 框架 × M 个数据源** 的集成地狱。

## MCP 的破局：引入 Client-Server 架构

Model Context Protocol (MCP) 的核心思想非常简单，但极其强大：**把“工具的实现”从 Host（Agent 代码）里剥离出去，变成一个独立的 Server。**

### 架构对比

**没有 MCP 之前（紧耦合）：**
```text
[ 大模型 ] <---> [ Cursor (内置了读文件、读GitHub、读Slack的代码) ] <---> [ 目标系统 ]
```

**有了 MCP 之后（松耦合）：**
```text
[ 大模型 ] <---> [ Cursor (MCP Client) ] 
                       |
                       | (标准 MCP 协议通信：stdio / SSE)
                       v
                 [ MCP Server (比如一个专门对接 GitHub 的进程) ] <---> [ GitHub API ]
                 [ MCP Server (比如一个专门读本地文件的进程) ] <---> [ 本地文件系统 ]
```

## MCP Server 的三大核心能力

一个标准的 MCP Server 可以向 Client 暴露三种东西：
1. **Resources (资源)**：像文件系统一样的只读数据（比如 `file:///users/didi/logs`）。
2. **Prompts (提示词模板)**：预设好的常用指令模板。
3. **Tools (工具)**：可以执行动作的函数（比如 `create_github_issue`），带有完整的 JSON Schema 描述。

## 运行流程

1. **发现阶段**：Cursor 启动时，连接到 GitHub MCP Server，问：“你有什么工具？” Server 回答：“我有 `create_issue`，它的 Schema 是...”。
2. **注册阶段**：Cursor 把这些 Schema 收集起来，连同用户的提问一起发给大模型。
3. **执行阶段**：大模型返回 `{"name": "create_issue", ...}`。Cursor 看到这个，知道它属于 GitHub Server，于是通过 MCP 协议把调用请求转发给 Server。
4. **返回阶段**：Server 执行完毕，把结果返回给 Cursor，Cursor 再发给大模型。
