# RAG 链：Embedding + VectorStore + Retriever

> RAG（Retrieval-Augmented Generation）让 Agent 能检索私有文档，突破模型训练截止日期的限制

## 核心思想

模型的知识是静态的（截止训练日期），无法回答私有文档内容。RAG 的解法：**检索相关文档 → 塞进 Prompt → 让模型基于文档回答**。

## Embedding：文字 → 向量的转换

Embedding 模型是一个训练好的神经网络，把"语义相似"的文本映射成"距离相近"的向量：

```
"怎么写泛型函数？"    → [0.21, -0.85, 0.38, ...] (1536维)
"TypeScript 泛型入门" → [0.23, -0.87, 0.41, ...] ← 距离很近 ✓ 语义相似

"怎么写泛型函数？"    → [0.21, -0.85, 0.38, ...]
"明天天气怎么样？"    → [-0.72, 0.33, -0.91, ...] ← 距离很远 ✓ 语义不相关
```

> **重要约束**：建库和查询必须用同一个 Embedding 模型。不同模型的"坐标系"完全不同，混用会导致向量无法比较。

## 完整流程

### 离线阶段：建库

```
1000 篇文章（原始文本）
  ↓ Embedding 模型（每篇 → 向量）
VectorStore（存储：原文 + 对应向量）
```

### 在线阶段：查询

```
用户问题（字符串）
  ↓ 同一个 Embedding 模型（问题 → 向量）
  ↓ 在 VectorStore 里找距离最近的 N 个向量
Top K 相关文档（Document[]）
  ↓ 格式化成字符串（context）
  ↓ 塞进 Prompt（question + context）
  ↓ 大模型生成答案
最终回答
```

## LangChain 代码实现

### 三大组件

```typescript
import { OpenAIEmbeddings } from "@langchain/openai"
import { MemoryVectorStore } from "langchain/vectorstores/memory"
import { Document } from "@langchain/core/documents"

// ① Embedding 模型（文字 → 向量）
const embeddings = new OpenAIEmbeddings()

// ② VectorStore（存向量 + 原文）
const vectorStore = await MemoryVectorStore.fromDocuments(
  [
    new Document({ pageContent: "TypeScript 泛型入门..." }),
    new Document({ pageContent: "React Hooks 详解..." }),
    // ...1000 篇
  ],
  embeddings // 用这个模型把每篇文章转成向量
)

// ③ Retriever（根据问题检索，返回 Top K 文档）
const retriever = vectorStore.asRetriever({ k: 3 })

// retriever 类型：Runnable<string, Document[]>
const docs = await retriever.invoke("怎么写泛型函数？")
// docs: Document[]（最相关的 3 篇文章原文）
```

### 完整 RAG Chain

```typescript
import { RunnablePassthrough } from "@langchain/core/runnables"
import { StringOutputParser } from "@langchain/core/output_parsers"

const prompt = ChatPromptTemplate.fromMessages([
  ["system", "根据以下上下文回答问题，如果不知道就说不知道。\n\n上下文：{context}"],
  ["human", "{question}"],
])

// Document[] → 拼接成字符串（注意：Document 不是 string，要取 .pageContent）
const formatDocs = (docs: Document[]) =>
  docs.map(d => d.pageContent).join("\n---\n")

const ragChain = RunnablePassthrough.assign({
  context: retriever.pipe(formatDocs), // 问题 → 检索 → 格式化文本
}).pipe(prompt)
  .pipe(model)
  .pipe(new StringOutputParser())

const answer = await ragChain.invoke({ question: "怎么写泛型函数？" })
```

### RunnablePassthrough.assign() 的作用

```
输入：{ question: "怎么写泛型函数？" }
  ↓ RunnablePassthrough.assign({ context: retriever.pipe(formatDocs) })
输出：{
  question: "怎么写泛型函数？",  ← 原样保留
  context: "TypeScript 泛型入门...\n---\nTypeScript 高级类型..." ← 追加
}
  ↓ prompt（能同时访问 question 和 context）
```

## 数据类型流

```
string（用户问题）
  ↓ retriever
Document[]（相关文档对象，原文在 .pageContent 里）
  ↓ formatDocs（map + join）
string（context，拼接好的上下文）
  ↓ prompt（和 question 合并格式化）
BaseMessage[]
  ↓ model
AIMessage
  ↓ StringOutputParser
string（最终回答）
```

## 常见错误

| 错误 | 原因 | 修复 |
|------|------|------|
| `docs.join('\n---\n')` | `Document` 不是 `string`，会变成 `[object Object]...` | `docs.map(d => d.pageContent).join('\n---\n')` |
| 建库和查询用不同 Embedding 模型 | 向量空间不同，距离计算无意义 | 统一使用同一个 `embeddings` 实例 |
| 把 `retriever.invoke()` 的结果直接塞进 prompt | 缺少格式化步骤，Document 对象无法放进字符串模板 | 先 `.pipe(formatDocs)` 转成字符串 |

## 选择指南：何时用 RAG

| 场景 | 方案 |
|------|------|
| 回答公开知识、通用问题 | 直接用 LLM，无需 RAG |
| 回答私有文档、内部知识库 | RAG |
| 需要引用最新信息（超出训练截止日期） | RAG 或 Tool（搜索 API） |
| 文档量大（>100篇），需精准检索 | RAG + 分块（chunking）策略 |
