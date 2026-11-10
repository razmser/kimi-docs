> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Update project

> Updates project name and/or description; at least one field is required.

> [!NOTE]
>
> This endpoint requires an Admin Key with the `org:write` scope, and the key owner must still be an organization admin at call time (otherwise `403 owner_not_admin`).


## OpenAPI

````yaml PATCH /v1/admin/organizations/projects/{pid}
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
  /v1/admin/organizations/projects/{pid}:
    patch:
      tags:
        - Organization
      summary: Update organization project
      description: Updates project name and/or description; at least one field is required.
      operationId: updateOrgProject
      parameters:
        - name: pid
          in: path
          required: true
          schema:
            type: string
          description: Project ID.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrgProjectUpdateRequest'
      responses:
        '200':
          description: Updated project.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrgProject'
        '400':
          description: Invalid request parameters (illegal/missing fields).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrgWriteErrorResponse'
        '401':
          description: >-
            Authentication failed (the Admin API key is invalid,
            deleted/expired, or its owner is no longer active).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrgWriteErrorResponse'
        '403':
          description: >-
            Insufficient permission (missing the org:write scope, or the key
            owner is not an organization admin).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrgWriteErrorResponse'
        '404':
          description: Project not found (invalid pid or not in this organization).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrgWriteErrorResponse'
        '429':
          description: >-
            Rate limit exceeded (requests-per-minute cap or concurrency cap
            hit).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrgWriteErrorResponse'
        '500':
          description: Internal server error (contact support with the request_id).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrgWriteErrorResponse'
      security:
        - AdminBearerAuth: []
components:
  schemas:
    OrgProjectUpdateRequest:
      type: object
      additionalProperties: false
      properties:
        name:
          type: string
          maxLength: 20
          minLength: 1
          example: Data Team (new)
          description: >-
            New project name. 1-20 characters, leading/trailing whitespace is
            not allowed.
        description:
          type: string
          maxLength: 50
          description: New project description. Up to 50 characters.
      description: At least one field is required.
    OrgProject:
      type: object
      required:
        - id
        - name
        - created_at
      additionalProperties: false
      properties:
        id:
          type: string
          description: Project ID.
        name:
          type: string
          description: Project name.
        description:
          type: string
          description: Project description.
        archived_at:
          type: string
          format: date-time
          nullable: true
          description: When the project was archived; null when not archived.
        daily_consume_limit:
          type: integer
          format: uint64
          minimum: 0
          nullable: true
          description: Daily spend limit (milli-cent); null means unlimited.
        monthly_consume_limit:
          type: integer
          format: uint64
          minimum: 0
          nullable: true
          description: Monthly spend limit (milli-cent); null means unlimited.
        max_token_per_minute:
          type: integer
          format: uint64
          minimum: 0
          nullable: true
          description: >-
            Project-level tokens-per-minute cap (TPM); null clears the project
            override and inherits the organization rate-limit configuration (not
            fully unlimited).
        created_at:
          type: string
          format: date-time
          description: When the project was created.
    OrgWriteErrorResponse:
      type: object
      required:
        - error
      additionalProperties: false
      properties:
        error:
          $ref: '#/components/schemas/OrgWriteErrorBody'
    OrgWriteErrorBody:
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
          description: Organization write-domain error codes (subset).
          enum:
            - invalid_request
            - invalid_cursor
            - invalid_api_key
            - api_key_inactive
            - insufficient_scope
            - resource_not_found
            - member_not_found
            - project_not_found
            - key_not_found
            - invitation_not_found
            - last_admin_protected
            - owner_not_admin
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
