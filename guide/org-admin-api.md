> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Manage Your Organization with the Admin API

> Connect the Admin API to your existing systems to automate member onboarding and offboarding, project and access management, financial reconciliation, and security auditing.

When multiple teams in an enterprise use the Kimi API, the day-to-day work of managing accounts, access, costs, and security increases as well. When employees join or leave, admins need to adjust access and clean up keys. Finance teams need to reconcile bills and usage regularly, while security teams need to review audit logs and investigate unusual activity.

Previously, this work had to be done manually in the console before the results were synced to internal systems. Administrators had to switch between two systems and repeat the same steps, which also made omissions more likely.

The Kimi Open Platform now provides the **Admin API** for organization management. Admins can integrate member, project, and key management, as well as billing and usage queries, into their own systems and automate tasks that previously required manual work in the console. This reduces repetitive work, **simplifies reconciliation, and makes access and keys easier to manage**.

**The Admin API is free to use**, and calls do not incur additional charges. Register on [platform.kimi.ai](https://platform.kimi.ai) and complete organization verification to get started. For detailed endpoint definitions, parameters, and response formats, see the [Admin API reference](../admin-api/overview.md).

> [!WARNING]
>
> Use an Admin API key to call the Admin API. Admin API keys and inference API keys are not interchangeable.

![Overview of organization resources and data available through the Admin API](https://mintcdn.com/moonshotai/oGQupEA7X2J7JWcr/assets/pics/org-admin-api/overview.png?fit=max\&auto=format\&n=oGQupEA7X2J7JWcr\&q=85\&s=f2453a8ef64565d1315e0e82163cd0a9)

## Example use cases

| Scenario | Recommended Admin API approach | Benefit |
| - | - | - |
| New employee onboarding | Create an organization invitation and add the member to the appropriate project after they accept it. | Reduce manual invitations and project setup. |
| Employee offboarding | Before removing a member from the organization, revoke their inference API keys. | Avoid leaving active credentials or access behind. |
| Financial reconciliation | Sync daily bills, product bills, and request details to your finance system on a schedule. | Reduce manual exports and make it easier to reconcile charges and per-request token usage. |
| Cost allocation | Aggregate bills and usage by project and map them to departments or business lines. | Track costs and usage by team. |
| Balance monitoring and top-up reminders | Check the account balance periodically and trigger an alert or initiate a top-up approval workflow when it falls below a configured threshold. | Identify low balances early and arrange top-ups. |
| 429 troubleshooting | Query organization rate limits and cross-check usage data against application request logs to identify the cause. | Determine whether rate limiting caused the error and narrow down the possible causes. |
| Security auditing | Regularly retrieve Admin API key information and audit logs for your internal audit process. | Spot unusual activity and permission changes. |

## Quickstart

### Complete organization verification

In the console, open **Organization → Organization Verification** and complete organization verification. The Admin API is enabled automatically afterward; no separate application is required.

### Create an Admin API key in the console

Admin API keys can only be created in the console.

As an organization admin, open **Organization → Admin API Keys** and create a key. Select scopes based on what you need to do:

| What you need to do | Required scope |
| - | - |
| View members, projects, inference API keys, and other organization information | Organization read (`org:read`) |
| Manage members, invitations, projects, project members, and inference API keys | Organization read and write (`org:write`, which includes organization read) |
| Query bills, account balance, and usage | Billing read (`billing:read`) |
| Query audit logs | Audit read (`audit:read`) |

If a key needs multiple capabilities, select all required scopes.

Copy and store the full key immediately. It is shown only once, at creation or rotation. Do not hard-code it in source code or share it with others.

Set the environment variable used in the example below:

```bash theme={null}
export MOONSHOT_ADMIN_KEY="replace-with-your-admin-api-key"
```

### Verify the connection

After creating an Admin API key with the `org:read` scope, call the organization endpoint to verify the connection.

```bash theme={null}
curl https://api.moonshot.ai/v1/admin/organizations \
  --header "Authorization: Bearer $MOONSHOT_ADMIN_KEY"
```

If the request returns information about the current organization, the key is working. Each Admin API key is bound to exactly one organization, so you do not need to specify an organization ID in the request.

## Manage organization resources

The write operations below require `org:write`, and the key owner must still be an organization admin when the request is made.

### Manage organization members

You can use the API to [create invitations](../admin-api/invite-create.md), [check invitation status](../admin-api/invite.md), [list organization members](../admin-api/members.md), [update member roles](../admin-api/member-update-role.md), and [remove members](../admin-api/member-remove.md). After an invitation is accepted, assign project access to the new member.

Before removing a member, revoke their inference API keys first. After a member is removed from the organization, their Admin API keys stop working, but their project inference API keys are not revoked automatically.

### Manage projects and budgets

You can use the API to [create](../admin-api/project-create.md), [update](../admin-api/project-update.md), and [archive projects](../admin-api/project-archive.md), and to [set daily and monthly project spending budgets](../admin-api/project-spend-limits-update.md). Project spending accumulates over the Asia/Shanghai calendar day and month for the daily and monthly budgets, respectively. When spending reaches a configured budget greater than 0, subsequent API requests for that project are rejected. Consumption statistics and limit enforcement may be delayed, and charges incurred during the delay must still be paid.

Before archiving a project, confirm that no active workloads depend on it.

### Manage project members

You can use the API to [add project members](../admin-api/project-member-add.md), [update their project roles](../admin-api/project-member-update-role.md), and [remove them from projects](../admin-api/project-member-remove.md). Only existing organization members can be added.

### Manage inference API keys

You can use the API to [query](../admin-api/project-api-keys.md), [rename](../admin-api/api-key-rename.md), and [revoke](../admin-api/api-key-revoke.md) existing inference API keys, but not to create them. Before revoking a key, confirm its project and owner to avoid disrupting active workloads.

## Query organization data

### Query bills and request details

Use [daily bills](../admin-api/billing-daily-bills.md) and [product bills](../admin-api/billing-product-bills.md) as the source of truth for billing amounts. They support financial reconciliation and cost allocation by project.

[Request details](../admin-api/billing-request-details.md) provide per-request token usage.

Daily bills, product bills, and request details for a given day become available at 07:00 (UTC+8) the next day. [Interrupted request details](../admin-api/billing-interrupted-requests.md) are not subject to this delay; same-day records are available as they are written.

### Query usage, balance, and organization rate limits

* [Usage aggregation](../admin-api/usage.md): Query token usage and request counts by minute, hour, or day to analyze usage trends and project usage.
* [Account balance](../admin-api/billing-account.md): Check the balance periodically and send an alert or initiate a top-up approval workflow when it falls below your internal threshold.
* [Organization rate limits](../admin-api/rate-limits.md): When inference requests return 429, first check the organization rate limits, then use usage data and request logs to help determine whether the errors were caused by rate limiting.

### Query audit logs

[Audit logs](../admin-api/audit-logs.md) cover console admin actions, sign-in events, and Admin API operations. Use them to investigate suspicious sign-ins, permission changes, and key management operations.

An organization admin must enable audit logging in the console first. Only events generated after it is enabled are available. Earlier events are not backfilled, and no events are recorded while it is disabled.

## Usage notes

* **Review Admin API key ownership**: If a key owner leaves the organization or their account is deactivated, their Admin API keys stop working. Review key owners and scopes regularly, and reassign related automation tasks promptly.
* **Handle Admin API rate limiting**: All Admin API keys in the same organization share the rate limit for each endpoint. When an Admin API request returns 429, increase the wait time between retries. Adding more Admin API keys does not increase the limit.

## Related documentation

* [Admin API reference](../admin-api/overview.md)
* [Manage your organization in the console](org-best-practice.md)

## FAQ

> [accordion] Where are Admin API keys and inference API keys created?
>
> Both are created in the console. Organization admins create Admin API keys under **Organization → Admin API Keys**. Inference API keys are created under **Organization → API Keys**. The Admin API can query, rename, and revoke existing inference API keys, but cannot create them.

> [accordion] What happens when I revoke an inference API key or archive a project?
>
> Revoking an inference API key takes effect immediately, and subsequent model calls with that key are rejected. After a project is archived, model calls made with its inference API keys are also rejected. Before automating either action, confirm which keys, projects, and workloads will be affected.

> [accordion] When are bills and request details available?
>
> Daily bills, product bills, and request details for a given day become available at 07:00 (UTC+8) the next day. Interrupted request details are not subject to this delay; same-day records are available as they are written.

> [accordion] How do I check token usage for a single request using a request ID?
>
> When calling an inference API, pass your own request ID in the `X-Msh-Client-Request-Id` header. Retrieve request details for the relevant time range, then match each `client_request_id` value to the corresponding request ID in your internal logs to find the request's token usage.
>
> Request details provide usage data, such as token counts, but not monetary amounts. For billing amounts, refer to daily bills and product bills. Request details and interrupted request details are retained for 90 days, and each query can cover at most 31 days.

> [accordion] Do I need to enable audit logs in advance?
>
> Yes. Your organization must enable audit logging in the console before events become available. Only events generated after it is enabled are recorded; earlier events are not backfilled, and no events are recorded while it is disabled. If you need audit logs, enable them when you start using the Admin API.
