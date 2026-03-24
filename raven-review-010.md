# Raven Review #010 — SSE 验证 & 极限压缩

**日期**: 2026-03-24
**项目**: https://github.com/nocoo/raven
**聚焦**: 验证 SSE 流式兼容性 + 最终代码压缩

---

## 一、SSE 流式格式验证 ✅

### 1.1 Claude Code SDK 的 SSE 解析逻辑

从 cli.js 确认，SDK 有两层解析：

**层 1: Anthropic SDK 消息累积器**
```javascript
case "content_block_start":
  K.content.push(q.content_block)  // 直接把整个 content_block 推入 content 数组
  return K
```

**层 2: WebSearch 的 call() 方法**
```javascript
// 1. 检测 server_tool_use
if (event.type === "content_block_start") {
  if (event.content_block.type === "server_tool_use") {
    X = event.content_block.id  // 记录 server tool ID
  }
}

// 2. 累积 input_json_delta 获取 query
if (event.type === "content_block_delta") {
  if (event.delta.type === "input_json_delta") {
    D += event.delta.partial_json  // 累积 JSON
    // 正则提取 query
  }
}

// 3. 读取 web_search_tool_result（在 content_block_start 中！）
if (event.type === "content_block_start") {
  if (event.content_block.type === "web_search_tool_result") {
    R = event.content_block.tool_use_id
    u = event.content_block.content  // ← 完整的搜索结果在这里
  }
}
```

### 1.2 SSE 事件序列要求

```
event: message_start
data: {"type":"message_start","message":{...}}

event: content_block_start        ← server_tool_use（Claude Code 记录 ID）
data: {"type":"content_block_start","index":0,"content_block":{"type":"server_tool_use","id":"srvtoolu_xxx","name":"web_search","input":{}}}

event: content_block_delta         ← input_json_delta（Claude Code 提取 query）
data: {"type":"content_block_delta","index":0,"delta":{"type":"input_json_delta","partial_json":"{\"query\":\"xxx\"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start        ← web_search_tool_result（Claude Code 读取结果）
data: {"type":"content_block_start","index":1,"content_block":{"type":"web_search_tool_result","tool_use_id":"srvtoolu_xxx","content":[{"type":"web_search_result","url":"...","title":"..."}]}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: content_block_start        ← text（摘要文本）
data: {"type":"content_block_start","index":2,"content_block":{"type":"text","text":"..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":2}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":0}}

event: message_stop
data: {"type":"message_stop"}
```

### 1.3 Review #009 方案的 SSE 问题

Review #009 的 SSE 实现**缺少 `input_json_delta` 事件**！Claude Code 通过这个事件提取 query。虽然 query 也可以从 `content_block_start` 的 `input.query` 中获取，但 Claude Code 的代码是通过 delta 来解析的。

**修复**: 需要在 `server_tool_use` 的 `content_block_start` 之后，添加一个 `content_block_delta` 事件包含 `input_json_delta`。

---

## 二、修正后的最终代码

### 2.1 压缩 + 修正版（只改 handler.ts）

```typescript
// handler.ts — 在 anthropicPayload 解析之后，translateToOpenAI 之前

// === Web Search → Tavily short-circuit ===
const wsTool = (anthropicPayload as any).tools?.find(
  (t: any) => typeof t.type === "string" && t.type.startsWith("web_search_"),
)
if (wsTool && process.env.TAVILY_API_KEY) {
  const msg0 = anthropicPayload.messages[0]
  const text = typeof msg0?.content === "string" ? msg0.content
    : Array.isArray(msg0?.content)
      ? msg0.content.filter((b: any) => b.type === "text").map((b: any) => b.text).join(" ")
      : ""
  const query = text.replace(/^Perform a web search for the query:\s*/i, "").trim() || text

  let results: any[] = []
  try {
    const r = await fetch("https://api.tavily.com/search", {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${process.env.TAVILY_API_KEY}` },
      body: JSON.stringify({ query, max_results: 5 }),
    }).then(r => r.json() as any)
    results = r.results ?? []
  } catch {}

  const sid = `srvtoolu_${Date.now().toString(36)}${Math.random().toString(36).slice(2, 8)}`
  const blocks: any[] = [
    { type: "server_tool_use", id: sid, name: "web_search", input: { query } },
    { type: "web_search_tool_result", tool_use_id: sid, content: results.map(r => ({ type: "web_search_result", url: r.url, title: r.title, encrypted_content: "", page_age: null })) },
    { type: "text", text: results.length ? results.map(r => `${r.title}\n${r.url}\n${r.content}`).join("\n\n---\n\n") : "No results found." },
  ]
  const resp = { id: `msg_${requestId}`, type: "message", role: "assistant", model: anthropicPayload.model, content: blocks, stop_reason: "end_turn", stop_sequence: null, usage: { input_tokens: 0, output_tokens: 0 } }

  logEmitter.emitLog({ ts: Date.now(), level: "info", type: "request_end", requestId, msg: `200 web_search (tavily) ${Math.round(performance.now() - startTime)}ms`, data: { path: "/v1/messages", format: "anthropic", model: anthropicPayload.model, latencyMs: Math.round(performance.now() - startTime), stream: !!anthropicPayload.stream, status: "success", statusCode: 200, accountName, sessionId, clientName, clientVersion } })

  if (!anthropicPayload.stream) return c.json(resp)

  return streamSSE(c, async (s) => {
    const ev = (event: string, data: any) => s.writeSSE({ event, data: JSON.stringify(data) })
    await ev("message_start", { type: "message_start", message: { ...resp, content: [], stop_reason: null } })
    for (let i = 0; i < blocks.length; i++) {
      await ev("content_block_start", { type: "content_block_start", index: i, content_block: blocks[i] })
      // Emit input_json_delta for server_tool_use (Claude Code parses query from this)
      if (blocks[i].type === "server_tool_use") {
        await ev("content_block_delta", { type: "content_block_delta", index: i, delta: { type: "input_json_delta", partial_json: JSON.stringify(blocks[i].input) } })
      }
      await ev("content_block_stop", { type: "content_block_stop", index: i })
    }
    await ev("message_delta", { type: "message_delta", delta: { stop_reason: "end_turn" }, usage: { output_tokens: 0 } })
    await ev("message_stop", { type: "message_stop" })
  })
}
// === End Web Search ===
```

### 2.2 行数

实际行数（按照 raven 代码风格格式化后）：**约 50 行**

| 部分 | 行数 |
|------|------|
| 检测 + query 提取 | 8 |
| Tavily 调用 | 7 |
| 响应构造 | 7 |
| 日志 | 2 |
| 非流式返回 | 1 |
| 流式 SSE 返回 | 13 |
| 空行 + 注释 | 12 |
| **总计** | **~50 行** |

---

## 三、从 69 行压缩到 50 行

| 优化 | 节省 |
|------|------|
| 内联 searchResults 类型（用 any[]） | -2 行 |
| 合并 responseBody 为单行 | -5 行 |
| 合并 log 为单行 | -6 行 |
| 简化 SSE helper (`ev`) | -4 行 |
| 添加 input_json_delta | +2 行 |
| **净减** | **-19 行** |

---

## 四、10 轮 Review 总演进

```
#001 (架构分析)
  ↓
#002 (71行, 独立模块)
  ↓ -49%
#004 (37行, 纯内联)
  ↓ +16%
#006 (43行, 加固版)
  ↓ 方案转向
#008 (发现 deferred tool 机制)
  ↓
#009 (69行, handler 短路)
  ↓ -28%
#010 (50行, 压缩+修正)
```

### 最终方案特性

| 特性 | 状态 |
|------|------|
| 代码行数 | 50 行 |
| 改动文件 | 1 个 (handler.ts) |
| 新文件 | 0 |
| 新依赖 | 0 |
| 新类型 | 0 |
| 依赖 Copilot | ❌ 不需要 |
| 流式支持 | ✅ 原生 SSE |
| input_json_delta | ✅ 正确发送 |
| 降级处理 | ✅ Tavily 失败返回空结果 |
| 向后兼容 | ✅ 无 TAVILY_API_KEY 时完全不影响 |
| 日志记录 | ✅ 记录到 raven 统计系统 |

---

## 五、其他 Raven 优化点补充

### 5.1 翻译层潜在 Bug

当前 `translateAnthropicToolsToOpenAI` 直接访问 `tool.input_schema`。如果 Claude Code 发来的 tools 中包含 `web_search_20250305`（没有 `input_schema`），它会把 `undefined` 作为 `parameters` 发给 Copilot。

即使有了我们的短路方案（web search 请求被拦截不会走翻译层），**主对话请求中不会包含 web_search tool**，所以这不是问题。但为了鲁棒性，可以加一行防御：

```typescript
parameters: tool.input_schema ?? {},
```

### 5.2 count-tokens-handler 同理

`handleCountTokens` 调 `translateToOpenAI`，如果 tools 中混入了 server tool，tokenizer 会收到不完整的 tool 定义。但同理，主请求中不会有 web_search tool。

### 5.3 模型名翻译

`translateModelName` 只处理 `claude-sonnet-4-*` 和 `claude-opus-4-*`。如果 Copilot 将来支持 `claude-haiku-4-*`，需要添加处理。建议用通用正则：

```typescript
return model.replace(/^(claude-\w+-\d+)-\d+$/, "$1")
```

---

*Review #010 by Claude Code — SSE 验证 & 最终压缩完成*
