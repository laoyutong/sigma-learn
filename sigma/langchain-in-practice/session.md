# Session: LangChain 实战
## Learner Profile
- Level: beginner (LangChain Code)
- Language: zh
- Started: 2026-03-21
- Background: 已掌握 AI Agent 理论 + 从零实现 Agent 的原理，理解 ReAct、Function Calling、状态机等核心概念
- Code Language: TypeScript
- Visual Preference: 每个概念生成配套的 visuals/*.md Markdown 总结文件（不需要 HTML），内容要完整详尽，不精简，包含完整代码示例、对比说明、流程图（用文字/ASCII）、选择指南等

## Concept Map
| # | Concept | Prerequisites | Status | Score | Last Reviewed | Review Interval |
|---|---------|---------------|--------|-------|---------------|-----------------|
| 1 | LangChain 架构与 LCEL 概览 | - | mastered | 100% | 2026-03-21 | 1d |
| 2 | ChatModel + PromptTemplate：第一条 Chain | 1 | mastered | 100% | 2026-03-21 | 1d |
| 3 | Runnable 协议：.invoke() / .stream() / .batch() | 2 | mastered | 100% | 2026-03-21 | 1d |
| 4 | 工具定义：@tool 装饰器与参数 Schema | 1, 2 | mastered | 95% | 2026-03-21 | 1d |
| 5 | 构建 Agent：create_tool_calling_agent + AgentExecutor | 3, 4 | mastered | 95% | 2026-03-21 | 1d |
| 6 | 对话记忆：RunnableWithMessageHistory | 3, 5 | mastered | 100% | 2026-03-21 | 1d |
| 7 | RAG 链：Embedding + VectorStore + Retriever | 3, 4 | mastered | 95% | 2026-03-21 | 1d |
| 8 | 迈向 LangGraph：AgentExecutor 的局限与图结构的优势 | 5, 6, 7 | mastered | 100% | 2026-03-21 | 1d |

## Misconceptions

| # | Concept | Misconception | Root Cause | Status | Counter-Example Used |
|---|---------|---------------|------------|--------|---------------------|

## Session Log
- [2026-03-21] Diagnosed level: beginner (LangChain Code), has strong conceptual AI Agent background
- [2026-03-21] SR review passed: ReAct while loop exit condition, Supervisor-Worker routing
- [2026-03-21] Concept 1: started tutoring
- [2026-03-21] Concept 1: mastered (100%). Learner correctly identified all three core components, understood Runnable protocol, discriminated chain variants by use case, and wrote correct LCEL chain from scratch.
- [2026-03-21] Concept 2: started tutoring
- [2026-03-21] Concept 2: mastered (100%). Learner correctly identified fromMessages vs fromTemplate difference, understood prompt.invoke() returns messages not AI response, and wrote MessagesPlaceholder template correctly.
- [2026-03-21] Concept 3: started tutoring
- [2026-03-21] Concept 3: mastered (100%). Learner correctly identified stream vs batch use cases, understood chunk type is determined by last component, and wrote batch call correctly.
- [2026-03-21] Concept 4: started tutoring
- [2026-03-21] Concept 4: mastered (95%). Learner correctly understood description vs .describe(), identified .optional() behavior, corrected misconception about .describe() not setting default values. Minor: used .optional().default() instead of just .default().
- [2026-03-21] Concept 5: started tutoring
- [2026-03-21] Concept 5: mastered (95%). Learner correctly identified agent=thinking/executor=execution loop separation. Needed correction on while loop count (3→4 rounds for 3 tool calls). Wrote createToolCallingAgent + AgentExecutor correctly.
- [2026-03-21] Concept 6: started tutoring
- [2026-03-21] Concept 6: mastered (100%). Learner correctly understood store saves full turn (Q+A), sessionId isolation, and wrote getMessageHistory pattern correctly.
- [2026-03-21] Concept 7: started tutoring
- [2026-03-21] Concept 7: mastered (95%). Learner correctly identified Embedding model as semantic converter, understood same-model constraint, grasped RAG flow. Minor: tried docs.join() instead of docs.map(d=>d.pageContent).join() — treated Document as string.
- [2026-03-21] Misconception logged (minor): Document[] treated as string[] in .join(). Root cause: unfamiliarity with Document type wrapper. Resolved immediately.
- [2026-03-21] Concept 8: started tutoring
- [2026-03-21] Concept 8: mastered (100%). Learner correctly identified AgentExecutor's fixed linear loop limitation, understood LangGraph's conditional edges and custom nodes. All 8 concepts completed!
