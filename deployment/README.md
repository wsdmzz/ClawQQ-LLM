## claw-qq-llm 集成（脱敏模板 + 运行说明）

**中文说明**：[README.zh-CN.md](README.zh-CN.md)

本目录用于保存你在服务器上完成的「OpenClaw 网关 + QQ 机器人 + 自定义 OpenAI 兼容 LLM API（vLLM）」对接方案。

注意：

- 此处文件已脱敏，不包含真实 `apiKey` / `clientSecret` / gateway token。
- 你的 OpenClaw 真实运行配置仍在服务器的 `~/.openclaw/` 下生效，本目录只提供“模板/片段/说明”用于同步与复现。

### Verified stack (vLLM tool calling + OpenClaw + QQ)

This repo documents a working setup:

| Part      | Notes                                                                                                            |
| --------- | ---------------------------------------------------------------------------------------------------------------- |
| Model     | **Qwen2.5-14B-Instruct-AWQ** (`--served-model-name` must match OpenClaw model id)                                |
| vLLM      | `--enable-auto-tool-choice` + `--tool-call-parser hermes`                                                        |
| Self-test | Both **`tool_choice: required`** and **`auto`** return proper **`message.tool_calls`**                           |
| OpenClaw  | Provider `api`: **`openai-completions`**, `baseUrl`: **`http://127.0.0.1:8001/v1`**, `tools.profile`: **`full`** |
| QQ        | `channels.qqbot` per snippet; restart gateway after changes                                                      |

**Note:** `Qwen2.5-Coder-*` as the primary model often emits tool-like text in `content` instead of structured `tool_calls`. Prefer **Instruct** for local 14B + agents. See `vllm-openai-compatible.md` for curl and serve examples.

## 1) vLLM 启动参数（必须）

你的 LLM 后端需要支持 OpenAI 兼容的 `POST /v1/chat/completions`，并且当请求里携带 `tools`、`tool_choice: "auto"` 时不能报错：

> `"auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set`

因此建议启动时显式加入：

- `--enable-auto-tool-choice`
- `--tool-call-parser` (**Qwen2.5**: **`hermes`**; other families per vLLM docs)

参考 `deployment/vllm-openai-compatible.md`。

## 2) OpenClaw 配置：provider 87（自定义 OpenAI 兼容 API）

在你的 `~/.openclaw/openclaw.json` 里，按 `deployment/openclaw.json.snippet-qqbot-custom-api.json` 对 provider 87 配置：

- `models.providers.87.baseUrl`：使用 **`http://127.0.0.1:8001/v1`**（不要带 `/chat`）
- `models.providers.87.api`：`openai-completions`
- `models.providers.87.apiKey`：填入你的 LLM Bearer token（示例中用占位符）
- 确保 `agents.defaults.model.primary` 指向：`87/Qwen2.5-14B-Instruct-AWQ`（与 vLLM `--served-model-name` 一致）

### 2.1) Tools profile（完整工具权限）

若需要完整工具能力，在 `~/.openclaw/openclaw.json` 中设置 `tools.profile` 为 `full`（与 `openclaw config set tools.profile full` 等价）。模板 `openclaw.json.snippet-qqbot-custom-api.json` 已包含该字段。

修改后请**重启 Gateway** 再生效（例如 `openclaw gateway restart`，或你环境中的 systemd / 进程管理方式）。

## 3) QQ 机器人接入

按 `deployment/openclaw.json.snippet-qqbot-custom-api.json` 配置 `channels.qqbot`：

- `appId` / `clientSecret` 填入你自己的 QQ 机器人凭据

## 4) 遇到 Context overflow（上下文过长）

当 QQ 对话历史积累很长时，模型会返回类似：
`Context overflow: prompt too large for the model. Try /reset (or /new)...`

处理方式：

1. 在当前 QQ 会话里发送 `/reset` 或 `/new` 清理会话上下文
2. 尽量不要把超长的调试/上下文原文反复塞进同一会话线程
