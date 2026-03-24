# Raven Review #001 — 架构分析 & Web Search 拦截方案

**日期**: 2026-03-23
**项目**: https://github.com/nocoo/raven
**版本**: 当前 main 分支

---

## 一、核心问题理解

**目标**: 在 raven 代理上拦截 Claude Code 的 web search 调用，替换为 Tavily 搜索。

**关键发现**: Raven 当前是 **GitHub Copilot 代理**（Anthropic 格式 ↔ OpenAI 格式翻译），不是 Anthropic API 直连代理。但 Claude Code 通过 `ANTHROPIC_BASE_URL` 连接 raven，所以 raven 确实是 Claude Code 和上游之间的中间层，可以在此拦截。

---

## 二、Anthropic Web Search 机制分析

### 2.1 Server Tool 机制

Claude Code 的 web search **不是** 普通的 client tool（`tool_use`），而是 **Anthropic Server Tool**：

| 属性 | Client Tool | Server Tool (web_search) |
|------|------------|--------------------------|
| 请求中的类型 | `{"name": "xxx", "input_schema": {...}}` | `{"type": "web_search_20250305", "name": "web_search"}` |
| 响应 block | `tool_use` | `server_tool_use` |
| ID 前缀 | `toolu_` | `srvtoolu_` |
| 结果 block | `tool_result`（客户端回传） | `web_search_tool_result`（服务端内联返回） |
| 谁执行 | 客户端代码 | Anthropic 服务端 |

### 2.2 请求格式

Claude Code 发送的请求中，`tools` 数组会包含：

```json
{
  "type": "web_search_20250305",
  "name": "web_search",
  "max_uses": 5
}
```

这个 tool **没有** `input_schema`，因为它不是普通 function tool。

### 2.3 响应格式

Anthropic 返回的响应 `content` 数组中会出现：

```json
[
  { "type": "text", "text": "让我搜索一下..." },
  {
    "type": "server_tool_use",
    "id": "srvtoolu_xxx",
    "name": "web_search",
    "input": { "query": "搜索关键词" }
  },
  {
    "type": "web_search_tool_result",
    "tool_use_id": "srvtoolu_xxx",
    "content": [
      {
        "type": "web_search_result",
        "url": "https://example.com",
        "title": "Example",
        "encrypted_content": "EqgfCioIARgB...",
        "page_age": "March 2026"
      }
    ]
  },
  { "type": "text", "text": "根据搜索结果...", "citations": [...] }
]
```

### 2.4 关键约束

- `encrypted_content` 是加密的，只有 Anthropic 服务端能解读
- 整个 server_tool_use → web_search_tool_result 流程在**同一个 API 响应**内完成
- 客户端不需要也不能参与 server tool 的执行

---

## 三、Raven 当前架构对 Web Search 的影响

### 3.1 当前请求流

```
Claude Code → POST /v1/messages (Anthropic 格式，含 web_search server tool)
    ↓
Raven: translateToOpenAI()
    ↓ 问题：web_search_20250305 类型的 tool 没有 input_schema，
    ↓ translateAnthropicToolsToOpenAI() 会把它当普通 function tool 翻译
    ↓
Copilot API (OpenAI 格式) — Copilot 不认识 web_search，会忽略
    ↓
响应中不会包含 web search 结果
```

### 3.2 现有代码的盲点

1. **`anthropic-types.ts`** — `AnthropicTool` 只定义了 `{name, description?, input_schema}` 格式，没有 `type` 字段支持 server tool
2. **`non-stream-translation.ts`** — `translateAnthropicToolsToOpenAI()` 把所有 tool 当 function tool 翻译
3. **`handler.ts`** — 直接把翻译后的 payload 发给 Copilot，没有中间拦截点
4. **`stream-translation.ts`** — 只处理 `tool_use` / `text` 类型的 content block，没有 `server_tool_use` / `web_search_tool_result`

---

## 四、拦截方案设计

### 方案 A: 最精简 — 在 handler.ts 中拦截（推荐 ⭐）

**核心思路**: 把 `web_search` 从 server tool 转换为普通 client tool 的工作流。当模型返回 `tool_use` (name=web_search) 时，proxy 自动调用 Tavily 并注入结果。

但**问题在于**: raven 把请求转发给 Copilot（不是 Anthropic），Copilot 的模型不会主动调 web_search。

### 方案 B: 不经过 Copilot，由 Proxy 直接处理 web search 循环

**核心思路**:
1. 从 tools 中剥离 `web_search` server tool
2. 注入一个普通的 `web_search` function tool 定义
3. 正常转发给 Copilot
4. 如果 Copilot 模型返回 `tool_use` name=`web_search` → proxy 自动调 Tavily → 把结果注入回对话，重新请求 Copilot
5. 最终返回给 Claude Code

**问题**: Copilot 的 Claude 模型不一定会调自定义的 web_search tool。这依赖 system prompt 引导。

### 方案 C: 最实际 — Proxy 直接转发 Anthropic API 用于 web search（最佳 ⭐⭐⭐）

**核心思路**：当检测到请求中包含 `web_search_20250305` 类型的 tool 时，**不走 Copilot 翻译流程**，而是直接转发给 Anthropic API。

等等——这需要 Anthropic API key，但如果用户本来就是用 Copilot 免费额度，就没有 Anthropic key。

### 方案 D: 最简单可行 — 在 Proxy 层模拟 server tool 行为（最佳 ⭐⭐⭐⭐）

**核心思路**：
1. 从请求的 tools 中**剥离** `web_search_20250305` 类型的 server tool
2. 注入一个**同名的普通 function tool** `web_search`，带 input_schema `{query: string}`
3. 正常翻译并转发给 Copilot
4. 当 Copilot 模型响应中出现 `tool_calls` (name=`web_search`) 时：
   - Proxy **拦截**此 tool call
   - 调用 **Tavily API** 搜索
   - 构造 `web_search_tool_result` 格式的结果
   - **不返回** tool_use 给客户端，而是将结果作为 tool message 追加到对话中，**再次请求 Copilot**
   - 最终把 Copilot 的最终回复翻译回 Anthropic 格式返回给客户端
5. 或者更简单：直接在 Anthropic 响应中注入 `server_tool_use` + `web_search_tool_result` blocks

**但这很复杂**，因为涉及多轮调用和流式处理。

### 方案 E: 真正最简 — 直接在响应翻译层合成 web search 结果（终极推荐 ⭐⭐⭐⭐⭐）

#### 思路

与其搞复杂的多轮调用，不如：

1. **请求侧**: 将 `web_search_20250305` server tool → 普通 function tool `web_search({query: string})`
2. **响应侧**: 当 Copilot 返回 `tool_calls` 包含 `web_search` 时：
   - **非流式**: proxy 拦截，调 Tavily，把 tool_call + tool_result 追加到 messages，再次请求 Copilot 拿最终回答
   - **流式**: 同理但需更复杂的流拼接（先缓冲到 tool call 完整，调 Tavily，再发第二个请求并流式转发）

#### 最精简代码骨架

```typescript
// 新文件: packages/proxy/src/routes/messages/web-search.ts

const TAVILY_API_KEY = process.env.TAVILY_API_KEY

interface TavilyResult {
  title: string
  url: string
  content: string
  score: number
}

// 1. 请求预处理：剥离 server tool，注入 function tool
export function preprocessWebSearch(payload: AnthropicMessagesPayload) {
  if (!payload.tools) return { payload, hasWebSearch: false }

  const serverToolIdx = payload.tools.findIndex(
    (t: any) => t.type?.startsWith("web_search_")
  )
  if (serverToolIdx === -1) return { payload, hasWebSearch: false }

  // 剥离 server tool，注入普通 function tool
  const newTools = [...payload.tools]
  newTools.splice(serverToolIdx, 1, {
    name: "web_search",
    description: "Search the web for current information. Use this when you need up-to-date information.",
    input_schema: {
      type: "object",
      properties: {
        query: { type: "string", description: "Search query" }
      },
      required: ["query"]
    }
  })

  return {
    payload: { ...payload, tools: newTools },
    hasWebSearch: true
  }
}

// 2. 调用 Tavily
async function searchTavily(query: string): Promise<TavilyResult[]> {
  const res = await fetch("https://api.tavily.com/search", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${TAVILY_API_KEY}`
    },
    body: JSON.stringify({ query, max_results: 5 })
  })
  const data = await res.json()
  return data.results
}

// 3. 响应后处理：检测 web_search tool_call，调 Tavily，再次请求
export async function handleWebSearchToolCall(
  openAIResponse: ChatCompletionResponse,
  originalOpenAIPayload: ChatCompletionsPayload
): Promise<ChatCompletionResponse | null> {
  const choice = openAIResponse.choices[0]
  if (!choice?.message?.tool_calls) return null

  const webSearchCall = choice.message.tool_calls.find(
    tc => tc.function.name === "web_search"
  )
  if (!webSearchCall) return null

  // 调 Tavily
  const args = JSON.parse(webSearchCall.function.arguments)
  const results = await searchTavily(args.query)
  const resultText = results.map(r =>
    `[${r.title}](${r.url})\n${r.content}`
  ).join("\n\n---\n\n")

  // 构建第二轮请求
  const followUp = {
    ...originalOpenAIPayload,
    messages: [
      ...originalOpenAIPayload.messages,
      choice.message, // assistant with tool_calls
      {
        role: "tool",
        tool_call_id: webSearchCall.id,
        content: resultText
      }
    ]
  }

  return createChatCompletions(followUp) // 第二轮调用
}
```

#### 改动量评估

| 文件 | 改动 |
|------|------|
| 新增 `web-search.ts` | ~60 行（Tavily 调用 + 请求预处理） |
| 修改 `handler.ts` | ~20 行（调用预处理 + 后处理） |
| 修改 `anthropic-types.ts` | ~5 行（添加 server tool type） |
| 修改 `.env.example` | 1 行（`TAVILY_API_KEY`） |
| **总计** | **~86 行** |

---

## 五、优化点汇总

### 5.1 必须修复

| # | 问题 | 优先级 |
|---|------|--------|
| 1 | `AnthropicTool` 类型缺少 `type` 字段，无法识别 server tool | 🔴 高 |
| 2 | `translateAnthropicToolsToOpenAI()` 不过滤 server tool | 🔴 高 |
| 3 | 没有 `server_tool_use` / `web_search_tool_result` 的类型定义和处理 | 🔴 高 |

### 5.2 建议优化

| # | 优化点 | 优先级 |
|---|--------|--------|
| 4 | handler 没有预处理/后处理 hook，不便扩展 | 🟡 中 |
| 5 | 流式 web search 拦截需要缓冲机制 | 🟡 中 |
| 6 | 配置系统未预留 Tavily API key 位置 | 🟢 低 |
| 7 | 没有 web search 结果缓存（相同 query 重复搜索） | 🟢 低 |

### 5.3 代码质量观察

| # | 观察 | 评价 |
|---|------|------|
| 8 | 全局 state 单例模式清晰 | ✅ 好 |
| 9 | 翻译层代码结构清晰，非流式和流式分开处理 | ✅ 好 |
| 10 | OPT-1/2/3 可配置优化标志设计优雅 | ✅ 好 |
| 11 | 错误处理覆盖了流式/非流式两种场景 | ✅ 好 |
| 12 | `translateModelName` 硬编码 claude-sonnet/opus，不够通用 | ⚠️ 可改进 |

---

## 六、下一步行动

1. ✅ 完成架构分析 (本次 review)
2. 🔄 设计最精简的非流式 web search 拦截实现
3. 🔄 设计流式 web search 拦截实现
4. 🔄 评估是否需要缓存层
5. 🔄 编写具体的代码修改方案

---

*Review by Claude Code — 持续优化中*
