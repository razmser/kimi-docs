> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Model Parameter Reference

> Compare Chat Completions API parameter defaults and constraints for the K3 series, Kimi K2.7 Code, and Kimi K2.6.

## Parameter Comparison

The K3 series column applies to `kimi-k3`, `kimi-k3.1-pro`, and `kimi-k3.1-pro-highspeed`. `kimi-k2.7-code-highspeed` has the same parameter constraints as `kimi-k2.7-code`.

| Parameter | K3 series | `kimi-k2.7-code` | `kimi-k2.6` |
| - | - | - | - |
| Context window | 1M tokens | 256K tokens | 256K tokens |
| `thinking` | Not supported | Optional. Thinking and Preserved Thinking are always on. | Optional. Thinking is on by default; reasoning from previous turns is not preserved by default. |
| `reasoning_effort` | `"low"` / `"high"` / `"max"` (default `"max"`) | Not supported | Not supported |
| [`tool_choice`](../guide/use-tool-choice.md) | `"auto"` / `"none"` / `"required"` | Passing `"required"` returns an error. | Passing `"required"` returns an error. |
| `temperature` | Fixed at `1.0` | Fixed at `1.0` | Fixed at `1.0` with thinking on and `0.6` with thinking off |
| `top_p` | Fixed at `0.95` | Fixed at `0.95` | Fixed at `0.95` |
| `n` | Fixed at `1` | Fixed at `1` | Fixed at `1` |
| `presence_penalty` / `frequency_penalty` | Fixed at `0` | Fixed at `0` | Fixed at `0` |

> [!NOTE]
>
> "Fixed" means the parameter cannot be modified: passing any other value returns an error, so do not pass it explicitly. When `temperature` is close to 0, `n` can only be 1. Otherwise, the API returns `invalid_request_error`.

### `thinking`

To turn off thinking for `kimi-k2.6`, set `thinking.type` to `"disabled"` in the request.

```bash theme={null}
curl https://api.moonshot.ai/v1/chat/completions \
  --header "Authorization: Bearer $MOONSHOT_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{
    "model": "kimi-k2.6",
    "messages": [{"role": "user", "content": "What is 1 + 1?"}],
    "thinking": {"type": "disabled"}
  }'
```

With thinking off, the response does not include `reasoning_content`. For supported values of `thinking.type` and `thinking.keep`, see [Chat Completions](chat.md).

### `reasoning_effort`

K3 always reasons with Preserved Thinking enabled. Configure its reasoning effort with the top-level `reasoning_effort` request field, which supports `"low"`, `"high"`, and `"max"` (default `"max"`). See [Reasoning Effort](../guide/use-reasoning-effort.md).

> [!WARNING]
>
> Switching levels invalidates prefix-cache hits. Decide on the `effort` level before the conversation starts and avoid switching it mid-session.

### FAQ

**How do I switch from `kimi-k2.6` or `kimi-k2.7-code` to `kimi-k3`?**

Set `model` to `kimi-k3` and remove `thinking` from the request. Use the top-level `reasoning_effort` field to set the reasoning effort. In multi-turn conversations and tool calls, pass the complete `assistant` message returned by the API back in `messages`, including `reasoning_content`.

**My code uses OpenAI's `reasoning_effort` — do I need to change it for `kimi-k3`?**

No. K3 supports top-level `reasoning_effort` with `"low"`, `"high"`, and `"max"` as accepted values and `"max"` as the default.
