# claw-qq-llm 集成说明（中文）

本目录用于保存「OpenClaw 网关 + QQ 机器人 + 自定义 OpenAI 兼容 LLM API（vLLM）」的对接方案与**脱敏模板**。

## 重要说明

- 仓库里的示例文件已**脱敏**，不包含真实的 `apiKey`、`clientSecret`、网关 `token`。
- 你机器上**正在运行**的配置仍在 `~/.openclaw/`（如 `openclaw.json`、agent 下的 `models.json` 等），**修改本仓库里的模板不会影响线上服务**；上线时请把占位符替换为你自己的密钥并合并到实际配置中。

## 1. vLLM 启动参数（建议必配）

LLM 需支持 OpenAI 兼容接口：`POST /v1/chat/completions`。当 OpenClaw 发起带 `tools` 且 `tool_choice: "auto"` 的请求时，若 vLLM 未按规范启动，可能报错：

> `"auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set`

建议在 vLLM 启动命令中增加：

- `--enable-auto-tool-choice`
- `--tool-call-parser`

详细说明与 `curl` 自测示例见：`vllm-openai-compatible.md`。

## 2. OpenClaw：自定义 API（示例 provider `87`）

在 `~/.openclaw/openclaw.json` 中参考片段文件：`openclaw.json.snippet-qqbot-custom-api.json`。

要点：

- `models.providers.87.baseUrl` 填 **`http://127.0.0.1:8001/v1`**（**不要**写成带 `/chat` 的路径，否则可能与客户端拼接路径重复导致 404）。
- `models.providers.87.api`：`openai-completions`。
- `models.providers.87.apiKey`：填你的 LLM 侧 Bearer Token（模板里为占位符）。
- `agents.defaults.model.primary` 指向例如：`87/DeepSeek-R1-Distill-Qwen-14B`（与你在 provider 里登记的模型 id 一致）。

若 QQ 仍走 agent 且使用了**缓存**的模型配置，请同时核对 `~/.openclaw/agents/main/agent/models.json` 中对应 provider 的 `baseUrl` / `apiKey` 是否与 `openclaw.json` 一致，可参考：`models.json.snippet-provider-87.json`。

## 2.1 工具权限（`tools.profile`）

若需要完整工具能力（与 `openclaw config set tools.profile full` 等价），在 `~/.openclaw/openclaw.json` 中加入：

```json
"tools": {
  "profile": "full"
}
```

片段文件 `openclaw.json.snippet-qqbot-custom-api.json` 已包含上述字段。修改后请**重启网关**再生效（例如 `openclaw gateway restart`；若你使用的 CLI 版本不支持子命令，可用 systemd：`systemctl --user restart openclaw`，以本机实际安装为准）。

## 3. QQ 机器人

在同一套 `openclaw.json` 片段中配置 `channels.qqbot`：

- `appId`、`clientSecret`：在 QQ 开放平台获取的机器人凭据（勿提交到公开仓库）。

## 4. 上下文溢出（Context overflow）

当会话历史过长时，可能出现：

`Context overflow: prompt too large for the model. Try /reset (or /new)...`

处理建议：

1. 在 QQ 对话中发送 **`/reset`** 或 **`/new`** 开启新会话。
2. 避免在同一会话里反复粘贴超长日志或整段上下文。

---

**英文说明**：见同目录下的 `README.md`。
