---
title: "从零理解 AI Agent 底层机制"
description: "用 25 个文件手写一个完整 Agent：ReAct Loop、Function Calling、MCP 协议、会话管理"
pubDatetime: 2026-06-07T04:00:00.000Z
featured: true
tags:
  - AI Agent
  - 实战
---

> 参考实现：[agent-lab](https://github.com/your-repo/agent-lab) — 25 个 Java 文件，无框架依赖。

---

## 1. Agent 不是魔法

**Agent = LLM + Tools + Loop**

```
用户提问
    │
    ▼
┌─────────┐    带 tools 定义     ┌─────────┐
│  Agent  │ ──────────────────▶ │   LLM   │
│  Loop   │ ◀────────────────── │         │
└─────────┘   content / tool_calls └─────────┘
    │                                    │
    │ tool_calls                         │
    ▼                                    │
┌─────────┐                              │
│  Tools  │ ─── 执行结果回传 ─────────────┘
└─────────┘
    │
    ▼
最终答案 → 用户
```

没有黑盒。就是：**循环调 LLM，按需执行工具，直到 LLM 说"答完了"。**

---

## 2. ReAct Loop（核心循环）

ReAct = **Re**asoning + **Act**ing。问 LLM → 执行工具 → 再问 LLM。

```java
// agent-lab: core/AgentLoop.java（简化）
public String run(List<Message> messages, List<Tool> tools) {
    while (true) {
        LlmResponse resp = llm.chat(messages, tools);

        if (resp.hasToolCalls()) {
            // LLM 决定调用工具
            messages.add(resp.toAssistantMessage());
            for (ToolCall call : resp.getToolCalls()) {
                String result = toolRegistry.execute(call);
                messages.add(Message.tool(call.getId(), result));
            }
            continue;  // 带着工具结果继续推理
        }

        // LLM 返回纯文本 → 结束
        return resp.getContent();
    }
}
```

关键路径：

| 步骤 | 动作 | 消息 role |
|------|------|-----------|
| 1 | 用户提问 | `user` |
| 2 | LLM 返回 `tool_calls` | `assistant` |
| 3 | 执行工具，结果写入历史 | `tool` |
| 4 | 重复 2-3，直到 LLM 返回 `content` | `assistant` |

---

## 3. Function Calling 协议

本质：**约定好的 JSON 格式**，让 LLM 输出结构化的工具调用请求。

### Request — 告诉 LLM 有哪些工具

```json
{
  "model": "gpt-4o",
  "messages": [
    { "role": "user", "content": "北京今天天气怎么样？" }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "查询指定城市天气",
        "parameters": {
          "type": "object",
          "properties": {
            "city": { "type": "string", "description": "城市名" }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

### Response — LLM 决定调哪个工具

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\": \"北京\"}"
        }
      }]
    }
  }]
}
```

### 回传工具结果

```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "北京晴，25°C，微风"
}
```

`agent-lab` 中 `llm/OpenAiClient.java` 负责序列化/反序列化这套 JSON。换模型（Claude、DeepSeek）只需换 Client，Loop 不变。

---

## 4. Tool 注册

LLM 不会魔法般知道你有什么工具。**你必须把工具定义塞进每次请求的 `tools` 字段。**

```java
// agent-lab: tool/Tool.java
public interface Tool {
    String getName();
    String getDescription();
    JsonSchema getParameters();   // JSON Schema
    String execute(Map<String, Object> args);
}
```

```java
// agent-lab: tool/ToolRegistry.java
public class ToolRegistry {
    private final Map<String, Tool> tools = new HashMap<>();

    public void register(Tool tool) { tools.put(tool.getName(), tool); }

    // 导出为 OpenAI tools 格式，随每次请求发给 LLM
    public List<ToolDefinition> toOpenAiFormat() { /* name + description + schema */ }

    public String execute(ToolCall call) {
        return tools.get(call.getName()).execute(call.getArguments());
    }
}
```

---

## 5. MCP = 工具的 USB 接口

手写 `Tool` 接口够用，但每个 Agent 框架各写一套，工具无法复用。**MCP（Model Context Protocol）** 解决这个问题：统一工具的发现、调用、权限协议。

```
Agent 应用                    MCP Server（独立进程）
    │                              │
    │  JSON-RPC: tools/list        │
    │ ────────────────────────────▶│  返回可用工具列表
    │                              │
    │  JSON-RPC: tools/call        │
    │ ────────────────────────────▶│  执行工具，返回结果
    │◀──────────────────────────── │
```

```json
// tools/list → 发现
{ "method": "tools/list" }
→ { "tools": [{ "name": "search_web", "inputSchema": { ... } }] }

// tools/call → 执行
{ "method": "tools/call", "params": { "name": "search_web", "arguments": { "query": "AI Agent" } } }
→ { "content": [{ "type": "text", "text": "..." }] }
```

`mcp/McpClient.java` 把 MCP 工具桥接进 `ToolRegistry`——对 Loop 来说，远程工具和本地工具无区别。

---

## 6. 会话管理

多轮对话 = **消息列表不断追加**。没有隐藏状态，全在 `messages` 数组里。

```java
// agent-lab: session/ChatSession.java
public class ChatSession {
    private final List<Message> messages = new ArrayList<>();

    public ChatSession(String systemPrompt) {
        messages.add(Message.system(systemPrompt));
    }

    public String chat(String userInput) {
        messages.add(Message.user(userInput));
        String reply = agentLoop.run(messages, toolRegistry.toOpenAiFormat());
        messages.add(Message.assistant(reply));
        return reply;
    }
}
```

一轮完整对话的消息序列：

```
[
  { role: "system",    content: "你是展会情报助手..." },
  { role: "user",      content: "CES 2026 有哪些参展商？" },
  { role: "assistant", tool_calls: [{ name: "search_exhibitors", ... }] },
  { role: "tool",      tool_call_id: "call_xxx", content: "[{name:...}, ...]" },
  { role: "assistant", content: "CES 2026 主要参展商包括..." },
  { role: "user",      content: "其中做支付的有几家？" },        ← 第二轮
  { role: "assistant", content: "根据上面的列表，支付相关..." }
]
```

`tool` 消息必须紧跟带 `tool_calls` 的 `assistant`，顺序不能乱。

---

## 7. 一图总结

```
┌──────────────────────────────────────────────────────────────┐
│                        Agent 应用                            │
│                                                              │
│  ┌──────────┐    ┌─────────────┐    ┌──────────────────┐   │
│  │ ChatSession │──▶│  AgentLoop  │──▶│   LlmClient      │   │
│  │ (消息历史)  │◀──│  (ReAct)    │◀──│  (HTTP → LLM)    │   │
│  └──────────┘    └──────┬──────┘    └──────────────────┘   │
│                         │                                    │
│                  tool_calls?                                 │
│                    ┌────┴────┐                               │
│                    ▼         ▼                               │
│            ┌────────────┐ ┌──────────┐                      │
│            │ToolRegistry│ │ McpClient │                      │
│            │ (内建工具)  │ │(远程工具) │                      │
│            └─────┬──────┘ └────┬─────┘                      │
│                  ▼             ▼                             │
│            WeatherTool    MCP Server ──▶ 文件/DB/API/...    │
│            SearchTool     (JSON-RPC)                         │
│            CalcTool                                          │
└──────────────────────────────────────────────────────────────┘
```

### agent-lab 文件地图

| 模块 | 路径 | 职责 |
|------|------|------|
| 主循环 | `core/AgentLoop.java` | ReAct 循环 |
| LLM 客户端 | `llm/OpenAiClient.java` | Function Calling JSON 收发 |
| 工具接口 | `tool/Tool.java` | 工具契约 |
| 工具注册 | `tool/ToolRegistry.java` | Schema 导出 + 执行分发 |
| MCP 桥接 | `mcp/McpClient.java` | 远程工具接入 |
| 会话 | `session/ChatSession.java` | 多轮消息管理 |
| 消息模型 | `session/Message.java` | role/content/tool_calls |

**25 个文件，零框架。** 拆开看，Agent 就是：循环 + JSON 协议 + 工具注册表 + 消息列表。剩下的都是工程细节。
