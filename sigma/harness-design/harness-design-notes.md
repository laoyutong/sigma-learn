# Harness Design (脚手架设计) 学习笔记

**来源**: [Harness design for long-running application development | Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps)
**学习日期**: 2026-03-30

---

## 核心痛点：为什么天真的实现会失败？
当让 LLM Agent 执行长周期、复杂的编码任务时，仅仅依赖一个足够大的上下文窗口（Context Window）是不够的。主要原因在于：
* **上下文焦虑 (Context Anxiety)**：随着上下文被填满，模型会降低关注度，草率收尾，导致出现预期之外的代码实现。
* **自我评估的盲区**：模型在评估自己的工作时，往往倾向于盲目赞美，无法客观发现缺陷。

---

## 核心机制与架构设计

### 1. Context Engineering & Resets (上下文工程与重置)
* **概念**：与其不断总结压缩（Compaction）历史对话，不如在关键节点**彻底重置（Reset）**上下文，启动一个全新的 Agent 实例。
* **优势**：彻底重置后，可用的上下文窗口更大，同时清除了历史对话中的干扰信息和路径依赖。新 Agent 只需要通过“交接棒”（Structured Artifacts，包含工作目的、旧 Agent 的工作内容等）即可完美接手。

### 2. Task Decomposition / Sprints (任务拆解与冲刺)
* **概念**：将宏大的产品规格（Spec）拆解为一个个独立的冲刺阶段（Sprints），每次只专注一个功能。
* **优势**：
  * **修复成本低**：状态被隔离。如果某个 Sprint 失败，只需回滚到上一个 Sprint 的结束状态，不影响其他模块。
  * **职责清晰**：Planner（规划者）只需关注高层功能定义，不需要把技术细节写死，避免了上下文占用过高和早期决策失误导致的级联错误。

### 3. The GAN Pattern: Generator vs Evaluator (生成与对抗模式)
* **概念**：引入多智能体协作，将“写代码”和“检查代码”分离。
  * **Generator (生成者)**：负责编写代码。
  * **Evaluator (评估者/QA)**：负责审查和测试代码。
* **优势**：打破了单个 Agent “自卖自夸”的盲区，通过外部评价推动代码质量的跃迁。

### 4. Verification & Playwright (客观验证与工具赋能)
* **概念**：Evaluator 不能只做静态代码审查，必须配备真实的测试工具（如 Playwright MCP）。
* **实践**：Evaluator 像真实用户一样打开浏览器、点击按钮、测试边缘情况（例如：密码错误时是否有提示交互）。如果发现问题，会针对性地降低 Functionality（功能性）等维度的分数，并给出明确反馈。

### 5. Sprint Contracts (冲刺契约)
* **概念**：在 Generator 写任何代码之前，它必须先和 Evaluator 进行“谈判”，就“什么是完成”达成一份颗粒度极高的测试契约。任何一条标准未满足，整个 Sprint 判定为失败。
* **优势**：实现了**延迟决策（Late Binding）**。高层次的 Spec 配合颗粒度的契约，避免了初期把所有工作量和细节堆积在 Planner 里导致的上下文爆炸。

### 6. Harness Evolution (脚手架的演进与瘦身)
* **概念**：随着底层模型能力增强（如 Claude 4.5 Opus），模型不再容易产生上下文焦虑。此时可以逐步移除强制的 Sprint 拆分和 Context Resets。
* **保留的核心**：即使模型变强，**Planner** 和 **Evaluator** 依然不可或缺。
  * **Planner**：负责任务拆分和宏观方向，使执行更加可控，防止范围缩水。
  * **Evaluator**：负责功能验证，守住质量底线。
* **结论**：随着模型变强，脚手架不会消失，而是会“移动”——从弥补短期记忆缺陷，转向提供高层认知和客观验证。

---

## 💡 总结：Harness Design 的本质
Harness Design 不是为了写更多的代码，而是为了**管理模型的认知边界**。它通过以下机制将 LLM 从一个“单次回答机”变成了一个“长效工程系统”：
1. **状态隔离** (Resets & Sprints)
2. **职责分离** (Planner vs Generator vs Evaluator)
3. **客观约束** (Sprint Contracts & Playwright)