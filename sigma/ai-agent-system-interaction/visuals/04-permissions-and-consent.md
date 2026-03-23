# 04. 权限管理与用户授权 (Consent & Scopes)

## 核心矛盾：能力的双刃剑

当 Agent 通过 MCP 或本地工具获得了操作系统的能力（如执行 Shell 命令、修改文件、操作数据库）后，它就变成了一把双刃剑。
- **好的一面**：它可以全自动帮你完成复杂的任务。
- **坏的一面**：如果大模型产生幻觉（Hallucination），或者遭遇了提示词注入攻击（Prompt Injection），它可能会执行毁灭性的操作（比如 `rm -rf /`，或者把你的私钥发给黑客）。

## 权限管理的三道防线

为了防止 Agent “暴走”，现代 Agent 系统通常在 Host 层设计了严格的权限管理机制。

### 1. 静态作用域 (Scopes / Allowed Directories)
在 Agent 启动时，就把它“框”在一个特定的范围内。
- **文件系统限制**：只允许读写特定的工作目录（Workspace），任何试图跳出该目录的操作（如 `cd ..` 或绝对路径 `/etc/passwd`）都会在 Host 层被直接拦截，根本不会发给操作系统。
- **网络限制**：只允许访问特定的白名单域名（如 `github.com`, `npmjs.org`），拦截对内网 IP（如 `127.0.0.1`）或未知域名的访问。

### 2. 动态用户授权 (User Consent / Human-in-the-loop)
对于高危操作，不能让 Agent 悄悄执行，必须把控制权交还给人类。
- **读操作（Read）**：通常可以静默执行（除非涉及敏感文件）。
- **写/执行操作（Write/Execute）**：当大模型返回了执行 Shell 命令的 JSON 时，Host 不会立刻执行，而是暂停（Pause），在 UI 上弹出一个确认框，展示即将执行的命令。只有用户点击“Approve（允许）”后，Host 才会真正调用系统 API。

### 3. 最小权限原则 (Principle of Least Privilege)
- 如果一个任务只需要读文件，绝不给它写文件的工具。
- 在 MCP 架构中，Client 可以选择性地只挂载特定的 Server，或者只启用 Server 暴露的部分 Tools。

## 典型工作流示例

```text
1. LLM 决定执行: {"name": "shell", "args": {"cmd": "npm publish"}}
2. Host 拦截: 这是一个高危操作！
3. Host 暂停执行，向 UI 发送请求: "Agent 想要执行 npm publish，是否允许？"
4. 用户在界面上点击 [允许]
5. Host 恢复执行，调用底层 shell 运行命令
6. Host 将命令输出结果返回给 LLM
```
