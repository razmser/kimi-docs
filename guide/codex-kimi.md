> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Use Kimi K3.1 Pro in Codex

> Connect Codex directly to Kimi K3.1 Pro through the Kimi Responses API: set an API key, edit `~/.codex/config.toml`, and verify the connection in both Codex Desktop and the CLI.

Kimi Open Platform natively supports the [Responses API](../api/responses.md) used by Codex, so Codex can use the `kimi-k3.1-pro` model directly — no protocol conversion or local proxy is required.

> [!NOTE]
>
> Codex CLI supports text and image input, but does not provide a native video input channel — you cannot submit a video file directly as multimodal input to the model. This is a limitation of Codex CLI's input layer, not of the Kimi K3.1 Pro model — the Kimi K3.1 Pro API natively supports video input. Call `kimi-k3.1-pro` directly as described in [Vision Input](use-kimi-vision-model.md) for full video understanding, with no manual frame extraction required.

## Prerequisites

Before you start, complete the following preparations. Follow the corresponding official instructions for installation and account-related operations; this guide does not repeat those procedures.

> [card] [Install Codex CLI](https://developers.openai.com/codex/cli)
>
> Follow the official Codex documentation and start Codex CLI at least once. To use the Codex desktop app, [download it from the official site](https://chatgpt.com/codex) or run `codex app` after installing the CLI.

> [card] [Create an API key](https://platform.kimi.ai/console/api-keys)
>
> Create and save an API key in Kimi Open Platform.

## Step 1: Configure the API key

Codex reads the API key from an environment variable. Do not write the key into `config.toml`. To keep the key out of your shell history, enter it as follows:

> [tab] macOS / Linux
>
> ```bash theme={null}
> echo "Paste your Kimi API key and press Enter (input is hidden):"
> read -s KIMI_API_KEY
> export KIMI_API_KEY
> ```
>
> This only applies to the current terminal session. To persist it, add the `export` command to `~/.zshrc` (or `~/.bashrc` if you use bash on Linux). The file stores the key in plain text — set permissions accordingly.

> [tab] Windows (PowerShell)
>
> ```powershell theme={null}
> $env:KIMI_API_KEY="YOUR_KIMI_API_KEY"
> ```
>
> To persist it across sessions, add `KIMI_API_KEY` under **Settings > System > About > Advanced system settings > Environment Variables**.

## Step 2: Add Kimi as a model provider

Open `~/.codex/config.toml` (on Windows: `%USERPROFILE%\.codex\config.toml`) and add the following configuration. If `model` or `model_provider` already exist, replace their values:

```toml theme={null}
model = "kimi-k3.1-pro"
model_provider = "kimi"
model_context_window = 1048576

[model_providers.kimi]
name = "Kimi"
base_url = "https://api.moonshot.ai/v1"
env_key = "KIMI_API_KEY"
wire_api = "responses"
```

| Setting | Type | Purpose |
| - | - | - |
| `wire_api = "responses"` | `string` | Connects to Kimi through the native Responses API — the key setting for the direct connection |
| `env_key = "KIMI_API_KEY"` | `string` | Name of the environment variable Codex reads the API key from |
| `model_context_window = 1048576` | `integer` | Matches the 1M context window of `kimi-k3.1-pro`; without it, Codex falls back to default model metadata, which can degrade performance |

## Use Codex Desktop

After completing the API key and provider configuration above, quit and restart Desktop so it reloads `~/.codex/config.toml`.

After Desktop starts, open the model picker and select `kimi-k3.1-pro`. The interface may show **Custom**, but requests still use the `kimi-k3.1-pro` you configured.

![The Desktop composer shows the "Custom" model label](https://mintcdn.com/moonshotai/vwk1GToU6RD4oxdy/assets/pics/codex-kimi/desktop-custom-provider.png?fit=max\&auto=format\&n=vwk1GToU6RD4oxdy\&q=85\&s=480b290d9937f00555e18c267dad6991)

Send a simple request like `hi` — a normal reply means the basic connection works:

![Kimi replies to a simple greeting in Desktop](https://mintcdn.com/moonshotai/vwk1GToU6RD4oxdy/assets/pics/codex-kimi/desktop-hello-verify.png?fit=max\&auto=format\&n=vwk1GToU6RD4oxdy\&q=85\&s=3cff1702002ee35765243561141feff1)

Next, send a task that exercises Codex's agent capabilities:

```text theme={null}
Inspect this repository and summarize its structure.
```

If Desktop continues generating a final answer after the tool results come back, model calls and tool calling are working properly.

## Use Codex CLI

Codex CLI shares the same user-level configuration as Desktop, so the configuration above applies to it as well. Enter your project directory and start Codex (if Codex CLI is already running, exit the current session first so it reloads the configuration):

```bash theme={null}
cd /path/to/your/project
codex
```

After startup, confirm that Codex CLI shows `kimi-k3.1-pro` as the current model:

![Confirm kimi-k3.1-pro as the current model in Codex CLI](https://mintcdn.com/moonshotai/GuRAyEnWJ2mAZCpu/assets/pics/codex-kimi/verify-codex-cli.png?fit=max\&auto=format\&n=GuRAyEnWJ2mAZCpu\&q=85\&s=9fc744c9a811045db4e8f4bc28d47284)

Send a simple request (for example `hello`). A normal reply confirms that Codex is connected through the Kimi Responses API.

Under the hood, Codex sends requests to `POST https://api.moonshot.ai/v1/responses`. For request and response schema details, see the [Responses API reference](../api/responses.md).

## Troubleshooting

> [accordion] 401 Unauthorized
>
> The API key is invalid, or the key and the base\_url belong to different platforms — API keys created on [platform.kimi.ai](https://platform.kimi.ai) only work with `https://api.moonshot.ai/v1`. Also confirm that `KIMI_API_KEY` is available in the environment used to start Codex or Desktop. For CLI, check it in the terminal where you start Codex with `test -n "$KIMI_API_KEY" && echo set || echo missing`.

> [accordion] 400 web\_search.search\_context\_size is not supported
>
> The request includes the `search_context_size` parameter, which is not supported yet — remove it. Codex does not send this parameter by default, and the built-in web\_search tool works out of the box.

> [accordion] 404 on /v1/responses
>
> base\_url is wrong — make sure it is exactly `https://api.moonshot.ai/v1` (with the /v1 suffix). If you previously connected through CC Switch or another local router, also confirm base\_url no longer points at a local address such as `http://127.0.0.1:...`.

> [accordion] 429 Rate Limit
>
> You have hit a rate or concurrency limit. See [Rate limits](../pricing/limits.md) for your tier's quotas.

> [accordion] Warning Model metadata for kimi-k3.1-pro not found
>
> kimi-k3.1-pro is not in Codex's built-in model catalog. This warning is expected and does not affect usage — the `model_context_window = 1048576` from Step 2 already ensures the context window is treated as 1M.

> [accordion] Configuration changes do not take effect
>
> Codex only reads config.toml at startup — exit and restart it. Also confirm you edited `~/.codex/config.toml` itself and that no `-c` flags or profiles are overriding it. If you previously connected through CC Switch, also turn off **Codex** under its **Settings > Routing** — otherwise it keeps rewriting config.toml and overwrites the new configuration.

> [accordion] Other HTTP errors
>
> See [Error codes](../api/errors.md) for the meaning of each status code.
