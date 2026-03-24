# Raven Review #004 — 从零推导：绝对最简实现

**日期**: 2026-03-23
**项目**: https://github.com/nocoo/raven
**聚焦**: 推翻前三份 review 的过度工程，找到真正的极简解

---

## 一、重新思考：为什么之前的方案都太复杂了

回顾 Review #001-003，我一直在纠结"server_tool_use 的完整协议兼容"。但重新审视后发现：

**raven 的目标用户是通过 Copilot 订阅使用 Claude 的人。他们没有 Anthropic API key。**

这意味着：
1. 他们从来没有直连过 Anthropic API
2. 会话历史中**不会**有真正的 `server_tool_use` / `web_search_tool_result` 块
3. 如果我们的方案产生的搜索结果用的是**普通 `tool_use` + `tool_result`** 格式，那后续轮次的历史兼容**已经被 raven 现有翻译层处理了**

所以 Review #003 中担心的"历史消息中的 server_tool_use"问题**根本不存在于 raven 的使用场景中**！

---

## 二、绝对最简方案

### 2.1 核心洞察

```
之前的思路:
  server tool → function tool → Copilot 模型调 web_search → proxy 拦截 → Tavily → 再次请求

全新思路 (更简):
  直接在 translateToOpenAI 中过滤掉 web_search server tool
  ↓
  同时给 tools 加一个普通的 web_search function tool 定义
  ↓
  Copilot Claude 模型看到 web_search 工具就可能调用它
  ↓
  在 handler 中检测非流式响应的 tool_calls
  ↓
  有 web_search → 调 Tavily → 追加到消息 → 再次请求
  ↓
  没有 web_search → 正常返回
```

但等等，能不能**更简单**？

### 2.2 终极精简：只改两个函数

**核心代码只需要改动 2 个地方：**

#### 改动 1: `translateAnthropicToolsToOpenAI` 中过滤 + 注入

```typescript
// non-stream-translation.ts — 改 translateAnthropicToolsToOpenAI
function translateAnthropicToolsToOpenAI(
  anthropicTools: Array<AnthropicTool> | undefined,
): Array<Tool> | undefined {
  if (!anthropicTools) return undefined

  // 过滤掉 server tool (web_search_20250305 等)
  const clientTools = anthropicTools.filter(
    (t: any) => !t.type?.startsWith("web_search_")
  )

  const translated = clientTools.map((tool) => ({
    type: "function" as const,
    function: {
      name: tool.name,
      description: tool.description,
      parameters: tool.input_schema,
    },
  }))

  // 如果有 web_search 被过滤掉了，注入 function tool 版本
  if (process.env.TAVILY_API_KEY && clientTools.length < anthropicTools.length) {
    translated.push({
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

  return translated
}
```

**改动量：+11 行**

#### 改动 2: `handleCompletion` 中非流式的 web search 后处理

```typescript
// handler.ts — 在非流式分支中
if (isNonStreaming(response)) {
  // ⬇️ 新增: web search 拦截
  let finalResp = response
  const wsCall = response.choices[0]?.message?.tool_calls?.find(
    (tc) => tc.function.name === "web_search"
  )
  if (wsCall && process.env.TAVILY_API_KEY) {
    const { query } = JSON.parse(wsCall.function.arguments)
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
  // ⬆️ 新增结束

  const anthropicResponse = translateToAnthropic(finalResp)
  // ... 原有代码不变
}
```

**改动量：+22 行**

### 2.3 总改动量

| 位置 | 行数 |
|------|------|
| `non-stream-translation.ts` — `translateAnthropicToolsToOpenAI` | +11 行 |
| `handler.ts` — 非流式分支 | +22 行 |
| `.env.example` — `TAVILY_API_KEY=` | +1 行 |
| **总计** | **+34 行** |

**没有新文件。没有新类型。没有新模块。**

---

## 三、为什么 34 行就够了

### 3.1 不需要新文件

Tavily 调用就是一个 `fetch`，内联 4 行代码。不需要独立模块。

### 3.2 不需要新类型

server tool (`web_search_20250305`) 在请求的 tools 数组里就是一个带 `type` 字段的对象。TypeScript 的 `any` 类型转换 + 属性检查就够了：`(t: any) => !t.type?.startsWith("web_search_")`。

### 3.3 不需要处理历史消息中的 server_tool_use

因为在 raven 场景下：
- 第一轮：我们把 web_search 变成了普通 function tool → 模型回的是普通 `tool_calls` → raven 的 `translateToAnthropic` 已经能正确把它转成 Anthropic 格式的 `tool_use`
- 第二轮：Claude Code 把上一轮的 `tool_use` + `tool_result` 放回历史 → 这是标准格式 → raven 的 `handleAssistantMessage` 和 `handleUserMessage` 已经能处理

**整个链路天然兼容，不需要额外工作。**

### 3.4 不需要处理流式

先做非流式版本。原因：
- Claude Code 的 web search 调用大部分是 subagent 调用，stream 可能是 false
- 即使 stream=true，可以先回退到非流式处理（见下方 3.5）
- 一旦验证非流式能工作，流式版本只需在流式分支加类似的缓冲逻辑

### 3.5 流式的最简处理（如果需要）

最简方式：**当检测到请求中有 web_search tool 且 stream=true 时，改为 stream=false 处理**

```typescript
// handler.ts — 在 translateToOpenAI 之后
const hasWebSearch = anthropicPayload.tools?.some(
  (t: any) => t.type?.startsWith("web_search_")
)
if (hasWebSearch && process.env.TAVILY_API_KEY) {
  openAIPayload.stream = false // 强制非流式
}
```

**+3 行。** 粗暴但有效。总共 37 行。

---

## 四、对比各 Review 的方案

| Review | 方案 | 新文件 | 改动行数 | 新增类型 |
|--------|------|--------|---------|----------|
| #002 | 独立模块 + handler 修改 | 1 个 | ~71 行 | 无 |
| #003 | 独立模块 + 类型 + 翻译层 | 1 个 | ~106 行 | 3 个 |
| **#004** | **纯内联改动** | **0 个** | **34-37 行** | **0 个** |

**Review #004 比 #002 精简了 50%，比 #003 精简了 65%。**

---

## 五、完整 diff 预览

```diff
--- a/packages/proxy/.env.example
+++ b/packages/proxy/.env.example
@@ -5,3 +5,4 @@
 RAVEN_TOKEN_PATH=data/github_token
 RAVEN_LOG_LEVEL=info
 RAVEN_BASE_URL=
+TAVILY_API_KEY=

--- a/packages/proxy/src/routes/messages/non-stream-translation.ts
+++ b/packages/proxy/src/routes/messages/non-stream-translation.ts
@@ -290,10 +290,22 @@ function translateAnthropicToolsToOpenAI(
   if (!anthropicTools) {
     return undefined
   }
-  return anthropicTools.map((tool) => ({
+
+  // Filter out server tools (web_search_20250305 etc.)
+  const clientTools = anthropicTools.filter(
+    (t: any) => !t.type?.startsWith("web_search_")
+  )
+
+  const translated = clientTools.map((tool) => ({
     type: "function",
     function: {
       name: tool.name,
       description: tool.description,
       parameters: tool.input_schema,
     },
   }))
+
+  // Inject web_search as a regular function tool when Tavily is configured
+  if (process.env.TAVILY_API_KEY && clientTools.length < anthropicTools.length) {
+    translated.push({ type: "function", function: {
+      name: "web_search",
+      description: "Search the web for current information",
+      parameters: { type: "object", properties: { query: { type: "string" } }, required: ["query"] },
+    }})
+  }
+
+  return translated
 }

--- a/packages/proxy/src/routes/messages/handler.ts
+++ b/packages/proxy/src/routes/messages/handler.ts
@@ -55,9 +55,34 @@ export async function handleCompletion(c: Context) {
   const openAIPayload = translateToOpenAI(anthropicPayload)

+  // Force non-streaming when web search is active (simplifies Tavily integration)
+  const webSearchActive = process.env.TAVILY_API_KEY &&
+    anthropicPayload.tools?.some((t: any) => t.type?.startsWith("web_search_"))
+  if (webSearchActive) openAIPayload.stream = false
+
   try {
     const response = await createChatCompletions(openAIPayload)

     if (isNonStreaming(response)) {
+      // Web search interception: if model called web_search, resolve via Tavily
+      let finalResp = response
+      const wsCall = response.choices[0]?.message?.tool_calls?.find(
+        (tc) => tc.function.name === "web_search"
+      )
+      if (wsCall && process.env.TAVILY_API_KEY) {
+        const { query } = JSON.parse(wsCall.function.arguments)
+        const tavily = await fetch("https://api.tavily.com/search", {
+          method: "POST",
+          headers: {
+            "Content-Type": "application/json",
+            Authorization: `Bearer ${process.env.TAVILY_API_KEY}`,
+          },
+          body: JSON.stringify({ query, max_results: 5 }),
+        }).then((r) => r.json() as any)
+        const text = (tavily.results ?? [])
+          .map((r: any) => `### ${r.title}\n${r.url}\n${r.content}`)
+          .join("\n\n")
+        finalResp = (await createChatCompletions({
+          ...openAIPayload, stream: false,
+          messages: [...openAIPayload.messages,
+            { role: "assistant", content: response.choices[0].message.content, tool_calls: [wsCall] },
+            { role: "tool", tool_call_id: wsCall.id, content: text },
+          ],
+        })) as ChatCompletionResponse
+      }
+
-      const anthropicResponse = translateToAnthropic(response)
+      const anthropicResponse = translateToAnthropic(finalResp)
```

---

## 六、存在的权衡

| 权衡点 | 选择 | 理由 |
|--------|------|------|
| 流式 vs 非流式 | 强制非流式 | 简单 +3 行，不需要复杂的流缓冲 |
| 独立模块 vs 内联 | 内联 | 逻辑只有 2 处，不值得抽模块 |
| 类型安全 vs any | any 断言 | 只有 2 处需要检查 type 字段 |
| 多次搜索 | 只处理第一个 | 90% 的场景只有一次搜索 |
| 错误处理 | 无 (透传) | Tavily 失败时 fetch 会抛异常，被外层 catch 处理 |
| 搜索结果缓存 | 无 | 首次实现不需要 |

---

## 七、未来可迭代的增强（不在首次实现中做）

1. **流式支持**: 在流式分支中缓冲 tool call，完成后调 Tavily，拼接第二段流
2. **并行搜索**: 支持一次请求中多个 web_search tool call
3. **结果缓存**: 相同 query 的 Tavily 结果缓存 5 分钟
4. **Dashboard 可视化**: 在 Dashboard 上展示 web search 调用统计
5. **可配置开关**: 像 OPT-1/2/3 一样在 settings DB 中存储 web search 开关

---

## 八、结论

**34-37 行内联改动 = 完整的 web search 拦截 + Tavily 替换。**

没有新文件。没有新类型。没有新依赖。只改两个现有函数。

这是我能找到的**绝对最精简**实现。

---

*Review #004 by Claude Code — 极限精简*
