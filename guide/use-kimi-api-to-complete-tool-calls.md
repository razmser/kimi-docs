> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Use Kimi API for Tool Calls

> Define, register, and execute Kimi API `tool_calls`, return tool results across turns, and review streaming entry points and common pitfalls.

Tool calls (`tool_calls`) let Kimi models move from "talking" to "doing": based on the conversation context, the model decides whether to invoke a tool and generates JSON arguments, your application executes the tool and returns the output, and the model produces the final answer from that result. With `tool_calls`, Kimi models can search the web, query databases, or interact with external services. This page walks through defining, registering, and executing tools with an online search example, then points to streaming tool-call guides and message-pairing rules.

## The Complete Flow of a Tool Call

A `tool_calls` round trip consists of six steps:

1. Define your tools using JSON Schema;
2. Pass the tool definitions to the model via the `tools` parameter (you can register multiple tools in one request);
3. Based on the conversation context, the model decides which tool or tools to invoke — or chooses not to call any tool;
4. When calling tools, the model outputs the target tool names and JSON-formatted arguments;
5. Your application executes the corresponding tools with those arguments and sends the results back to the model;
6. The model generates the final user-facing reply based on the tool execution results.

> [!TIP]
>
> If your application mounts dozens or hundreds of tools, use [Dynamically Loaded Tools](use-dynamic-tool-loading.md) to inject tool definitions on demand instead of sending the full list in every request — this substantially reduces token consumption and improves tool-selection accuracy.

## Give the Model Internet Access with Tool Calls

A model's built-in knowledge comes from its training data, so it cannot answer time-sensitive questions on its own. Below, we define two tools — a "search engine" and a "web page reader" — so the model can look up current information online before answering.

### Define Tools with JSON Schema

When looking up information on the web, a person typically enters a query into a search engine, scans the results, and opens one or more pages to read the details. Abstracting those two actions into JSON Schema tool definitions gives the model the same workflow:

> [JSON Schema](https://json-schema.org/) is a vocabulary that you can use to annotate and validate JSON documents.

Consider the following minimal JSON Schema:

```json theme={null}
{
    "type": "object",
    "properties": {
        "name": {
            "type": "string"
        }
    }
}
```

This schema describes a JSON object with a `name` property of type `string`, such as:

```json theme={null}
{
    "name": "Hei"
}
```

Describing tools with JSON Schema tells the model what parameters each function expects, along with each parameter's type and purpose. Let's define the `search` and `crawl` tools:

> [tab] python
>
> ```python expandable theme={null}
> tools = [
>     {
>         "type": "function",  # Fixed as "function" for custom function tools
>         "function": {  # Define the function metadata and parameter schema under "function"
>             "name": "search",  # Use ASCII letters, digits, hyphens, and underscores
>             "description": (
>                 "Search the internet using a search engine.\n\n"
>                 "Call this tool when your knowledge cannot answer the user's question or when the user asks for an online search. "
>                 "Extract the search query from the conversation as the value of query.\n"
>                 "Search results include each page's title, URL, and snippet."
>             ),  # Describe what the function does and when to call it so the model selects tools accurately
>             "parameters": {  # Declare accepted parameters with JSON Schema
>                 "type": "object",  # Top-level type must be "object"
>                 "required": ["query"],  # List required parameter names
>                 "properties": {  # Define each parameter by name
>                     "query": {
>                         "type": "string",
>                         "description": "The search query extracted from the user's question or conversation context.",
>                     }
>                 },
>             },
>         },
>     },
>     {
>         "type": "function",
>         "function": {
>             "name": "crawl",
>             "description": "Fetch the text content of a web page given its URL.",
>             "parameters": {
>                 "type": "object",
>                 "required": ["url"],
>                 "properties": {
>                     "url": {
>                         "type": "string",
>                         "description": "The URL of the web page to fetch, typically obtained from search results.",
>                     }
>                 },
>             },
>         },
>     },
> ]
> ```

> [tab] node.js
>
> ```js expandable theme={null}
> const tools = [
>     {
>         type: "function", // Fixed as "function" for custom function tools
>         function: { // Define the function metadata and parameter schema under "function"
>             name: "search", // Use ASCII letters, digits, hyphens, and underscores
>             description: [
>                 "Search the internet using a search engine.",
>                 "",
>                 "Call this tool when your knowledge cannot answer the user's question or when the user asks for an online search. Extract the search query from the conversation as the value of query.",
>                 "Search results include each page's title, URL, and snippet.",
>             ].join("\n"), // Describe what the function does and when to call it so the model selects tools accurately
>             parameters: { // Declare accepted parameters with JSON Schema
>                 type: "object", // Top-level type must be "object"
>                 required: ["query"], // List required parameter names
>                 properties: { // Define each parameter by name
>                     query: {
>                         type: "string",
>                         description: "The search query extracted from the user's question or conversation context.",
>                     },
>                 },
>             },
>         },
>     },
>     {
>         type: "function",
>         function: {
>             name: "crawl",
>             description: "Fetch the text content of a web page given its URL.",
>             parameters: {
>                 type: "object",
>                 required: ["url"],
>                 properties: {
>                     url: {
>                         type: "string",
>                         description: "The URL of the web page to fetch, typically obtained from search results.",
>                     },
>                 },
>             },
>         },
>     },
> ];
> ```

Every custom function tool follows this structure:

```json theme={null}
{
    "type": "function",
    "function": {
        "name": "NAME",
        "description": "DESCRIPTION",
        "parameters": {
            "type": "object",
            "properties": {}
        }
    }
}
```

You define `name`, `description`, and `parameters.properties`: `description` explains what the tool does and when to use it, while `parameters` specifies the input schema. **When calling the tool, the model generates a JSON object conforming to `parameters` and returns it in `arguments`.**

### Register the Tools with the Model

Pass the `search` tool in the `tools` parameter to verify that the model invokes it as expected:

The examples on this page use the latest model `kimi-k3.1-pro` by default. K3.1 Pro configures reasoning effort with the top-level `reasoning_effort` request field (supports `"low"` / `"high"` / `"max"`, default `"max"`). To use another model such as `kimi-k3.1-pro-highspeed`, replace the `model` field and adjust the parameter configuration for that model. See the [Model Parameter Reference](../api/models-overview.md).

> [tab] python
>
> ```python expandable theme={null}
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
>             "name": "search",
>             "description": (
>                 "Search the internet using a search engine.\n\n"
>                 "Call this tool when your knowledge cannot answer the user's question or when the user asks for an online search. "
>                 "Extract the search query from the conversation as the value of query.\n"
>                 "Search results include each page's title, URL, and snippet."
>             ),
>             "parameters": {
>                 "type": "object",
>                 "required": ["query"],
>                 "properties": {
>                     "query": {
>                         "type": "string",
>                         "description": "The search query extracted from the user's question or conversation context.",
>                     }
>                 },
>             },
>         },
>     },
>     # You can also register the crawl tool for the full workflow:
>     # {
>     #     "type": "function",
>     #     "function": {
>     #         "name": "crawl",
>     #         "description": "Fetch the text content of a web page given its URL.",
>     #         "parameters": {
>     #             "type": "object",
>     #             "required": ["url"],
>     #             "properties": {
>     #                 "url": {
>     #                     "type": "string",
>     #                     "description": "The URL of the web page to fetch, typically obtained from search results.",
>     #                 }
>     #             },
>     #         },
>     #     },
>     # },
> ]
>
> completion = client.chat.completions.create(
>     model="kimi-k3.1-pro",
>     messages=[
>         {"role": "system", "content": "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated."},
>         {"role": "user", "content": "Please search the internet for 'Context Caching' and tell me what it is."},  # Ask the model to search online
>     ],
>     tools=tools,  # <-- Pass the tool definitions via tools
> )
>
> print(completion.choices[0].model_dump_json(indent=4))
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
>             name: "search",
>             description: [
>                 "Search the internet using a search engine.",
>                 "",
>                 "Call this tool when your knowledge cannot answer the user's question or when the user asks for an online search. Extract the search query from the conversation as the value of query.",
>                 "Search results include each page's title, URL, and snippet.",
>             ].join("\n"),
>             parameters: {
>                 type: "object",
>                 required: ["query"],
>                 properties: {
>                     query: {
>                         type: "string",
>                         description: "The search query extracted from the user's question or conversation context.",
>                     },
>                 },
>             },
>         },
>     },
>     // You can also register the crawl tool for the full workflow:
>     // {
>     //     type: "function",
>     //     function: {
>     //         name: "crawl",
>     //         description: "Fetch the text content of a web page given its URL.",
>     //         parameters: {
>     //             type: "object",
>     //             required: ["url"],
>     //             properties: {
>     //                 url: {
>     //                     type: "string",
>     //                     description: "The URL of the web page to fetch, typically obtained from search results.",
>     //                 },
>     //             },
>     //         },
>     //     },
>     // },
> ];
>
> async function main() {
>     const completion = await client.chat.completions.create({
>         model: "kimi-k3.1-pro",
>         messages: [
>             { role: "system", content: "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated." },
>             { role: "user", content: "Please search the internet for 'Context Caching' and tell me what it is." }, // Ask the model to search online
>         ],
>         tools: tools, // <-- Pass the tool definitions via tools
>     });
>
>     console.log(JSON.stringify(completion.choices[0], null, 4));
> }
>
> main();
> ```

The model responds with a choice like the following (with thinking models, `message` also includes `reasoning_content`):

```json theme={null}
{
    "finish_reason": "tool_calls",
    "message": {
        "content": "",
        "role": "assistant",
        "tool_calls": [
            {
                "id": "search:0",
                "function": {
                    "arguments": "{\n    \"query\": \"Context Caching\"\n}",
                    "name": "search"
                },
                "type": "function"
            }
        ]
    }
}
```

A `finish_reason` of `tool_calls` indicates that the model is requesting tool execution rather than returning a final answer.

At this stage `message.content` is empty because the model is waiting for tool output before composing its reply. The `tool_calls` array lists every call requested in this turn — **the model can invoke multiple tools in parallel, whether different tools or the same tool with different arguments**. Each entry contains a unique `id`, the target `function.name`, and `function.arguments` as a JSON-serialized object (`type` is `function` for custom function tools).

Next, execute the requested tools with those arguments.

### Execute the Tools and Return the Results

Kimi does not execute external code on your behalf — your application runs the tools and sends back their output. Consider a typical assistant application with three participants: **the user, your application, and the Kimi API. When the user asks a question, your application calls the Kimi API with `tools`; the model responds with `tool_calls`; your application runs the corresponding local or remote functions and appends their results to `messages`; and the Kimi API generates the final answer (`finish_reason=stop`), which your application displays to the user.** The intermediate tool calls remain transparent to the end user, who sees only the final response.

The complete example below implements this multi-step tool execution loop:

> [tab] python
>
> ```python expandable theme={null}
> import json
> import os
> from typing import Any
>
> import httpx
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
>             "name": "search",
>             "description": (
>                 "Search the internet using a search engine.\n\n"
>                 "Call this tool when your knowledge cannot answer the user's question or when the user asks for an online search. "
>                 "Extract the search query from the conversation as the value of query.\n"
>                 "Search results include each page's title, URL, and snippet."
>             ),
>             "parameters": {
>                 "type": "object",
>                 "required": ["query"],
>                 "properties": {
>                     "query": {
>                         "type": "string",
>                         "description": "The search query extracted from the user's question or conversation context.",
>                     }
>                 },
>             },
>         },
>     },
>     {
>         "type": "function",
>         "function": {
>             "name": "crawl",
>             "description": "Fetch the text content of a web page given its URL.",
>             "parameters": {
>                 "type": "object",
>                 "required": ["url"],
>                 "properties": {
>                     "url": {
>                         "type": "string",
>                         "description": "The URL of the web page to fetch, typically obtained from search results.",
>                     }
>                 },
>             },
>         },
>     },
> ]
>
>
> def search_impl(query: str) -> list[dict[str, Any]]:
>     """
>     Query your chosen search engine API (such as Bing) and return a list of dicts with title, URL, and snippet.
>     Add authentication, validation, and error handling for production use.
>     """
>     r = httpx.get("https://your.search.api", params={"query": query})
>     return r.json()
>
>
> def search(arguments: dict[str, Any]) -> dict[str, Any]:
>     query = arguments["query"]
>     result = search_impl(query)
>     return {"result": result}
>
>
> def crawl_impl(url: str) -> str:
>     """
>     Fetch web page text for the given URL. Clean HTML or strip boilerplate as needed in production.
>     """
>     r = httpx.get(url)
>     return r.text
>
>
> def crawl(arguments: dict[str, Any]) -> dict[str, Any]:
>     url = arguments["url"]
>     content = crawl_impl(url)
>     return {"content": content}
>
>
> # Map each tool name to its local implementation for dispatch
> tool_map = {
>     "search": search,
>     "crawl": crawl,
> }
>
> messages: list[Any] = [
>     {
>         "role": "system",
>         "content": "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated.",
>     },
>     {"role": "user", "content": "Please search online for Context Caching and tell me what it is."},
> ]
>
> finish_reason = None
>
> # Repeat until the model finishes without requesting more tools (finish_reason == "stop"):
> # 1. When finish_reason == "tool_calls", append choice.message as-is (preserving tool_calls and reasoning_content);
> # 2. Execute each requested tool, append a role="tool" message with its result, and call the API again;
> # 3. When finish_reason == "stop", exit the loop and print the final answer.
> while finish_reason is None or finish_reason == "tool_calls":
>     completion = client.chat.completions.create(
>         model="kimi-k3.1-pro",
>         messages=messages,
>         tools=tools,
>     )
>     choice = completion.choices[0]
>     finish_reason = choice.finish_reason
>     if finish_reason == "tool_calls":
>         messages.append(choice.message)  # <-- Preserve the assistant message (including tool_calls and reasoning_content)
>         for tool_call in choice.message.tool_calls:
>             tool_call_name = tool_call.function.name
>             tool_call_arguments = json.loads(tool_call.function.arguments)
>             tool_function = tool_map[tool_call_name]
>             tool_result = tool_function(tool_call_arguments)
>
>             messages.append({
>                 "role": "tool",
>                 "tool_call_id": tool_call.id,
>                 "name": tool_call_name,
>                 "content": json.dumps(tool_result),  # <-- Serialize tool output as a string for content
>             })
>
> print(choice.message.content)
> ```

> [tab] node.js
>
> ```js expandable theme={null}
> const axios = require("axios");
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
>             name: "search",
>             description:
>                 "Search the internet using a search engine.\n\nCall this tool when your knowledge cannot answer the user's question or when the user asks for an online search. Extract the search query from the conversation as the value of query.\nSearch results include each page's title, URL, and snippet.",
>             parameters: {
>                 type: "object",
>                 required: ["query"],
>                 properties: {
>                     query: {
>                         type: "string",
>                         description: "The search query extracted from the user's question or conversation context.",
>                     },
>                 },
>             },
>         },
>     },
>     {
>         type: "function",
>         function: {
>             name: "crawl",
>             description: "Fetch the text content of a web page given its URL.",
>             parameters: {
>                 type: "object",
>                 required: ["url"],
>                 properties: {
>                     url: {
>                         type: "string",
>                         description: "The URL of the web page to fetch, typically obtained from search results.",
>                     },
>                 },
>             },
>         },
>     },
> ];
>
> async function searchImpl(query) {
>     const response = await axios.get("https://your.search.api", { params: { query } });
>     return response.data;
> }
>
> async function search(args) {
>     const result = await searchImpl(args.query);
>     return { result };
> }
>
> async function crawlImpl(url) {
>     const response = await axios.get(url);
>     return response.data;
> }
>
> async function crawl(args) {
>     const content = await crawlImpl(args.url);
>     return { content };
> }
>
> const toolMap = {
>     search: search,
>     crawl: crawl,
> };
>
> const messages = [
>     { role: "system", content: "You are Kimi, an AI assistant provided by Moonshot AI. You are proficient in Chinese and English conversations. You provide users with safe, helpful, and accurate answers, and refuse to answer questions involving terrorism, racism, or explicit violence. Moonshot AI is a proper noun and should not be translated." },
>     { role: "user", content: "Please search online for Context Caching and tell me what it is." },
> ];
>
> async function main() {
>     let finishReason = null;
>     let choice;
>
>     while (finishReason === null || finishReason === "tool_calls") {
>         const completion = await client.chat.completions.create({
>             model: "kimi-k3.1-pro",
>             messages: messages,
>             tools: tools,
>         });
>         choice = completion.choices[0];
>         finishReason = choice.finish_reason;
>         if (finishReason === "tool_calls") {
>             messages.push(choice.message); // <-- Preserve the assistant message (including tool_calls and reasoning_content)
>             for (const toolCall of choice.message.tool_calls) {
>                 const toolCallName = toolCall.function.name;
>                 const toolCallArguments = JSON.parse(toolCall.function.arguments);
>                 const toolFunction = toolMap[toolCallName];
>                 const toolResult = await toolFunction(toolCallArguments);
>
>                 messages.push({
>                     role: "tool",
>                     tool_call_id: toolCall.id,
>                     name: toolCallName,
>                     content: JSON.stringify(toolResult), // <-- Serialize tool output as a string for content
>                 });
>             }
>         }
>     }
>     console.log(choice.message.content);
> }
>
> main();
> ```

We use a `while` loop because answering a question often takes more than one round of tool calls. For an online search, the model typically calls `search` first to get candidate URLs and then calls `crawl` in parallel to read the relevant pages. The `messages` history evolves like this:

```text theme={null}
system: prompt                                                                                               # System prompt
user: prompt                                                                                                 # User question
assistant: tool_call(name=search, arguments={query: query})                                                  # Model invokes a single tool
tool: search_result(tool_call_id=tool_call.id, name=search)                                                  # Submit the search result
assistant: tool_call_1(name=crawl, arguments={url: url_1}), tool_call_2(name=crawl, arguments={url: url_2})  # Model invokes multiple tools in parallel
tool: crawl_content(tool_call_id=tool_call_1.id, name=crawl)                                                 # Submit result for tool_call_1
tool: crawl_content(tool_call_id=tool_call_2.id, name=crawl)                                                 # Submit result for tool_call_2
assistant: message_content(finish_reason=stop)                                                               # Model produces final reply, ending the turn
```

With your own `search` and `crawl` implementations plugged in, the model can search the web and answer questions from live pages end to end.

## Tool Calls in Streaming Output

The examples above use non-streaming requests. Tool calls also work with streaming enabled (`stream=true`): the model delivers tool arguments across multiple fragments, and your client reassembles those fragments by index and waits until the stream closes cleanly and passes validation before executing any tool. For protocol-specific event structures and complete streaming code examples, see the Streaming guide:

* **Chat Completions (`POST /v1/chat/completions`)**: streams `function.arguments` fragments inside `delta.tool_calls` keyed by tool `index`. See [Streaming — Handle Tool Calls](utilize-the-streaming-output-feature-of-kimi-api.md#handle-tool-calls).
* **Messages API (`POST /anthropic/v1/messages`)**: enables fine-grained tool streaming automatically, delivering argument fragments via `input_json_delta.partial_json` in `content_block_delta` events keyed by content block `index`. See [Streaming — Messages API](utilize-the-streaming-output-feature-of-kimi-api.md#messages-api).

## Use tool\_calls Instead of function\_call

`tool_calls` evolved from the legacy `function_call` interface (`function_call` can be viewed as a single-call subset of `tool_calls`). Because OpenAI has deprecated `function_call` and `functions`, the Kimi API no longer supports `function_call`; always use `tool_calls` instead. Compared with `function_call`, `tool_calls` offers two main benefits:

* **Parallel execution**: the model can return multiple `tool_calls` in a single response, allowing your application to run independent calls concurrently and reduce wall-clock latency;
* **Fewer round trips**: when multiple tool calls do not depend on one another, the model tends to batch them in a single turn, saving repeated prompt token overhead compared with sequential `function_call` turns.

## Notes

* When `finish_reason=tool_calls`, `message.content` is occasionally non-empty: the model may briefly explain which tools it is calling and why. During long-running or multi-step tool workflows, surfacing this text can reassure users while they wait and make it easier to spot and interrupt unintended tool selections;
* Tool definitions in `tools` count toward input tokens. Ensure that the combined tokens of `tools` and `messages` stay within the model's context window.

### Keep Every tool\_call Matched to a tool Message

With tool calls, `messages` no longer alternates strictly between `user` and `assistant`:

```text theme={null}
system: ...
user: ...
assistant: ...
user: ...
assistant: ...
```

Instead, an `assistant` tool-call message is followed by one or more `tool` messages:

```text theme={null}
system: ...
user: ...
assistant: ...
tool: ...
tool: ...
assistant: ...
```

Whenever the model returns `tool_calls`, follow that assistant message with one `role=tool` message for every `tool_call`, each carrying the matching `tool_call_id`. A mismatch in count or `tool_call_id` values causes the next request to fail.

### Troubleshoot the tool\_call\_id not found Error

If you encounter a `tool_call_id not found` error, check whether you forgot to append the `role=assistant` message returned by the API before appending the `role=tool` messages. The required sequence is:

```text theme={null}
system: ...
user: ...
assistant: ...  # <-- Do not omit this assistant message containing tool_calls
tool: ...
tool: ...
assistant: ...
```

Appending `choice.message` to `messages` before adding the `role=tool` results prevents this error.

*Note: Every assistant message preceding `role=tool` messages must retain the full `tool_calls` field returned by the API (and, for thinking models, the `reasoning_content` field — see [Preserved Thinking](use-thinking-models.md)). Append `choice.message` as-is to avoid dropping required fields.*
