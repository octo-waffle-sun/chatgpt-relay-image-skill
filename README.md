# ChatGPT Relay Image Skill

> Build a dedicated Codex image-generation skill for any OpenAI-compatible relay or API proxy.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![OpenAI Compatible](https://img.shields.io/badge/API-OpenAI%20Compatible-10a37f)](#supported-apis)
[![Documentation](https://img.shields.io/badge/docs-English-blue)](README.md)
[![简体中文](https://img.shields.io/badge/语言-简体中文-red)](README.zh-CN.md)

**English** · [简体中文](README.zh-CN.md)

## Overview

Chat and coding requests may work correctly through an OpenAI-compatible relay while image generation still fails with an error such as:

```text
OPENAI_API_KEY was not found
```

This repository provides a copy-and-paste prompt for creating a dedicated Codex skill that calls the relay's image endpoints directly. The generated skill uses the relay URL, reads a user-selected API-key environment variable, supports image generation and editing, and saves returned images locally.

This repository does **not** provide an API service, API key, account, or prebuilt skill. It provides a safer, reusable method for asking Codex to build the skill for your own relay.

## The Problem

An OpenAI-compatible relay is often configured as the model provider for chat or coding. Image generation can still fail because the image workflow may:

- expect `OPENAI_API_KEY` instead of the relay credential used by the configured model provider;
- use the official OpenAI image endpoint instead of the relay endpoint;
- fail to discover `/v1/images/generations` and `/v1/images/edits` automatically;
- send an editing request as JSON instead of `multipart/form-data`;
- receive `b64_json` while expecting a downloadable URL, or the reverse;
- run in the Codex desktop environment without access to variables exported in a different shell.

The result is confusing: the relay works for normal Codex requests, but image generation reports a missing key, an authentication error, or an unsupported endpoint.

## The Solution

Create a dedicated Codex skill that owns the complete image workflow:

1. Call the relay's exact image-generation endpoint.
2. Call the relay's exact image-editing endpoint for reference images.
3. Read the relay key from an explicit environment variable.
4. Send the correct JSON or multipart request format.
5. Save both base64 and URL responses locally.
6. Return the absolute paths of generated files.

## Supported APIs

The method is designed for services that expose OpenAI-compatible Images API routes, including relay services, API proxies, and self-hosted gateways.

Typical routes are:

```text
POST https://your-relay.example/v1/images/generations
POST https://your-relay.example/v1/images/edits
```

The exact model names and supported parameters depend on your provider.

## Preparation

Collect the following information from your relay dashboard or documentation:

| Value | Example |
| --- | --- |
| Generation endpoint | `https://your-relay.example/v1/images/generations` |
| Editing endpoint | `https://your-relay.example/v1/images/edits` |
| API key | Create a user key in the relay dashboard |
| Default model | `gpt-image-2` or another supported image model |
| Skill name | `relay-image-generation` |
| Output directory | `output/relay-images` |

Confirm that the API-key group is allowed to use image generation. A valid chat key does not always have permission to use an image model.

## Store the API Key Locally

Do not paste a real API key into this repository, the prompt, `SKILL.md`, or a script. Store it in a local environment variable instead:

```bash
export RELAY_IMAGE_API_KEY="your-relay-api-key"
```

If Codex Desktop was opened from the macOS graphical interface, it may not inherit variables from your terminal. Set the variable for applications launched by macOS, then fully restart Codex:

```bash
launchctl setenv RELAY_IMAGE_API_KEY "your-relay-api-key"
```

Use a secret manager when possible, avoid saving secrets in shell history, and rotate any key that is accidentally exposed.

## Create the Skill

1. Replace every `{{PLACEHOLDER}}` in the prompt below.
2. Paste the completed prompt into Codex.
3. Allow Codex to create and validate the global skill.
4. Restart Codex if the new skill is not immediately discovered.
5. Test with a low-cost prompt after confirming the endpoint, model, and billing settings.

## Copy-and-Paste Prompt

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

### Example Placeholder Values

```text
{{SKILL_NAME}}          = relay-image-generation
{{GENERATION_ENDPOINT}} = https://your-relay.example/v1/images/generations
{{EDIT_ENDPOINT}}       = https://your-relay.example/v1/images/edits
{{API_KEY_ENV_VAR}}     = RELAY_IMAGE_API_KEY
{{DEFAULT_MODEL}}       = gpt-image-2
{{OUTPUT_DIRECTORY}}    = output/relay-images
```

## Troubleshooting

### `OPENAI_API_KEY was not found`

Confirm that the generated skill reads the environment variable named in the prompt, such as `RELAY_IMAGE_API_KEY`. If Codex Desktop was already running when the variable was set, fully quit and reopen the application.

### HTTP 401 or 403

Verify the relay key, its assigned group, account status, balance, and image-generation permissions. Do not assume that a key authorized for chat is authorized for images.

### HTTP 404

Verify the complete endpoint path. The base chat URL is not the generation endpoint. Most compatible services use `/v1/images/generations` and `/v1/images/edits`.

### Model not found or unsupported

Check the relay's model list and image-model mapping. Replace the default model with the exact model name supported by the API-key group.

### Image editing fails while generation works

Confirm that the editing request uses `multipart/form-data`, the source file exists, and the provider supports image editing for the selected model.

### The response contains no saved image

Inspect whether the provider returns `data[].b64_json`, `data[].url`, or a provider-specific schema. Update the generated script only if the relay is not actually OpenAI-compatible.

## Security Notes

- Use HTTPS for public relay endpoints.
- Never commit API keys, OAuth tokens, cookies, or exported account files.
- Give each user a separate relay API key with an appropriate quota.
- Rotate keys after accidental disclosure.
- Review generated scripts before running them.
- Treat image prompts and uploaded reference images as data visible to the relay operator.

## License

Released under the [MIT License](LICENSE).
