# Raven Review #006 — Edge Case 压力测试 & 方案加固

**日期**: 2026-03-23
**项目**: https://github.com/nocoo/raven
**聚焦**: 对 Review #005 的 37 行方案做 edge case 压力测试，逐个判断哪些需要处理

---

## 一、Edge Case 清单 & 逐一判决

深度分析发现了 20 个 edge case。但**不是所有都需要在首次实现中解决**。逐一裁决：

### 🔴 可能导致失败的

| # | Edge Case | 是否需要处理？ | 理由 |
|---|-----------|---------------|------|
| 1 | **混合 tool_calls (web_search + 其他 tool)** | ❌ 不需要 | Copilot 返回的 tool_calls 要么全是客户端 tool（client 执行），要么包含 web_search。如果 web_search 和其他 tool 混在一起，我们只处理 web_search，**其他 tool 在二次请求中仍然在 tools 列表里**——模型可以再次决定调用它们 |
| 2 | **web_search 无限循环** | ⚠️ 需要简单限制 | 加一个 `maxWebSearchRetries = 3` 的计数器即可。**+2 行** |
| 3 | **count-tokens-handler 也调 translateToOpenAI** | ❌ 不需要 | count_tokens 只是估算 token 数，少算/多算一个 web_search tool 的定义（约 30 token）完全可以接受 |
| 4 | **metrics 使用 response.usage 而非 finalResp.usage** | ⚠️ 需要修复 | 把 metrics 提取改为用 `finalResp` 而非 `response`。这不是额外行数——只是变量名改了 |
| 5 | **resolvedModel 追踪** | ❌ 不需要 | 两次请求用的是同一个 model，Copilot 不会悄悄换模型 |

### 🟡 次要问题

| # | Edge Case | 是否需要处理？ | 理由 |
|---|-----------|---------------|------|
| 6 | **TypeScript 类型** | ❌ 不需要 | 用 `as any` 或类型断言即可，Message interface 已有 tool_calls 字段 |
| 7 | **TAVILY_API_KEY 设了但没 web_search tool** | ❌ 自动安全 | `clientTools.length < anthropicTools.length` 只有真正过滤了 server tool 才成立 |
| 8 | **stream=false 是否正确走非流式分支** | ✅ 已验证安全 | `createChatCompletions` 在 `stream=false` 时返回 JSON，`isNonStreaming` 正确判断 |
| 10 | **Tavily API 失败** | ⚠️ 需要基本处理 | 加 try-catch，Tavily 失败时返回 "Search failed" 作为 tool result。**+3 行** |
| 14 | **第二次请求的 token 计量** | ❌ 可以接受 | 用 finalResp 的 usage 已经包含了两次的总 input tokens |
| 15 | **流式模式** | ✅ 已处理 | 强制 stream=false 解决 |

### 🟢 不需要处理的

| # | 理由 |
|---|------|
| 9, 11-13, 16-20 | 都是"未来增强"或"极端边界"，首次实现不需要考虑 |

---

## 二、需要的额外改动

基于 edge case 分析，Review #005 的方案需要加 **2 个小改动**：

### 2.1 循环限制（+2 行）

```typescript
// 在 web search 拦截逻辑外层
let webSearchRetries = 0
// 在检测 wsCall 的条件中加：
if (wsCall && process.env.TAVILY_API_KEY && webSearchRetries++ < 3) {
```

### 2.2 Tavily 错误处理（+3 行）

```typescript
// 包裹 Tavily fetch
let resultText = "Web search failed. Please answer based on your training data."
try {
  const tavily = await fetch(...).then(r => r.json() as any)
  resultText = (tavily.results ?? []).map(...).join("\n\n")
} catch {}
```

### 2.3 metrics 修正（0 额外行，只改变量名）

```typescript
// 原来: const cachedTokens = response.usage?...
// 改为: const cachedTokens = finalResp.usage?...
```

---

## 三、修正后的最终代码

### 文件 1: `non-stream-translation.ts` — translateAnthropicToolsToOpenAI（不变，+11 行）

与 Review #005 完全相同。

### 文件 2: `handler.ts` — 完整的修改后非流式分支

```typescript
// === 新增：强制非流式 ===
const hasWebSearchTool = process.env.TAVILY_API_KEY
  && anthropicPayload.tools?.some((t: any) => t.type?.startsWith("web_search_"))
if (hasWebSearchTool) openAIPayload.stream = false

try {
  let response = await createChatCompletions(openAIPayload)

  if (isNonStreaming(response)) {
    // === 新增：web search 拦截循环 ===
    let finalResp = response as ChatCompletionResponse
    let wsRetries = 0
    while (wsRetries < 3) {
      const wsCall = finalResp.choices[0]?.message?.tool_calls?.find(
        (tc) => tc.function.name === "web_search",
      )
      if (!wsCall || !process.env.TAVILY_API_KEY) break
      wsRetries++

      let resultText = "Web search unavailable."
      try {
        const tavily = await fetch("https://api.tavily.com/search", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${process.env.TAVILY_API_KEY}`,
          },
          body: JSON.stringify({ query: JSON.parse(wsCall.function.arguments).query, max_results: 5 }),
        }).then((r) => r.json() as any)
        resultText = (tavily.results ?? [])
          .map((r: any) => `### ${r.title}\n${r.url}\n${r.content}`)
          .join("\n\n")
      } catch { /* Tavily down — use fallback text */ }

      finalResp = (await createChatCompletions({
        ...openAIPayload,
        stream: false,
        messages: [
          ...openAIPayload.messages,
          { role: "assistant", content: finalResp.choices[0].message.content, tool_calls: [wsCall] },
          { role: "tool", tool_call_id: wsCall.id, content: resultText },
        ],
      })) as ChatCompletionResponse
    }
    // === 新增结束 ===

    const anthropicResponse = translateToAnthropic(finalResp)
    const latencyMs = Math.round(performance.now() - startTime)
    const cachedTokens = finalResp.usage?.prompt_tokens_details?.cached_tokens ?? 0
    const inputTokens = (finalResp.usage?.prompt_tokens ?? 0) - cachedTokens
    const outputTokens = finalResp.usage?.completion_tokens ?? 0
    // ... 后续 log 和 return 不变，只是用 finalResp 代替 response
```

---

## 四、最终行数统计

| 文件 | 改动 | 行数 |
|------|------|------|
| `non-stream-translation.ts` — tool 过滤 + 注入 | 原有 | +11 行 |
| `handler.ts` — 强制非流式 | 原有 | +3 行 |
| `handler.ts` — web search 循环（含重试 + 错误处理） | 扩展 | +28 行 |
| `.env.example` | 原有 | +1 行 |
| **总计** | | **+43 行** |

**从 37 行增加到 43 行**，换来了：
- ✅ 循环搜索支持（最多 3 次）
- ✅ Tavily 降级处理（失败不崩溃）
- ✅ 正确的 metrics 追踪（用 finalResp）
- ✅ 不是 while(true)，有明确退出条件

---

## 五、方案演进总结

| Review | 行数 | 特点 |
|--------|------|------|
| #002 | 71 行 | 独立模块，过度工程 |
| #003 | 106 行 | 类型完备，更过度 |
| #004 | 37 行 | 纯内联，极简 |
| #005 | 37 行 | 确认收敛 |
| **#006** | **43 行** | **加固版：+重试 +降级 +metrics修正** |

### 43 行 = 功能完整 + 生产可用的最小实现

---

## 六、其他代码优化点（非 web search 相关）

顺便记录 raven 代码中的其他可改进之处：

### 6.1 翻译层

| 文件 | 优化点 | 优先级 |
|------|--------|--------|
| `non-stream-translation.ts:53-58` | `translateModelName` 硬编码 sonnet/opus，应支持 haiku 和未来模型 | 🟡 低 |
| `non-stream-translation.ts:197-199` | `handleAssistantMessage` 中 `toolUseBlocks` 只匹配 `type === "tool_use"`，应同时匹配 `server_tool_use`（鲁棒性） | 🟡 中 |
| `anthropic-types.ts:83-87` | `AnthropicTool` 缺少 `type` 字段，无法类型安全地区分 server tool | 🟢 低 |

### 6.2 Handler

| 文件 | 优化点 | 优先级 |
|------|--------|--------|
| `handler.ts:62-65` | metrics 提取和 log 是重复代码（非流式和流式两套），可抽取公共函数 | 🟢 低 |
| `handler.ts:160-174` | 错误处理只在外层 catch，没有针对特定上游错误的重试 | 🟢 低 |

### 6.3 基础设施

| 位置 | 优化点 | 优先级 |
|------|--------|--------|
| `config.ts` | 没有验证环境变量格式（如 port 是否为数字） | 🟢 低 |
| `state.ts` | 全局单例没有 freeze/readonly 保护 | 🟢 低 |
| `middleware.ts` | `timingSafeEqual` 是自定义实现，可用 `crypto.timingSafeEqual` | 🟡 中 |

---

*Review #006 by Claude Code — 压力测试 & 加固完成*
