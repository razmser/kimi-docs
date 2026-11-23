> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Manage Your Organization in the Console

> Understand the hierarchy of organizations, projects, and API keys and the difference between v1 and v2 keys, and use the console to set project budgets and rate limits, manage members, configure IP whitelist access policies, and review organization and project consumption.

On Kimi Open Platform, an organization is the top-level unit for account and resource management. You can create multiple projects in an organization and multiple API keys in each project.

## Basic Concepts: Organizations, Projects, and API Keys

| Level | Description |
| - | - |
| Organization | The top-level management unit, with a unique organization ID |
| Project | A resource and consumption management unit within an organization |
| API Key | An API credential within a project. You can create multiple API keys in each project |

To find your organization ID, open **General** under **Organization** in the console.

All projects in an organization share the organization's account balance and rate limits. API usage is tracked separately for each project, so you can monitor and manage spending by project.

### Project and API Key Limits

The project and API key limits depend on your organization verification type:

| Organization Type | Project Limit | API Key Limit |
| - | - | - |
| Default | 20 | 50 |
| Enterprise Verified | 50 | 100 |

If you need higher limits, fill out the [Contact Sales](https://www.kimi.ai/contact-sales) form.

### Capability differences between API Key v1 and v2

API keys come in two versions. New keys are v2 and use the `sk-ms-` prefix, with support for expiration, disable/enable, IP whitelist access policies, and source-IP lookup. v1 keys remain usable but do not support these capabilities and cannot be upgraded in place, so create a v2 key to replace them when needed. Check a key's version in the **Version** column of the **API Keys** list.

| Version | Lifecycle and management | Security capabilities |
| - | - | - |
| v1 | Can remain in use; supports rename and delete | No expiration, disable/enable, or IP whitelist |
| v2 | New keys use the `sk-ms-` prefix; supports rename, disable, enable, and delete | Expiration configured at creation, IP whitelist, and source-IP lookup |

#### v2 key expiration

* Expiration can be configured only when creating a v2 key. Choose 3 hours, 1 day, 7 days, 30 days, a custom expiration, or never; the creation page previews the expiration time in UTC+8.
* Expiration cannot be changed or cleared after creation.
* An expired key becomes invalid permanently and cannot be recovered; create a new key instead.
* Expiration takes effect at the configured time, with no additional cache window.

#### IP whitelist access policies for v2 keys

By default, a v2 key uses the organization default access policy and can also add one custom policy. The CIDR ranges from both policies take effect as a union. The organization default policy is built in, cannot be deleted, and may be empty. A custom policy can be bound to multiple v2 keys, but a custom policy with associated keys cannot be deleted.

When the effective list is empty, requests are not restricted by source IP, and any IP can call the API. Confirm that this is intended. For configuration permissions, public IPv4 and CIDR requirements, entry limits, and detailed steps, see [Configure IP Whitelist Access Policies](#configure-ip-whitelist-access-policies).

#### Source IP lookup for v2 keys

For v2 keys, open **Source IPs** from the console key list to view source IP addresses and request counts by UTC+8 calendar day. The default view is today; historical dates are available, and future dates cannot be selected.

#### Effective timing and inference call behavior for v2 keys

| Scenario | Inference call behavior or effective timing |
| - | - |
| Key disabled, deleted, or expired | HTTP 401, `incorrect_api_key_error` |
| Source IP not allowed by the whitelist | HTTP 403, `permission_denied_error`, with the message `Your IP is not allowed to access this organization` |
| Disable, enable, delete, policy rebinding, or whitelist modification | The change can be delayed by caches: up to about 30 seconds on the console side and up to about 60 seconds across the API call path |
| Key expiration | Takes effect at the expiration time, with no additional cache window; irreversible |

## Example Use Cases

### Multi-Member Collaboration in a Research Group

An enterprise-verified research group is working on topics A and B. The group administrator creates a project for each topic and invites members to the appropriate projects. Each member creates their own API key in each project they participate in:

```text theme={null}
Organization: Research group
├── Topic A project
│   ├── Member A's API key
│   └── Member B's API key
└── Topic B project
    ├── Member A's API key
    └── Member C's API key
```

Member A works on both topics and therefore creates one API key in each project. Consumption for the two topics is recorded separately in their respective projects, making it easier to track costs by topic. Consumption analysis by API key also helps attribute consumption to individual members. Consumption is tracked separately by project, but all projects still share the organization's account balance.

### Separate Production and Testing Environments

When a product handles both production traffic and automated testing, you can create separate production and testing projects and use each project's API keys for the corresponding calls. Consumption for production and testing is then recorded separately in their respective projects.

Set production budgets and TPM limits based on actual production usage. For the testing project, you can set a lower daily consumption budget to reduce spending from faulty test scripts. Each project's available TPM still cannot exceed the organization's TPM.

## Organization Management

### Manage Organization Members

Only enterprise-verified organizations can invite members. Before inviting members, visit [Set Organization Information](https://platform.kimi.ai/console/general) to keep your organization's information up to date and complete enterprise verification.

To invite a member, open **Members** under **Organization**. The platform generates a dedicated invitation link. The invitee can use the link to register and log in to the Open Platform and join your organization.

| Role | Permissions |
| - | - |
| Organization Administrator | The organization creator is the default administrator. Administrators can create projects, invite and manage members, and issue invoices |
| General Member | Can view projects. General members must be invited to join a project to gain access to its resources |

### Configure Organization Balance Alerts

To prevent service interruptions due to an insufficient account balance, configure balance alerts in [Organization Management Settings](https://platform.kimi.ai/console/account).

The default alert threshold is \$5 and can be customized. When your account balance falls below the threshold, the platform sends an email to the organization's registered email address.

### Configure IP Whitelist Access Policies

IP whitelists are an organization-level security setting on Kimi Open Platform, available to organizations that have completed personal or enterprise verification. Access policies apply only to v2 inference API keys; v1 keys do not support IP whitelists.

The **Access policies** page centrally manages two types of IP whitelist policies:

| Policy | Description |
| :- | :- |
| Organization default policy | Built-in and cannot be deleted. Applies to all v2 keys in the organization, and new keys use it by default. It may be empty; when empty, requests are not restricted by source IP. |
| Custom policy | Created for specific business needs and can be bound to multiple v2 keys. Its CIDR ranges take effect as a union with the organization default policy. A custom policy with associated keys cannot be deleted. |

#### Entry and Permissions

After logging in to [Kimi Open Platform](https://platform.kimi.ai/), open **Access policies** under **Organization** in the left navigation.

The entry and configuration permissions are as follows:

| Organization Status | Entry Displayed | Who Can Configure |
| :- | :- | :- |
| Organization verification not completed | No | None |
| Organization verification completed | Yes | Current account |
| Enterprise Verified | Yes | Organization creator, organization administrator |

If you do not see the **Access policies** entry, check whether organization verification has been completed and whether your account has the required permission. For enterprise organizations, only the organization creator and organization administrators can configure access policies.

#### Create a Custom Policy

Click **Create policy**, enter a policy name, an optional description, and the **IP Whitelist**, then save. Please note:

* Add entries row by row, or switch to batch paste to enter multiple entries at once.
* Only public IPv4 addresses and valid IPv4 CIDR ranges are supported. A single IP address is automatically treated as `/32`. IPv6 and private ranges are not supported.
* Entries are normalized and deduplicated on save, and ranges contained within broader ranges are removed.
* Standard organizations can configure up to 100 entries, and enterprise organizations up to 1,000 entries.

Example:

```text theme={null}
203.0.113.4
198.51.100.0/24
52.94.76.0/22
13.107.42.0/24
```

You can also write them as:

```text theme={null}
203.0.113.4, 198.51.100.0/24 52.94.76.0/22 13.107.42.0/24
```

Before configuring the whitelist, make sure your service uses a fixed public IPv4 egress address. If your service is hosted by a cloud provider or calls the Kimi API through a proxy gateway, NAT gateway, or corporate network, enter the actual public egress IP address used for those calls.

#### Format Validation

The system validates the input before saving. The following items cannot be saved:

* IPv6 addresses;
* private IPv4 addresses or private IPv4 ranges;
* loopback addresses, link-local addresses, multicast addresses, reserved addresses, and other non-public IPv4 addresses;
* invalid CIDR ranges;
* lists that exceed the entry limit.

Common invalid examples:

| Type | Examples |
| :- | :- |
| Private address or range | `192.168.1.1`, `10.0.0.1`, `172.16.0.0/12` |
| Carrier-grade NAT shared address | `100.64.0.0/10` |
| Loopback address | `127.0.0.0/8` |
| Link-local address | `169.254.0.0/16` |
| Multicast or reserved address | `224.0.0.0/4`, `240.0.0.0/4` |

If the input contains invalid items, the page shows an error message and marks the invalid entries. Fix the issues before saving.

#### Edit the Organization Default Policy

The organization default policy applies to all v2 keys in the organization, and new keys use it by default. Click **Edit** on the default policy to modify its **IP Whitelist** list. Saving overwrites the existing configuration, so make sure the list contains every range you want to keep.

After you clear the list and save, the default policy becomes empty, and v2 keys are no longer restricted by source IP through the default policy.

> [!WARNING]
>
> A key's effective list is the union of the organization default policy and any bound custom policy. When the effective list is empty, requests are not restricted by source IP, and any IP can call the API. Confirm that this is intended.

#### Access Policies When Creating a v2 Key

When you create a key under **API Keys**, the new key is v2 and uses the organization default access policy by default. You can also add one custom policy, and the CIDR ranges take effect as a union. If the effective CIDR list is empty, the creation dialog warns that requests are not restricted by source IP. Confirm you understand the security risk before proceeding.

For the effective timing and call errors associated with v2 key status and whitelist changes, see [Effective timing and inference call behavior for v2 keys](#effective-timing-and-inference-call-behavior-for-v2-keys).

### View Organization Consumption

The console provides Account Overview and Project Overview pages for reviewing consumption at each level.

Click the current project name at the top of the page, select **Organization Overview** from the project menu, then open **Overview** under **Billing** in the left navigation to view total organization consumption and consumption analysis by product, model, project, and API key.

![Account Overview in Organization Overview](https://mintcdn.com/moonshotai/U7MjKBKSGsjeb87u/assets/pics/org-best-practice/organization-consumption.png?fit=max\&auto=format\&n=U7MjKBKSGsjeb87u\&q=85\&s=028414dcf46dd4200d7e60abc633b8d2)

## Project Lifecycle Management

### Create a Project

1. Click the current project name at the top of the page to open the project menu, then click **Create Project**.
2. Enter a project name and description, then confirm.

![Create Project entry in the project menu](https://mintcdn.com/moonshotai/U7MjKBKSGsjeb87u/assets/pics/org-best-practice/create-project.png?fit=max\&auto=format\&n=U7MjKBKSGsjeb87u\&q=85\&s=d94e270495414cc8b1a546292da7d9be)

If the console prompts you to add funds to your account, do so before creating a new project.

After creating a project, open **Project Settings** to configure its budgets and rate limits.

### Set Consumption Budgets and the TPM Limit

Switch to the target project from the project menu at the top of the page, then open **Project Settings** in the left navigation and find **Project Budget/Rate Limit Settings**:

| Setting | Behavior |
| - | - |
| Monthly and daily consumption budgets | Daily and monthly budgets accumulate over the calendar day and month. When spending reaches a configured budget greater than 0, subsequent API requests for that project are rejected. Consumption statistics and limit enforcement may be delayed, and charges incurred during the delay must still be paid. |
| Project TPM rate limit | Limits the number of tokens the project can use per minute. Requests that exceed the limit are denied. The project's available TPM cannot exceed the organization's TPM. If the configured value is higher than the organization's TPM, the organization's TPM still applies. |

For more information about organization rate limits, see [Recharge and Rate Limiting](../pricing/limits.md).

![Project budget and rate limit settings](https://mintcdn.com/moonshotai/AP772Is8kVTEjpLL/assets/pics/project_settings_en.png?fit=max\&auto=format\&n=AP772Is8kVTEjpLL\&q=85\&s=2fdf298f09969087e641d28340d82feb)

### Manage Project Members and API Keys

Organization administrators can create projects and invite organization members to join them. Project members can create API keys within a project to use its resources.

| Role | Permissions |
| - | - |
| Project Administrator | Can manage project budgets and rate limits, invite project members, and create API keys |
| General Project Member | Can view the project and create API keys |

Do not share API keys. Each project member should create their own API key within the project so that your organization can manage project resources effectively. When a member is removed from a project, all API keys created by that member are invalidated.

### View Project Consumption

Click the current project name at the top of the page, select the target project from the project menu, then open **Project Overview** under **Project Information** in the left navigation to view monthly consumption, daily consumption, project rate limits, and consumption analysis by product, model, and API key.

![Project Overview with consumption details and rate limits](https://mintcdn.com/moonshotai/U7MjKBKSGsjeb87u/assets/pics/org-best-practice/project-overview.png?fit=max\&auto=format\&n=U7MjKBKSGsjeb87u\&q=85\&s=e2825decfe41d48c997f624d4ae7b484)

### Archive a Project

If you no longer need a project, you can archive it. Archived projects appear on the **Archived** tab of the project list.

In the left navigation, open **Projects** under **Organization**. Find the target project in the project list and click **Settings**.

![Settings entry in the project list](https://mintcdn.com/moonshotai/U7MjKBKSGsjeb87u/assets/pics/org-best-practice/project-list-settings.png?fit=max\&auto=format\&n=U7MjKBKSGsjeb87u\&q=85\&s=30a80bf7f8dc9b5bdb63a5b12c1d3303)

On the **Project Settings** page, click **Archive Project** in the upper-right corner.

## FAQ

> [accordion] Do projects have separate account balances?
>
> No. All projects in the same organization share the organization's account balance.

> [accordion] Do projects have independent rate limits?
>
> Not entirely. All projects in an organization share the organization's rate limits. You can set a TPM limit for each project, but the project's available TPM cannot exceed the organization's TPM.

> [accordion] What happens to a member's API keys when the member is removed from a project?
>
> All API keys created by that member are invalidated.

> [accordion] What happens when a project reaches its consumption budget?
>
> Project budgets are used to track and manage spending. When spending reaches a configured daily or monthly budget greater than 0, subsequent API requests for that project are rejected. Consumption statistics and limit enforcement may be delayed, and charges incurred during the delay must still be paid.

> [accordion] Why might project consumption data be delayed?
>
> Consumption statistics may take about 10 minutes to update, so recent usage may not appear immediately.
