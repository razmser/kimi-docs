> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Features overview

> Explore Kimi Open Platform capabilities for model inference, tool calling, integrations, and operations, with links to the relevant guides.

Use this page to discover the capabilities available on Kimi Open Platform and open the relevant documentation. If you are new to the platform, start with the [Quickstart](get-your-api-key.md). For endpoint definitions, see the [API overview](api/overview.md).

See each linked page for supported models, API formats, regional availability, and pricing.

## Model inference features

### Model capabilities

| Feature | Description |
| - | - |
| [Thinking models](guide/use-thinking-models.md) | Use reasoning for complex tasks, multi-step tool calls, and agent workflows, while preserving reasoning content correctly across conversation turns. |
| ↳ [Reasoning effort](guide/use-reasoning-effort.md) | Adjust reasoning depth, latency, and token usage with the levels supported by each model. |
| [Multi-turn conversations](guide/engage-in-multi-turn-conversations-using-kimi-api.md) | Maintain message history and control context length with a stateless API. |
| [Streaming output](guide/utilize-the-streaming-output-feature-of-kimi-api.md) | Receive incremental responses over SSE to reduce time to first token and process output as it arrives. |
| [Multimodal input](guide/use-kimi-vision-model.md) | Send text, images, and videos to supported Kimi models. |
| [Structured Output](guide/response_format.md) | Constrain response fields and types with JSON Schema for reliable downstream parsing. |
| [JSON Mode](guide/use-json-mode-feature-of-kimi-api.md) | Require a valid JSON object without constraining its exact field structure. |
| [Partial Mode](guide/use-partial-mode-feature-of-kimi-api.md) | Continue a given assistant prefix, fix the opening of a response, or resume truncated output. |
| [Context caching](guide/context-caching.md) | Reuse repeated request prefixes and reduce input costs with a 5m or 1h cache TTL. |

### API formats

| Feature | Description |
| - | - |
| [Chat Completions API](api/chat.md) | Use the OpenAI-compatible format for conversations, structured output, and tool calling. |
| [Responses API](api/responses.md) | Use the OpenAI Responses-compatible format to generate text or JSON and call function tools or server-side web search. |
| [Messages API](api/messages.md) | Call Kimi with the Anthropic Messages-compatible format through tools such as the Anthropic SDK and Claude Code. |

### Tool calling

| Feature | Description |
| - | - |
| [Custom tool calling](guide/use-kimi-api-to-complete-tool-calls.md) | Define function tools, execute model-requested calls in your application, and return the results to the model. |
| ↳ [Tool choice](guide/use-tool-choice.md) | Use `tool_choice` to let the model select tools automatically, require a call, or disable tool use. |
| ↳ [Dynamic tool loading](guide/use-dynamic-tool-loading.md) | Append tool definitions on demand to reduce token usage and preserve prefix caching. |
| ↳ [Kimi K3 tool calling best practices](guide/kimi-k3-tool-calling-best-practice.md) | Design tool workflows with dynamic loading, `tool_choice`, and reasoning effort. |
| [Official tools](guide/use-official-tools.md) | Configure and call official Kimi Open Platform tools through the Chat Completions API. |
| [Web search](guide/best-practices-for-web-search.md) | Choose a search API, control costs, and improve queries and parameters for your scenario. |
| ↳ [Web Search Basic](api/tools-search.md) | Get structured search results such as page titles, summaries, sites, and URLs through a standalone endpoint. |
| ↳ [Web Search Pro](api/tools-search-pro.md) | Get relevance-ranked content chunks and restrict results by site and time range. |
| [URL Fetch](api/tools-fetch.md) | Fetch the title and Markdown body of a specified web page. |

### Core workflows

| Feature | Description |
| - | - |
| [File-based Q\&A](guide/use-kimi-api-for-file-based-qa.md) | Upload and extract file content, then add it to a conversation for single-file or multi-file Q\&A. |
| ↳ [Files API](api/files-upload.md) | Upload files and retrieve file IDs, with endpoints to list, retrieve, download, and delete files. |
| [Batch API](guide/use-batch-api.md) | Upload JSONL requests and asynchronously create, monitor, and download large-scale batch results. |
| [Console batch inference](guide/use-batch-inference.md) | Create, monitor, and download batch inference results from the console without writing code. |
| [Automatic reconnection](guide/auto-reconnect.md) | Reconnect interrupted streaming requests and use Partial Mode to resume generation. |
| [Token counting](api/estimate.md) | Estimate the token count of message content before sending a request. |

### Integrations

| Feature | Description |
| - | - |
| [Kimi Code CLI](guide/kimi-code-cli.md) | Connect Kimi Code CLI with an Open Platform API key, switch models, and update the CLI. |
| [Codex](guide/codex-kimi.md) | Connect Codex to Kimi through the Responses API and verify your configuration. |
| [Claude Code](guide/claude-code-kimi.md) | Connect Kimi API to Claude Code and verify model selection and configuration. |
| [OpenClaw](guide/use-kimi-in-openclaw.md) | Connect Kimi API in OpenClaw and build cross-platform agents. |
| [OpenCode](guide/open-code.md) | Connect Kimi models in OpenCode and configure reasoning effort. |
| [Hermes Agent](guide/use-kimi-in-hermes-agent.md) | Connect Kimi in Hermes Agent for text and multimodal understanding. |
| [ModelScope MCP](guide/configure-the-modelscope-mcp-server.md) | Sync and enable ModelScope-hosted MCP servers in Playground. |

### Debugging and operations

| Feature | Description |
| - | - |
| [Playground](guide/use-playground-to-debug-the-model.md) | Compare models, adjust parameters, test tool calls, and convert debugging results into API requests. |
| [MoonPalace](guide/use-moonpalace.md) | Debug Kimi API requests and manage conversations, parameters, files, tool calls, and context caching. |
| [Organization, project, and API key management](guide/org-best-practice.md) | Manage organization and project hierarchies, budgets, rate limits, members, IP allowlists, and usage analytics. |
| [Admin API](guide/org-admin-api.md) | Automate member, project, and key management, plus billing, usage, and audit queries, in your existing enterprise systems. |
| [Zero Data Retention (ZDR)](guide/zero-data-retention.md) | Learn about the zero data retention mode available to enterprise customers. |

## Related indexes

| Index | Description |
| - | - |
| [Model list](models.md) | View currently available models and migration guidance for retired models. |
| [API overview](api/overview.md) | Review service URLs, authentication, API compatibility, and the complete endpoint list. |
| [Model parameter reference](api/models-overview.md) | Compare default parameters, supported ranges, and usage constraints across model families. |
| [Model inference pricing](pricing/chat.md) | Review pricing for model tokens and context caching. |
| [Top-up and rate limits](pricing/limits.md) | Review top-up methods, rate limits, and related rules. |
| [Troubleshooting](guide/troubleshooting.md) | Resolve common issues with accounts, billing, authentication, model parameters, rate limits, and connectivity. |
