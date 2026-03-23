# AI Agent 系统交互学习路线图

- [x] **1. Agent 与系统的物理边界 (Host vs Target)**: 明确大模型、运行环境（Host）与目标系统（Target）的界限。
- [x] **2. 工具协议的底层逻辑 (JSON Schema 到执行)**: 探索 LLM 输出的文本如何安全地映射为本地的函数调用。
- [x] **3. MCP 架构深度解析 (Client-Server 模式)**: 深入理解 Model Context Protocol 的设计哲学（Resources, Prompts, Tools）。
- [x] **4. 权限管理与用户授权 (Consent & Scopes)**: Agent 替用户执行高危操作时的授权机制。
- [x] **5. 沙箱与安全隔离 (Docker, WASM, 提权防御)**: 当 Agent 拥有执行代码能力时，如何防止系统被破坏。
