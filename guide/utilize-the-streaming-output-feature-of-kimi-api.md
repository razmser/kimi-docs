> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Use the Streaming Feature of the Kimi API

> Use Kimi API server-sent events to reduce time to first token, parse incremental output, and capture final usage data.

After receiving a question, the Kimi large language model first performs inference and then generates the answer one Token at a time. Streaming sends Tokens to the client as soon as a certain number of them (usually 1 Token) is generated, instead of waiting until the full response is complete. Waiting for the complete response usually takes several seconds — for complex questions and long replies it can stretch to 10 or even 20 seconds; with streaming, users see the first Token immediately, which significantly reduces wait time. When you chat with [Kimi AI Assistant](https://kimi.ai), the reply appears character by character — that is streaming in action.

This page covers streaming for two APIs. Because their fields, event models, and terminal conditions differ, their parsing code is not interchangeable:

| Item | [Chat Completions](#chat-completions) | [Messages API](#messages-api) |
| - | - | - |
| Endpoint | `POST /v1/chat/completions` | `POST /anthropic/v1/messages` |
| Client options | OpenAI SDK or raw HTTP/SSE | Anthropic SDK or raw HTTP/SSE |
| Reasoning and text deltas | `delta.reasoning_content`, `delta.content` | `thinking_delta` (with `signature_delta`), `text_delta` |
| Tool argument deltas | `delta.tool_calls[].function.arguments` (assembled by tool `index`) | `input_json_delta.partial_json` (fine-grained tool streaming, assembled by content block `index`) |
| Terminal signal | `finish_reason` chunk followed by `data: [DONE]` in raw SSE | `message_delta` carrying `stop_reason`, followed by `message_stop` |

## Chat Completions

### Quickstart

Set `stream=True` (`"stream": true` in a JSON request body) to enable streaming. The OpenAI SDK returns an iterable that yields data chunks one by one. Each chunk has a structure similar to a non-streaming completion, except the `message` field is replaced by an incremental `delta` field. The following examples read `delta.content` and print answer text as it arrives:

The examples on this page use the latest model `kimi-k3.1-pro` by default. K3.1 Pro configures reasoning effort with the top-level `reasoning_effort` request field (supports `"low"` / `"high"` / `"max"`, default `"max"`). To use another model such as `kimi-k3.1-pro-highspeed`, replace the `model` field and adjust the parameter configuration for that model. See the [Model Parameter Reference](../api/models-overview.md).

> [tab] python
>
> ```python theme={null}
> import os
> from openai import OpenAI
>
> client = OpenAI(
>     api_key=os.environ["MOONSHOT_API_KEY"],  # Set the MOONSHOT_API_KEY environment variable before running this example
>     base_url="https://api.moonshot.ai/v1",
> )
>
> stream = client.chat.completions.create(
>     model="kimi-k3.1-pro",
>     messages=[
>         {"role": "user", "content": "Hello, briefly explain streaming."},
>     ],
>     stream=True,  # <-- Enable streaming output
> )
>
> for chunk in stream:
>     if not chunk.choices:
>         continue
>     content = chunk.choices[0].delta.content
>     if content:
>         print(content, end="")
> ```

> [tab] node.js
>
> ```js theme={null}
> const OpenAI = require("openai");
>
> const client = new OpenAI({
>     apiKey: process.env.MOONSHOT_API_KEY, // Set the MOONSHOT_API_KEY environment variable before running this example
>     baseURL: "https://api.moonshot.ai/v1",
> });
>
> async function main() {
>     const stream = await client.chat.completions.create({
>         model: "kimi-k3.1-pro",
>         messages: [
>             { role: "user", content: "Hello, briefly explain streaming." },
>         ],
>         stream: true, // <-- Enable streaming output
>     });
>
>     for await (const chunk of stream) {
>         if (chunk.choices.length === 0) continue;
>         const content = chunk.choices[0].delta.content;
>         if (content) process.stdout.write(content);
>     }
> }
>
> main();
> ```

### Read Reasoning Content

Thinking models emit `reasoning_content` before `content` and `tool_calls`. The OpenAI SDK type definitions do not declare this field; in Python, read it with `hasattr`/`getattr`. Reasoning content arrives in fragments, so append each fragment instead of overwriting previous content:

> [tab] python
>
> ```python theme={null}
> reasoning_content = ""
>
> for chunk in stream:
>     if not chunk.choices:
>         continue
>     delta = chunk.choices[0].delta
>     if hasattr(delta, "reasoning_content"):
>         fragment = getattr(delta, "reasoning_content")
>         if fragment:
>             reasoning_content += fragment
>             print(fragment, end="")
> ```

> [tab] node.js
>
> ```js theme={null}
> let reasoningContent = "";
>
> for await (const chunk of stream) {
>     if (chunk.choices.length === 0) continue;
>     const delta = chunk.choices[0].delta;
>     if (delta.reasoning_content) {
>         reasoningContent += delta.reasoning_content;
>         process.stdout.write(delta.reasoning_content);
>     }
> }
> ```

### Handle Tool Calls

When the model requests tool execution, `delta.tool_calls` carries tool-call fragments:

* Fragments of the same tool call share the same `index`; keep a separate buffer per `index` when multiple tools are called.
* `id`, `type`, and `function.name` appear only once, in the first fragment of that tool call.
* `function.arguments` arrives as JSON string fragments. Append the fragments without overwriting previous content, and parse the JSON only after the stream ends.

The following example shows how to reconstruct tool calls by `index` with the OpenAI SDK:

> [tab] python
>
> ```python expandable theme={null}
> import json
> import os
> from openai import OpenAI
>
> client = OpenAI(
>     api_key=os.environ["MOONSHOT_API_KEY"],  # Set the MOONSHOT_API_KEY environment variable before running this example
>     base_url="https://api.moonshot.ai/v1",
> )
>
> tools = [
>     {
>         "type": "function",
>         "function": {
>             "name": "get_weather",
>             "description": "Get the current weather of a city.",
>             "parameters": {
>                 "type": "object",
>                 "required": ["city"],
>                 "properties": {
>                     "city": {
>                         "type": "string",
>                         "description": "The name of the city, e.g. Beijing",
>                     }
>                 },
>             },
>         },
>     }
> ]
>
> stream = client.chat.completions.create(
>     model="kimi-k3.1-pro",
>     messages=[
>         {"role": "system", "content": "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated."},
>         {"role": "user", "content": "What is the weather like in Beijing today?"},
>     ],
>     tools=tools,
>     stream=True,
> )
>
> finish_reason = None
> reasoning_content = ""  # <-- Reasoning content from thinking models; preserve when returning tool results
> tool_calls = []         # <-- Reconstructed tool calls, one entry per index
>
> for chunk in stream:
>     if not chunk.choices:
>         continue
>
>     choice = chunk.choices[0]
>     delta = choice.delta
>     if choice.finish_reason:
>         finish_reason = choice.finish_reason
>
>     if hasattr(delta, "reasoning_content"):
>         fragment = getattr(delta, "reasoning_content")
>         if fragment:
>             reasoning_content += fragment
>
>     for tool_call in delta.tool_calls or []:
>         index = tool_call.index  # <-- Fragments of the same tool call share the same index
>         while len(tool_calls) <= index:
>             tool_calls.append({"id": "", "type": "", "name": "", "arguments": ""})
>         current = tool_calls[index]
>         if tool_call.id:  # <-- id, type, and name arrive only in the first fragment
>             current["id"] = tool_call.id
>         if tool_call.type:
>             current["type"] = tool_call.type
>         if tool_call.function:
>             if tool_call.function.name:
>                 current["name"] = tool_call.function.name
>             if tool_call.function.arguments:
>                 current["arguments"] += tool_call.function.arguments  # <-- Append, never overwrite
>
> if finish_reason == "tool_calls":
>     for tool_call in tool_calls:
>         arguments = json.loads(tool_call["arguments"])  # <-- Parse JSON only after all fragments are assembled
>         print(f"tool_call: {tool_call['id']} {tool_call['name']}({arguments})")
> ```

> [tab] node.js
>
> ```js expandable theme={null}
> const OpenAI = require("openai");
>
> const client = new OpenAI({
>     apiKey: process.env.MOONSHOT_API_KEY, // Set the MOONSHOT_API_KEY environment variable before running this example
>     baseURL: "https://api.moonshot.ai/v1",
> });
>
> const tools = [
>     {
>         type: "function",
>         function: {
>             name: "get_weather",
>             description: "Get the current weather of a city.",
>             parameters: {
>                 type: "object",
>                 required: ["city"],
>                 properties: {
>                     city: {
>                         type: "string",
>                         description: "The name of the city, e.g. Beijing",
>                     },
>                 },
>             },
>         },
>     },
> ];
>
> async function main() {
>     const stream = await client.chat.completions.create({
>         model: "kimi-k3.1-pro",
>         messages: [
>             { role: "system", content: "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated." },
>             { role: "user", content: "What is the weather like in Beijing today?" },
>         ],
>         tools: tools,
>         stream: true,
>     });
>
>     let finishReason;
>     let reasoningContent = ""; // <-- Reasoning content from thinking models; preserve when returning tool results
>     const toolCalls = [];      // <-- Reconstructed tool calls, one entry per index
>
>     for await (const chunk of stream) {
>         if (chunk.choices.length === 0) continue;
>
>         const choice = chunk.choices[0];
>         const delta = choice.delta;
>         if (choice.finish_reason) finishReason = choice.finish_reason;
>         if (delta.reasoning_content) reasoningContent += delta.reasoning_content;
>
>         for (const toolCall of delta.tool_calls ?? []) {
>             const index = toolCall.index; // <-- Fragments of the same tool call share the same index
>             while (toolCalls.length <= index) {
>                 toolCalls.push({ id: "", type: "", name: "", arguments: "" });
>             }
>             const current = toolCalls[index];
>             if (toolCall.id) current.id = toolCall.id; // <-- id, type, and name arrive only in the first fragment
>             if (toolCall.type) current.type = toolCall.type;
>             if (toolCall.function) {
>                 if (toolCall.function.name) current.name = toolCall.function.name;
>                 if (toolCall.function.arguments) {
>                     current.arguments += toolCall.function.arguments; // <-- Append, never overwrite
>                 }
>             }
>         }
>     }
>
>     if (finishReason === "tool_calls") {
>         for (const toolCall of toolCalls) {
>             const args = JSON.parse(toolCall.arguments); // <-- Parse JSON only after all fragments are assembled
>             console.log(`tool_call: ${toolCall.id} ${toolCall.name}(${JSON.stringify(args)})`);
>         }
>     }
> }
>
> main();
> ```

When `finish_reason` is `tool_calls`, the model is asking your application to execute tools. After executing them, append the assistant message (including its `tool_calls` and, for thinking models, the `reasoning_content` assembled as shown in the previous section — see [Preserved Thinking](use-thinking-models.md)) together with the tool results to `messages`, then call the API again. For the complete tool-definition, execution-loop, and result-return workflow, see [Use Kimi API for Tool Calls](use-kimi-api-to-complete-tool-calls.md). `tool_call.type` matches the declared tool type (`function` or `builtin_function`).

### Raw SSE Format and Reading Without an SDK

With streaming enabled, the endpoint returns `Content-Type: text/event-stream` ([SSE](https://kimi.ai/share/cr7boh3dqn37a5q9tds0)) instead of `Content-Type: application/json`, pushing frames to the client as tokens are generated. A raw SSE response body from `POST /v1/chat/completions` looks like this:

```text theme={null}
data: {"id":"cmpl-1305b94c570f447fbde3180560736287","object":"chat.completion.chunk","created":1698999575,"model":"kimi-k3.1-pro","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"cmpl-1305b94c570f447fbde3180560736287","object":"chat.completion.chunk","created":1698999575,"model":"kimi-k3.1-pro","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

...

data: {"id":"cmpl-1305b94c570f447fbde3180560736287","object":"chat.completion.chunk","created":1698999575,"model":"kimi-k3.1-pro","choices":[{"index":0,"delta":{"content":"."},"finish_reason":null}]}

data: {"id":"cmpl-1305b94c570f447fbde3180560736287","object":"chat.completion.chunk","created":1698999575,"model":"kimi-k3.1-pro","choices":[{"index":0,"delta":{},"finish_reason":"stop","usage":{"prompt_tokens":19,"completion_tokens":13,"total_tokens":32}}]}

data: {"id":"cmpl-1305b94c570f447fbde3180560736287","object":"chat.completion.chunk","created":1698999575,"model":"kimi-k3.1-pro","choices":[],"usage":{"prompt_tokens":19,"completion_tokens":13,"total_tokens":32}}

data: [DONE]
```

Each chunk starts with `data: `, followed by a JSON object, and ends with two newline characters (`\n\n`). Once all chunks have been transmitted, the server sends `data: [DONE]` to signal completion, at which point you can close the connection.

*Note: Always use `data: [DONE]` to determine whether transmission is complete rather than relying on `finish_reason` alone. Until `data: [DONE]` arrives, treat the message as **incomplete** even if `finish_reason=stop` has already been received.*

Incremental fields appear across chunks as follows:

* `role` appears only in the first chunk and is omitted from subsequent chunks;
* For thinking models, `reasoning_content` arrives chunk by chunk ahead of `content`;
* `content` delivers answer text fragments chunk by chunk;
* When the model calls tools, `delta.tool_calls` delivers fragments keyed by `index` (`id`, `type`, and `function.name` in the first fragment, `function.arguments` appended across subsequent fragments);
* When `stream_options: {"include_usage": true}` is passed, the server sends one final statistics chunk with an empty `choices` array before `data: [DONE]`, placing total token counts in the top-level `usage` field.

In languages without an official SDK, or when you need custom transport control, you can read the HTTP stream directly:

> [tab] python
>
> ```python expandable theme={null}
> import json
> import os
> import httpx
>
> request_data = {
>     "model": "kimi-k3.1-pro",
>     "messages": [
>         {"role": "user", "content": "Hello, briefly explain streaming."},
>     ],
>     "stream": True,
>     "stream_options": {"include_usage": True},
> }
>
> headers = {
>     "Authorization": f"Bearer {os.environ['MOONSHOT_API_KEY']}",
> }
>
> usage = None
>
> # Use httpx.stream to process SSE lines as they arrive instead of buffering the entire response
> with httpx.stream(
>     "POST",
>     "https://api.moonshot.ai/v1/chat/completions",
>     headers=headers,
>     json=request_data,
> ) as response:
>     response.raise_for_status()
>
>     for line in response.iter_lines():
>         if not line.startswith("data:"):
>             continue
>
>         payload = line[5:].lstrip()
>         if payload == "[DONE]":
>             break
>
>         chunk = json.loads(payload)
>         choices = chunk.get("choices", [])
>         chunk_usage = chunk.get("usage")
>         if chunk_usage is None and choices:
>             chunk_usage = choices[0].get("usage")
>         if chunk_usage:
>             usage = chunk_usage
>         if not choices:
>             continue
>
>         choice = choices[0]
>         delta = choice["delta"]
>         role = delta.get("role")
>         if role:
>             print("role:", role)
>         content = delta.get("content")
>         if content:
>             print(content, end="")
>
> if usage:
>     print("\ntotal_tokens:", usage["total_tokens"])
> ```

> [tab] node.js
>
> ```js expandable theme={null}
> const axios = require("axios");
>
> const requestData = {
>     model: "kimi-k3.1-pro",
>     messages: [
>         { role: "user", content: "Hello, briefly explain streaming." },
>     ],
>     stream: true,
>     stream_options: { include_usage: true },
> };
>
> axios.post("https://api.moonshot.ai/v1/chat/completions", requestData, {
>     headers: {
>         "Authorization": `Bearer ${process.env.MOONSHOT_API_KEY}`,
>     },
>     responseType: "stream",
> }).then((response) => {
>     let buffer = "";
>     let usage;
>     response.data.on("data", (rawChunk) => {
>         buffer += rawChunk.toString();
>         const events = buffer.split(/\r?\n\r?\n/);
>         buffer = events.pop() ?? "";
>
>         for (const event of events) {
>             const dataLines = event
>                 .split(/\r?\n/)
>                 .filter((line) => line.startsWith("data:"))
>                 .map((line) => line.slice(5).trimStart());
>             if (dataLines.length === 0) continue;
>
>             const payload = dataLines.join("\n");
>             if (payload === "[DONE]") {
>                 if (usage) console.log("\ntotal_tokens:", usage.total_tokens);
>                 response.data.destroy();
>                 return;
>             }
>
>             const chunk = JSON.parse(payload);
>             const choices = chunk.choices || [];
>             const chunkUsage = chunk.usage ?? choices[0]?.usage;
>             if (chunkUsage) usage = chunkUsage;
>             if (choices.length === 0) {
>                 continue;
>             }
>             const delta = choices[0].delta;
>             if (delta.role) {
>                 console.log("role:", delta.role);
>             }
>             if (delta.content) {
>                 process.stdout.write(delta.content);
>             }
>         }
>     });
> }).catch((error) => {
>     console.error("Error in request:", error);
> });
> ```

Regardless of programming language, reading Chat Completions SSE directly follows four steps:

1. Send an HTTP request with `"stream": true` in the request body;
2. Verify that the response `Content-Type` header is `text/event-stream`;
3. Split frames on `\n\n`, extract `data:` lines, and deserialize each JSON payload;
4. Stop reading when `data: [DONE]` arrives.

### Count Token Usage

There are two ways to count token usage for streaming requests. We recommend passing `stream_options: {"include_usage": true}` (`{"include_usage": True}` in the Python SDK) and reading `prompt_tokens`, `completion_tokens`, and `total_tokens` from the top-level `usage` field of the final statistics chunk right before `data: [DONE]`:

```text theme={null}
...

data: {"id":"cmpl-1305b94c570f447fbde3180560736287","object":"chat.completion.chunk","created":1698999575,"model":"kimi-k3.1-pro","choices":[],"usage":{"prompt_tokens":19,"completion_tokens":13,"total_tokens":32}}
                                                                                                                                             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                                                                                                                             Read the top-level usage field of the final statistics chunk
data: [DONE]
```

> [!NOTE]
>
> The final statistics chunk does not contain model output, so its `choices` array is empty. Do not assume that every chunk has `choices[0]`; check that `choices` is non-empty first, and read total usage from `chunk.usage` on the final chunk.

If a stream is interrupted by a network drop, client cancellation, or an unexpected error before the final chunk arrives, the server-reported `usage` will not be available. As a fallback, buffer the fragments received so far and call the [Estimate Tokens endpoint](../api/estimate.md) after the stream closes:

> [tab] python
>
> ```python expandable theme={null}
> import os
> import httpx
> from openai import OpenAI
>
> client = OpenAI(
>     api_key=os.environ["MOONSHOT_API_KEY"],  # Set the MOONSHOT_API_KEY environment variable before running this example
>     base_url="https://api.moonshot.ai/v1",
> )
>
> stream = client.chat.completions.create(
>     model="kimi-k3.1-pro",
>     messages=[
>         {"role": "system", "content": "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated."},
>         {"role": "user", "content": "Hello, my name is Li Lei. What is 1+1?"},
>     ],
>     stream=True,
> )
>
>
> def estimate_token_count(text: str) -> int:
>     headers = {
>         "Authorization": f"Bearer {os.environ['MOONSHOT_API_KEY']}",
>     }
>     payload = {
>         "model": "kimi-k3.1-pro",
>         "messages": [
>             {"role": "user", "content": text},
>         ],
>     }
>     r = httpx.post(
>         "https://api.moonshot.ai/v1/tokenizers/estimate-token-count",
>         headers=headers,
>         json=payload,
>     )
>     r.raise_for_status()
>     return r.json()["data"]["total_tokens"]
>
>
> output_fragments = []
> for chunk in stream:
>     if not chunk.choices:
>         continue
>     delta = chunk.choices[0].delta
>     if hasattr(delta, "reasoning_content"):
>         reasoning_fragment = getattr(delta, "reasoning_content")
>         if reasoning_fragment:
>             output_fragments.append(reasoning_fragment)
>     if delta.content:
>         output_fragments.append(delta.content)
>
> print("completion_tokens:", estimate_token_count("".join(output_fragments)))
> ```

> [tab] node.js
>
> ```js expandable theme={null}
> const axios = require("axios");
> const OpenAI = require("openai");
>
> const client = new OpenAI({
>     apiKey: process.env.MOONSHOT_API_KEY,
>     baseURL: "https://api.moonshot.ai/v1",
> });
>
> async function estimateTokenCount(inputMessages) {
>     const headers = {
>         "Authorization": `Bearer ${process.env.MOONSHOT_API_KEY}`,
>     };
>     const payload = {
>         model: "kimi-k3.1-pro",
>         messages: inputMessages,
>     };
>     const r = await axios.post(
>         "https://api.moonshot.ai/v1/tokenizers/estimate-token-count",
>         payload,
>         { headers }
>     );
>     return r.data.data.total_tokens;
> }
>
> async function main() {
>     const stream = await client.chat.completions.create({
>         model: "kimi-k3.1-pro",
>         messages: [
>             { role: "system", content: "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated." },
>             { role: "user", content: "Hello, my name is Li Lei. What is 1+1?" },
>         ],
>         stream: true,
>     });
>
>     const outputFragments = [];
>     for await (const chunk of stream) {
>         if (chunk.choices.length === 0) continue;
>         const delta = chunk.choices[0].delta;
>         if (delta.reasoning_content) {
>             outputFragments.push(delta.reasoning_content);
>         }
>         if (delta.content) {
>             outputFragments.push(delta.content);
>         }
>     }
>
>     console.log("completion_tokens:", await estimateTokenCount([
>         { role: "user", content: outputFragments.join("") },
>     ]));
> }
>
> main();
> ```

### Stop Streaming Output

To stop receiving output early, close the underlying HTTP connection or break out of the iteration loop:

```python theme={null}
for chunk in stream:
    if condition:
        break
```

### Multiple Responses (`n` Parameter)

> [!NOTE]
>
> `kimi-k3.1-pro` and `kimi-k3` fix `n` at `1` and do not support returning multiple responses in a single request. Passing an `n` greater than 1 returns a 400 error (`invalid n: only 1 is allowed for this model`) for both streaming and non-streaming requests. See the [Model Parameter Reference](../api/models-overview.md) for per-model parameter constraints.

## Messages API

`POST /anthropic/v1/messages` follows the Anthropic Messages event stream model. When `stream=true` is set, the server pushes SSE events in the order `message_start` → content block events (`content_block_start` / `content_block_delta` / `content_block_stop`) → `message_delta` → `message_stop`, ending with `message_stop` (no `data: [DONE]` sentinel is sent).

A streaming message can contain up to three types of content blocks (`content_block`) in sequence, each delivering fragments through its corresponding `content_block_delta` subtype:

* **Thinking block (`thinking`)**: delivers reasoning text via `thinking_delta` and signatures via `signature_delta`;
* **Text block (`text`)**: delivers user-facing answer text via `text_delta`;
* **Tool use block (`tool_use`)**: delivers JSON string fragments of tool arguments via `input_json_delta.partial_json`.

Token usage is reported across two events: `message_start` carries the initial input token counts, and `message_delta` carries the final cumulative `usage` statistics (including input, output, and cache details) along with `stop_reason`. See [Messages API](../api/messages.md) for complete parameter and event schemas.

### Fine-grained Tool Streaming

When you declare `tools` (each with `name`, `description`, and `input_schema`) and set `stream=true` on the Messages API, the model streams JSON fragments of tool arguments to the client via `input_json_delta.partial_json` as they are generated — known in the Anthropic ecosystem as **fine-grained tool streaming**. This behavior is enabled automatically whenever `stream=true`; you do not need to pass `eager_input_streaming` or request separate access.

With fine-grained tool streaming, clients can render read-only previews of long code files, documents, or complex configurations in real time and warm up validators or execution environments in advance, **without executing tools before validation completes**. Because reasoning and output pacing vary across models, clients must not assume a fixed number of fragments or arrival interval, and this mode does not guarantee lower end-to-end request latency. Clients that do not need live previews can continue waiting for the stream to finish before processing arguments.

A Messages API SSE stream containing a tool call looks like this (each SSE frame carries both an `event:` line and a `data:` line, where `event` matches `data.type`):

```text theme={null}
event: message_start
data: {"type":"message_start","message":{"id":"msg_01...","type":"message","role":"assistant","model":"kimi-k3.1-pro","content":[],"stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":128,"output_tokens":0}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"tool_use","id":"toolu_01...","name":"write_file","input":{}}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"input_json_delta","partial_json":"{\"filename\":\"streaming.txt\","}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"input_json_delta","partial_json":"\"content\":\"AAAA...\"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use","stop_sequence":null},"usage":{"input_tokens":128,"output_tokens":64}}

event: message_stop
data: {"type":"message_stop"}
```

Follow these rules when assembling tool arguments:

* **Keep an isolated buffer per `index`**: When a `content_block_start` with `content_block.type == "tool_use"` arrives, allocate a dedicated buffer for `event.index`. The initial `input: {}` is an empty placeholder and does not contain actual arguments.
* **`index` is not a zero-based tool ordinal**: `index` identifies the content block's position across the entire message. Preceding `thinking` and `text` blocks also occupy indices (so the first tool block may start at `index=1` or `index=2`). When the model invokes multiple tools in parallel, store each buffer under its own `index` to avoid mixing fragments.
* **Append only — do not parse individual fragments**: On each `input_json_delta`, append `partial_json` to the string buffer for that `index`. Never overwrite prior fragments or run `JSON.parse` on incomplete slices.

### Use the Anthropic SDK

When using the official Anthropic SDK, set `base_url` (`baseURL` in Node.js) to `https://api.moonshot.ai/anthropic` and pass `stream=True` (`stream: true` in Node.js) to `messages.create` to assemble `input_json_delta` events as they arrive (this example only prints the reconstructed arguments and does not execute the tool):

> [tab] python
>
> ```python expandable theme={null}
> import json
> import os
> from anthropic import Anthropic
>
> client = Anthropic(
>     api_key=os.environ["MOONSHOT_API_KEY"],  # Set the MOONSHOT_API_KEY environment variable before running this example
>     base_url="https://api.moonshot.ai/anthropic",
> )
>
> stream = client.messages.create(
>     model="kimi-k3.1-pro",
>     max_tokens=4096,
>     messages=[{
>         "role": "user",
>         "content": "Call write_file once with filename set to streaming.txt and content set to 160 uppercase A characters. Do not output ordinary text.",
>     }],
>     tools=[{
>         "name": "write_file",
>         "description": "Write a text file. This example only demonstrates argument streaming and does not execute the tool.",
>         "input_schema": {
>             "type": "object",
>             "properties": {
>                 "filename": {"type": "string"},
>                 "content": {"type": "string"},
>             },
>             "required": ["filename", "content"],
>             "additionalProperties": False,
>         },
>     }],
>     stream=True,
> )
>
> tool_inputs = {}
> stop_reason = None
> message_stopped = False
>
> for event in stream:
>     if event.type == "content_block_start":
>         block = event.content_block
>         if block.type == "tool_use":
>             tool_inputs[event.index] = {
>                 "id": block.id,
>                 "name": block.name,
>                 "partial_json": "",
>                 "closed": False,
>             }
>     elif event.type == "content_block_delta":
>         if event.delta.type == "input_json_delta":
>             tool_inputs[event.index]["partial_json"] += event.delta.partial_json
>             # Update a read-only preview here; do not execute the tool.
>     elif event.type == "content_block_stop":
>         call = tool_inputs.get(event.index)
>         if call is not None:
>             call["closed"] = True
>     elif event.type == "message_delta":
>         stop_reason = event.delta.stop_reason
>     elif event.type == "message_stop":
>         message_stopped = True
>
> if not message_stopped or stop_reason != "tool_use":
>     raise RuntimeError("Tool arguments are incomplete; do not execute the tool.")
>
> for call in tool_inputs.values():
>     if not call["closed"]:
>         raise RuntimeError("The tool argument block is incomplete; do not execute the tool.")
>     arguments = json.loads(call["partial_json"])
>     # Validate arguments against input_schema and check business permissions before execution.
>     print(call["id"], call["name"], arguments)
> ```

> [tab] node.js
>
> ```js expandable theme={null}
> const Anthropic = require("@anthropic-ai/sdk");
>
> const client = new Anthropic({
>     apiKey: process.env.MOONSHOT_API_KEY, // Set the MOONSHOT_API_KEY environment variable before running this example
>     baseURL: "https://api.moonshot.ai/anthropic",
> });
>
> async function main() {
>     const stream = await client.messages.create({
>         model: "kimi-k3.1-pro",
>         max_tokens: 4096,
>         messages: [{
>             role: "user",
>             content: "Call write_file once with filename set to streaming.txt and content set to 160 uppercase A characters. Do not output ordinary text.",
>         }],
>         tools: [{
>             name: "write_file",
>             description: "Write a text file. This example only demonstrates argument streaming and does not execute the tool.",
>             input_schema: {
>                 type: "object",
>                 properties: {
>                     filename: { type: "string" },
>                     content: { type: "string" },
>                 },
>                 required: ["filename", "content"],
>                 additionalProperties: false,
>             },
>         }],
>         stream: true,
>     });
>
>     const toolInputs = new Map();
>     let stopReason = null;
>     let messageStopped = false;
>
>     for await (const event of stream) {
>         if (event.type === "content_block_start") {
>             const block = event.content_block;
>             if (block.type === "tool_use") {
>                 toolInputs.set(event.index, {
>                     id: block.id,
>                     name: block.name,
>                     partialJson: "",
>                     closed: false,
>                 });
>             }
>         } else if (event.type === "content_block_delta") {
>             if (event.delta.type === "input_json_delta") {
>                 const call = toolInputs.get(event.index);
>                 call.partialJson += event.delta.partial_json;
>                 // Update a read-only preview here; do not execute the tool.
>             }
>         } else if (event.type === "content_block_stop") {
>             const call = toolInputs.get(event.index);
>             if (call) call.closed = true;
>         } else if (event.type === "message_delta") {
>             stopReason = event.delta.stop_reason;
>         } else if (event.type === "message_stop") {
>             messageStopped = true;
>         }
>     }
>
>     if (!messageStopped || stopReason !== "tool_use") {
>         throw new Error("Tool arguments are incomplete; do not execute the tool.");
>     }
>
>     for (const call of toolInputs.values()) {
>         if (!call.closed) throw new Error("The tool argument block is incomplete; do not execute the tool.");
>         const args = JSON.parse(call.partialJson);
>         // Validate args against input_schema and check business permissions before execution.
>         console.log(call.id, call.name, args);
>     }
> }
>
> main().catch((error) => {
>     console.error(error);
>     process.exitCode = 1;
> });
> ```

### Read SSE Directly Without an SDK

When calling `POST https://api.moonshot.ai/anthropic/v1/messages` over raw HTTP, send `Authorization: Bearer $MOONSHOT_API_KEY` and `anthropic-version: 2023-06-01` in the request headers and apply the same event-handling rules:

> [tab] python
>
> ```python expandable theme={null}
> import json
> import os
> import httpx
>
> request_data = {
>     "model": "kimi-k3.1-pro",
>     "max_tokens": 4096,
>     "messages": [{
>         "role": "user",
>         "content": "Call write_file once with filename set to streaming.txt and content set to 160 uppercase A characters. Do not output ordinary text.",
>     }],
>     "tools": [{
>         "name": "write_file",
>         "description": "Write a text file. This example only demonstrates argument streaming and does not execute the tool.",
>         "input_schema": {
>             "type": "object",
>             "properties": {
>                 "filename": {"type": "string"},
>                 "content": {"type": "string"},
>             },
>             "required": ["filename", "content"],
>             "additionalProperties": False,
>         },
>     }],
>     "stream": True,
> }
> headers = {
>     "Authorization": f"Bearer {os.environ['MOONSHOT_API_KEY']}",
>     "anthropic-version": "2023-06-01",
> }
>
> tool_inputs = {}
> stop_reason = None
> message_stopped = False
>
> with httpx.stream(
>     "POST",
>     "https://api.moonshot.ai/anthropic/v1/messages",
>     headers=headers,
>     json=request_data,
>     timeout=120,
> ) as response:
>     response.raise_for_status()
>     for line in response.iter_lines():
>         if not line.startswith("data:"):
>             continue
>         event = json.loads(line[5:].lstrip())
>         event_type = event.get("type")
>
>         if event_type == "content_block_start":
>             block = event["content_block"]
>             if block["type"] == "tool_use":
>                 tool_inputs[event["index"]] = {
>                     "id": block["id"],
>                     "name": block["name"],
>                     "partial_json": "",
>                     "closed": False,
>                 }
>         elif event_type == "content_block_delta":
>             delta = event["delta"]
>             if delta["type"] == "input_json_delta":
>                 tool_inputs[event["index"]]["partial_json"] += delta["partial_json"]
>                 # Update a read-only preview here; do not execute the tool.
>         elif event_type == "content_block_stop":
>             call = tool_inputs.get(event["index"])
>             if call is not None:
>                 call["closed"] = True
>         elif event_type == "message_delta":
>             stop_reason = event["delta"]["stop_reason"]
>         elif event_type == "message_stop":
>             message_stopped = True
>             break
>
> if not message_stopped or stop_reason != "tool_use":
>     raise RuntimeError("Tool arguments are incomplete; do not execute the tool.")
>
> for call in tool_inputs.values():
>     if not call["closed"]:
>         raise RuntimeError("The tool argument block is incomplete; do not execute the tool.")
>     arguments = json.loads(call["partial_json"])
>     # Validate arguments against input_schema and check business permissions before execution.
>     print(call["id"], call["name"], arguments)
> ```

> [tab] node.js
>
> ```js expandable theme={null}
> async function main() {
>     const requestData = {
>         model: "kimi-k3.1-pro",
>         max_tokens: 4096,
>         messages: [{
>             role: "user",
>             content: "Call write_file once with filename set to streaming.txt and content set to 160 uppercase A characters. Do not output ordinary text.",
>         }],
>         tools: [{
>             name: "write_file",
>             description: "Write a text file. This example only demonstrates argument streaming and does not execute the tool.",
>             input_schema: {
>                 type: "object",
>                 properties: {
>                     filename: { type: "string" },
>                     content: { type: "string" },
>                 },
>                 required: ["filename", "content"],
>                 additionalProperties: false,
>             },
>         }],
>         stream: true,
>     };
>
>     const toolInputs = new Map();
>     let stopReason = null;
>     let messageStopped = false;
>
>     function handleEvent(event) {
>         if (event.type === "content_block_start") {
>             const block = event.content_block;
>             if (block.type === "tool_use") {
>                 toolInputs.set(event.index, {
>                     id: block.id,
>                     name: block.name,
>                     partialJson: "",
>                     closed: false,
>                 });
>             }
>         } else if (event.type === "content_block_delta") {
>             const delta = event.delta;
>             if (delta.type === "input_json_delta") {
>                 const call = toolInputs.get(event.index);
>                 call.partialJson += delta.partial_json;
>                 // Update a read-only preview here; do not execute the tool.
>             }
>         } else if (event.type === "content_block_stop") {
>             const call = toolInputs.get(event.index);
>             if (call) call.closed = true;
>         } else if (event.type === "message_delta") {
>             stopReason = event.delta.stop_reason;
>         } else if (event.type === "message_stop") {
>             messageStopped = true;
>         }
>     }
>
>     function processFrame(frame) {
>         const data = frame
>             .split(/\r?\n/)
>             .filter((line) => line.startsWith("data:"))
>             .map((line) => line.slice(5).trimStart())
>             .join("\n");
>         if (data) handleEvent(JSON.parse(data));
>     }
>
>     const response = await fetch("https://api.moonshot.ai/anthropic/v1/messages", {
>         method: "POST",
>         headers: {
>             "Authorization": `Bearer ${process.env.MOONSHOT_API_KEY}`,
>             "Content-Type": "application/json",
>             "anthropic-version": "2023-06-01",
>         },
>         body: JSON.stringify(requestData),
>     });
>     if (!response.ok) throw new Error(await response.text());
>     if (!response.body) throw new Error("The response does not contain an SSE stream.");
>
>     const decoder = new TextDecoder();
>     let buffer = "";
>     for await (const chunk of response.body) {
>         buffer += decoder.decode(chunk, { stream: true });
>         const frames = buffer.split(/\r?\n\r?\n/);
>         buffer = frames.pop() ?? "";
>         for (const frame of frames) processFrame(frame);
>     }
>     buffer += decoder.decode();
>     if (buffer.trim()) processFrame(buffer);
>
>     if (!messageStopped || stopReason !== "tool_use") {
>         throw new Error("Tool arguments are incomplete; do not execute the tool.");
>     }
>
>     for (const call of toolInputs.values()) {
>         if (!call.closed) throw new Error("The tool argument block is incomplete; do not execute the tool.");
>         const args = JSON.parse(call.partialJson);
>         // Validate args against input_schema and check business permissions before execution.
>         console.log(call.id, call.name, args);
>     }
> }
>
> main().catch((error) => {
>     console.error(error);
>     process.exitCode = 1;
> });
> ```

### Execution Requirements, Error Handling, and Billing

Your application remains responsible for executing tools. Before running any tool, verify all of the following conditions:

1. `message_stop` has arrived and the stream has closed cleanly;
2. The final `stop_reason` is `tool_use`;
3. The corresponding tool block has received `content_block_stop`;
4. The concatenated string parses as valid JSON;
5. The parsed arguments pass `input_schema`, business permission, and safety checks.

Even if you parse a block's buffer as soon as its `content_block_stop` arrives, wait until the entire message satisfies every check above before triggering side effects — `content_block_stop` only marks the end of the current block and does not by itself authorize execution. When `stop_reason=max_tokens`, `message_stop` is missing, or the network connection drops, the received arguments may be truncated. Do not execute the tool; retain diagnostic details, then increase `max_tokens` or retry the request rather than auto-completing partial JSON.

> [!NOTE]
>
> Fine-grained tool streaming is the standard streaming behavior of the Messages API and is not billed separately. Input tokens, output tokens, and context caching are billed at [the selected model's rates](../pricing/chat.md), and thinking tokens count toward output tokens. Compute, storage, network, and third-party service costs incurred when your application executes tools are outside Kimi model inference charges.
