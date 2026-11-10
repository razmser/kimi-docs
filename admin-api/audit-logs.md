> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# List audit log events

> Queries audit events in the organization (console management operations, login events, and Admin API operations), newest first, with cursor pagination.

> [!NOTE]
>
> This endpoint requires an Admin Key with the `audit:read` scope.
>
> Audit logging is enabled per organization: an organization admin turns it on under Console → Organization → Admin API Keys → the audit log card. While disabled, this endpoint returns an empty list — indistinguishable from "no events". Once enabled, events become queryable from that moment on: nothing is backfilled, and gaps while disabled are unrecoverable. Every call to this endpoint itself records an `audit_log.read` audit event.


## OpenAPI

````yaml GET /v1/admin/organizations/audit_logs
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
  /v1/admin/organizations/audit_logs:
    get:
      tags:
        - Audit Logs
      summary: List audit log events
      description: >-
        Queries audit events in the organization (console management operations,
        login events, and Admin API operations), newest first, with cursor
        pagination.
      operationId: listAuditLogsV2
      parameters:
        - $ref: '#/components/parameters/Start'
        - $ref: '#/components/parameters/End'
        - name: event_types
          in: query
          required: false
          schema:
            type: array
            items:
              $ref: '#/components/schemas/AuditEventType'
          style: form
          explode: true
          description: >-
            Event type filter. Only events whose `event_type` is in this list
            are returned. Accepts repeated parameters or a comma-separated
            value.
        - name: actor_id
          in: query
          required: false
          schema:
            type: string
          description: Actor id filter (uid or admin key id).
        - name: actor_type
          in: query
          required: false
          schema:
            type: string
            enum:
              - user
              - api_key
          description: Actor type filter.
        - name: target_type
          in: query
          required: false
          schema:
            $ref: '#/components/schemas/AuditTargetType'
          description: Target type filter.
        - name: target_id
          in: query
          required: false
          schema:
            type: string
          description: Target id filter.
        - name: result
          in: query
          required: false
          schema:
            type: string
            enum:
              - success
              - failed
          description: Result filter.
        - name: limit
          in: query
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
          description: Page size. Defaults to 20, maximum 100.
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
          description: Audit events, newest first.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuditLogsResponse'
        '400':
          description: >-
            Invalid request parameters (illegal time window, invalid enum value,
            or illegal limit/cursor).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuditErrorResponse'
        '401':
          description: >-
            Authentication failed (the Admin API key is invalid,
            deleted/expired, or its owner is no longer active).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuditErrorResponse'
        '403':
          description: Insufficient scope (the Admin Key lacks the audit:read scope).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuditErrorResponse'
        '404':
          description: >-
            Not found (no audit data for the organization, or the cursor points
            to nothing).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuditErrorResponse'
        '429':
          description: >-
            Rate limit exceeded (requests-per-minute cap or concurrency cap
            hit).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuditErrorResponse'
        '500':
          description: Internal server error (contact support with the request_id).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuditErrorResponse'
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
  schemas:
    AuditEventType:
      type: string
      enum:
        - api_key.created
        - api_key.updated
        - api_key.deleted
        - admin_api_key.created
        - admin_api_key.updated
        - admin_api_key.deleted
        - organization.member_invited
        - organization.member_joined
        - organization.member_role_updated
        - organization.member_removed
        - organization.ip_whitelist_updated
        - organization.audit_log_enabled
        - organization.audit_log_disabled
        - project.created
        - project.updated
        - project.archived
        - project.member_added
        - project.member_role_updated
        - project.member_removed
        - project.spend_limit_updated
        - project.rate_limit_updated
        - audit_log.read
        - auth.login_succeeded
        - auth.login_failed
      description: >-
        Audit event type. New values may be added over time.


        | Value | Meaning |

        | --- | --- |

        | `api_key.created` | Business API key created. |

        | `api_key.updated` | Business API key updated (rename /
        enable-disable). |

        | `api_key.deleted` | Business API key deleted (revoked). |

        | `admin_api_key.created` | Admin key created (console self-serve
        issuance). |

        | `admin_api_key.updated` | Admin key updated (name / scopes / expiry);
        rotation reuses this event with `detail.action=rotate`. |

        | `admin_api_key.deleted` | Admin key deleted (soft-deleted, kept
        grayed-out in the list). |

        | `organization.member_invited` | Organization member invitation
        created. |

        | `organization.member_joined` | Invitee accepted the invitation and
        joined the organization. |

        | `organization.member_role_updated` | Organization member role changed
        (admin ↔ member). |

        | `organization.member_removed` | Organization member removed. |

        | `organization.ip_whitelist_updated` | Organization IP whitelist
        overwritten as a whole. |

        | `organization.audit_log_enabled` | Organization audit log enabled (by
        an org admin). |

        | `organization.audit_log_disabled` | Organization audit log disabled. |

        | `project.created` | Project created. |

        | `project.updated` | Project updated (name / description). |

        | `project.archived` | Project archived. |

        | `project.member_added` | Project member added (upsert semantics). |

        | `project.member_role_updated` | Project member role changed. |

        | `project.member_removed` | Project member removed. |

        | `project.spend_limit_updated` | Project spend limits changed. |

        | `project.rate_limit_updated` | Project rate limits (TPM dimension)
        changed. |

        | `audit_log.read` | Audit logs queried (self-audit; one event per `GET
        audit_logs` call). |

        | `auth.login_succeeded` | Login succeeded. |

        | `auth.login_failed` | Login failed (the only failed-result event
        retained, for security audit). |
    AuditTargetType:
      type: string
      enum:
        - api_key
        - admin_api_key
        - member
        - project
        - invite
        - ip_whitelist
        - user
        - organization
        - audit_log
      description: |-
        Audit event target type.

        | Value | Meaning |
        | --- | --- |
        | `api_key` | Business API key (`ak-` prefixed). |
        | `admin_api_key` | Admin key (`admk_` prefixed). |
        | `member` | Organization / project member (uid). |
        | `project` | Project (`proj-` prefixed). |
        | `invite` | Invitation. |
        | `ip_whitelist` | Organization IP whitelist. |
        | `user` | Platform user (login event subject). |
        | `organization` | Organization. |
        | `audit_log` | Audit log (self-audit event target). |
    AuditLogsResponse:
      type: object
      required:
        - data
        - has_more
        - next
      additionalProperties: false
      properties:
        data:
          type: array
          description: Audit events.
          items:
            $ref: '#/components/schemas/AuditLogEvent'
        has_more:
          type: boolean
          description: Whether there is a next page.
        next:
          type: string
          nullable: true
          description: >-
            Cursor for the next page. Pass it back verbatim as the `cursor`
            query parameter.
    AuditErrorResponse:
      type: object
      required:
        - error
      additionalProperties: false
      properties:
        error:
          $ref: '#/components/schemas/AuditErrorBody'
    AuditLogEvent:
      type: object
      required:
        - id
        - event_type
        - organization_id
        - actor
        - result
        - created_at
      additionalProperties: false
      properties:
        id:
          type: string
          description: Event id (audit_<ms-hex><rand-hex>, time-ordered).
        event_type:
          $ref: '#/components/schemas/AuditEventType'
        organization_id:
          type: string
          description: ID of the organization the event belongs to.
        project_id:
          type: string
          nullable: true
          description: Project the event relates to; null when unrelated to a project.
        actor:
          $ref: '#/components/schemas/AuditLogActor'
        target:
          $ref: '#/components/schemas/AuditLogTarget'
        result:
          type: string
          enum:
            - success
            - failed
          description: >-
            Operation result. Management operations only record successful
            events; failed logins (`auth.login_failed`) are also recorded.
        error_code:
          type: string
          description: Stable error code when result=failed.
        error_message:
          type: string
          description: Truncated error message when result=failed (max 256).
        detail:
          type: object
          additionalProperties: true
          description: >-
            Event payload (before/after values, action parameters; trimmed per
            event type, sensitive data excluded).
        created_at:
          type: string
          format: date-time
          description: When the event occurred.
    AuditErrorBody:
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
          description: Audit-domain error codes (subset).
          enum:
            - invalid_request
            - invalid_date_range
            - date_range_too_large
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
    AuditLogActor:
      type: object
      required:
        - type
        - id
        - name
      additionalProperties: false
      properties:
        type:
          type: string
          enum:
            - user
            - api_key
          description: 'Actor type: `user` (platform user) or `api_key` (Admin Key).'
        id:
          type: string
          description: Actor ID (uid or Admin Key ID); empty for anonymous (failed login).
        name:
          type: string
          description: Snapshot name at event time (masked account for failed login).
        ip:
          type: string
          description: Source IP of the operation.
        user_agent:
          type: string
          description: Client User-Agent.
    AuditLogTarget:
      type: object
      additionalProperties: false
      properties:
        type:
          $ref: '#/components/schemas/AuditTargetType'
        id:
          type: string
          description: Target object ID.
        name:
          type: string
          description: Target object name snapshot at event time.
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
