> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Web Search Pro

> Run a web search via the /v1/tools/search_pro endpoint with site and time-range constraints, plus structured content chunks.

On top of Web Search Basic, this endpoint lets you restrict results to specific sites with `sites` (multiple sites are OR'd, up to 5), constrain the time range with `time_window`, and return structured content chunks (`chunks`) for each result. Ideal for retrieval scenarios that require control over sources and freshness.

`start` and `end` of `time_window` accept `YYYY`, `YYYY-MM`, and `YYYY-MM-DD` formats. Both are normalized to the first day of their period before comparison, and `start` must not be later than `end`.

> [accordion] Usage Example
>
> ```python python expandable theme={null}
> import os
> import requests
>
> api_key = os.environ.get("MOONSHOT_API_KEY")
> url = "https://api.moonshot.ai/v1/tools/search_pro"
>
> response = requests.post(
>     url,
>     headers={"Authorization": f"Bearer {api_key}"},
>     json={
>         "text_query": "Kimi K2 model release",
>         "limit": 5,
>         "sites": ["moonshot.ai", "kimi.com"],
>         "time_window": {"start": "2026-01", "end": "2026-09"},
>     },
> )
> print(response.json())
> ```
>
> ```bash curl expandable theme={null}
> curl https://api.moonshot.ai/v1/tools/search_pro \
>   -H "Content-Type: application/json" \
>   -H "Authorization: Bearer $MOONSHOT_API_KEY" \
>   -d '{"text_query": "Kimi K2 model release", "limit": 5, "sites": ["moonshot.ai", "kimi.com"], "time_window": {"start": "2026-01", "end": "2026-09"}}'
> ```
>
> ```javascript node.js expandable theme={null}
> const apiKey = process.env.MOONSHOT_API_KEY;
>
> async function main() {
>     const response = await fetch("https://api.moonshot.ai/v1/tools/search_pro", {
>         method: "POST",
>         headers: {
>             "Content-Type": "application/json",
>             Authorization: `Bearer ${apiKey}`,
>         },
>         body: JSON.stringify({
>             text_query: "Kimi K2 model release",
>             limit: 5,
>             sites: ["moonshot.ai", "kimi.com"],
>             time_window: { start: "2026-01", end: "2026-09" },
>         }),
>     });
>     const data = await response.json();
>     console.log(data);
> }
>
> main();
> ```

> [accordion] Response Fields
>
> | Field | Type | Description |
> | - | - | - |
> | `search_results` | array\[object] | List of search results; an empty array when nothing matches |
> | `search_results[].authority` | string | Authority level of the source |
> | `search_results[].date` | string | Date of the search result |
> | `search_results[].icon` | string | Site icon URL |
> | `search_results[].mime` | string | MIME type of the content |
> | `search_results[].site_name` | string | Site name |
> | `search_results[].snippet` | string | Snippet of the search result |
> | `search_results[].title` | string | Title of the search result |
> | `search_results[].url` | string | URL of the search result |
> | `search_results[].chunks` | array\[object] | Structured content chunks collected for each page; an empty array when no content is available |
> | `search_results[].chunks[].text` | string | Content of the chunk |
> | `search_results[].chunks[].score` | number | Relevance score of the chunk to the query |
>
> Except for `chunks`, the string fields above are returned as empty strings when no data is available.
>
> **Response Example**
>
> ```json theme={null}
> {
>     "search_results": [
>         {
>             "authority": "S",
>             "date": "2026-06-01",
>             "icon": "https://platform.kimi.ai/favicon.ico",
>             "mime": "text/html",
>             "site_name": "Kimi API Open Platform",
>             "snippet": "Documentation entry of the Kimi API open platform.",
>             "title": "API Overview - Kimi API Open Platform",
>             "url": "https://platform.kimi.ai/docs/api/overview",
>             "chunks": [
>                 {
>                     "text": "The Kimi API provides chat completions, files, and batch endpoints, compatible with OpenAI and Anthropic protocols.",
>                     "score": 1.23
>                 }
>             ]
>         }
>     ]
> }
> ```
>
> **Common Response Headers**
>
> | Header | Description |
> | - | - |
> | `X-Msh-Track-Id` | Request ID. If the client sends this header, its value is reused; otherwise the server generates one. Provide this ID when troubleshooting |
> | `X-Msh-Chat-Id` | Session ID, always `toolgw-{X-Msh-Track-Id}`; provide both IDs when troubleshooting |
>
> All responses, including error responses, carry these two headers.

> [accordion] Error Codes
>
> | HTTP Status | error.type | Typical message | Description |
> | - | - | - | - |
> | 400 | `invalid_request` | `invalid request body` | The request body is not valid JSON |
> | 400 | `invalid_request` | `text_query is required` | Missing search query text |
> | 400 | `invalid_request` | `timeout_seconds must be less than or equal to 60` | `timeout_seconds` must be between 1 and 60 |
> | 400 | `invalid_request` | `limit must be between 1 and 20` | `limit` must be between 1 and 20 |
> | 400 | `invalid_request` | `sites must contain at most 5 entries` | `sites` accepts at most 5 entries |
> | 400 | `invalid_request` | `site must not contain whitespace or parentheses` | A `sites` entry must not contain whitespace or parentheses |
> | 400 | `invalid_request` | `time_window.start is invalid, expect YYYY / YYYY-MM / YYYY-MM-DD` | Invalid `time_window` date format (same for `end`) |
> | 400 | `invalid_request` | `time_window.start must not be after time_window.end` | After normalization to the first day of each period, `start` must not be later than `end` |
> | 401 | - | No error body | Missing or invalid API key |
> | 403 | - | No error body | Account inactive or suspended |
> | 408 | `client_canceled` | `client canceled the request` | The client disconnected before the server responded |
> | 429 | `rate_limited` | `project qps limit exceeded` | Rate or concurrency limit exceeded; the response carries `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers, plus `X-RateLimit-Reset` when a per-second limit is hit |
> | 429 | `rate_limit_unavailable` | `rate limit store unavailable` | Rate-limit service temporarily unavailable; retry later |
> | 500 | `internal_error` | Raw internal error text | Internal server error; retry later, and contact support with the `X-Msh-Track-Id` if it persists |
> | 502 | `upstream_failed` | `upstream service failed` | Service temporarily unavailable; retry later |
> | 504 | `timeout` | `request timeout` | The search timed out; increase `timeout_seconds` or simplify the query and retry |

> [!NOTE]
>
> Billing: you are charged once per successful call (HTTP 200) that returns a non-empty `search_results` array; failed or empty-result calls are free. For pricing details, see [WebSearch Pricing](../pricing/websearch.md).


## OpenAPI

````yaml POST /v1/tools/search_pro
openapi: 3.1.0
info:
  title: Moonshot AI API
  version: 1.0.0
  description: API for Moonshot AI / Kimi large language model services
servers:
  - url: https://api.moonshot.ai
    description: Production
security: []
paths:
  /v1/tools/search_pro:
    post:
      tags:
        - Tools
      summary: Web Search Pro
      description: >-
        Web Search Basic with site and time-range constraints, plus structured
        content chunks (`chunks`) for each result. You are charged once per
        successful call that returns at least one result; failed or empty-result
        calls are free. See [WebSearch Pricing](/pricing/tools).
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ToolsSearchProRequest'
      responses:
        '200':
          description: Search results with content snippets
          headers:
            X-Msh-Track-Id:
              description: >-
                Request ID. If the client sends this header, its value is
                reused; otherwise the server generates one. Provide this ID when
                troubleshooting.
              schema:
                type: string
            X-Msh-Chat-Id:
              description: >-
                Session ID, always `toolgw-{X-Msh-Track-Id}`; provide both IDs
                when troubleshooting.
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ToolsSearchProResponse'
        '400':
          description: Bad request - Invalid parameters or missing required fields
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: >-
            Unauthorized - Invalid or missing API key (status code only, no
            error body)
        '403':
          description: >-
            Forbidden - account inactive or suspended (status code only, no
            error body)
        '408':
          description: Client canceled the request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: >-
            Too many requests - rate or concurrency limit exceeded, or
            rate-limit service temporarily unavailable
          headers:
            X-RateLimit-Limit:
              description: The rate limit that was hit.
              schema:
                type: string
            X-RateLimit-Remaining:
              description: Remaining quota in the current window.
              schema:
                type: string
            X-RateLimit-Reset:
              description: >-
                When the rate-limit window resets (Unix timestamp in seconds;
                returned when a per-second rate limit is hit).
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '502':
          description: Service temporarily unavailable; retry later
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '504':
          description: Request timed out
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
      security:
        - bearerAuth: []
components:
  schemas:
    ToolsSearchProRequest:
      type: object
      properties:
        text_query:
          type: string
          description: Search query text. Must not be empty.
        timeout_seconds:
          type: integer
          minimum: 1
          maximum: 60
          description: >-
            Search timeout in seconds, from 1 to 60. If omitted, no per-request
            timeout is applied.
        limit:
          type: integer
          minimum: 1
          maximum: 20
          default: 5
          description: Maximum number of results to return, from 1 to 20. Defaults to 5.
        sites:
          type: array
          items:
            type: string
          maxItems: 5
          description: >-
            Restrict results to the given sites. Multiple sites are OR'd, up to
            5 entries; each entry must be non-empty and must not contain
            whitespace or parentheses.
        time_window:
          $ref: '#/components/schemas/ToolsSearchTimeWindow'
      required:
        - text_query
    ToolsSearchProResponse:
      type: object
      properties:
        search_results:
          type: array
          items:
            $ref: '#/components/schemas/ToolsSearchProResult'
          description: List of search results; an empty array when nothing matches.
      required:
        - search_results
    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            message:
              type: string
              description: Error message describing what went wrong
            type:
              type: string
              description: Error type
            code:
              type: string
              description: Error code
          required:
            - message
      required:
        - error
    ToolsSearchTimeWindow:
      type: object
      description: >-
        Time-window constraint on result dates. start and end are each
        normalized to the first day of their period before comparison; start
        must not be later than end.
      properties:
        start:
          type: string
          description: >-
            Lower bound of the time window, in YYYY / YYYY-MM / YYYY-MM-DD
            format.
        end:
          type: string
          description: >-
            Upper bound of the time window, in YYYY / YYYY-MM / YYYY-MM-DD
            format.
    ToolsSearchProResult:
      type: object
      properties:
        authority:
          type: string
          description: Authority level of the source.
        date:
          type: string
          description: Date of the search result.
        icon:
          type: string
          description: Site icon URL.
        mime:
          type: string
          description: MIME type of the content.
        site_name:
          type: string
          description: Site name.
        snippet:
          type: string
          description: Snippet of the search result.
        title:
          type: string
          description: Title of the search result.
        url:
          type: string
          description: URL of the search result.
        chunks:
          type: array
          items:
            $ref: '#/components/schemas/ToolsSearchProChunk'
          description: >-
            Structured content chunks collected for each page; an empty array
            when no content is available.
      required:
        - authority
        - date
        - icon
        - mime
        - site_name
        - snippet
        - title
        - url
        - chunks
    ToolsSearchProChunk:
      type: object
      properties:
        text:
          type: string
          description: Content of the chunk.
        score:
          type: number
          format: float
          description: Relevance score of the chunk to the query.
      required:
        - text
        - score
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      description: >-
        The Authorization header expects a Bearer token. Use an MOONSHOT_API_KEY
        as the token. This is a server-side secret key. Generate one on the [API
        keys page](https://platform.kimi.ai/console/api-keys) in your dashboard.

````
