# ChatGPT 中转站生图 Skill

> 为任何兼容 OpenAI API 的中转站或 API 代理创建专用的 Codex 生图 Skill。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![OpenAI Compatible](https://img.shields.io/badge/API-OpenAI%20Compatible-10a37f)](#支持的接口)
[![Documentation](https://img.shields.io/badge/docs-English-blue)](README.md)
[![简体中文](https://img.shields.io/badge/语言-简体中文-red)](README.zh-CN.md)

[English](README.md) · **简体中文**

## 项目简介

通过兼容 OpenAI API 的中转站使用聊天和编程模型时，普通请求可能完全正常，但生图仍然报错：

```text
OPENAI_API_KEY was not found
```

本仓库提供一段可以直接复制的提示词，用来创建一个专门调用中转站图片接口的 Codex Skill。生成的 Skill 会使用中转站地址，从指定的环境变量读取 API Key，同时支持文生图、参考图编辑，并将返回的图片保存到本地。

本仓库**不提供**中转服务、API Key、账号或已经生成好的 Skill，只提供一种更安全、可复用的 Skill 创建方法。

## 当前痛点

中转站通常只作为聊天或编程模型的提供商进行配置，但图片工作流可能仍然：

- 只读取 `OPENAI_API_KEY`，没有使用中转站的凭据；
- 请求 OpenAI 官方图片接口，而不是中转站接口；
- 无法自动发现 `/v1/images/generations` 和 `/v1/images/edits`；
- 把图片编辑请求错误地发送为 JSON，而不是 `multipart/form-data`；
- 只处理图片 URL，却没有处理 `b64_json`，或者情况相反；
- Codex 桌面端没有读取到在另一个终端会话中设置的环境变量。

因此会出现一种很容易混淆的情况：中转站在 Codex 中聊天正常，但生图却提示缺少 Key、鉴权失败或接口不受支持。

## 解决思路

创建一个独立的 Codex Skill，完整接管图片调用流程：

1. 调用中转站准确的生图接口。
2. 用户提供参考图时调用准确的图片编辑接口。
3. 从明确指定的环境变量读取中转站 API Key。
4. 根据接口分别发送 JSON 或 multipart 请求。
5. 同时支持 Base64 和 URL 图片结果。
6. 返回所有生成文件的绝对路径。

## 支持的接口

该方法适用于提供 OpenAI 兼容 Images API 的中转站、API 代理和自建网关。

常见接口格式：

```text
POST https://your-relay.example/v1/images/generations
POST https://your-relay.example/v1/images/edits
```

具体模型名称和可用参数取决于你的服务商。

## 准备信息

从中转站后台或接口文档中获取：

| 信息 | 示例 |
| --- | --- |
| 生图接口 | `https://your-relay.example/v1/images/generations` |
| 图片编辑接口 | `https://your-relay.example/v1/images/edits` |
| API Key | 在中转站后台创建用户 Key |
| 默认模型 | `gpt-image-2` 或其他受支持的图片模型 |
| Skill 名称 | `relay-image-generation` |
| 输出目录 | `output/relay-images` |

同时确认 API Key 所属分组已经开启图片生成权限。可以聊天的 Key 不一定可以调用图片模型。

## 在本机保存 API Key

不要把真实 API Key 写入本仓库、提示词、`SKILL.md` 或脚本。应在本机设置环境变量：

```bash
export RELAY_IMAGE_API_KEY="your-relay-api-key"
```

如果 Codex Desktop 是从 macOS 图形界面启动的，它可能无法继承终端变量。可以为 macOS 启动的应用设置变量，然后完全退出并重新打开 Codex：

```bash
launchctl setenv RELAY_IMAGE_API_KEY "your-relay-api-key"
```

建议使用密码管理工具，避免在 Shell 历史记录中保存密钥；如果密钥意外泄露，应立即轮换。

## 创建 Skill

1. 替换下方提示词中的全部 `{{PLACEHOLDER}}`。
2. 把修改后的提示词粘贴到 Codex。
3. 允许 Codex 创建并验证全局 Skill。
4. 如果新 Skill 没有立即出现，重新启动 Codex。
5. 确认接口、模型和计费设置后，再使用低成本提示词进行测试。

## 可复制的提示词

请直接使用下面的英文提示词：

````text
Create or update a global Codex skill named {{SKILL_NAME}} for image generation and image editing through an OpenAI-compatible relay.

Provider configuration:

- Image generation endpoint: {{GENERATION_ENDPOINT}}
- Image editing endpoint: {{EDIT_ENDPOINT}}
- API key environment variable: {{API_KEY_ENV_VAR}}
- Default image model: {{DEFAULT_MODEL}}
- Default output directory: {{OUTPUT_DIRECTORY}}

Before making changes, check whether a skill with the same name already exists. If it exists, inspect it and update it without removing unrelated functionality.

Build the skill using the standard Codex skill structure:

- `SKILL.md` with valid YAML frontmatter containing only `name` and `description`
- `agents/openai.yaml` with an English display name, short description, and default prompt
- a reusable script in `scripts/` that uses only the Python standard library when practical

Functional requirements:

1. Use the generation endpoint for text-only image requests.
2. Use the editing endpoint when the user supplies one or more reference images or asks to modify an existing image.
3. Send generation requests as `application/json`.
4. Send editing requests as `multipart/form-data` and upload the source image correctly.
5. Authenticate with `Authorization: Bearer <key>`, reading the key only from `{{API_KEY_ENV_VAR}}`.
6. Never write the API key into the skill, source code, logs, generated files, terminal output, or chat messages.
7. If the environment variable is missing, stop and clearly explain how to set `{{API_KEY_ENV_VAR}}`. Do not fall back to an unrelated account automatically.
8. Support these optional parameters when supplied by the user: `model`, `size`, `quality`, `n`, `output_format`, and `background`.
9. Default to `{{DEFAULT_MODEL}}` only when the user does not specify a model. Do not silently replace unsupported user values.
10. Support both common response formats:
    - decode and save `data[].b64_json`;
    - download and save `data[].url`.
11. Create `{{OUTPUT_DIRECTORY}}` automatically, avoid overwriting existing files, and return the absolute path of every saved image.
12. Surface the provider's HTTP status and error message when a request fails, but redact credentials and sensitive headers.
13. Warn before sending credentials to a public non-HTTPS endpoint.
14. Validate the completed skill with the official skill validator.
15. Test parsing, multipart construction, base64 saving, and URL saving locally without making a paid API request. Do not make a real image request unless I explicitly approve it.

Include concise English usage examples for both text-to-image generation and reference-image editing.
````

### 占位符示例

```text
{{SKILL_NAME}}          = relay-image-generation
{{GENERATION_ENDPOINT}} = https://your-relay.example/v1/images/generations
{{EDIT_ENDPOINT}}       = https://your-relay.example/v1/images/edits
{{API_KEY_ENV_VAR}}     = RELAY_IMAGE_API_KEY
{{DEFAULT_MODEL}}       = gpt-image-2
{{OUTPUT_DIRECTORY}}    = output/relay-images
```

## 常见问题

### `OPENAI_API_KEY was not found`

确认生成的 Skill 读取的是提示词中指定的变量，例如 `RELAY_IMAGE_API_KEY`。如果设置变量时 Codex Desktop 已经在运行，请完全退出后重新打开。

### HTTP 401 或 403

检查中转站 Key、所属分组、账号状态、余额和图片生成权限。不要默认认为可以聊天的 Key 一定可以生图。

### HTTP 404

检查完整接口路径。聊天 Base URL 并不是生图接口。多数兼容服务使用 `/v1/images/generations` 和 `/v1/images/edits`。

### 找不到模型或模型不受支持

检查中转站模型列表和模型映射，把默认模型改成 API Key 所属分组实际支持的模型名称。

### 文生图正常，但图片编辑失败

确认编辑请求使用 `multipart/form-data`、源图片确实存在，并且所选模型支持图片编辑。

### 接口成功但没有保存图片

确认服务商返回的是 `data[].b64_json`、`data[].url` 还是自定义结构。如果不是 OpenAI 兼容格式，需要相应修改生成的脚本。

## 安全提示

- 公网中转站必须优先使用 HTTPS。
- 不要提交 API Key、OAuth Token、Cookie 或账号导出文件。
- 为不同用户创建独立 API Key 并设置合理额度。
- 密钥泄露后立即轮换。
- 运行生成的脚本前先检查代码。
- 中转站运营者可能看到图片提示词和上传的参考图。

## 许可证

本项目采用 [MIT License](LICENSE)。
