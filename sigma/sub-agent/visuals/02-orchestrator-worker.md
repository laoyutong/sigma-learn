# Orchestrator-Worker 核心模型

## 核心职责划分

```
┌─────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                         │
│                                                         │
│  职责1: 任务分解     职责2: 输出契约     职责3: 结果聚合  │
│  "分析三家车企"  →  定义返回 Schema  →  合并成报告       │
│                                                         │
│  ⚠️ 不干预: Worker 怎么查数据、调几个 API、中间步骤是啥  │
└──────────┬─────────────────────┬────────────────────────┘
           │ 下发任务 + Schema   │ 接收结构化结果
    ┌──────┴──────┐        ┌────┴──────────┐       ┌─────────────┐
    │  Worker A   │        │   Worker B    │       │  Worker C   │
    │  特斯拉分析  │        │  比亚迪分析   │       │ Rivian 分析 │
    │             │        │               │       │             │
    │ 自己的 ReAct│        │ 自己的 ReAct  │       │自己的 ReAct │
    │ 循环，自主  │        │ 循环，自主    │       │循环，自主   │
    │ 调工具查数据│        │ 调工具查数据  │       │调工具查数据 │
    └──────┬──────┘        └────┬──────────┘       └──────┬──────┘
           │                    │                          │
           └────────────────────┴──────────────────────────┘
                    全部返回同一个 Schema ↓

        { company: string, revenue: number, yoy_growth: number,
          gross_margin: number, key_risks: string[] }
```

## Output Contract 的重要性

### 没有 Output Contract（灾难）
```
Worker A 返回: "特斯拉 2024 年收入为 973 亿美元，同比增长 1%"
Worker B 返回: { "revenue": "¥6940亿", "growth": "增长35%" }
Worker C 返回: "Rivian 营收约 17 亿美元"

→ Orchestrator 无法自动合并，货币单位不统一，格式各异
```

### 有 Output Contract（正确）
```typescript
// Orchestrator 定义契约
interface CompanyAnalysis {
  company: string;
  revenue_usd_billion: number;  // 统一用十亿美元
  yoy_growth_percent: number;
  gross_margin_percent: number;
  key_risks: string[];          // 最多3条
}

// 发给每个 Worker 的 prompt 包含：
// "你的分析结果必须返回以下 JSON 格式：[CompanyAnalysis schema]"

// Worker A 返回:
{ company: "Tesla", revenue_usd_billion: 97.3, yoy_growth_percent: 1,
  gross_margin_percent: 17.9, key_risks: ["需求疲软", "价格战", "竞争加剧"] }

// Worker B 返回:
{ company: "BYD", revenue_usd_billion: 107.2, yoy_growth_percent: 28,
  gross_margin_percent: 20.1, key_risks: ["出海政策风险", "汇率", "毛利承压"] }

// → Orchestrator 可以直接 JSON.parse() 后对比合并 ✓
```

## 完整架构代码（TypeScript）

```typescript
// ============ Output Contract ============
interface CompanyAnalysis {
  company: string;
  revenue_usd_billion: number;
  yoy_growth_percent: number;
  gross_margin_percent: number;
  key_risks: string[];
}

// ============ Worker Agent ============
async function workerAgent(company: string): Promise<CompanyAnalysis> {
  const systemPrompt = `你是一个专业的财务分析师。
分析用户指定的公司，返回严格符合以下 JSON Schema 的结果，不要有任何其他文字：
${JSON.stringify(CompanyAnalysisSchema)}`;

  // Worker 有自己的 ReAct 循环：
  // 1. 调 search_tool 查财报
  // 2. 调 calculator_tool 算同比增长
  // 3. 自主决定查几次、怎么查
  // 4. 最终返回结构化结果
  const result = await runAgentLoop(systemPrompt, company);
  return JSON.parse(result) as CompanyAnalysis;
}

// ============ Orchestrator ============
async function orchestrator(task: string): Promise<string> {
  // Step 1: 任务分解（这里由 LLM 动态决定，或硬编码）
  const companies = ["Tesla", "BYD", "Rivian"];

  // Step 2: 并行派发（关键！不是串行）
  const results = await Promise.all(
    companies.map(company => workerAgent(company))
  );

  // Step 3: 结果聚合（因为有 Output Contract，聚合变得简单）
  return synthesizeReport(results);
}
```

## 关键原则总结

| 维度 | Orchestrator | Worker |
|------|-------------|--------|
| 知道什么 | 任务全貌、业务目标 | 单一子任务的执行细节 |
| 控制什么 | 输出 Schema、任务路由 | 自己的 ReAct 循环 |
| 不管什么 | Worker 内部怎么执行 | 其他 Worker 在做什么 |
| 失败时 | 重试/换 Worker/降级 | 上报错误给 Orchestrator |

## 与单 Agent 对比

| 维度 | 单 Agent | Orchestrator-Worker |
|------|----------|---------------------|
| 并行性 | ❌ 串行 | ✅ Promise.all 并行 |
| 上下文 | 全部挤在一个 context | 每个 Worker 独立 context |
| 窗口耗尽 | 容易触发 | 分散开，每个 Worker 只看自己的数据 |
| 容错 | 一步失败全盘重来 | 单个 Worker 失败可单独重试 |
| 专业化 | 一个 prompt 做所有事 | 每个 Worker 有专属 system prompt |
