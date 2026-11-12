> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Quickstart

> Create an API Key and complete your first Kimi API call.

Kimi API Platform provides access to Kimi K3.1 Pro and other models. You can call these models through [Chat Completions API](api/chat.md), [Responses API](api/responses.md), or [Messages API](api/messages.md).

To get started, create an API Key, choose a model, and configure `base_url`. You can then send requests directly over HTTP or use the OpenAI or Anthropic SDK.

> [!NOTE]
>
> [Kimi K3.1 Pro](guide/kimi-k3-1-pro-quickstart.md) and Kimi K3.1 Pro Highspeed are now available. As Kimi's most capable flagship model to date, K3.1 Pro supports a 1M-token context window and native visual understanding, with significant improvements in coding, 3D creation, financial research, and modeling. It is well suited for programming agents such as [Kimi Code](guide/kimi-code-cli.md), [Codex](guide/codex-kimi.md), and [Claude Code](guide/claude-code-kimi.md), as well as knowledge work and deep reasoning. Join the [Kimi developer community](api/join-the-community.md) and let's build with Kimi!

## Get started

**1. Get an API Key**

Visit the [Kimi API Platform](https://platform.kimi.ai/), sign in, then go to [API Keys](https://platform.kimi.ai/console/api-keys) to create and copy an API Key.

Keep your API Key secure. Do not share it with others or hard-code it directly in your application. We recommend storing it in an environment variable:

```bash theme={null}
export MOONSHOT_API_KEY="YOUR_KIMI_API_KEY"
```

> [card] [Kimi API Platform](https://platform.kimi.ai/)
>
> Sign in to the platform to access the console, development workspace, and user center.

> [card] [API Keys](https://platform.kimi.ai/console/api-keys)
>
> Create, copy, and manage the keys used for API calls.

**2. Choose a model**

For a quickstart, we recommend starting with Kimi K3.1 Pro; choose the Kimi K3.1 Pro high-speed variant when you need faster output.

> [card] [Kimi K3.1 Pro](guide/kimi-k3-1-pro-quickstart.md)
>
> `kimi-k3.1-pro` is the first flagship of our new K3.1 series — a 3T-class model with a 1M-token context window and major gains in coding, 3D creation, and financial research and modeling.

> [card] [Kimi K3.1 Pro Highspeed](guide/kimi-k3-1-pro-quickstart.md)
>
> `kimi-k3.1-pro-highspeed` is the high-speed variant of Kimi K3.1 Pro — same capabilities at about 2-3× the speed, for scenarios that need faster output.

**3. Choose an integration method**

Kimi API is compatible with both the OpenAI and Anthropic API formats, so you can choose the integration method that best fits your technology stack.

> [card] [Chat Completions API](api/chat.md)
>
> OpenAI-compatible format, with support for the OpenAI SDK and HTTP access from any language.

> [card] [Responses API](api/responses.md)
>
> OpenAI-compatible format for generating text/JSON output or calling function tools.

> [card] [Messages API](api/messages.md)
>
> Anthropic Messages API compatible, for integrating with the Anthropic SDK, Claude Code, and similar tools.

> [card] [Playground](https://platform.kimi.ai/playground)
>
> Test prompts, model behavior, and business examples without writing code.

**4. Make your first call**

Set `MOONSHOT_API_KEY` to the API Key you created in step 1; the examples read the key from this environment variable.

Before running the examples, prepare:

1. Python 3.9 or later, or Node.js 18 or later.
2. OpenAI SDK 1.0.0 or later. Kimi API is compatible with the OpenAI API format, so you can call Kimi directly with the OpenAI Python or Node.js SDK.
   ```text theme={null}
   pip install --upgrade 'openai>=1.0' # Python
   npm install openai@latest # Node.js
   ```

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
>     messages=[
>         {"role": "system", "content": "You are Kimi, an AI assistant provided by Moonshot AI. You are especially good at conversations in Chinese and English. You provide users with safe, helpful, and accurate answers. You also refuse to answer any questions involving terrorism, racism, pornography, violence, or similar harmful content. Moonshot AI is a proper noun and must not be translated into other languages."},
>         {"role": "user", "content": "Hi, my name is Li Lei. What is 1+1?"}
>     ]
> )
>
> print(completion.choices[0].message.content)
> ```

> [tab] curl
>
> ```bash theme={null}
> curl https://api.moonshot.ai/v1/chat/completions \
>     -H "Content-Type: application/json" \
>     -H "Authorization: Bearer $MOONSHOT_API_KEY" \
>     -d '{
>         "model": "kimi-k3.1-pro",
>         "messages": [
>             {"role": "system", "content": "You are Kimi, an AI assistant provided by Moonshot AI. You are especially good at conversations in Chinese and English. You provide users with safe, helpful, and accurate answers. You also refuse to answer any questions involving terrorism, racism, pornography, violence, or similar harmful content. Moonshot AI is a proper noun and must not be translated into other languages."},
>             {"role": "user", "content": "Hi, my name is Li Lei. What is 1+1?"}
>         ]
>    }'
> ```

> [tab] node.js
>
> ```js theme={null}
> const OpenAI = require("openai");
>
> const client = new OpenAI({
>   apiKey: process.env.MOONSHOT_API_KEY,
>   baseURL: "https://api.moonshot.ai/v1",
> });
>
> async function main() {
>   const completion = await client.chat.completions.create({
>     model: "kimi-k3.1-pro",
>     messages: [
>       {"role": "system", "content": "You are Kimi, an AI assistant provided by Moonshot AI. You are especially good at conversations in Chinese and English. You provide users with safe, helpful, and accurate answers. You also refuse to answer any questions involving terrorism, racism, pornography, violence, or similar harmful content. Moonshot AI is a proper noun and must not be translated into other languages."},
>       {"role": "user", "content": "Hi, my name is Li Lei. What is 1+1?"}
>     ]
>   });
>
>   console.log(completion.choices[0].message.content);
> }
>
> main();
> ```

If the code runs successfully without errors, you will see output similar to:

```text theme={null}
Hello, Li Lei! 1+1 equals 2. This is a basic arithmetic question. If you have any other questions or need help, feel free to let me know.
```

*Note: Because Kimi models are nondeterministic, the actual response may not exactly match the example above.*

## Model parameter notes

The examples on this page use the latest model `kimi-k3.1-pro` by default. K3.1 Pro configures reasoning effort with the top-level `reasoning_effort` request field (supports `"low"` / `"high"` / `"max"`, default `"max"`). To use another model such as `kimi-k3.1-pro-highspeed`, replace the `model` field and adjust the parameter configuration for that model. See the [Model Parameter Reference](api/models-overview.md).

## Explore more features

> [card] [Streaming output](guide/utilize-the-streaming-output-feature-of-kimi-api.md)
>
> Enable `stream` to receive tokens as they are generated. Useful for chat, code generation, and long-form text output.

> [card] [Multi-turn chat](guide/engage-in-multi-turn-conversations-using-kimi-api.md)
>
> Maintain a `messages` list to preserve context history, letting the model remember the conversation.

> [card] [Multimodal input](guide/use-kimi-vision-model.md)
>
> Kimi K3.1 Pro and Kimi K3 support text, image, and video input.

> [card] [Tool calls](guide/use-kimi-api-to-complete-tool-calls.md)
>
> Let the model call external functions or APIs for Agent tasks, web search, and complex workflows.

> [card] [JSON Mode](guide/use-json-mode-feature-of-kimi-api.md)
>
> Force the model to output valid JSON for structured data extraction and downstream integration.

> [card] [Thinking models](guide/use-thinking-models.md)
>
> Use reasoning capabilities for complex tasks, multi-step tool use, and Agent workflows.

## Related resources

> [card] [Kimi K3.1 Pro model](guide/kimi-k3-1-pro-quickstart.md)
>
> View Kimi K3.1 Pro capabilities, calling examples, and best practices.

> [card] [Kimi K3.1 Pro Highspeed](pricing/chat.md)
>
> View pricing for Kimi K3.1 Pro Highspeed.

> [card] [Model list](models.md)
>
> View currently available model names and descriptions.

> [card] [Join the Kimi Developer Community](api/join-the-community.md)
>
> Ask questions, share feedback, and showcase what you are building with Kimi.
