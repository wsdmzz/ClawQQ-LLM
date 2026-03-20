# vLLM OpenAI 兼容接口（含 tool_choice:auto 支持）

你的 vLLM 后端需要兼容：

- `POST /v1/chat/completions`
- Bearer `Authorization: Bearer <token>`
- 当请求体包含 `tools` 且 `tool_choice: "auto"` 时，不要返回类似：
  `"auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set`

## 启动参数建议

如果你在启动 vLLM 时使用 OpenAI 兼容模式，请确保包含以下参数：

- `--enable-auto-tool-choice`
- `--tool-call-parser`

然后确认本机可访问：

```bash
curl -v -X POST http://127.0.0.1:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_LLM_BEARER_TOKEN>" \
  -d '{
    "model": "DeepSeek-R1-Distill-Qwen-14B",
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
