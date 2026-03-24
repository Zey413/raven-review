# Raven Review — Web Search Interception Analysis

对 [nocoo/raven](https://github.com/nocoo/raven) 项目的持续 code review，聚焦于在 raven proxy 上拦截 Claude Code 的 web search 调用并替换为 Tavily 搜索引擎。

## 核心结论

**43 行代码改动** = 完整的 Claude Code WebSearch → Tavily 替换方案。

- 0 新文件、0 新依赖、0 新类型
- 向后兼容（没有 `TAVILY_API_KEY` 时行为不变）
- 含重试 (max 3)、降级 (Tavily 挂了不崩)、正确的 metrics 追踪

## Review 系列

| 文件 | 焦点 | 方案行数 |
|------|------|---------|
| [raven-review-001.md](raven-review-001.md) | 架构分析 & 候选方案 | — |
| [raven-review-002.md](raven-review-002.md) | 独立模块方案 | ~71 行 |
| [raven-review-003.md](raven-review-003.md) | 会话历史兼容 & 鲁棒版 | ~106 行 |
| [raven-review-004.md](raven-review-004.md) | **突破：纯内联极简** | ~37 行 |
| [raven-review-005.md](raven-review-005.md) | 极限压缩确认 | ~37 行 |
| [raven-review-006.md](raven-review-006.md) | Edge case 压力测试 & 加固 | ~43 行 |
| [raven-review-007.md](raven-review-007.md) | 核心假设验证 & 定稿 | **43 行 (最终)** |

## 快速使用

参见 [patch/web-search-tavily.patch](patch/web-search-tavily.patch)：

```bash
cd /path/to/raven
git apply /path/to/web-search-tavily.patch
echo "TAVILY_API_KEY=tvly-your-key" >> packages/proxy/.env.local
bun run dev
```

## 原理

```
Claude Code → POST /v1/messages (含 web_search_20250305 server tool)
    ↓
Raven: 剥离 server tool → 注入 web_search function tool
    ↓
Copilot API: Claude 模型返回 tool_calls [{name: "web_search", args: {query: "..."}}]
    ↓
Raven: 拦截 → 调 Tavily API → 把结果作为 tool result 追加 → 再次请求 Copilot
    ↓
Copilot API: 基于搜索结果生成最终回答
    ↓
Raven: 翻译回 Anthropic 格式 → 返回给 Claude Code
```

## License

MIT
