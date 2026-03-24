# Raven Review #002 — 最精简 Web Search 拦截实现方案

**日期**: 2026-03-23
**项目**: https://github.com/nocoo/raven
**聚焦**: 找到最简单的代码实现来拦截 Claude Code 的 web search 并用 Tavily 替代

---

## 一、重新审视问题

经过 Review #001 的分析，我重新思考了一个关键问题：

**Raven 代理的是 Copilot API，不是 Anthropic API。**

Claude Code 发送 `web_search_20250305` server tool → Raven → 翻译成 OpenAI 格式 → Copilot API。

Copilot API 运行的是 Copilot 版 Claude 模型。这些模型**可能会也可能不会**主动调用 `web_search` function tool。但关键是：
- 我们可以在 tools 中注入 web_search 的 function 定义
- Copilot 的 Claude 模型**会**根据 tool 定义来决定是否调用
- 当模型返回 tool_calls 包含 web_search 时，proxy 拦截并调 Tavily

---

## 二、最终最精简方案：非流式先行

### 2.1 方案总览

```
Claude Code 请求:
  tools: [{ type: "web_search_20250305", name: "web_search" }, ...普通tools...]
       ↓
  Raven 预处理 (≈10行):
    - 检测并移除 web_search server tool
    - 注入等效的 function tool 定义
       ↓
  translateToOpenAI() → Copilot API (正常流程)
       ↓
  Copilot 响应: 可能包含 tool_calls[{name: "web_search", args: {query: "..."}}]
       ↓
  Raven 后处理 (≈30行):
    - 检测 web_search tool call
    - 调 Tavily API
    - 把 tool result 追加到 messages
    - 再次调 Copilot 拿最终回复
       ↓
  translateToAnthropic() → 返回给 Claude Code
```

### 2.2 为什么这是最简的

| 对比 | 改动量 | 复杂度 |
|------|--------|--------|
| 方案 A: 转发给真 Anthropic API | 需要额外 API key，改架构 | 🔴 高 |
| 方案 B: 模拟完整 server_tool_use 协议 | 需要造 encrypted_content | 🔴 高 |
| 方案 C: 在翻译层合成响应 | 需要改翻译层 | 🟡 中 |
| **方案 D: 在 handler 层拦截 (本方案)** | **只改 handler + 新增 1 个文件** | 🟢 **最低** |

---

## 三、具体代码实现

### 3.1 新增文件: `packages/proxy/src/routes/messages/web-search.ts`

**完整实现 — 仅 ~55 行核心代码：**

```typescript
import type { AnthropicMessagesPayload, AnthropicTool } from "./anthropic-types"
import type {
  ChatCompletionResponse,
  ChatCompletionsPayload,
} from "~/services/copilot/create-chat-completions"
import { logger } from "~/util/logger"

const TAVILY_API_KEY = process.env.TAVILY_API_KEY

const WEB_SEARCH_TOOL: AnthropicTool = {
  name: "web_search",
  description: "Search the web for up-to-date information.",
  input_schema: {
    type: "object",
    properties: { query: { type: "string" } },
    required: ["query"],
  },
}

// ---- 请求预处理 ----

export function stripServerTools(payload: AnthropicMessagesPayload): {
  payload: AnthropicMessagesPayload
  webSearchEnabled: boolean
} {
  if (!TAVILY_API_KEY || !payload.tools) {
    return { payload, webSearchEnabled: false }
  }

  const hasServerTool = payload.tools.some(
    (t: any) => typeof t.type === "string" && t.type.startsWith("web_search_")
  )
  if (!hasServerTool) return { payload, webSearchEnabled: false }

  const clientTools = payload.tools.filter(
    (t: any) => !t.type?.startsWith("web_search_")
  )
  clientTools.push(WEB_SEARCH_TOOL)

  return {
    payload: { ...payload, tools: clientTools },
    webSearchEnabled: true,
  }
}

// ---- Tavily 搜索 ----

async function tavilySearch(query: string): Promise<string> {
  const res = await fetch("https://api.tavily.com/search", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${TAVILY_API_KEY}`,
    },
    body: JSON.stringify({ query, max_results: 5 }),
  })
  const data = (await res.json()) as any
  if (!data.results?.length) return "No results found."

  return data.results
    .map((r: any) => `### ${r.title}\n${r.url}\n${r.content}`)
    .join("\n\n")
}

// ---- 响应后处理 (非流式) ----

export async function resolveWebSearch(
  response: ChatCompletionResponse,
  openAIPayload: ChatCompletionsPayload,
  callUpstream: (p: ChatCompletionsPayload) => Promise<any>,
): Promise<ChatCompletionResponse | null> {
  const toolCalls = response.choices[0]?.message?.tool_calls
  if (!toolCalls) return null

  const wsCall = toolCalls.find((tc) => tc.function.name === "web_search")
  if (!wsCall) return null

  const { query } = JSON.parse(wsCall.function.arguments) as { query: string }
  logger.info(`Web search via Tavily: "${query}"`)

  const searchResult = await tavilySearch(query)

  // 构建第二轮: 追加 assistant(tool_call) + tool(result)
  const followUp: ChatCompletionsPayload = {
    ...openAIPayload,
    stream: false, // 第二轮强制非流式
    messages: [
      ...openAIPayload.messages,
      {
        role: "assistant",
        content: response.choices[0].message.content,
        tool_calls: [wsCall],
      },
      {
        role: "tool",
        tool_call_id: wsCall.id,
        content: searchResult,
      },
    ],
  }

  return (await callUpstream(followUp)) as ChatCompletionResponse
}
```

### 3.2 修改文件: `handler.ts` — 仅改约 15 行

```diff
+ import { stripServerTools, resolveWebSearch } from "./web-search"

  export async function handleCompletion(c: Context) {
    // ... 现有代码 ...
    const anthropicPayload = await c.req.json<AnthropicMessagesPayload>()
+
+   // Web search: 剥离 server tool, 注入 function tool
+   const { payload: processedPayload, webSearchEnabled } =
+     stripServerTools(anthropicPayload)
+
-   const openAIPayload = translateToOpenAI(anthropicPayload)
+   const openAIPayload = translateToOpenAI(processedPayload)

    try {
      const response = await createChatCompletions(openAIPayload)

      if (isNonStreaming(response)) {
+       // Web search: 如果模型调了 web_search, 拦截并走 Tavily
+       let finalResponse = response
+       if (webSearchEnabled) {
+         const resolved = await resolveWebSearch(
+           response, openAIPayload, createChatCompletions
+         )
+         if (resolved) finalResponse = resolved
+       }
+
-       const anthropicResponse = translateToAnthropic(response)
+       const anthropicResponse = translateToAnthropic(finalResponse)
        // ... 其余不变 ...
      }
```

### 3.3 改动总结

| 文件 | 改动类型 | 行数 |
|------|---------|------|
| `web-search.ts` (新增) | 新文件 | ~55 行 |
| `handler.ts` | 添加 import + 预处理 + 后处理 | ~15 行 |
| `.env.example` | 添加 `TAVILY_API_KEY=` | 1 行 |
| **总计** | | **~71 行** |

---

## 四、流式处理方案（第二阶段）

非流式版本工作后，流式版本的思路：

### 4.1 缓冲策略

```
流式场景下:
1. 开始正常流式转发 Copilot 的 SSE 响应
2. 如果检测到 finish_reason=tool_calls 且包含 web_search:
   a. 不发 message_delta/message_stop 给客户端
   b. 缓冲 tool_call 信息
   c. 调 Tavily
   d. 发起第二轮请求 (流式)
   e. 继续流式转发第二轮的响应
3. 如果没有 web_search tool call，正常流式转发
```

### 4.2 复杂度评估

流式拦截的主要挑战：
- 需要缓冲 tool_call 的 arguments（因为是分块到达的）
- 需要在 finish_reason 到来时决策
- 需要拼接两段流为一个连续的 Anthropic 流
- content_block_index 需要跨两段流正确编号

**建议**: 先实现非流式版本验证可行性，再做流式版本。实际上，web search 场景中 Claude Code 可能本身就用非流式调用（`stream: false`），因为 web search 通常是 sub-agent 调用。

### 4.3 最简流式方案（如果必须）

```typescript
// 在 handler.ts 的流式分支中:
// 1. 不直接流式返回，先收集完整响应
// 2. 检查是否有 web_search tool call
// 3. 如果有: 调 Tavily → 第二轮 → 再流式返回
// 4. 如果没有: 原样翻译返回（但已失去流式优势）

// 更好的方案: 使用 TransformStream 管道
// 让大部分内容正常流过，只在需要时缓冲
```

---

## 五、对比已有实现 (CCAG)

[CCAG (Claude Code AWS Gateway)](https://lib.rs/crates/ccag-cli) 是一个 Rust 实现的类似代理，它的 web search 拦截方式：

| 方面 | CCAG | Raven 方案 |
|------|------|-----------|
| 语言 | Rust | TypeScript |
| 代码量 | ~200 行 Rust | ~71 行 TS |
| Server tool 处理 | 剥离 + 注入 | 同 |
| 搜索引擎 | Tavily / DuckDuckGo / Serper | Tavily |
| 流式处理 | 完整实现 | 先非流式 |
| 集成位置 | 独立代理 | 嵌入 raven |

**我们的方案更精简**，因为：
1. 复用 raven 现有的翻译层和 Copilot 调用基础设施
2. 只需要在 handler 层加一层拦截
3. TypeScript 本身比 Rust 更紧凑

---

## 六、进一步精简的可能性

### 6.1 极限精简版（~40 行）

如果不需要模块化，可以直接在 `handler.ts` 中内联所有逻辑：

```typescript
// 在 handler.ts 的 translateToOpenAI 之前:
const tavilyKey = process.env.TAVILY_API_KEY
if (tavilyKey && anthropicPayload.tools?.some((t: any) => t.type?.startsWith("web_search_"))) {
  anthropicPayload.tools = anthropicPayload.tools
    .filter((t: any) => !t.type?.startsWith("web_search_"))
    .concat([{
      name: "web_search",
      description: "Search the web",
      input_schema: { type: "object", properties: { query: { type: "string" } }, required: ["query"] }
    }])
}

// 在 isNonStreaming 分支中, translateToAnthropic 之前:
const wsCall = response.choices[0]?.message?.tool_calls?.find(tc => tc.function.name === "web_search")
if (wsCall && tavilyKey) {
  const { query } = JSON.parse(wsCall.function.arguments)
  const tavilyRes = await fetch("https://api.tavily.com/search", {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${tavilyKey}` },
    body: JSON.stringify({ query, max_results: 5 })
  }).then(r => r.json())
  const resultText = tavilyRes.results.map(r => `${r.title}\n${r.url}\n${r.content}`).join("\n---\n")
  response = await createChatCompletions({
    ...openAIPayload, stream: false,
    messages: [...openAIPayload.messages, response.choices[0].message, { role: "tool", tool_call_id: wsCall.id, content: resultText }]
  })
}
```

**这仅 ~20 行内联代码。** 但可维护性较差。

### 6.2 推荐的平衡点

独立文件 `web-search.ts`（~55 行）+ handler 修改（~15 行）= **~71 行总改动**

这是**可维护性和精简性的最佳平衡**。

---

## 七、待验证的问题

1. **Copilot 的 Claude 模型是否会调用自定义 web_search tool？** — 需要实测
2. **Claude Code 对 web_search 结果格式的期望** — 普通 text 内容 vs encrypted_content
3. **流式场景的比例** — 如果大部分 web search 是非流式，先做非流式够用
4. **多次 web search** — 模型可能一次调多个 web_search，需要处理并行搜索
5. **tool_calls 中混合 web_search 和其他 tool** — 需要只拦截 web_search，其他正常返回

---

## 八、结论

**最精简实现 = 71 行 TypeScript 代码**（新增 1 文件 + 修改 1 文件 + 1 行 env）

核心逻辑仅 3 步：
1. 请求预处理：server tool → function tool（10 行）
2. Tavily 搜索：调 API 拿结果（15 行）
3. 响应后处理：检测 tool call → 搜索 → 二次请求（30 行）

---

*Review #002 by Claude Code — 持续优化中*
