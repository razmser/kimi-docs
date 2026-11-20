> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Programmatic Tool Calling

> Define a client-side code execution tool so the model can write Python code to batch function calls and aggregate results in an isolated container, reducing round trips and context length.

When a task requires querying records in batches, passing data across tools, or filtering and aggregating multiple items, having the model issue standard tool calls one by one adds extra round trips and fills the `messages` history with raw records. Instead, you can define a client-side `run_code` tool so the model can write a short Python script that loops over tool functions and computes the result inside an isolated container, returning only the final aggregated output to the model.

## Implement the client-side code execution tool

This example builds on the [tool-calling workflow](use-kimi-api-to-complete-tool-calls.md). Make sure Docker is installed locally and set your API key in the `MOONSHOT_API_KEY` environment variable before running the code:

```python expandable theme={null}
import json
import os
import subprocess
from typing import Any
import uuid
from openai import OpenAI

Message = dict[str, Any]
MODEL = "kimi-k3.1-pro"
BASE_URL = "https://api.moonshot.ai/v1"
TIMEOUT_SECONDS = 10

# Preloaded sandbox functions
SANDBOX_TOOLS = '''
ORDERS = {
    "ord_1": {"id": "ord_1", "status": "paid", "amount": 120, "detail": "raw-log-alpha"},
    "ord_2": {"id": "ord_2", "status": "refunded", "amount": 80, "detail": "raw-log-beta"},
    "ord_3": {"id": "ord_3", "status": "paid", "amount": 230, "detail": "raw-log-gamma"},
}

def list_order_ids() -> list[str]:
    return list(ORDERS.keys())

def get_order(order_id: str) -> dict:
    return ORDERS[order_id]
'''

# Code execution tool definition
RUN_CODE_TOOL: dict[str, Any] = {
    "type": "function",
    "function": {
        "name": "run_code",
        "description": (
            "Run Python code in an isolated container to batch function calls and aggregate results.\n"
            "The following read-only functions are preloaded in the container (no import needed):\n"
            "- list_order_ids() -> list[str]: returns all order IDs\n"
            "- get_order(order_id: str) -> dict: returns an order dict with id, status ('paid' or 'refunded'), amount (int), and detail (str)\n"
            "Use print() to output only the final aggregated result; do not print raw records."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "code": {"type": "string", "description": "Python 3 code to execute; use print() to output the final result"},
            },
            "required": ["code"],
            "additionalProperties": False,
        },
    },
}


# Request helper
def complete(messages: list[Message], **parameters: Any) -> dict[str, Any]:
    client = OpenAI(
        api_key=os.environ["MOONSHOT_API_KEY"],
        base_url=BASE_URL,
    )
    response = client.chat.completions.create(
        model=MODEL,
        messages=messages,
        max_tokens=8192,
        **parameters,
    )
    return response.model_dump()


# Container execution function
def run_code(code: str, timeout: int = TIMEOUT_SECONDS) -> str:
    if not code.strip():
        return "Error: code must be a non-empty string"
    container_name = f"kimi-ptc-{uuid.uuid4().hex[:12]}"
    script = f"{SANDBOX_TOOLS}\n{code}"
    try:
        completed = subprocess.run(
            [
                "docker", "run", "--rm",
                "--name", container_name,
                "--network", "none",
                "-i", "python:3.12-slim",
                "python", "-I", "-",
            ],
            input=script,
            text=True,
            capture_output=True,
            timeout=timeout,
            check=False,
        )
    except subprocess.TimeoutExpired:
        subprocess.run(["docker", "rm", "-f", container_name], capture_output=True, check=False)
        return f"Error: code execution timed out after {timeout}s"
    except OSError as exc:
        return f"Error: failed to start container: {exc}"

    if completed.returncode != 0:
        stderr = completed.stderr.strip() or f"exit code {completed.returncode}"
        return f"Error: {stderr}"
    return completed.stdout.strip() or "(no output)"
```

The table below summarizes each part of the code:

| Component | Identifier | Role |
| - | - | - |
| Container execution function | `run_code()` | Runs model-generated code in a disposable Docker container with networking disabled and a 10-second timeout; returns standard output on success, or cleans up the container and returns an error string on timeout or failure |
| Request helper | `complete()` | Calls `/v1/chat/completions` through the OpenAI SDK |
| Code execution tool definition | `RUN_CODE_TOOL` | Declares `run_code` as a function tool and documents the preloaded function signatures, return fields, and requirement to print only aggregated results |
| Preloaded sandbox functions | `SANDBOX_TOOLS` | Read-only helper functions (`list_order_ids()` and `get_order()`) preloaded inside the container for the generated script to call |

## Run generated code in your tool-calling loop

Pass `RUN_CODE_TOOL` in `tools`. When the model returns a `run_code` call, execute the generated Python code in the container and append the output back to `messages`:

```python theme={null}
while True:
    response: dict[str, Any] = complete(
        messages=messages,
        tools=[RUN_CODE_TOOL],  # Expose the code execution tool to the model
    )
    assistant: Message = response["choices"][0]["message"]
    messages.append(assistant)  # Keep the complete assistant message returned by the model

    tool_calls: list[dict[str, Any]] = assistant.get("tool_calls") or []
    if not tool_calls:
        print(assistant.get("content"))
        break

    for call in tool_calls:
        name: str = call["function"]["name"]
        arguments: dict[str, Any] = json.loads(call["function"]["arguments"])
        if name == "run_code":
            # Execute the generated code in an isolated container and capture the aggregated output
            output: str = run_code(arguments["code"])
        else:
            output = f"Error: unknown tool {name}"

        # Append the output (or error message) by tool_call_id and continue the loop
        messages.append({
            "role": "tool",
            "tool_call_id": call["id"],
            "content": output,
        })
```

## Frequently asked questions

> [accordion] In a real application, how does generated code access my own data or functions?
>
> The example embeds `ORDERS` and helper functions in `SANDBOX_TOOLS` for demonstration. In production, generated code can access your business data in two common ways:
>
> 1. **Pass a data snapshot**: Before starting the sandbox, serialize the records needed for the current step as JSON and pass them in through standard input or a read-only file. The generated script operates only on the snapshot and never accesses external data. This approach works best when the required dataset is small and known in advance.
> 2. **Forward queries through a controlled bridge**: Place only lightweight stub functions inside the sandbox. When generated code calls a stub, the stub forwards the request to your host service over a local socket or internal read-only endpoint, and the host runs the real query and returns the result. Database passwords and API keys stay on the host and never enter the sandbox. This approach is better suited to large datasets or workflows that need to retrieve data during computation.

> [accordion] Do I have to use Docker as the execution environment?
>
> Model-generated code must run in an isolated environment. Calling `exec()` in the host process runs that code with host privileges: it can read and write host files and read credentials from environment variables such as `MOONSHOT_API_KEY`, and an infinite loop or memory spike could stall the host process.
>
> The example uses a network-disabled disposable Docker container (`--rm --network none`) because Docker is the easiest way to run an isolated environment locally. In production, you can swap in a faster isolated runtime such as a V8 isolate, WebAssembly sandbox, Firecracker microVM, or managed code sandbox service. Whichever runtime you choose, it must meet three requirements: generated code cannot access host credentials, each run has a timeout, and execution errors are returned to the model as regular `tool` results.
