> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# URL Fetch

> Fetch the content of a URL via the /v1/tools/fetch endpoint and get the page title and body text in Markdown.

Fetch the content of a URL and get the page title and body text in Markdown. Ideal for content extraction scenarios such as web reading and data curation.

> [!WARNING]
>
> Only `http` and `https` URLs are supported. URLs blocked by security risk control return a 403 `security_risk` error; pages with no extractable content return a 404 `markdown_not_found` error.

> [accordion] Usage Example
>
> ```python python expandable theme={null}
> import os
> import requests
>
> api_key = os.environ.get("MOONSHOT_API_KEY")
> url = "https://api.moonshot.ai/v1/tools/fetch"
>
> response = requests.post(
>     url,
>     headers={"Authorization": f"Bearer {api_key}"},
>     json={"url": "https://platform.kimi.ai/docs/api/overview"},
> )
> print(response.json())
> ```
>
> ```bash curl expandable theme={null}
> curl https://api.moonshot.ai/v1/tools/fetch \
>   -H "Content-Type: application/json" \
>   -H "Authorization: Bearer $MOONSHOT_API_KEY" \
>   -d '{"url": "https://platform.kimi.ai/docs/api/overview"}'
> ```
>
> ```javascript node.js expandable theme={null}
> const apiKey = process.env.MOONSHOT_API_KEY;
>
> async function main() {
>     const response = await fetch("https://api.moonshot.ai/v1/tools/fetch", {
>         method: "POST",
>         headers: {
>             "Content-Type": "application/json",
>             Authorization: `Bearer ${apiKey}`,
>         },
>         body: JSON.stringify({
>             url: "https://platform.kimi.ai/docs/api/overview",
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
> | `url` | string | The fetched URL |
> | `markdown` | string | Fetched content in Markdown. Text and images appear in page order; images are embedded as placeholders (see the example below), and blocks are separated by blank lines |
> | `title` | string | Page title |
>
> **Response Example**
>
> ```json theme={null}
> {
>     "url": "https://platform.kimi.ai/docs/api/overview",
>     "markdown": "# API Overview\n\nDocumentation entry of the Kimi API open platform.",
>     "title": "API Overview - Kimi API Open Platform"
> }
> ```
>
> **Markdown content example**
>
> ```text theme={null}
> # API Overview
>
> Documentation entry of the Kimi API open platform.
>
> ![image1](https://platform.kimi.ai/assets/logo/dark.svg)
>
> More content...
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
> | 400 | `invalid_url` | `The provided URL is invalid: only http and https are supported` | The URL is empty or uses a scheme other than http/https |
> | 400 | `invalid_url` | `The provided URL is invalid: missing host` | The URL has no host |
> | 401 | - | No error body | Missing or invalid API key |
> | 403 | - | No error body | Account inactive or suspended |
> | 403 | `security_risk` | `We consider the current URL poses a security risk and are unable to provide fetch service at this time.` | The URL was blocked by security risk control; use a different URL |
> | 404 | `markdown_not_found` | `No text/markdown content found for the current URL.` | No extractable content found on the page |
> | 408 | `client_canceled` | `client canceled the request` | The client disconnected before the server responded |
> | 429 | `rate_limited` | `project concurrency limit exceeded` | Rate or concurrency limit exceeded; the response carries `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers, plus `X-RateLimit-Reset` when a per-second limit is hit |
> | 429 | `rate_limit_unavailable` | `rate limit store unavailable` | Rate-limit service temporarily unavailable; retry later |
> | 500 | `internal_error` | Raw internal error text | Internal server error; retry later, and contact support with the `X-Msh-Track-Id` if it persists |
> | 502 | `upstream_failed` | `upstream service failed` | Service temporarily unavailable; retry later |
> | 504 | `timeout` | `request timeout` | The fetch timed out; retry later |

> [!NOTE]
>
> Billing: you are charged once per successful call (HTTP 200) that returns non-blank `markdown` content; failed calls or pages with no extractable content are free. For pricing details, see [WebSearch Pricing](../pricing/websearch.md).


## OpenAPI

````yaml POST /v1/tools/fetch
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
  /v1/tools/fetch:
    post:
      tags:
        - Tools
      summary: URL Fetch
      description: >-
        Fetch the content of a URL and get the page title and body text in
        Markdown (text and images concatenated in order). Only http and https
        URLs are supported. You are charged once per successful call that
        extracts content; failed or empty-content calls are free. See [WebSearch
        Pricing](/pricing/tools).
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ToolsFetchRequest'
      responses:
        '200':
          description: Fetch result
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
                $ref: '#/components/schemas/ToolsFetchResponse'
        '400':
          description: Bad request - Invalid parameters or malformed URL
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
            Forbidden - account inactive or suspended (no error body), or the
            URL was blocked by security risk control (returns a security_risk
            error)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: No extractable content found for the URL (markdown_not_found)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
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
    ToolsFetchRequest:
      type: object
      properties:
        url:
          type: string
          format: uri
          description: URL of the page to fetch. Only http and https are supported.
      required:
        - url
    ToolsFetchResponse:
      type: object
      properties:
        url:
          type: string
          description: The fetched URL.
        markdown:
          type: string
          description: >-
            Fetched content in Markdown. Text and images are concatenated in
            page order, with images rendered as `![imageN](url)` placeholders.
        title:
          type: string
          description: Page title.
      required:
        - url
        - markdown
        - title
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
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      description: >-
        The Authorization header expects a Bearer token. Use an MOONSHOT_API_KEY
        as the token. This is a server-side secret key. Generate one on the [API
        keys page](https://platform.kimi.ai/console/api-keys) in your dashboard.

````
