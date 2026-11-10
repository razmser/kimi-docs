> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Use Kimi Models in OpenCode

> Install OpenCode, connect it to the Kimi Open Platform through built-in authentication, and use Kimi K3 with its thinking-effort variants.

[OpenCode](https://opencode.ai/) is an open-source programming agent. This guide shows how to connect OpenCode to the Kimi Open Platform through built-in authentication and use the `kimi-k3` model with its 1M-token context window.

> [!NOTE]
>
> This guide is based on OpenCode 1.18.3. Its interface, configuration options, and supported capabilities may change between versions.

## Prerequisites

Complete these prerequisites before you start. Follow the linked official guides for installation and account setup.

> [card] [Install OpenCode](https://opencode.ai/docs/)
>
> Install or update OpenCode with the official documentation.

> [card] [Create an API key](https://platform.kimi.ai/console/api-keys)
>
> Create a Kimi Open Platform API key and keep it private.

> [card] [Check account settings](account-and-payments.md)
>
> Confirm that the account has available balance, then check limits, budgets, and organization settings.

Kimi K3 requires available balance in your account; vouchers granted from new-user verification cannot be used for Kimi K3. Rate limits vary by user tier. See [Rate Limits](../pricing/limits.md). If your organization uses an IP whitelist, see [Manage Your Organization in the Console](org-best-practice.md) and add the public egress IPv4 address of the computer that calls the API.

## Step 1: Configure the API key

Run `opencode auth login` and select **Moonshot AI** in the provider list:

```text theme={null}
$ opencode auth login
┌  Add credential
│
◆  Select provider
│  Search: Moon█ (2 matches)
│  ● Moonshot AI
│  ↑/↓ to select • Enter: confirm • Type: to search
└
```

Then paste your Kimi Open Platform API key and press `Enter`:

```text theme={null}
$ opencode auth login
┌  Add credential
│
◇  Select provider
│  Moonshot AI
│
◇  Enter your API key
│  ▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪▪
│
└  Done
```

> [!WARNING]
>
> Do not put the API key in configuration files, screenshots, or a Git repository. This guide uses a Global Kimi Open Platform key; do not substitute a key from another Kimi service or region.

## Step 2: Select the Kimi K3 model

Run `opencode` to start OpenCode:

```bash theme={null}
opencode
```

Run the `/models` command in the input field:

![Run the /models command](https://mintcdn.com/moonshotai/gDhPv8C9OsSiwQmv/assets/pics/opencode/models-command.png?fit=max\&auto=format\&n=gDhPv8C9OsSiwQmv\&q=85\&s=746dbb26a7cbfbce1efcb5705fbbf198)

Search for and select **Kimi K3** in the Select model dialog:

![Select the Kimi K3 model](https://mintcdn.com/moonshotai/gDhPv8C9OsSiwQmv/assets/pics/opencode/select-model.png?fit=max\&auto=format\&n=gDhPv8C9OsSiwQmv\&q=85\&s=d40d1b661ad5e7740b7e60d4de166207)

## Step 3: Adjust the reasoning effort

Run the `/variants` command in the input field:

![Run the /variants command](https://mintcdn.com/moonshotai/gDhPv8C9OsSiwQmv/assets/pics/opencode/variants-command.png?fit=max\&auto=format\&n=gDhPv8C9OsSiwQmv\&q=85\&s=2965e82cf4c6c14ac67b42e7d287b01e)

Select **max** in the Select variant dialog:

![Select the max variant](https://mintcdn.com/moonshotai/gDhPv8C9OsSiwQmv/assets/pics/opencode/select-variant-max.png?fit=max\&auto=format\&n=gDhPv8C9OsSiwQmv\&q=85\&s=350bef5d3f0fe8a9d09ac739912637cc)

`kimi-k3` defaults to `max` reasoning effort and can also switch to `low` / `high`. See the [Model Parameter Reference](../api/models-overview.md).

When setup is complete, the status bar should show **Kimi K3**, **Moonshot AI**, and **max**:

![Final configuration](https://mintcdn.com/moonshotai/gDhPv8C9OsSiwQmv/assets/pics/opencode/final-config.jpg?fit=max\&auto=format\&n=gDhPv8C9OsSiwQmv\&q=85\&s=fd9615f2304b054f3bedc56656d97a30)

## Learn more

* [OpenCode documentation](https://opencode.ai/docs)
* [Kimi Open Platform quickstart](../get-your-api-key.md)
