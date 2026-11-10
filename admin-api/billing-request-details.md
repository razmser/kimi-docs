> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# List request details

> Per-request details for an organization over a time window (data for day T becomes available at 07:00 (UTC+8) on day T+1) with cursor pagination. Amounts are in milli-cent; supports project_id/key filters.

> [!NOTE]
>
> This endpoint requires an Admin Key with the `billing:read` scope.

> [!NOTE]
>
> **Reconcile row-by-row with `X-Msh-Client-Request-Id`**: when calling the inference API, pass your own request ID in the header (`X-Msh-Client-Request-Id: <your-request-id>`); our billing details persist it as `client_request_id`. To reconcile, pull this feed and match your request IDs against `client_request_id` — whether a request was recorded, and exactly how (token volume, off-peak hit or not) — row by row, with no need to infer from "time + tokens". For disconnect auditing use the dedicated `interrupted_requests` feed instead; for amounts, reconcile against daily/product bills (this feed reports tokens only, no amounts) — `client_request_id` links rows across both APIs exactly. Example: "Why was our gateway request `abc123` charged?" → look up its tokens via `client_request_id=abc123` in request\_details, then verify the day's amount in daily\_bills.


## OpenAPI

````yaml GET /v1/admin/organizations/billings/request_details
openapi: 3.0.3
info:
  title: Kimi Admin API
  version: 2026-10-20-beta
  description: >-
    ## Overview


    The Kimi Admin API lets organization administrators manage members,
    invitations, projects, and inference API keys programmatically, and query
    billing, usage, and audit logs — without signing in to the console. It is a
    control-plane API: Admin API credentials cannot call any inference endpoint,
    and inference API keys cannot call this API.


    The base URL is `https://api.moonshot.ai/v1`.


    <Note>

    The Admin API is in **beta** and opens to organizations that have completed
    organization verification. After completing organization verification under
    **Console → Organization → Organization Verification**, organization admins
    can create Admin API keys under **Organization → Admin API Keys**.

    </Note>


    ## Authentication


    Send the Admin API secret (`sk-admin-*`) as a Bearer token:


    ```sh

    curl https://api.moonshot.ai/v1/admin/organizations \
      --header "Authorization: Bearer $MOONSHOT_ADMIN_KEY"
    ```


    A key is bound to exactly one organization; the organization is always
    derived from the key — cross-organization access is impossible by
    construction. If the key owner leaves the organization or is deactivated,
    the key stops working within 30 seconds. Secrets are shown only once, at
    creation or rotation; store them safely.


    ## Scopes


    Every key carries scopes chosen at creation:


    | Scope | Type | Covers |

    | --- | --- | --- |

    | `billing:read` | read | Daily/product bills, request details, interrupted
    requests, account balance, usage aggregation |

    | `org:read` | read | Members, invitations, projects, project members,
    inference API key metadata, spend limits, organization rate limits |

    | `org:write` | write | Member/invitation/project/key writes (implies
    `org:read` — no separate grant needed) |

    | `audit:read` | read | Audit log events |

    | `all` | read+write | Everything above; new scopes are covered
    automatically |


    Write endpoints additionally require the key owner to be an organization
    admin at call time and are rate-limited more strictly (see below).


    ## Rate limits


    | Class | Limit |

    | --- | --- |

    | Read endpoints | 60 requests/minute per organization |

    | Write endpoints | 20 requests/minute per organization |

    | Request details, interrupted requests, usage aggregation | 5 concurrent
    requests per organization |


    Rate limits are counted per organization and endpoint: all Admin Keys in the
    same organization share the quota — more keys do not raise it. Exceeding a
    limit returns `429 rate_limit_exceeded` / `concurrency_limit_exceeded`.


    ## Pagination


    Paginated endpoints (daily/product bills, request details, interrupted
    requests, usage aggregation, audit logs) use cursor pagination: responses
    carry `has_more` and `next`; pass `next` back as the `cursor` query
    parameter. Page size by endpoint: daily/product bills and usage default 100,
    max 1000; request details and interrupted requests default 100, max 10000;
    audit logs default 20, max 100. The cursor carries only the page position —
    restate all filters on every page.


    Organization collection endpoints (members, invitations, projects, project
    members, API keys) return the full list in one response, with no pagination
    parameters. Query parameters outside the protocol (such as `page_size`) are
    silently ignored — double-check parameter names.


    ## Errors


    Errors follow a unified envelope:


    ```json

    {
      "error": {
        "type": "invalid_request_error",
        "code": "invalid_date_range",
        "message": "end must be greater than start.",
        "param": "end",
        "request_id": "..."
      }
    }

    ```


    `data_not_ready` gates the bill endpoints' readiness: current-day data
    becomes ready on T-1 and is queryable from 07:00 CST the next day.


    ## Data conventions


    - Monetary amounts are in **milli-cent** (1 USD = 100,000 milli-cent).

    - Timestamps in query parameters are epoch **milliseconds**; timestamps in
    response bodies are RFC 3339 unless stated otherwise.

    - Time windows are left-closed, right-open, aligned to CST day boundaries;
    timestamps in query parameters are epoch milliseconds, and RFC 3339 in
    responses unless stated otherwise. Passing seconds returns empty data rather
    than an error — check your units.

    - Billing data becomes ready on T-1: the current day is queryable from 07:00
    CST the next day and returns `400 data_not_ready` earlier; interrupted
    request details are exempt (near real time).

    - Retention and window caps: daily/product bills 365 days; request details
    and interrupted requests 90 days with a 31-day per-call cap; audit logs 31
    days per call; usage aggregation windows by granularity (`1m` ≤24 hours,
    `1h` ≤31 days, `1d` ≤93 days, cut on Asia/Shanghai natural days).

    - Each organization holds at most 10 active Admin API keys.
servers:
  - url: https://api.moonshot.ai
    description: Production
security:
  - AdminBearerAuth: []
paths:
  /v1/admin/organizations/billings/request_details:
    get:
      tags:
        - Billing
      summary: List request details
      description: >-
        Per-request details for an organization over a time window (data for day
        T becomes available at 07:00 (UTC+8) on day T+1) with cursor pagination.
        Amounts are in milli-cent; supports project_id/key filters.
      operationId: listRequestDetailsV2
      parameters:
        - $ref: '#/components/parameters/Start'
        - $ref: '#/components/parameters/End'
        - $ref: '#/components/parameters/ProjectID'
        - $ref: '#/components/parameters/APIKey'
        - name: desc
          in: query
          required: false
          schema:
            type: boolean
          description: >-
            Whether to sort rows by server write time, descending. The sort key
            is the server write time, not the `time` field, so rows within the
            same second are not guaranteed to be monotonic by request time.
            Defaults to `true` (descending).
        - name: limit
          in: query
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 10000
            default: 100
          description: >-
            Page size. Defaults to 100 for interactive queries; raise it (up to
            10000) for bulk pulls — per-page latency may grow for large pages
            (not an SLA guarantee). 1000 rows.
        - name: cursor
          in: query
          required: false
          schema:
            type: string
          description: >-
            Cursor from the previous response `next` field. The cursor carries
            only the page position; all query parameters must be restated on
            every page. Keep them unchanged while paginating (except limit) —
            the server does not check them for consistency.
      responses:
        '200':
          description: Request details.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RequestDetailsResponseV2'
        '400':
          description: >-
            Invalid request parameters (illegal/out-of-retention time window,
            invalid enum value, or illegal limit/cursor).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BillingErrorResponse'
        '401':
          description: >-
            Authentication failed (the Admin API key is invalid,
            deleted/expired, or its owner is no longer active).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BillingErrorResponse'
        '403':
          description: Insufficient scope (the Admin Key lacks the billing:read scope).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BillingErrorResponse'
        '404':
          description: Resource not found (no data for this organization).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BillingErrorResponse'
        '429':
          description: >-
            Rate limit exceeded (requests-per-minute cap or concurrency cap
            hit).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BillingErrorResponse'
        '500':
          description: Internal server error (contact support with the request_id).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BillingErrorResponse'
      security:
        - AdminBearerAuth: []
components:
  parameters:
    Start:
      name: start
      in: query
      required: true
      schema:
        type: integer
        format: int64
        minimum: 0
      description: >-
        Start time in milliseconds, inclusive, e.g. `1790784000000` (2026-10-01
        00:00 CST).
      example: 1790784000000
    End:
      name: end
      in: query
      required: true
      schema:
        type: integer
        format: int64
        minimum: 0
      description: >-
        End time in milliseconds, exclusive, e.g. `1792598400000` (2026-10-22
        00:00 CST).
      example: 1792598400000
    ProjectID:
      name: project_id
      in: query
      required: false
      schema:
        type: array
        items:
          type: string
      style: form
      explode: true
      description: >-
        Project IDs. Every project must belong to the authenticated
        organization.
    APIKey:
      name: key
      in: query
      required: false
      schema:
        type: array
        items:
          type: string
      style: form
      explode: true
      description: >-
        Business API key IDs. Every key must belong to the authenticated
        organization.
  schemas:
    RequestDetailsResponseV2:
      type: object
      required:
        - data
        - has_more
        - next
      additionalProperties: false
      properties:
        data:
          type: array
          description: One row per request.
          items:
            $ref: '#/components/schemas/RequestDetailV2'
        has_more:
          type: boolean
          description: Whether there is a next page.
        next:
          type: string
          nullable: true
          description: >-
            Cursor for the next page. Pass it back verbatim as the `cursor`
            query parameter.
    BillingErrorResponse:
      type: object
      required:
        - error
      additionalProperties: false
      properties:
        error:
          $ref: '#/components/schemas/BillingErrorBody'
    RequestDetailV2:
      type: object
      required:
        - id
        - time
        - model_id
        - key
        - key_name
        - project_id
        - project_name
        - prompt_tokens
        - completion_tokens
        - cached_tokens
        - is_off_peak
        - off_peak_ratio
        - client_request_id
      additionalProperties: false
      properties:
        id:
          type: string
          description: Request ID.
        time:
          type: string
          format: date-time
          description: >-
            Request time. Rows are ordered by server write time, not by this
            field.
        model_id:
          type: string
          description: Model ID.
        key:
          type: string
          description: ID of the inference API key that made the request.
        key_name:
          type: string
          nullable: true
          description: Business API key name; null when unnamed.
        project_id:
          type: string
          description: Project ID.
        project_name:
          type: string
          nullable: true
          description: Project name.
        prompt_tokens:
          type: integer
          format: int64
          minimum: 0
          description: Prompt tokens.
        completion_tokens:
          type: integer
          format: int64
          minimum: 0
          description: Completion tokens.
        cached_tokens:
          type: integer
          format: int64
          minimum: 0
          description: Cached prompt tokens.
        is_off_peak:
          type: boolean
          description: >-
            Whether the request fell into an active off-peak discount window for
            the organization and model at its write-time bucket (rounded down to
            10 minutes, Asia/Shanghai). Present on every row; false when no
            off-peak rule hit.
        off_peak_ratio:
          type: string
          nullable: true
          description: >-
            Price multiplier applied when is_off_peak is true, as a decimal
            string (e.g. "0.3" means charged at 30% of the list price, i.e. a
            70% discount). Null when is_off_peak is false.
        cache_write_5m_tokens:
          type: integer
          format: int64
          minimum: 0
          description: >-
            Cache write tokens (5-minute TTL tier). Returned only for requests
            whose cache writes are metered by TTL tier; omitted for requests
            before tiered metering launched or without tier labels (field
            absent, not null).
        cache_write_1h_tokens:
          type: integer
          format: int64
          minimum: 0
          description: >-
            Cache write tokens (1-hour TTL tier). Returned only for requests
            whose cache writes are metered by TTL tier; omitted for requests
            before tiered metering launched or without tier labels (field
            absent, not null).
        client_request_id:
          type: string
          description: >-
            Client request ID propagated from the client's
            X-Msh-Client-Request-Id header, for exact row matching in
            reconciliation. Empty when the client did not pass one.
    BillingErrorBody:
      type: object
      required:
        - type
        - code
        - message
        - request_id
      additionalProperties: false
      properties:
        type:
          type: string
          example: invalid_request_error
        code:
          type: string
          description: Billing-domain error codes (subset).
          enum:
            - invalid_request
            - invalid_date_range
            - date_range_too_large
            - date_range_out_of_retention
            - data_not_ready
            - invalid_cursor
            - invalid_api_key
            - api_key_inactive
            - insufficient_scope
            - resource_not_found
            - rate_limit_exceeded
            - concurrency_limit_exceeded
            - internal_server_error
        message:
          type: string
        param:
          type: string
          nullable: true
        request_id:
          type: string
  securitySchemes:
    AdminBearerAuth:
      type: http
      scheme: bearer
      bearerFormat: Admin API Secret (sk-admin-*)
      description: >-
        Send the Admin API secret as a Bearer token in the Authorization header
        (`Bearer sk-admin-***`). Create Admin Keys under Console → Organization
        → Admin API Keys.

````
