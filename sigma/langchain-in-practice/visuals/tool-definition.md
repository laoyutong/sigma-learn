# 工具定义：tool() + Zod Schema

## 基本结构

```typescript
import { tool } from "@langchain/core/tools"
import { z } from "zod"

const getWeather = tool(
  async ({ city }) => {          // 参数类型由 schema 自动推断
    return `${city} 今天晴，25°C`
  },
  {
    name: "get_weather",         // 模型用来"点名"调用
    description: "查询指定城市的当前天气", // 模型读它决定是否调用
    schema: z.object({
      city: z.string().describe("城市名称，例如：北京、上海"),
    }),
  }
)
```

## Zod → JSON Schema 自动转换

你写的 Zod：

```typescript
z.object({
  city: z.string().describe("城市名称"),
  date: z.string().optional().describe("日期，默认今天"),
})
```

模型收到的 JSON Schema：

```json
{
  "type": "object",
  "properties": {
    "city": { "type": "string", "description": "城市名称" },
    "date": { "type": "string", "description": "日期，默认今天" }
  },
  "required": ["city"]
}
```

> **注意**：`.optional()` 的参数不出现在 `required` 数组里，模型可以选择不传。

## Zod 方法影响速查

| 方法 | 影响 TS 类型 | 影响 JSON Schema | 影响运行时校验 |
|------|------------|----------------|------------|
| `z.string()` | ✓ `string` | ✓ `"type":"string"` | ✓ 必须是字符串 |
| `z.number()` | ✓ `number` | ✓ `"type":"number"` | ✓ 必须是数字 |
| `.optional()` | ✓ `T \| undefined` | ✓ 不在 required 里 | ✓ 可以不传 |
| `.describe("...")` | ✗ 无影响 | ✓ 加 description 字段 | ✗ 无影响 |
| `.default(val)` | ✓ 不含 undefined | ✓ 加 default 字段 | ✓ 未传时用默认值 |

> **重要**：`.describe()` 里写的文字只是给模型的说明，和代码行为无关。想要真正的默认值用 `.default(val)`，不是 `.describe("默认 xxx")`。

## 工具调用的完整生命周期

```
① 注册  model.bindTools([tool1, tool2])
        └─ Zod schema → JSON Schema → 随每次请求发给模型

② 决策  模型读 name + description → 决定调用哪个 → 返回 tool_calls

③ 执行  LangChain 用 Zod 校验参数 → 调用函数 → 结果 append 回 messages
```

## 两个 description 的区别

- **工具级 `description`**：模型决定**要不要**调用这个工具
- **参数级 `.describe()`**：模型决定**传什么值**给这个参数
