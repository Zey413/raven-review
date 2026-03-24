# Raven Review #005 — 极限压缩：能否更少？

**日期**: 2026-03-23
**项目**: https://github.com/nocoo/raven
**聚焦**: 在 Review #004 基础上寻找进一步压缩的可能性

---

## 一、Review #004 的 34 行是否还能压缩？

重新审视每一行：

### 1.1 translateAnthropicToolsToOpenAI 的改动（11 行）

```typescript
// 现在的实现（11 行新增）
const clientTools = anthropicTools.filter((t: any) => !t.type?.startsWith("web_search_"))
const translated = clientTools.map(...)
if (process.env.TAVILY_API_KEY && clientTools.length < anthropicTools.length) {
  translated.push({ type: "function", function: {
    name: "web_search",
    description: "Search the web for current information",
    parameters: { type: "object", properties: { query: { type: "string" } }, required: ["query"] },
  }})
}
return translated
```

**能压缩吗？** 可以压到 7 行：

```typescript
const hasSrvTool = anthropicTools.some((t: any) => t.type?.startsWith("web_search_"))
const tools = anthropicTools.filter((t: any) => !t.type?.startsWith("web_search_")).map(tool => ({
  type: "function" as const, function: { name: tool.name, description: tool.description, parameters: tool.input_schema },
}))
if (hasSrvTool && process.env.TAVILY_API_KEY) tools.push({ type: "function", function: {
  name: "web_search", description: "Search the web", parameters: { type: "object", properties: { query: { type: "string" } }, required: ["query"] },
}})
return tools
```

但可读性下降。**保持 11 行更好。**

### 1.2 handler.ts 的改动（22 行 + 3 行流式处理）

Tavily fetch + 结果格式化 + 二次请求 = 最少需要约 15 行。
加上 tool call 检测和 response 替换 = ~22 行。

**能进一步压缩吗？**

试试看：

```typescript
const ws = response.choices[0]?.message?.tool_calls?.find(tc => tc.function.name === "web_search")
if (ws && process.env.TAVILY_API_KEY) {
  const q = JSON.parse(ws.function.arguments).query
  const t = await fetch("https://api.tavily.com/search", { method: "POST", headers: { "Content-Type": "application/json", Authorization: `Bearer ${process.env.TAVILY_API_KEY}` }, body: JSON.stringify({ query: q, max_results: 5 }) }).then(r => r.json() as any)
  const txt = (t.results ?? []).map((r: any) => `${r.title}\n${r.url}\n${r.content}`).join("\n---\n")
  response = await createChatCompletions({ ...openAIPayload, stream: false, messages: [...openAIPayload.messages, { role: "assistant", content: response.choices[0].message.content, tool_calls: [ws] }, { role: "tool", tool_call_id: ws.id, content: txt }] }) as ChatCompletionResponse
}
```

**7 行！** 虽然每行很长，但逻辑完全正确。

### 1.3 极限版总行数

| 位置 | 行数 |
|------|------|
| `non-stream-translation.ts` | +7 行 |
| `handler.ts` — web search 后处理 | +7 行 |
| `handler.ts` — 强制非流式 | +2 行 |
| `.env.example` | +1 行 |
| **总计** | **+17 行** |

---

## 二、但 17 行不一定是最好的

极限压缩到 17 行牺牲了：
- 可读性
- 错误可追踪性
- 单行长度超标

**最佳平衡点仍然是 Review #004 的 ~37 行。**

---

## 三、还有没有完全不同的思路？

### 3.1 思路 A：Hono 中间件方式

能否写一个 Hono middleware 来处理，而不是改 handler？

```typescript
// 作为中间件插在 messages route 之前
app.use("/v1/messages", async (c, next) => {
  // 克隆请求体，修改 tools，设置回请求
  // ... 但 Hono 不方便修改请求体
  await next()
  // 修改响应... 但流式响应不好改
})
```

**结论：不行。** 中间件方式需要修改请求体和响应体，在 Hono 中比直接改 handler 更复杂。

### 3.2 思路 B：在 translateToOpenAI 层一次性解决

如果在 `translateToOpenAI` 中就记录"有 web search"标记，然后在 `translateToAnthropic` 中注入搜索结果？

**问题：** 搜索是异步的（要调 Tavily），而 translateToAnthropic 是同步函数。不匹配。

### 3.3 思路 C：新路由完全替代 messages handler

写一个全新的 handler 来替代 messages handler，先处理 web search，再调原 handler？

**问题：** 比直接改 handler 更复杂，因为需要重新处理请求/响应。

### 3.4 思路 D：MCP Server（不改 raven）

**不在 raven 中做任何改动**，而是：
1. 从 raven 的 `translateAnthropicToolsToOpenAI` 中只加 2 行过滤 server tool（防止翻译出错）
2. 配置一个 Tavily MCP Server 给 Claude Code

```json
// Claude Code 的 MCP 配置
{
  "mcpServers": {
    "tavily": {
      "command": "npx",
      "args": ["-y", "@anthropics/mcp-server-tavily"],
      "env": { "TAVILY_API_KEY": "tvly-xxx" }
    }
  }
}
```

**raven 改动量：2 行（过滤 server tool）。**

但这要求用户额外配置 MCP，不够"透明"。而且 Claude Code 的 `WebSearch` 内建能力不会自动使用 MCP 工具——它们是独立的。

---

## 四、最终答案

### 经过 5 轮 review，结论不变：

**最简实现 = 在 raven 的 2 个现有文件中内联修改，共 ~37 行。**

具体是：
1. `non-stream-translation.ts` 的 `translateAnthropicToolsToOpenAI` 函数：过滤 server tool + 注入 function tool（**+11 行**）
2. `handler.ts` 的 `handleCompletion` 函数：检测 web_search tool call → 调 Tavily → 二次请求（**+22 行**）
3. `handler.ts`：当有 web search 时强制 stream=false（**+3 行**）
4. `.env.example`：加 `TAVILY_API_KEY=`（**+1 行**）

**没有更简的方式了。**

### 为什么？

因为这个方案已经做到了：
- ✅ 零新文件
- ✅ 零新类型
- ✅ 零新依赖
- ✅ 复用所有现有基础设施（翻译层、Copilot 调用、日志、错误处理）
- ✅ 完全向后兼容（没 TAVILY_API_KEY 时行为完全不变）
- ✅ 会话历史天然兼容（用的是标准 tool_use/tool_result 格式）

唯一的妥协是强制非流式（+3 行），这可以在后续迭代中优化为真正的流式处理。

---

## 五、完整的最终代码清单

### 文件 1: `packages/proxy/src/routes/messages/non-stream-translation.ts`

**改动位置**: `translateAnthropicToolsToOpenAI` 函数（第 290-304 行）

**改动前**:
```typescript
function translateAnthropicToolsToOpenAI(
  anthropicTools: Array<AnthropicTool> | undefined,
): Array<Tool> | undefined {
  if (!anthropicTools) {
    return undefined
  }
  return anthropicTools.map((tool) => ({
    type: "function",
    function: {
      name: tool.name,
      description: tool.description,
      parameters: tool.input_schema,
    },
  }))
}
```

**改动后**:
```typescript
function translateAnthropicToolsToOpenAI(
  anthropicTools: Array<AnthropicTool> | undefined,
): Array<Tool> | undefined {
  if (!anthropicTools) {
    return undefined
  }
  // Strip server tools (web_search_20250305 etc.) — they have no input_schema
  const clientTools = anthropicTools.filter(
    (t: any) => !t.type?.startsWith("web_search_"),
  )
  const tools = clientTools.map((tool) => ({
    type: "function" as const,
    function: {
      name: tool.name,
      description: tool.description,
      parameters: tool.input_schema,
    },
  }))
  // When Tavily is configured, inject web_search as a regular function tool
  if (process.env.TAVILY_API_KEY && clientTools.length < anthropicTools.length) {
    tools.push({
      type: "function",
      function: {
        name: "web_search",
        description: "Search the web for current information",
        parameters: {
          type: "object",
          properties: { query: { type: "string" } },
          required: ["query"],
        },
      },
    })
  }
  return tools
}
```

### 文件 2: `packages/proxy/src/routes/messages/handler.ts`

**改动 A — 第 55 行后，translateToOpenAI 之后**:
```typescript
  const openAIPayload = translateToOpenAI(anthropicPayload)

  // Force non-streaming when web search tool is present (Tavily needs full response)
  const hasWebSearchTool = process.env.TAVILY_API_KEY
    && anthropicPayload.tools?.some((t: any) => t.type?.startsWith("web_search_"))
  if (hasWebSearchTool) openAIPayload.stream = false
```

**改动 B — 第 60 行，isNonStreaming 分支开始处**:
```typescript
    if (isNonStreaming(response)) {
      // Web search: intercept tool call, resolve via Tavily, re-request
      let finalResp = response
      const wsCall = response.choices[0]?.message?.tool_calls?.find(
        (tc) => tc.function.name === "web_search",
      )
      if (wsCall && process.env.TAVILY_API_KEY) {
        const { query } = JSON.parse(wsCall.function.arguments) as { query: string }
        const tavily = await fetch("https://api.tavily.com/search", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${process.env.TAVILY_API_KEY}`,
          },
          body: JSON.stringify({ query, max_results: 5 }),
        }).then((r) => r.json() as any)

        const resultText = (tavily.results ?? [])
          .map((r: any) => `### ${r.title}\n${r.url}\n${r.content}`)
          .join("\n\n")

        finalResp = (await createChatCompletions({
          ...openAIPayload,
          stream: false,
          messages: [
            ...openAIPayload.messages,
            { role: "assistant", content: response.choices[0].message.content, tool_calls: [wsCall] },
            { role: "tool", tool_call_id: wsCall.id, content: resultText },
          ],
        })) as ChatCompletionResponse
      }

      const anthropicResponse = translateToAnthropic(finalResp)
      // ... 后续用 finalResp 提取 metrics（而非 response）
```

### 文件 3: `packages/proxy/.env.example`

```
TAVILY_API_KEY=
```

---

*Review #005 by Claude Code — 极限压缩完成，结论收敛*
