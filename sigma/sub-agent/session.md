# Session: Sub-Agent 深度解析

## Learner Profile
- Level: intermediate-advanced（AI Agent 系列已全部掌握）
- Language: 中文
- Started: 2026-03-23

## Concept Map

| # | Concept | Prerequisites | Status | Score | Last Reviewed | Review Interval |
|---|---------|---------------|--------|-------|---------------|-----------------|
| 1 | 为什么需要 Sub-Agent（单 Agent 三大瓶颈） | - | mastered | 85% | 2026-03-23 | 1d |
| 2 | Orchestrator-Worker 核心模型 | 1 | mastered | 90% | 2026-03-23 | 1d |
| 3 | Agent 间通信机制（Tool Call vs Message Passing） | 2 | mastered | 90% | 2026-03-23 | 1d |
| 4 | 任务拆解与委派策略 | 2, 3 | mastered | 85% | 2026-03-23 | 1d |
| 5 | 并行 vs 串行执行与结果聚合 | 4 | mastered | 90% | 2026-03-23 | 1d |
| 6 | Sub-Agent 状态管理与上下文传递 | 5 | mastered | 90% | 2026-03-23 | 1d |
| 7 | 错误处理与 Sub-Agent 容错 | 6 | mastered | 90% | 2026-03-23 | 1d |
| 8 | 信任边界与权限隔离 | 7 | mastered | 95% | 2026-03-23 | 1d |
| 9 | 主流模式：Supervisor / Swarm / Hierarchical | 2, 4, 5 | mastered | 90% | 2026-03-23 | 1d |
| 10 | 工程实现（LangGraph / TypeScript 实战） | 全部 | mastered | 95% | 2026-03-23 | 1d |

## Misconceptions
| # | Concept | Misconception | Root Cause | Status | Counter-Example Used |
|---|---------|---------------|------------|--------|---------------------|
| - | - | - | - | - | - |

## Session Log
- [2026-03-23] Diagnosed: 全新领域，从零开始
- [2026-03-23] Concept 1: 诊断阶段 — 学员自主推导出并行性和上下文污染两个核心瓶颈，补充了上下文窗口耗尽，得分 70%
