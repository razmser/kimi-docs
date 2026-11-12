> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Thinking Models

> Understand Kimi thinking modes, model selection, `reasoning_content`, multi-turn retention, tool calling, and reasoning-token billing.

Thinking models use reasoning tokens to "think" before producing a final answer — breaking down the problem, planning steps, and evaluating alternatives. The reasoning process is returned in the response's `reasoning_content` field. Thinking before answering improves performance on complex reasoning, code generation, and multi-step tool calling, at the cost of higher latency and token usage.

## Choose the right thinking model

This page uses the following thinking models:

* **`kimi-k3.1-pro`**: the flagship thinking model; reasoning and Preserved Thinking are always on, and `reasoning_content` may be returned. Configure its reasoning effort with the top-level `reasoning_effort` request field, which supports `"low"` / `"high"` / `"max"` (default `"max"`).
* **`kimi-k3`**: supports a 1M-token context window and follows the same thinking behavior as K3.1 Pro.

| Request field | `kimi-k3.1-pro` | `kimi-k3` |
| - | - | - |
| `reasoning_effort` | `"low"` / `"high"` / `"max"` (default `"max"`) | `"low"` / `"high"` / `"max"` (default `"max"`) |
| `thinking` | Not supported | Not supported |

If you are doing benchmark testing with kimi api, please refer to this [benchmark best practice](benchmark-best-practice.md).

## Basic calls

### Call kimi-k3.1-pro

`kimi-k3.1-pro` always reasons with Preserved Thinking always on; you do not need to (and should not) pass the `thinking` parameter — just set `model`, and optionally adjust [reasoning effort](use-reasoning-effort.md) with the top-level `reasoning_effort` field:

> [tab] curl
>
> ```bash theme={null}
> $ curl https://api.moonshot.ai/v1/chat/completions \
>     -H "Content-Type: application/json" \
>     -H "Authorization: Bearer $MOONSHOT_API_KEY" \
>     -d '{
>         "model": "kimi-k3.1-pro",
>         "messages": [
>             {
>                 "role": "user",
>                 "content": "Prove that the square root of 2 is irrational."
>             }
>         ]
>     }'
> ```

> [tab] python
>
> ```python theme={null}
> import os
> from openai import OpenAI
>
> client = OpenAI(
>     api_key=os.environ["MOONSHOT_API_KEY"],
>     base_url="https://api.moonshot.ai/v1",
> )
>
> completion = client.chat.completions.create(
>     model="kimi-k3.1-pro",
>     messages=[{"role": "user", "content": "Prove that the square root of 2 is irrational."}],
> )
>
> message = completion.choices[0].message
> if hasattr(message, "reasoning_content"):
>     print(getattr(message, "reasoning_content"))
> print(message.content)
> ```

For multi-turn conversations and tool calls, pass the complete assistant message returned by the API back to `messages` as-is (including `reasoning_content`); see [Preserved Thinking](#preserved-thinking). For more K3.1 Pro usage, see the [Kimi K3.1 Pro Quickstart](kimi-k3-1-pro-quickstart.md).

## Control thinking behavior

### K3.1 Pro: adjust reasoning effort with `reasoning_effort`

`kimi-k3.1-pro` always reasons and does not support the `thinking` parameter. Adjust reasoning effort with the top-level `reasoning_effort` field (`"low"` / `"high"` / `"max"`, default `"max"`); see [Reasoning Effort](use-reasoning-effort.md) for usage and examples.

## Read reasoning\_content from the response

When using a thinking model with thinking enabled, the API response carries the model's reasoning in the `reasoning_content` field. When reading this field:

* In the OpenAI SDK, `ChoiceDelta` and `ChatCompletionMessage` types do not provide a `reasoning_content` field directly, so you cannot access it via `.reasoning_content`. You must use `hasattr(obj, "reasoning_content")` to check if the field exists, and if so, use `getattr(obj, "reasoning_content")` to retrieve its value.
* If you use other frameworks or directly interface with the HTTP API, you can directly obtain the `reasoning_content` field at the same level as the `content` field.
* In streaming output (`stream=True`), the `reasoning_content` field will always appear before the `content` field. In your business logic, you can detect if the `content` field has been output to determine if the reasoning (inference process) is finished.
* Tokens in `reasoning_content` are also controlled by the `max_tokens` parameter: the sum of tokens in `reasoning_content` and `content` must be less than or equal to `max_tokens`.

## Configure multi-step tool calls

Thinking models can perform deep reasoning across multiple tool calls to tackle complex tasks. To get reliable results, **always follow these configuration rules when using thinking models:**

* In both single-task tool-call loops and multi-turn conversations, pass the complete assistant message back as-is, including `reasoning_content`. `kimi-k3.1-pro` and `kimi-k3` always enable Preserved Thinking.
* Set `max_tokens >= 16000` to ensure the full `reasoning_content` and `content` can be returned without truncation.
* **Do not set `temperature`.** `kimi-k3.1-pro` and `kimi-k3` fix `temperature` at `1.0`; do not pass it explicitly (see [Model Parameter Reference](../api/models-overview.md)).
* Enable streaming (`stream=True`). Because thinking models return both `reasoning_content` and regular `content`, the response is larger than usual. Streaming delivers a better user experience and helps avoid network-timeout issues.

### Complete example: generate a daily news report

The example below demonstrates a "Daily News Report Generation" scenario. The model will sequentially call official tools like `date` (to get the date) and `web_search` (to search today's news), and will present deep reasoning throughout this process:

```python expandable theme={null}
import os
import json
import httpx
import openai


class FormulaChatClient:
    def __init__(self, base_url: str, api_key: str):
        """Initialize Formula client"""
        self.base_url = base_url
        self.api_key = api_key
        self.openai = openai.Client(
            base_url=base_url,
            api_key=api_key,
        )
        self.httpx = httpx.Client(
            base_url=base_url,
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30.0,
        )
        # Use kimi-k3.1-pro; reasoning and Preserved Thinking are always enabled
        self.model = "kimi-k3.1-pro"

    def get_tools(self, formula_uri: str):
        """Get tool definitions from Formula API"""
        response = self.httpx.get(f"/formulas/{formula_uri}/tools")
        response.raise_for_status()
        
        try:
            return response.json().get("tools", [])
        except json.JSONDecodeError as e:
            print(f"Error: Unable to parse JSON (status code: {response.status_code})")
            print(f"Response content: {response.text[:500]}")
            raise

    def call_tool(self, formula_uri: str, function: str, args: dict):
        """Call an official tool"""
        response = self.httpx.post(
            f"/formulas/{formula_uri}/fibers",
            json={"name": function, "arguments": json.dumps(args)},
        )
        response.raise_for_status()
        fiber = response.json()
        
        if fiber.get("status", "") == "succeeded":
            return fiber["context"].get("output") or fiber["context"].get("encrypted_output")
        
        if "error" in fiber:
            return f"Error: {fiber['error']}"
        if "error" in fiber.get("context", {}):
            return f"Error: {fiber['context']['error']}"
        return "Error: Unknown error"

    def close(self):
        """Close the client connection"""
        self.httpx.close()


# Initialize client
base_url = os.getenv("MOONSHOT_BASE_URL", "https://api.moonshot.ai/v1")
api_key = os.getenv("MOONSHOT_API_KEY")

if not api_key:
    raise ValueError("MOONSHOT_API_KEY environment variable not set. Please set your API key.")

print(f"Base URL: {base_url}")
print(f"API Key: {api_key[:10]}...{api_key[-10:] if len(api_key) > 20 else api_key}\n")

client = FormulaChatClient(base_url, api_key)

# Define the official tool Formula URIs to use
formula_uris = [
    "moonshot/date:latest",
    "moonshot/web-search:latest"
]

# Load all tool definitions and build mapping
print("Loading official tools...")
all_tools = []
tool_to_uri = {}  # function.name -> formula_uri

for uri in formula_uris:
    try:
        tools = client.get_tools(uri)
        for tool in tools:
            func = tool.get("function")
            if func:
                func_name = func.get("name")
                if func_name:
                    tool_to_uri[func_name] = uri
                    all_tools.append(tool)
                    print(f"  Loaded tool: {func_name} from {uri}")
    except Exception as e:
        print(f"  Warning: Failed to load tool {uri}: {e}")
        continue

print(f"Loaded {len(all_tools)} tools in total\n")

if not all_tools:
    raise ValueError("No tools loaded. Please check API key and network connection.")

# Initialize message list
messages = [
    {
        "role": "system",
        "content": "You are Kimi, a professional news analyst. You excel at collecting, analyzing, and organizing information to generate high-quality news reports.",
    },
]

# User request to generate today's news report
user_request = "Please help me generate a daily news report including important technology, economy, and society news."
messages.append({
    "role": "user",
    "content": user_request
})

print(f"User request: {user_request}\n")

# Begin multi-step conversation loop
max_iterations = 10  # Prevent infinite loops
for iteration in range(max_iterations):
    try:
        completion = client.openai.chat.completions.create(
            model=client.model,
            messages=messages,
            max_tokens=1024 * 32,
            tools=all_tools,
        )
    except openai.AuthenticationError as e:
        print(f"Authentication error: {e}")
        print("Please check if the API key is correct and has the required permissions")
        raise
    except Exception as e:
        print(f"Error while calling the model: {e}")
        raise
    
    # Get response
    message = completion.choices[0].message
    
    # Print reasoning process
    if hasattr(message, "reasoning_content"):
        print(f"=============Reasoning round {iteration + 1} starts=============")
        reasoning = getattr(message, "reasoning_content")
        if reasoning:
            print(reasoning[:500] + "..." if len(reasoning) > 500 else reasoning)
        print(f"=============Reasoning round {iteration + 1} ends=============\n")
    
    # Add assistant message to context (preserve reasoning_content)
    messages.append(message)
    
    # If the model did not call any tools, conversation is done
    if not message.tool_calls:
        print("=============Final Answer=============")
        print(message.content)
        break
    
    # Handle tool calls
    print(f"The model decided to call {len(message.tool_calls)} tool(s):\n")
    
    for tool_call in message.tool_calls:
        func_name = tool_call.function.name
        args = json.loads(tool_call.function.arguments)
        
        print(f"Calling tool: {func_name}")
        print(f"Arguments: {json.dumps(args, ensure_ascii=False, indent=2)}")
        
        # Get corresponding formula_uri
        formula_uri = tool_to_uri.get(func_name)
        if not formula_uri:
            print(f"Error: Could not find Formula URI for tool {func_name}")
            continue
        
        # Call the tool
        result = client.call_tool(formula_uri, func_name, args)
        
        # Print result (truncate if too long)
        if len(str(result)) > 200:
            print(f"Tool result: {str(result)[:200]}...\n")
        else:
            print(f"Tool result: {result}\n")
        
        # Add tool result to message list
        tool_message = {
            "role": "tool",
            "tool_call_id": tool_call.id,
            "name": func_name,
            "content": result
        }
        messages.append(tool_message)

print("\nConversation completed!")

# Cleanup
client.close()
```

This process demonstrates how thinking models use deep reasoning to plan and execute complex multi-step tasks when thinking is enabled. Detailed reasoning steps (`reasoning_content`) are preserved in the context to ensure accurate tool use at every stage.

## Preserve thinking across turns (Preserved Thinking)

Preserved Thinking means passing the `reasoning_content` of previous turns through to the model in a multi-turn conversation so that the model can continue its prior reasoning context.

`kimi-k3.1-pro` and `kimi-k3` always enable Preserved Thinking. In multi-turn conversations and tool calls, append the complete assistant message returned by the API to `messages` as-is, including `reasoning_content`; do not keep only the final answer text.

> [!WARNING]
>
> `reasoning_content` consumes tokens. Historical reasoning content continues to occupy context space and is billed, so keep the context size under control.

## Frequently Asked Questions

### Q1: Why should I keep `reasoning_content`?

A: Keeping `reasoning_content` ensures continuity in multi-step reasoning, especially during tool calls. For K3.1 Pro and K3, multi-turn conversations and tool calls must pass the complete assistant message returned by the API back in `messages` as-is.

### Q2: Does `reasoning_content` consume extra tokens?

A: Yes, `reasoning_content` counts towards your input/output token quota. For detailed pricing, see [Pricing](../pricing/chat.md).
