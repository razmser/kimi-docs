> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Admin API overview

> Manage organization members, invitations, projects, and inference API keys programmatically, and query billing, usage, and audit logs.

The Kimi Admin API lets organization administrators manage members, invitations, projects, and inference API keys programmatically, and query billing, usage, and audit logs — without signing in to the console. It is a **control-plane** API: Admin API credentials cannot call any inference endpoint, and inference API keys cannot call this API.

> [!NOTE]
>
> The Admin API is in **beta** and is available to organizations that have completed organization verification. After completing organization verification under **Console → Organization → Organization Verification**, organization admins can create Admin API keys under **Organization → Admin API Keys**.

The base URL is `https://api.moonshot.ai`. All endpoints in this section live under `/v1/admin/organizations`.

## Authentication

Send the Admin API secret (`sk-admin-*`) as a Bearer token:

```sh theme={null}
curl https://api.moonshot.ai/v1/admin/organizations \
  --header "Authorization: Bearer $MOONSHOT_ADMIN_KEY"
```

A key is bound to exactly one organization; the organization is always derived from the key: you cannot specify or access another organization in a request. If the key owner leaves the organization or is deactivated, the key stops working (key deletion, rotation, owner and organization state changes all take effect within 30 seconds, matching the platform-wide auth cache TTL). Secrets are shown only once, at creation or rotation; store them safely. Each organization can hold at most 10 active Admin API keys.

## Scopes

Every key carries scopes chosen at creation:

| Scope | Type | Covers |
| - | - | - |
| `billing:read` | read | Daily/product bills, request details, interrupted requests, account balance, usage aggregation |
| `org:read` | read | Members, invitations, projects, project members, inference API key metadata, spend limits, organization rate limits |
| `org:write` | write | Write operations on members, invitations, projects, project members, inference API keys, and project spend limits (implies `org:read`, but not `billing:read` or `audit:read`) |
| `audit:read` | read | Audit log events |
| `all` | read+write | Everything above; new scopes are covered automatically |

## Roles and permissions

| Role | Scope | Permission difference |
| - | - | - |
| `admin` (organization) | Organization | Full organization management: members, invitations, projects, inference API keys, organization settings, spend limits, organization rate limits, Admin Key management |
| `member` (organization) | Organization | Use organization resources; project-level access is determined by the project role (can be added as project admin/member) |
| `admin` (project) | Project | Project management: project members, inference API keys, project spend limits and rate limits, project updates and archiving |
| `member` (project) | Project | Use project resources (inference calls and project viewing) |

Organization roles and project roles share the same names but have different meanings: `role` in organization-member endpoints refers to the **organization role**, while `role` in project-member endpoints refers to the **project role**. Only `admin` and `member` can be granted through the API.

Write endpoints additionally require the key owner to be an organization **admin** at call time and are rate-limited more strictly (see below).

## Rate limits

| Class | Limit |
| - | - |
| Read endpoints | 60 requests/minute per organization × endpoint (v2) |
| Write endpoints | 20 requests/minute per organization × endpoint (v2) |
| Request details, interrupted requests, usage aggregation | 5 concurrent requests per organization × endpoint |

Rate limits are counted per organization and endpoint (all v2 endpoints): all Admin Keys in the same organization share the quota — more keys do not raise it. The four frozen v1 endpoints (`/v1/organizations/billings/*`) keep key × endpoint granularity: 60 requests/minute per key and 5 concurrent requests per key. Exceeding a limit returns `429 rate_limit_exceeded` / `concurrency_limit_exceeded`. **Every endpoint may return 429; individual endpoint pages do not repeat it**. Responses do not currently include `Retry-After`; retry with exponential backoff (e.g. 1s/2s/4s/8s). For troubleshooting, contact support with the `request_id` from the response.

## Response format

Successful single-object endpoints (organization info, single member, account, rate limits, spend limit, a single invitation/key/project, etc.) return the object body directly, without an envelope. Collection endpoints return a `data` array; paginated endpoints additionally return `has_more` and `next`. Write operations return the resulting object; delete operations return `deleted=true` together with the resource ID (uid/key, per the endpoint schema).

## Pagination

Paginated endpoints (daily/product bills, request details, interrupted requests, usage aggregation, audit logs) use cursor pagination: responses carry `has_more` and `next`; pass `next` back as the `cursor` query parameter. Page size by endpoint: daily/product bills and usage default 100, max 1000; request details and interrupted requests default 100, max 10000; audit logs default 20, max 100. The cursor carries only the page position — restate all filters on every page. **If any filter changes, restart from the first page and do not reuse the previous cursor** (the server does not validate filter consistency, so reusing it mixes pages across queries). `limit` may be adjusted freely.

Organization collection endpoints (members, invitations, projects, project members, API keys) return the full list in one response, with no pagination parameters. Query parameters outside the protocol (such as `page_size`) are silently ignored — double-check parameter names.

## Errors

Errors follow a unified envelope. The error codes documented here are those reachable through the public Admin API; errors surfaced on console pages (such as the Admin Key management UI) are shown in the product and are outside this list:

```json theme={null}
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

`data_not_ready` gates the bill endpoints' readiness: data for day T becomes available at 07:00 (UTC+8) on day T+1.

## Data conventions

* Monetary amounts are in **milli-cent** (1 USD = 100,000 milli-cent).
* Time windows are left-closed, right-open, aligned to Asia/Shanghai (UTC+8) day boundaries; timestamps in query parameters must be epoch **milliseconds** (e.g. `1761379200000`), and RFC 3339 in responses unless stated otherwise. **Second-level timestamps are not valid input** (they are interpreted as a window near 1970 and may return empty results rather than an error) — check your units.
* Billing data for day T becomes available at 07:00 (UTC+8) on day T+1 and returns `400 data_not_ready` earlier; interrupted request details are exempt (near real time).
* Retention and windows: daily/product bills are retained for 365 days (window cap = retention); request details and interrupted requests are retained for 90 days with a single-query window of ≤31 days; audit logs have a single-query window of ≤31 days; usage aggregation windows vary by granularity (`1m` ≤24h, `1h` ≤31d, `1d` ≤93d).
