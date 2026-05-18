# PackyApi gpt-5.5 流式 Tool Calling 排查记录

## 背景

在 Next AI Draw.io 桌面应用中配置 PackyApi 的 `gpt-5.5` 后，模型校验可以通过，但正式生成或编辑图形时会出现 `network error`。本记录用于沉淀本地排查结论，便于后续继续定位或提交 issue。

## 正确配置

PackyApi 应按 OpenAI-compatible 接口配置：

```text
Provider: OpenAI
Base URL: https://api-slb.packyapi.com/v1
Model: gpt-5.5
```

不要配置为：

```text
Provider: Gateway
Base URL: https://api-slb.packyapi.com/v1
```

`Gateway` 在本项目中表示 Vercel AI Gateway。该模式会走 AI Gateway 协议路径，例如 `POST /v1/language-model`，而 PackyApi 暴露的是 OpenAI-compatible 接口，因此会出现：

```text
Invalid URL (POST /v1/language-model)
```

## 已验证现象

对 PackyApi 做了最小接口探测，结果如下：

```text
/v1/models
结果：成功
说明：响应中包含 gpt-5.5，supported_endpoint_types = ["openai"]
```

```text
非流式普通 Chat Completions
POST /v1/chat/completions
model = gpt-5.5
stream = false
结果：HTTP 200，能正常返回文本
```

```text
非流式 Tool Calling
POST /v1/chat/completions
model = gpt-5.5
stream = false
tools = [get_weather]
结果：HTTP 200，能正常返回 tool_calls
finish_reason = tool_calls
arguments = {"city":"Paris"}
```

```text
流式 Tool Calling
POST /v1/chat/completions
model = gpt-5.5
stream = true
tools = [get_weather]
结果：HTTP 200，但 SSE 中 tool_calls 的增量格式异常
```

## 关键异常

流式返回中，同一个工具调用的首个 chunk 类似：

```json
{
  "choices": [
    {
      "delta": {
        "tool_calls": [
          {
            "id": "call_xxx",
            "index": 0,
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": ""
            }
          }
        ]
      },
      "finish_reason": null,
      "index": 0
    }
  ]
}
```

但后续参数增量 chunk 变成：

```json
{
  "choices": [
    {
      "delta": {
        "tool_calls": [
          {
            "index": 1,
            "function": {
              "arguments": "{\""
            }
          }
        ]
      },
      "finish_reason": null,
      "index": 0
    }
  ]
}
```

后续 `"city"`、`"Paris"`、`"\"}"` 等参数片段也继续使用 `index: 1`。

## 初步结论

`gpt-5.5` 模型本身可以调用，PackyApi 的非流式 Tool Calling 也可用。问题集中在 `stream=true` 的 Tool Calling SSE 增量格式。

同一个 tool call 的后续增量应继续使用与首块一致的 `tool_calls[].index`，例如都为 `index: 0`。当前 PackyApi 返回首块 `index: 0`，参数增量却变成 `index: 1`，这会导致 Vercel AI SDK 或应用层在组装工具调用参数时把参数归到另一个不完整 tool call 上。

因此，应用中看到的 `network error` 更像是下游 SDK 或前端对流式 tool_calls 解析失败后的泛化错误，而不是模型不存在、API Key 无效或普通网络不可达。

## 涉及源码位置

模型校验入口：

```text
components/model-config-dialog.tsx
app/api/validate-model/route.ts
```

聊天流式生成入口：

```text
components/chat-panel.tsx
app/api/chat/route.ts
```

Provider 构造逻辑：

```text
lib/ai-providers.ts
```

其中 `openai + custom baseUrl` 会走 OpenAI-compatible Chat Completions 兼容路径；`gateway` 会走 Vercel AI Gateway 逻辑。

## 后续处理方向

1. 向 PackyApi 反馈：修复 `stream=true` 时 `tool_calls[].index` 与首个 tool call chunk 不一致的问题。
2. 在 Next AI Draw.io 中改善错误提示，避免把流式 tool_calls 解析失败泛化为 `network error`。
3. 评估是否为 OpenAI-compatible custom endpoint 增加非流式 fallback。
4. 如需兼容不标准上游，可考虑在应用或 SDK 适配层修正连续 tool_calls 增量的 index，但这属于兼容性兜底，不是根因修复。

