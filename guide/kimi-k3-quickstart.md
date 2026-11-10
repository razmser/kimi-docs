> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Kimi K3

> Explore Kimi K3 for long-horizon coding, knowledge work, deep reasoning, visual understanding, and a 1M-token context window.

## Overview

Kimi K3 is Kimi's previous-generation flagship model, with 2.8 trillion parameters. It is built on Kimi Delta Attention (KDA), a hybrid linear attention mechanism, and Attention Residuals, with native visual understanding and a 1M-token context window. It is the world's first open-source model in the 3-trillion-parameter class, designed for frontier intelligence scenarios including long-horizon coding, knowledge work, and reasoning.

For complete benchmarks and case studies, see the [technical blog](https://www.kimi.com/blog/kimi-k3). Kimi is currently working closely with inference partners and open-source maintainers to align technical details and ensure the model launches reliably across the ecosystem. The full model weights have been released. More details on architecture, training, and evaluation will be published with the Kimi K3 technical report.

### A 3-trillion-scale open-source model

Kimi K3 is the first open-source model to reach 2.8 trillion parameters. This is the latest step in Kimi's continued push of model-scale boundaries: in 9 of the past 12 months (2025/07–2026/07), Kimi models have maintained the frontier in open-source model scale.

![Open-source frontier model scale over time](https://mintcdn.com/moonshotai/nNBnxyDb94JSawL-/assets/pics/k3-opensource-progress.png?fit=max\&auto=format\&n=nNBnxyDb94JSawL-\&q=85\&s=0c723d1508cb0ebbaacd83a0d034223f)

Kimi K3 is built on Kimi Delta Attention (KDA) and Attention Residuals (AttnRes). Both architectural updates are designed to help information flow more smoothly through longer sequences and deeper models. We also further increased the sparsity of the Mixture of Experts (MoE): with the Stable LatentMoE framework, the model efficiently activates 16 out of 896 experts. Together with improvements in training methodology and data recipes, these structural advances give Kimi K3 roughly 2.5x the overall scaling efficiency of K2, converting compute into capability more effectively.

![Kimi K3 architecture](https://mintcdn.com/moonshotai/nNBnxyDb94JSawL-/assets/pics/k3-arch.png?fit=max\&auto=format\&n=nNBnxyDb94JSawL-\&q=85\&s=e9ea3491ca74fa231985986a1081e5d7)

### Coding

Kimi K3 has strong long-horizon coding capabilities. With minimal human supervision, it can sustain long-running engineering tasks, understand and work with large codebases, and coordinate terminal tools.

Kimi K3 also excels at tasks that combine software engineering and visual reasoning. It can use screenshots and visual feedback to improve workflows in game development, frontend engineering, CAD, and related scenarios.

### Knowledge work

Kimi K3 advances end-to-end knowledge work. Beyond public benchmarks, Kimi K3 (max) also shows consistent gains in our internal evaluations. These evaluations reflect recurring task patterns and challenges from real user-agent collaboration workflows. Kimi K3 demonstrates consistent advantages across production-oriented workflows, indicating broad improvements in agentic knowledge-work capabilities.

## Access requirements

Kimi K3 is a flagship model: it is unlocked after a successful top-up (minimum \$1). Your cumulative top-up amount also determines your account tier and rate limits (concurrency, RPM, TPM, TPD) — see [Recharge and Rate Limits](../pricing/limits.md).

## Get started

* [Playground](https://platform.kimi.ai/playground)
* [Get an API Key](https://platform.kimi.ai/console/api-keys)

The examples require Python 3.9+ and the OpenAI SDK. Install the SDK and initialize the client once; later Python examples reuse `client`.

```bash theme={null}
python3 -m pip install --upgrade 'openai>=1.0'
```

```python theme={null}
import os

from openai import OpenAI

client = OpenAI(
    api_key=os.environ["MOONSHOT_API_KEY"],
    base_url="https://api.moonshot.ai/v1",
)
```

## Basic call

> [tab] Python
>
> ```python theme={null}
> completion = client.chat.completions.create(
>     model="kimi-k3",
>     messages=[{"role": "user", "content": "Introduce Kimi K3 in one sentence."}],
> )
>
> print(completion.choices[0].message.content)
> ```

> [tab] cURL
>
> ```bash theme={null}
> curl https://api.moonshot.ai/v1/chat/completions \
>   --header "Authorization: Bearer $MOONSHOT_API_KEY" \
>   --header "Content-Type: application/json" \
>   --data '{
>     "model": "kimi-k3",
>     "messages": [{"role": "user", "content": "Introduce Kimi K3 in one sentence."}]
>   }'
> ```

## Reasoning effort

K3 always has thinking mode enabled and supports configuring its reasoning effort with the top-level `reasoning_effort` request field.

> [!NOTE]
>
> Reasoning effort supports `low`, `high`, and `max` (default `max`). See [Reasoning Effort](use-reasoning-effort.md) for usage.

```python theme={null}
completion = client.chat.completions.create(
    model="kimi-k3",
    reasoning_effort="max",
    messages=[{"role": "user", "content": "Prove that the square root of 2 is irrational."}],
)

print(completion.choices[0].message.content)
```

> [!NOTE]
>
> For multi-turn conversations and tool calls, add the complete assistant message returned by the API to the next request. Do not keep only `content`.

## Streaming

Streaming responses provide separate `reasoning_content` and final-answer `content` deltas. See [Streaming Output](utilize-the-streaming-output-feature-of-kimi-api.md) for details.

```python theme={null}
stream = client.chat.completions.create(
    model="kimi-k3",
    messages=[{"role": "user", "content": "Explain why the sky is blue."}],
    stream=True,
)

for chunk in stream:
    delta = chunk.choices[0].delta
    reasoning = getattr(delta, "reasoning_content", None)
    if reasoning:
        print(reasoning, end="", flush=True)
    if delta.content:
        print(delta.content, end="", flush=True)
```

## Vision input

For vision messages, `content` must be an array of objects, not a serialized string. See [Vision Input](use-kimi-vision-model.md) for formats and limits.

> [tab] Local image
>
> ```python theme={null}
> import base64
> from pathlib import Path
>
> image_data: str = base64.b64encode(Path("image.png").read_bytes()).decode()
> completion = client.chat.completions.create(
>     model="kimi-k3",
>     messages=[
>         {
>             "role": "user",
>             "content": [
>                 {
>                     "type": "image_url",
>                     "image_url": {"url": f"data:image/png;base64,{image_data}"},
>                 },
>                 {"type": "text", "text": "Describe this image."},
>             ],
>         }
>     ],
> )
>
> print(completion.choices[0].message.content)
> ```

> [tab] Video file
>
> ```python theme={null}
> from pathlib import Path
>
> video = client.files.create(file=Path("video.mp4"), purpose="video")
> try:
>     completion = client.chat.completions.create(
>         model="kimi-k3",
>         messages=[
>             {
>                 "role": "user",
>                 "content": [
>                     {
>                         "type": "video_url",
>                         "video_url": {"url": f"ms://{video.id}"},
>                     },
>                     {"type": "text", "text": "Summarize this video."},
>                 ],
>             }
>         ],
>     )
>     print(completion.choices[0].message.content)
> finally:
>     client.files.delete(video.id)
> ```

## Structured output

Use `json_schema` with `strict: true` to constrain the final `message.content`. Parse only that field, not `reasoning_content`.

> [accordion] Name and age schema
>
> ```python theme={null}
> import json
>
> completion = client.chat.completions.create(
>     model="kimi-k3",
>     messages=[
>         {"role": "user", "content": "Lin is 28 years old. Extract the name and age."}
>     ],
>     response_format={
>         "type": "json_schema",
>         "json_schema": {
>             "name": "person",
>             "strict": True,
>             "schema": {
>                 "type": "object",
>                 "properties": {
>                     "name": {"type": "string"},
>                     "age": {"type": "integer"},
>                 },
>                 "required": ["name", "age"],
>                 "additionalProperties": False,
>             },
>         },
>     },
> )
>
> person: dict[str, object] = json.loads(
>     completion.choices[0].message.content or "{}"
> )
> print(person)
> ```

See [Structured Output](response_format.md).

## Partial Mode

Add an assistant message with `partial=True` at the end of `messages` to continue from a text prefix. Prepend that prefix when displaying the final result.

```python theme={null}
prefix: str = "Conclusion: "
completion = client.chat.completions.create(
    model="kimi-k3",
    messages=[
        {"role": "user", "content": "In one sentence, explain why API compatibility matters."},
        {"role": "assistant", "content": prefix, "partial": True},
    ],
)

print(prefix + (completion.choices[0].message.content or ""))
```

See [Partial Mode](use-partial-mode-feature-of-kimi-api.md).

## Custom tools and `tool_choice`

Use `tool_choice="required"` on the first turn to require at least one tool call. After executing every call, return the complete assistant message and append one tool result with the matching `tool_call_id` for each call.

> [accordion] Minimal weather agent loop
>
> ```python theme={null}
> import json
> from typing import Any
>
> tools: list[dict[str, Any]] = [
>     {
>         "type": "function",
>         "function": {
>             "name": "get_weather",
>             "description": "Get the weather for a city",
>             "parameters": {
>                 "type": "object",
>                 "properties": {"city": {"type": "string"}},
>                 "required": ["city"],
>             },
>         },
>     }
> ]
> messages: list[Any] = [
>     {"role": "user", "content": "What is the weather in San Francisco today?"}
> ]
>
> first = client.chat.completions.create(
>     model="kimi-k3",
>     messages=messages,
>     tools=tools,
>     tool_choice="required",
> )
> assistant_message = first.choices[0].message
> messages.append(assistant_message)
>
> for tool_call in assistant_message.tool_calls or []:
>     arguments: dict[str, str] = json.loads(tool_call.function.arguments)
>     result: str = json.dumps(
>         {"city": arguments["city"], "weather": "sunny", "temperature_c": 24}
>     )
>     messages.append(
>         {"role": "tool", "tool_call_id": tool_call.id, "content": result}
>     )
>
> final = client.chat.completions.create(
>     model="kimi-k3",
>     messages=messages,
>     tools=tools,
> )
> print(final.choices[0].message.content)
> ```

See [Tool Choice](use-tool-choice.md).

## Dynamic tool loading

Place a complete tool definition in a `system` message without `content`. The tool becomes available from that message onward.

> [accordion] Load a calculator dynamically
>
> ```python theme={null}
> from typing import Any
>
> dynamic_messages: list[dict[str, Any]] = [
>     {"role": "user", "content": "Calculate 23 times 47."},
>     {
>         "role": "system",
>         "tools": [
>             {
>                 "type": "function",
>                 "function": {
>                     "name": "calculate",
>                     "description": "Evaluate an arithmetic expression",
>                     "parameters": {
>                         "type": "object",
>                         "properties": {
>                             "expression": {
>                                 "type": "string",
>                                 "description": "The arithmetic expression to evaluate",
>                             }
>                         },
>                         "required": ["expression"],
>                     },
>                 },
>             }
>         ],
>     },
> ]
> completion = client.chat.completions.create(
>     model="kimi-k3",
>     messages=dynamic_messages,
> )
>
> print(completion.choices[0].message.tool_calls)
> ```

* Include the complete `name`, `description`, and `parameters` definition.
* The declaration takes effect at its position in `messages`.
* Keep this message in later request history; the server does not retain it.

See [Dynamic Tool Loading](use-dynamic-tool-loading.md).

## 1M context and TTL caching

Context caching is automatic for regular model requests — no manual cache creation: once an eligible prefix is written to the cache, hits are billed at the cached-input price, and each write incurs a one-time charge. TTL supports two values, `5m` (default) and `1h`; choose one based on your request interval — the default `5m` is enough for rapid back-and-forth, while workloads with gaps over 5 minutes should pass `prompt_cache_options.ttl` to select the `1h` tier and avoid repeated writes after expiry. The cache is stored in blocks; any partial block that does not fill a whole block cannot be written to the cache and is billed as a cache miss.

```python theme={null}
from pathlib import Path

knowledge: str = Path("knowledge-base.md").read_text(encoding="utf-8")

for question in ["Summarize the key conclusions.", "List three implementation risks."]:
    completion = client.chat.completions.create(
        model="kimi-k3",
        messages=[
            {"role": "system", "content": knowledge},
            {"role": "user", "content": question},
        ],
    )
    print(completion.choices[0].message.content)
```

For help choosing a TTL, improving hit rates, and verifying usage, see [Context Caching](context-caching.md).

## Official tools

Official tools are integrated through Formula:

1. Fetch tool definitions from the Formula `/tools` endpoint.
2. Add those definitions to the Chat Completions `tools` field.
3. When the model returns `tool_calls`, submit each function name and arguments to the Formula `/fibers` endpoint.
4. Add the complete assistant message and Fiber output as the corresponding tool message.
5. Call Chat Completions again until the model returns a final answer.

See [Official Tools](use-official-tools.md) for the complete client and API contract.

## Important limits

* Reasoning effort is configured with the top-level `reasoning_effort` request field and supports `low`, `high`, and `max` (default `max`); K3 always has thinking mode enabled.
* `max_completion_tokens` defaults to 131072 and can be set up to 1048576.
* `temperature=1.0`, `top_p=0.95`, `n=1`, `presence_penalty=0`, and `frequency_penalty=0` are fixed; omit them from requests.
* Return the complete assistant message unchanged in multi-turn conversations and tool calls.
* Vision input does not support public image URLs. Use base64 or `ms://<file-id>`, and make `content` an array of objects.

## FAQ

> [accordion] How is Kimi K3 billed?
>
> Kimi K3 offers a 1M-token context and uses flat pay-as-you-go pricing — there is no tiering by context length. Billable items include cache writes (with separate TTL 5min and TTL 1h tiers), input (with separate rates for cache hits and misses), and output, all billed at uniform per-token prices. See [Kimi K3 pricing](../pricing/chat.md).

> [accordion] How do I turn off Kimi K3's chain-of-thought?
>
> You can't — K3 always thinks. If the reasoning takes too long, set `reasoning_effort` to `low` to reduce the reasoning effort. See [Reasoning Effort](use-reasoning-effort.md).

## Model Pricing

For token pricing details, refer to [Model Pricing](../pricing/chat.md).

## Related docs

> [card] [Reasoning Effort](use-reasoning-effort.md)
>
> Configure reasoning\_effort.

> [card] [Vision Input](use-kimi-vision-model.md)
>
> Send images and videos.

> [card] [Structured Output](response_format.md)
>
> Use strict JSON Schema.

> [card] [Partial Mode](use-partial-mode-feature-of-kimi-api.md)
>
> Continue from a prefix.

> [card] [Tool Choice](use-tool-choice.md)
>
> Control whether the model calls tools.

> [card] [Dynamic Tool Loading](use-dynamic-tool-loading.md)
>
> Inject tool definitions on demand.

> [card] [Tool Calling Best Practices](kimi-k3-tool-calling-best-practice.md)
>
> Combine tool-calling features.

> [card] [Official Tools](use-official-tools.md)
>
> Integrate Formula tools.

> [card] [Kimi K3 Pricing](../pricing/chat.md)
>
> Review input and output prices.
