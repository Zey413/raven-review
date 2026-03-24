# Raven Review #003 — 重大发现：会话历史问题 & 方案修正

**日期**: 2026-03-23
**项目**: https://github.com/nocoo/raven
**聚焦**: 基于业界实践重新评估最简拦截方案

---

## 一、关键新发现 ⚠️

经过深入调研 LiteLLM、LibreChat、Antigravity、OpenRouter 等项目的实际经验，发现了 Review #002 方案中遗漏的**致命问题**：

### 1.1 会话历史中的 server_tool_use 块

Claude Code 在多轮对话中**会把 `server_tool_use` + `web_search_tool_result` 块原封不动地放回 messages 历史**。即：

```json
{
  "messages": [
    { "role": "user", "content": "搜索 xxx" },
    { "role": "assistant", "content": [
      { "type": "text", "text": "让我搜索..." },
      { "type": "server_tool_use", "id": "srvtoolu_xxx", "name": "web_search", "input": {"query": "..."} },
      { "type": "web_search_tool_result", "tool_use_id": "srvtoolu_xxx", "content": [...] },
      { "type": "text", "text": "根据搜索结果...", "citations": [...] }
    ]},
    { "role": "user", "content": "继续讨论..." }  // ← 第二轮
  ]
}
```

### 1.2 翻译层的致命问题

Raven 当前的 `handleAssistantMessage()` 只识别 3 种 block 类型：
- `text` → 合并为文本
- `tool_use` → 转为 OpenAI tool_calls
- `thinking` → 合并为文本

**`server_tool_use` 和 `web_search_tool_result` 会被完全丢弃！**

更糟的是：
- `server_tool_use` 的 ID 前缀是 `srvtoolu_`（不是 `toolu_`）
- `web_search_tool_result` 的 `encrypted_content` 对非 Anthropic 后端无意义
- `text` 块中的 `citations` 字段也会丢失

### 1.3 业界项目全部踩坑

| 项目 | 问题 |
|------|------|
| **LiteLLM** | 3 个 issue，错误转换 server_tool_use → tool_use 导致 400 错误 |
| **LibreChat** | web_search_tool_result 未保留，第二轮调用 400 |
| **Antigravity** | 推荐直接禁用 web search + 改用 MCP server |
| **OpenRouter** | 只有原生 Anthropic provider 能正常工作 |

---

## 二、方案重新评估

### Review #002 方案的问题

Review #002 的方案（server tool → function tool → Tavily → 二次请求）有一个我之前没考虑到的问题：

**第一轮**可以工作。但**第二轮及之后**，Claude Code 会把第一轮的搜索结果放回 messages 历史。这些历史中可能包含：
- 我们自己构造的 `tool_use`（name=web_search）+ `tool_result` 块
- 这些需要正确翻译到 OpenAI 格式

好消息是：如果我们用的是**普通 tool_use**（不是 server_tool_use），那 raven 现有的翻译层已经能正确处理了！所以实际上 Review #002 的方案对后续轮次是**兼容的**。

但等等——还有一个问题。Claude Code 会记住之前用了 web_search，下一轮如果**不在 tools 中包含 web_search**，API 会报错。我们的方案始终注入 web_search function tool，所以这也没问题。

### 真正的问题

真正的问题是：**如果用户之前不通过 raven 做过 web search（比如直连 Anthropic API），然后切换到 raven，历史中已经有 `server_tool_use` / `web_search_tool_result` 块怎么办？**

答案：Claude Code 的每个会话是独立的。切换 `ANTHROPIC_BASE_URL` 后开新会话，不会有旧历史。所以这不是问题。

---

## 三、最终方案确认 ✅

Review #002 的方案 **基本正确**，但需要两个补充：

### 3.1 补充 1：处理历史消息中的 server_tool_use 块

虽然在我们的方案下，历史中不会出现真正的 `server_tool_use` 块（因为我们把它转成了普通 tool_use），但为了**鲁棒性**，应该在翻译层添加对这两种 block 的处理：

```typescript
// non-stream-translation.ts 的 handleAssistantMessage 中
// 添加对 server_tool_use 的处理（当作 tool_use）
case "server_tool_use":
  toolUseBlocks.push({
    type: "tool_use",
    id: block.id,
    name: block.name,
    input: block.input
  })
  break

// handleUserMessage 中添加对 web_search_tool_result 的处理
// 将其转为等效的 tool_result
case "web_search_tool_result":
  // 提取可读内容
  const resultText = block.content
    .filter(r => r.type === "web_search_result")
    .map(r => `${r.title}\n${r.url}`)
    .join("\n")
  toolResultBlocks.push({
    type: "tool_result",
    tool_use_id: block.tool_use_id,
    content: resultText
  })
  break
```

### 3.2 补充 2：处理 text 块中的 citations

citations 对 Copilot 无意义，直接**剥离**即可：

```typescript
// 在 mapContent 中
case "text":
  // 只取 text 字段，丢弃 citations
  return block.text
```

这已经是现有代码的行为了（`block.text`），所以不需要额外修改。✅

---

## 四、修正后的最终实现方案

### 4.1 改动清单（精简版）

| 文件 | 改动 | 行数 |
|------|------|------|
| **新增** `web-search.ts` | Tavily 搜索 + 请求预处理 | ~55 行 |
| **修改** `handler.ts` | 调用预处理 + 后处理 | ~15 行 |
| **修改** `anthropic-types.ts` | 添加 ServerToolUse + WebSearchToolResult 类型 | ~20 行 |
| **修改** `non-stream-translation.ts` | 在翻译中处理 server_tool_use 块 | ~15 行 |
| **修改** `.env.example` | 添加 TAVILY_API_KEY | 1 行 |
| **总计** | | **~106 行** |

### 4.2 核心文件不变

`web-search.ts` 与 Review #002 中完全相同（55 行），是最精简的。

### 4.3 类型定义补充

```typescript
// anthropic-types.ts 新增

export interface AnthropicServerToolUseBlock {
  type: "server_tool_use"
  id: string
  name: string
  input: Record<string, unknown>
}

export interface AnthropicWebSearchToolResultBlock {
  type: "web_search_tool_result"
  tool_use_id: string
  content: Array<{
    type: "web_search_result"
    url: string
    title: string
    encrypted_content?: string
    page_age?: string
  }>
}

// 扩展 AnthropicAssistantContentBlock 联合类型
export type AnthropicAssistantContentBlock =
  | AnthropicTextBlock
  | AnthropicToolUseBlock
  | AnthropicThinkingBlock
  | AnthropicServerToolUseBlock        // 新增
  | AnthropicWebSearchToolResultBlock  // 新增
```

### 4.4 翻译层修改

```typescript
// non-stream-translation.ts — handleAssistantMessage 中

// 现有代码只过滤 tool_use, text, thinking
// 需要同时处理 server_tool_use
const toolUseBlocks = message.content.filter(
  (block): block is AnthropicToolUseBlock | AnthropicServerToolUseBlock =>
    block.type === "tool_use" || block.type === "server_tool_use",
)

// web_search_tool_result 当作 text 或直接跳过
// （它的内容对 Copilot 无用，搜索结果文本已在后续 text block 中）
```

---

## 五、方案对比总结

| 维度 | Review #002 | Review #003 (修正) |
|------|-------------|-------------------|
| 核心逻辑 | 相同 | 相同 |
| 历史消息兼容 | ❌ 未考虑 | ✅ 处理 server_tool_use 块 |
| 类型安全 | ❌ 无类型 | ✅ 完整类型定义 |
| 总代码量 | ~71 行 | ~106 行 |
| 鲁棒性 | 🟡 基本可用 | 🟢 生产可用 |

**最精简的"能用"实现**: 71 行 (Review #002)
**最精简的"鲁棒"实现**: 106 行 (Review #003)

---

## 六、替代方案：纯剥离 + MCP Server

如果不想在 raven 中实现 Tavily 调用，最简单的方案是：

1. **在 raven 中只做剥离**（~30 行）：
   - 从 tools 中移除 `web_search_20250305`
   - 从历史消息中将 `server_tool_use` / `web_search_tool_result` 转为无害的 text
2. **配置一个 Tavily MCP Server**给 Claude Code：
   - Claude Code 本身支持 MCP
   - 用 `@anthropics/mcp-server-tavily` 或自建
   - Claude Code 会自动使用 MCP 提供的搜索能力

这种方案在 raven 侧的改动更少，但需要用户额外配置 MCP server。

---

## 七、推荐优先级

1. **首选**: Review #003 方案（106 行，在 raven 内完整解决）
2. **备选**: 纯剥离 + MCP Server（30 行 raven 改动 + MCP 配置）
3. **最简首次验证**: Review #002 方案（71 行，先跑通再迭代）

---

*Review #003 by Claude Code — 持续优化中*
