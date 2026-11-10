> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# API Overview

> Review Kimi API base URLs, authentication, request conventions, compatibility, and links to the main API endpoints.

## Service Address

```
https://api.moonshot.ai
```

The `base_url` differs by compatible protocol; see "Protocol Compatibility" below.

## Protocol Compatibility

The Kimi API is compatible with three API formats, so you can reuse the corresponding official SDKs and tools:

| Compatible API | base\_url | Endpoint | SDKs / Tools |
| - | - | - | - |
| OpenAI Chat Completions | `https://api.moonshot.ai/v1` | [`/chat/completions`](chat.md) | Official OpenAI SDKs (Python / Node.js), plus third-party tools and frameworks such as LangChain, Dify, and Coze |
| OpenAI Responses | `https://api.moonshot.ai/v1` | [`/responses`](responses.md) | Official OpenAI SDKs (Python / Node.js) |
| Anthropic Messages | `https://api.moonshot.ai/anthropic` | [`/messages`](messages.md) | Official Anthropic SDKs, Claude Code, and similar tools |

If your project already uses the official OpenAI or Anthropic APIs, simply replace `base_url` and the API Key with your Kimi configuration to migrate — no other code changes are needed.

> [!NOTE]
>
> When using the OpenAI-compatible endpoints, some parameters are Kimi-specific extensions: the `thinking` parameter needs to be passed via the SDK's `extra_body`; `partial` is a field on assistant messages within the messages array (`"partial": true`), not a top-level request parameter. See [Tool Use](https://platform.kimi.ai/docs/api/tool-use) and [Partial Mode](https://platform.kimi.ai/docs/api/partial) for details.

## Authentication

All API requests require an API Key in the HTTP header:

```
Authorization: Bearer $MOONSHOT_API_KEY
```

API Keys can be created and managed in the [Kimi Open Platform Console](https://platform.kimi.ai/console/api-keys).

> [!WARNING]
>
> Your API Key is sensitive. Do not expose it in client-side code, public repositories, or logs. Use environment variables to manage it.

When using an official SDK, pass your API Key and the protocol-specific `base_url` at initialization.

OpenAI-compatible endpoints:

```python Python theme={null}
import os

from openai import OpenAI

client = OpenAI(
    api_key=os.environ["MOONSHOT_API_KEY"],
    base_url="https://api.moonshot.ai/v1",
)
```

```javascript Node.js theme={null}
const OpenAI = require("openai");

const client = new OpenAI({
    apiKey: process.env.MOONSHOT_API_KEY,
    baseURL: "https://api.moonshot.ai/v1",
});
```

Anthropic-compatible endpoints:

```python Python theme={null}
import os

from anthropic import Anthropic

client = Anthropic(
    api_key=os.environ["MOONSHOT_API_KEY"],
    base_url="https://api.moonshot.ai/anthropic",
)
```

```javascript Node.js theme={null}
const Anthropic = require("@anthropic-ai/sdk");

const client = new Anthropic({
    apiKey: process.env.MOONSHOT_API_KEY,
    baseURL: "https://api.moonshot.ai/anthropic",
});
```

## Error Handling

Failed requests return a JSON error response with `error.type` and `error.message` fields. For the full list of error types, messages, and troubleshooting tips, see [Errors](errors.md).

## API Endpoints

| Endpoint | Method | Protocol | Description |
| - | - | - | - |
| `/v1/chat/completions` | POST | OpenAI | [Create Chat Completion](chat.md) |
| `/v1/responses` | POST | OpenAI | [Responses API](responses.md) |
| `/anthropic/v1/messages` | POST | Anthropic | [Messages API](messages.md) |
| `/v1/models` | GET | OpenAI | [List Models](list-models.md) |
| `/v1/tokenizers/estimate-token-count` | POST | OpenAI | [Estimate Tokens](estimate.md) |
| `/v1/users/me/balance` | GET | OpenAI | [Check Balance](balance.md) |
| `/v1/tools/search` | POST | Kimi | [Web Search Basic](tools-search.md) |
| `/v1/tools/search_pro` | POST | Kimi | [Web Search Pro](tools-search-pro.md) |
| `/v1/tools/fetch` | POST | Kimi | [URL Fetch](tools-fetch.md) |
| `/v1/files` | POST | OpenAI | [Upload File](files-upload.md) |
| `/v1/files` | GET | OpenAI | [List Files](files-list.md) |
| `/v1/files/{file_id}` | GET | OpenAI | [Get File Info](files-retrieve.md) |
| `/v1/files/{file_id}` | DELETE | OpenAI | [Delete File](files-delete.md) |
| `/v1/files/{file_id}/content` | GET | OpenAI | [Get File Content](files-content.md) |
| `/v1/batches` | POST | OpenAI | [Create Batch](batch-create.md) |
| `/v1/batches` | GET | OpenAI | [List Batches](batch-list.md) |
| `/v1/batches/{batch_id}` | GET | OpenAI | [Get Batch Details](batch-retrieve.md) |
| `/v1/batches/{batch_id}/cancel` | POST | OpenAI | [Cancel Batch](batch-cancel.md) |

## Next Steps

> [card] [Quickstart](https://platform.kimi.ai/docs/api/quickstart)
>
> Send your first API request

> [card] [Responses API](responses.md)
>
> OpenAI Responses-compatible endpoint

> [card] [Messages API](messages.md)
>
> Anthropic Messages-compatible endpoint

> [card] [List Models](list-models.md)
>
> See the currently available models
