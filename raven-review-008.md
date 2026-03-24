# Raven Review #008 — 源码级发现：Claude Code WebSearch 的真实机制

**日期**: 2026-03-24
**项目**: https://github.com/nocoo/raven
**聚焦**: 直接读 Claude Code 安装包源码 (cli.js)，找到 web search 的精确实现

---

## 一、颠覆性发现 🔴

通过直接阅读 `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js` (v2.1.81) 中的混淆源码，**发现了方案的关键前提假设需要修正**：

### 1.1 WebSearch 是 Claude Code 的 "Deferred Tool"

Claude Code 的 WebSearch **不是直接放在主对话的 tools 数组里的**。它的实现机制是：

```javascript
// Claude Code 源码中的关键结构
NR8 = {
  name: cT,                    // "WebSearch" (内部常量名)
  shouldDefer: true,           // ⚡ 关键：这是一个 deferred tool

  isEnabled() {
    let A = QA()               // 获取 API provider 类型
    let q = KK()               // 获取模型名
    if (A === "firstParty") return true    // Anthropic 直连：启用
    if (A === "vertex") return q.includes("claude-opus-4") || ...  // Vertex：部分模型
    if (A === "foundry") return true       // Foundry：启用
    return false               // ⚡ 其他 provider（包括自定义 base URL）：禁用？
  },

  async call(A, q, K, _, Y) {
    // ⚡ 关键发现：call() 方法发起一个 **独立的 API 请求**
    let $ = mB_(A)  // 创建 {type: "web_search_20250305", name: "web_search", max_uses: 8}

    let J = aZ6({
      messages: [userMessage],
      systemPrompt: "You are an assistant for performing a web search tool use",
      tools: [],                         // 空！普通 tools 为空
      extraToolSchemas: [$],             // ⚡ web_search 通过 extraToolSchemas 注入
      querySource: "web_search_tool",    // 标记为 web_search 来源
      // ... 其他配置
    })

    // 流式读取响应中的 server_tool_use 和 web_search_tool_result
    for await (let N of J) {
      // 解析 server_tool_use → 提取 query
      // 解析 web_search_tool_result → 提取搜索结果
    }

    return { data: BB_(Z, w, v) }  // 返回解析后的搜索结果
  }
}
```

### 1.2 工作流程解析

```
Claude Code 主循环：
  1. 模型决定调用 "WebSearch" tool (这是 Claude Code 内部的 tool, 不是 API tool)
  2. Claude Code 客户端拦截这个 tool_use
  3. 调用 NR8.call() → 发起一个 **全新的、独立的** API 请求
     - system prompt: "You are an assistant for performing a web search tool use"
     - messages: 只有一条用户消息 "Perform a web search for the query: xxx"
     - tools: []（空）
     - extraToolSchemas: [{type: "web_search_20250305", name: "web_search", max_uses: 8}]
  4. Anthropic API 返回 server_tool_use + web_search_tool_result
  5. Claude Code 解析结果，提取 title + url
  6. 返回给主循环作为 tool_result
```

### 1.3 这意味着什么？

**web_search_20250305 不是在主对话请求的 tools 数组里！**

它是通过 `extraToolSchemas` 在一个**独立的子请求**中发送的。主对话的 tools 数组只包含普通的 client tool（Read, Write, Bash, etc.）。

---

## 二、对 Raven 方案的影响

### 2.1 之前方案的问题

Review #004-#007 的方案假设：
> "Claude Code 把 web_search_20250305 放在主请求的 tools 数组里发给 raven"

**这个假设是错误的。** 实际流程：

```
主请求：POST /v1/messages
  tools: [Read, Write, Bash, Grep, ...]   ← 没有 web_search！

WebSearch 子请求：POST /v1/messages（独立请求！）
  tools: []
  extraToolSchemas: [{type: "web_search_20250305", ...}]
  messages: [{role: "user", content: "Perform a web search for: xxx"}]
```

### 2.2 但 isEnabled() 的判断

```javascript
isEnabled() {
  let A = QA()  // provider 类型
  if (A === "firstParty") return true
  if (A === "vertex") return ...
  if (A === "foundry") return true
  return false  // ⚡ 自定义 base URL 可能返回 false
}
```

`QA()` 函数检测 API provider 类型。当 `ANTHROPIC_BASE_URL=http://localhost:7033`（raven）时，Claude Code 可能认为这是 "firstParty"（因为有 ANTHROPIC_API_KEY），也可能认为这是 "custom"。

**如果返回 "firstParty"**：WebSearch 启用 → 发独立子请求到 raven → **raven 需要处理这个子请求**
**如果返回 false**：WebSearch 禁用 → 永远不会发搜索请求 → 方案无用

### 2.3 关键问题：QA() 怎么判断 provider？

需要进一步确认 QA() 的实现。但根据 Claude Code 的行为，设置 `ANTHROPIC_BASE_URL` + `ANTHROPIC_API_KEY` 时，它通常认为这是 "firstParty"。

---

## 三、修正后的拦截方案

### 3.1 新的请求特征

WebSearch 子请求的特征（与主请求不同）：
1. `messages` 只有 1 条：`"Perform a web search for the query: xxx"`
2. `tools` 为空数组 `[]`
3. 有 `extraToolSchemas` 字段包含 `{type: "web_search_20250305"}`
4. `systemPrompt` 包含 "You are an assistant for performing a web search tool use"
5. `querySource` 标记为 `"web_search_tool"`（可能在请求头或 metadata 中）

### 3.2 修正后的拦截策略

**不再在翻译层做！直接在 handler 层拦截整个请求！**

```
Claude Code → POST /v1/messages (WebSearch 子请求)
    ↓
Raven handler: 检测到这是 web search 子请求（通过 extraToolSchemas 或消息内容）
    ↓
不转发给 Copilot！直接调 Tavily → 构造 Anthropic 格式的响应 → 返回
```

### 3.3 这其实**更简单**！

因为不需要：
- ❌ 修改翻译层
- ❌ 拦截 tool_calls
- ❌ 二次请求 Copilot
- ❌ 处理流式/非流式切换

只需要：
- ✅ 检测请求是否包含 web_search server tool
- ✅ 提取搜索 query
- ✅ 调 Tavily
- ✅ 构造一个假的 Anthropic 响应返回

---

## 四、全新的极简实现

### 4.1 方案概述

在 `handler.ts` 的最开头，检测到 web search 子请求时，**完全短路**——不走翻译和 Copilot，直接返回搜索结果。

```typescript
// handler.ts — 在 translateToOpenAI 之前
const webSearchTool = anthropicPayload.tools?.find(
  (t: any) => t.type?.startsWith("web_search_")
) as any

if (webSearchTool && process.env.TAVILY_API_KEY) {
  // 这是 Claude Code 的 WebSearch 子请求 — 直接用 Tavily 处理
  const queryMatch = anthropicPayload.messages[0]?.content?.toString()
    .match(/search.*?(?:for|query):?\s*(.+)/i)
  const query = queryMatch?.[1]?.trim() || anthropicPayload.messages[0]?.content?.toString() || ""

  let searchResults: any[] = []
  try {
    const tavily = await fetch("https://api.tavily.com/search", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${process.env.TAVILY_API_KEY}`,
      },
      body: JSON.stringify({ query, max_results: 5 }),
    }).then(r => r.json() as any)
    searchResults = tavily.results ?? []
  } catch {}

  // 构造 Anthropic 格式的响应（模拟 server_tool_use + web_search_tool_result）
  const srvToolId = `srvtoolu_${crypto.randomUUID().replace(/-/g, "").slice(0, 24)}`
  return c.json({
    id: `msg_${crypto.randomUUID().replace(/-/g, "").slice(0, 20)}`,
    type: "message",
    role: "assistant",
    model: anthropicPayload.model,
    content: [
      { type: "text", text: `Searching for: ${query}` },
      {
        type: "server_tool_use",
        id: srvToolId,
        name: "web_search",
        input: { query },
      },
      {
        type: "web_search_tool_result",
        tool_use_id: srvToolId,
        content: searchResults.map(r => ({
          type: "web_search_result",
          url: r.url,
          title: r.title,
          encrypted_content: Buffer.from(r.content || "").toString("base64"),
          page_age: null,
        })),
      },
      {
        type: "text",
        text: searchResults.map(r => `${r.title}: ${r.content}`).join("\n\n"),
      },
    ],
    stop_reason: "end_turn",
    stop_sequence: null,
    usage: { input_tokens: 100, output_tokens: 200 },
  })
}
```

### 4.2 等等——encrypted_content 的问题

Claude Code 的 `BB_` 函数解析响应时：
```javascript
// Claude Code 只提取 title 和 url，忽略 encrypted_content！
let O = w.content.map(($) => ({ title: $.title, url: $.url }))
```

**Claude Code 根本不用 encrypted_content！** 它只取 title 和 url。所以我们不需要真正的加密内容。

但是...Claude Code 的主模型需要搜索结果的**文本内容**来生成回答。它是怎么获得的？

回看 `mapToolResultToToolResultBlockParam`：
```javascript
mapToolResultToToolResultBlockParam(A, q) {
  let { query: K, results: _ } = A
  let Y = `Web search results for query: "${K}"\n\n`
  // 把 results 格式化为文本给主模型
}
```

所以 Claude Code 把搜索结果**格式化为纯文本**作为 tool_result 传回主对话。它不依赖 encrypted_content！

### 4.3 简化：不需要模拟完整的 server_tool_use 协议

既然 Claude Code 的 `call()` 方法自己解析响应并提取 title/url，那我们只需要返回：
- `server_tool_use` block（Claude Code 会跳过这个）
- `web_search_tool_result` block（Claude Code 从这里提取 title + url）

**但有一个更大的问题：**Claude Code 的 `call()` 方法通过**流式**读取响应。它期望收到 SSE stream events，不是 JSON 响应！

回看代码：
```javascript
let J = aZ6({...})  // aZ6 返回一个 AsyncGenerator
for await (let N of J) {
  // 逐个处理 stream events
  if (N.type === "stream_event" && N.event?.type === "content_block_start") {
    // 处理 server_tool_use
    // 处理 web_search_tool_result
  }
}
```

所以 Claude Code **期望流式响应**。我们需要返回 SSE 格式。

### 4.4 但是...stream 参数呢？

`aZ6` 的调用中，stream 参数取决于 Claude Code 的配置。在实际请求中，`stream: true` 是默认行为。

让我重新想...

---

## 五、最终修正方案

### 5.1 关键认识

不管是流式还是非流式，raven 的 handler 已经能处理两种情况。问题是：
1. 请求到达 raven 时，有 `web_search_20250305` 在 tools（或 extraToolSchemas）中
2. Raven 当前会把它翻译成 OpenAI 格式发给 Copilot → Copilot 不认识 → 失败

### 5.2 最简方案（保持 handler 短路思路）

**在 handler 开头检测 web search 请求 → 直接调 Tavily → 返回 Anthropic 流式响应**

但这要求我们构造完整的 SSE stream，比较复杂。

### 5.3 退一步：最实际的方案

考虑到 extraToolSchemas 在 Anthropic SDK 中最终会被合并到 tools 数组里发送给 API，raven 收到的请求 **tools 数组里确实会包含 `{type: "web_search_20250305"}`**。

所以 Review #004-#007 的方案前提**依然成立**！只是需要理解的是：
- 主对话请求的 tools 里**没有** web_search
- WebSearch **子请求**的 tools 里**有** web_search_20250305

两种请求都会经过 raven 的 `/v1/messages` handler。

### 5.4 综合两种情况的统一方案

```
情况 1: 主对话请求（tools 里没有 web_search）→ 正常翻译转发
情况 2: WebSearch 子请求（tools 里有 web_search_20250305）→ 拦截 + Tavily
```

**Review #004-#007 的方案完全正确！** 它处理的就是情况 2。

唯一需要修正的是：**不需要注入 function tool 让 Copilot 的模型去调用 web_search**。而是应该**直接短路整个请求**，因为这个子请求的唯一目的就是做搜索。

---

## 六、终极方案：直接短路（比 43 行更简）

```typescript
// handler.ts — 在 translateToOpenAI 之前

// 检测 web search 子请求
if (process.env.TAVILY_API_KEY
  && anthropicPayload.tools?.some((t: any) => t.type?.startsWith("web_search_"))) {

  // 从消息中提取搜索 query
  const msg = anthropicPayload.messages[0]
  const content = typeof msg?.content === "string"
    ? msg.content
    : Array.isArray(msg?.content)
      ? msg.content.map((b: any) => b.text || "").join(" ")
      : ""
  const query = content.replace(/^Perform a web search for the query:\s*/i, "").trim()

  if (query) {
    // 直接调 Tavily
    let results: Array<{title: string, url: string, content: string}> = []
    try {
      const resp = await fetch("https://api.tavily.com/search", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${process.env.TAVILY_API_KEY}`,
        },
        body: JSON.stringify({ query, max_results: 5 }),
      })
      const data = await resp.json() as any
      results = data.results ?? []
    } catch {}

    // 构造 Anthropic 格式的 web search 响应
    const srvId = `srvtoolu_${Date.now().toString(36)}${Math.random().toString(36).slice(2, 8)}`
    const searchResults = results.map(r => ({
      type: "web_search_result",
      url: r.url,
      title: r.title,
      encrypted_content: "",  // Claude Code 不读这个字段
      page_age: null,
    }))

    // 构造文本摘要（Claude Code 会用这个作为 tool_result 返回主对话）
    const summaryText = results.length
      ? results.map(r => `**${r.title}**\n${r.url}\n${r.content}`).join("\n\n---\n\n")
      : "No search results found."

    return c.json({
      id: `msg_${Date.now().toString(36)}`,
      type: "message",
      role: "assistant",
      model: anthropicPayload.model,
      content: [
        { type: "server_tool_use", id: srvId, name: "web_search", input: { query } },
        { type: "web_search_tool_result", tool_use_id: srvId, content: searchResults },
        { type: "text", text: summaryText },
      ],
      stop_reason: "end_turn",
      stop_sequence: null,
      usage: { input_tokens: 0, output_tokens: 0 },
    })
  }
}
```

### 6.1 优势

- **完全短路**：不走翻译层、不调 Copilot、不需要二次请求
- **~35 行**：比之前的 43 行方案还少
- **不需要改翻译层**：`translateAnthropicToolsToOpenAI` 不用改
- **响应格式完美匹配 Claude Code 的解析逻辑**

### 6.2 但有个问题：流式呢？

Claude Code 的 WebSearch 子请求是流式的。上面返回的是 JSON，不是 SSE。

Claude Code 在 `stream: true` 时期望 SSE events。需要判断 `anthropicPayload.stream`。

如果 stream=true，需要用 `streamSSE` 返回 SSE 格式的事件。但这会增加代码量。

**最简处理**：如果请求 stream=true，先返回 JSON（Claude Code 的 SDK 应该能处理非流式响应，即使请求了 stream）。

不行，这不可靠。让我考虑另一个方式...

### 6.3 最终决策：短路 JSON + 流式 SSE 双支持

```typescript
if (!anthropicPayload.stream) {
  return c.json(response)  // 非流式：直接返回 JSON
}

// 流式：返回 SSE events
return streamSSE(c, async (sseStream) => {
  await sseStream.writeSSE({ event: "message_start", data: JSON.stringify({ type: "message_start", message: { ...response, content: [] } }) })
  for (let i = 0; i < response.content.length; i++) {
    await sseStream.writeSSE({ event: "content_block_start", data: JSON.stringify({ type: "content_block_start", index: i, content_block: response.content[i] }) })
    await sseStream.writeSSE({ event: "content_block_stop", data: JSON.stringify({ type: "content_block_stop", index: i }) })
  }
  await sseStream.writeSSE({ event: "message_delta", data: JSON.stringify({ type: "message_delta", delta: { stop_reason: "end_turn" }, usage: response.usage }) })
  await sseStream.writeSSE({ event: "message_stop", data: JSON.stringify({ type: "message_stop" }) })
})
```

这增加了约 10 行。总计约 45 行。

---

## 七、方案对比

| 方案 | 思路 | 行数 | 改动文件 | 是否需要 Copilot |
|------|------|------|---------|-----------------|
| Review #004-#007 | 翻译层过滤 + tool call 拦截 + 二次请求 | 43 行 | 2 个 | ✅ 需要 |
| **Review #008** | **handler 短路 + Tavily + 直接返回** | **~45 行** | **1 个** | **❌ 不需要** |

### 核心区别

- 旧方案：改翻译层 → 让 Copilot 的 Claude 调 web_search → 拦截 → Tavily → 再次请求
- **新方案：完全绕过 Copilot，直接在 proxy 层用 Tavily 替代**

新方案**不依赖 Copilot 模型是否会调用 web_search tool**，消除了最大的不确定性！

---

## 八、遗留问题

1. **Claude Code 是否真的把 web_search_20250305 放在请求的 tools 数组中？** — extraToolSchemas 在 SDK 中可能被合并到 tools，也可能作为单独字段。需要通过 raven 的日志实际观察。

2. **isEnabled() 在 raven 场景下是否返回 true？** — 取决于 QA() 的 provider 检测逻辑。如果设了 ANTHROPIC_API_KEY，很可能被识别为 firstParty。

3. **encrypted_content 为空是否会导致 Claude Code 报错？** — 根据源码分析，Claude Code 的 `BB_` 函数只提取 title 和 url，不读 encrypted_content。应该安全。

---

*Review #008 by Claude Code — 源码级逆向工程*
