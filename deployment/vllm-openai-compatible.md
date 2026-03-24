# vLLM OpenAI 兼容接口（含 tool_choice:auto 支持）

你的 vLLM 后端需要兼容：

- `POST /v1/chat/completions`
- Bearer `Authorization: Bearer <token>`
- 当请求体包含 `tools` 且 `tool_choice: "auto"` 时，不要返回类似：
  `"auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set`

## 启动参数建议

如果你在启动 vLLM 时使用 OpenAI 兼容模式，请确保包含以下参数：

- `--enable-auto-tool-choice`
- `--tool-call-parser`（取值需与模型族一致，见下）

### Qwen2.5（本仓库已验证）

- **推荐权重**：`Qwen2.5-14B-Instruct-AWQ`（对话 + 工具调用更稳）。
- **解析器**：`--tool-call-parser hermes`（与 [vLLM Tool Calling — Qwen Models](https://docs.vllm.ai/en/stable/features/tool_calling.html#qwen-models) 一致）。
- **不建议**：`Qwen2.5-Coder-*` 作 OpenClaw 主模型时，易出现 **`message.tool_calls` 为空**、仅在 `content` 里输出 `<tools>...</tools>` 文本的情况，网关无法执行工具。

示例（按本机路径与端口调整）：

```bash
vllm serve /path/to/Qwen2.5-14B-Instruct-AWQ \
  --host 0.0.0.0 --port 8001 \
  --tensor-parallel-size 2 \
  --served-model-name Qwen2.5-14B-Instruct-AWQ \
  --quantization awq \
  --api-key <YOUR_LLM_BEARER_TOKEN> \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```

`--served-model-name` 必须与 OpenClaw `models.providers.*.models[].id` 及 `curl` 里的 `"model"` **完全一致**。

## curl 自测（工具调用）

下列请求中 `"model"` 改为你的 `served-model-name`。

**1）`tool_choice: required`（易验证结构化 `tool_calls`）**

```bash
curl -sS -X POST http://127.0.0.1:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_LLM_BEARER_TOKEN>" \
  -d '{"model":"Qwen2.5-14B-Instruct-AWQ","messages":[{"role":"user","content":"查询杭州天气，必须使用 get_weather 工具。"}],"max_tokens":256,"tool_choice":"required","tools":[{"type":"function","function":{"name":"get_weather","description":"根据城市查询天气","parameters":{"type":"object","properties":{"location":{"type":"string"},"unit":{"type":"string","enum":["celsius","fahrenheit"]}},"required":["location","unit"]}}}]}' \
  | jq '{finish_reason: .choices[0].finish_reason, tool_calls: .choices[0].message.tool_calls, content: .choices[0].message.content}'
```

期望：`finish_reason` 为 `tool_calls`，`tool_calls` 非空，`content` 为空或 `null`。

**2）`tool_choice: auto`（与 OpenClaw 日常一致）**

将上式中的 `"tool_choice":"required"` 改为 `"tool_choice":"auto"` 即可；本仓库在 **Qwen2.5-14B-Instruct-AWQ + hermes** 下两种均已跑通。

## 其它模型（示例）

非 Qwen2.5 时 `tool-call-parser` 不同，例如部分 DeepSeek 权重使用 `deepseek_v32`，请参阅当前 vLLM 文档。最小冒烟请求示例：

```bash
curl -v -X POST http://127.0.0.1:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_LLM_BEARER_TOKEN>" \
  -d '{
    "model": "<SERVED_MODEL_NAME>",
    "messages": [{"role":"user","content":"你是谁"}],
    "max_tokens": 50,
    "tools": [{
      "type":"function",
      "function":{"name":"test","description":"test","parameters":{"type":"object","properties":{},"required":[]}}
    }],
    "tool_choice":"auto"
  }'
```

## OpenClaw baseUrl 要点

在 OpenClaw 的 `models.providers.<providerId>.baseUrl` 里应写：

- `http://127.0.0.1:8001/v1`

不要写成包含 `/chat` 的形式（否则可能出现重复路径导致 404）。

对接 vLLM 时 provider 建议使用 **`api: "openai-completions"`**（Chat Completions）。修改 `openclaw.json` 后请重启 Gateway；若使用 QQ 且存在 agent 缓存，请同步 `~/.openclaw/agents/main/agent/models.json` 中同一 provider 的 `baseUrl` / `api` / 模型 id。

## OpenClaw + QQ（已验证链路）

在 **`tools.profile: full`**、vLLM 工具参数正确、模型 id 一致的前提下，OpenClaw 网关 + QQ 通道可与上述 vLLM 协同工作（工具由 OpenClaw 执行，LLM 产出标准 `tool_calls`）。详见同目录 `README.zh-CN.md`。
