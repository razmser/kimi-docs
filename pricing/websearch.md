> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# WebSearch Pricing

> Review pricing, billing units, and usage notes for the Kimi web-search and fetch tools.

## Product Pricing

**Explanation: Prices exclude applicable taxes. Specific tax obligations are subject to local tax regulations and will be calculated at checkout based on your jurisdiction.**

Web search and fetch capabilities are billed per call; each channel is independent and billed separately:

<table>
  <thead>
    <tr />
  </thead>

  <tbody />
</table>

> [!NOTE]
>
> Failed requests, searches not triggered by the model, or empty results/content are not billed. When using web search via the Responses API or the Formula API, model input/output Tokens are still billed at the corresponding [model pricing](chat.md).

## Billing Details

### Web Search Basic / Web Search Pro

Calls to `/v1/tools/search` or `/v1/tools/search_pro`:

* One call charge applies when the request succeeds (HTTP 200) and the `search_results` field in the response is non-empty (at least one search result is returned).
* Requests that fail (invalid parameters, rate limiting, server errors, timeouts, etc.) or return no search results are not billed.
* Web Search Pro additionally supports restricting results to specific sites via `sites`, constraining the result time range via `time_window`, and returns structured content snippets (`chunks`) for each result.

For endpoint parameters and usage examples, see [Best Practices for Web Search](../guide/best-practices-for-web-search.md).

### URL Fetch

Calls to `/v1/tools/fetch`:

* One call charge applies when the request succeeds (HTTP 200) and the `markdown` field in the response is non-blank (page content was successfully extracted).
* Requests that fail (invalid URL, security risk control, rate limiting, server errors, timeouts, etc.), pages with no extractable content (404 `markdown_not_found`), or empty fetched content are not billed.

For endpoint parameters and usage examples, see [Best Practices for Web Search](../guide/best-practices-for-web-search.md).

### Responses API web\_search

Add `{"type": "web_search"}` to the `tools` array of `POST /v1/responses`:

* The server evaluates the input to determine whether a search is required; if so, it executes the search and injects the results into the model context.
* One call charge applies when the server executes a search (a `web_search_call` item is present in `output`); no charge applies if the model does not trigger a search.
* Including `image` in `search_content_types` (image search) does not incur an additional charge.

See [Responses API](../api/responses.md).

### Formula API web-search

On `kimi-k3`, each execution of the official `web-search` tool through the Formula API (`POST /v1/formulas/{uri}/fibers`) incurs one call charge. See [How to Use Official Tools in Kimi API](../guide/use-official-tools.md).
