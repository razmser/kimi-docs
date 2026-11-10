> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Recharge and Rate Limiting

> Review Kimi Open Platform recharge requirements, account tiers, RPM, TPM, and TPD limits, and options for requesting higher capacity.

Rate limits are applied based on each account's cumulative recharge amount to ensure fair resource distribution and prevent abuse. The limits are shown below.

* To prevent abuse, a minimum recharge of \$1 is required to start using the API; when your cumulative recharge reaches \$5, you receive a \$5 voucher.

<table>
  <thead>
    <tr />
  </thead>

  <tbody />
</table>

## Rate Limit Concepts

* Concurrency: The maximum number of requests that can be processed at the same time.
* RPM: Requests per minute — the maximum number of requests allowed per minute.
* TPM: Tokens per minute — the maximum number of tokens processed per minute.
* TPD: Tokens per day — the maximum number of tokens processed per day.
* Web Search QPS: The maximum number of requests per second you can send to the web search endpoints (`/v1/tools/search`, `/v1/tools/search_pro`). This quota is independent of the other columns: it neither consumes nor is consumed by your RPM, TPM, TPD, or concurrency quotas.

For details, see [Rate Limits](../introduction.md#rate-limits).

## Web Search Rate Limits

The web search endpoints (`/v1/tools/search` and `/v1/tools/search_pro`) are rate limited by the "Web Search QPS" column in the table above, with the following rules:

* Web Search QPS is independent of the other rate limits in the table: calling the search endpoints only consumes search QPS quota, not RPM, TPM, TPD, or concurrency quota; conversely, usage of chat and other endpoints never consumes search QPS.
* The two search endpoints are counted independently and do not consume each other's quota. For example, a Tier1 account can call `/v1/tools/search` 3 times per second and, at the same time, `/v1/tools/search_pro` 3 times per second.
* When a limit is exceeded, the endpoint returns HTTP 429 with `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` response headers, which you can use to back off and retry.

For endpoint pricing, see [WebSearch Pricing](websearch.md).

## Why We Apply Rate Limits

Rate limits are standard practice for API services, for several reasons:

* **Preventing abuse**: They stop malicious actors from overwhelming the API with requests and disrupting the service.
* **Ensuring fair access**: Limiting how many requests a single user can send keeps the API responsive for everyone.
* **Managing cluster load**: They absorb sudden traffic surges, keeping the service stable and consistent.

## Special Notes

* When cluster load reaches capacity, rate limits may be temporarily adjusted.
* Vouchers do not count towards the cumulative recharge total.
* When the system detects abnormal activity on an account, a risk-control rate-limiting policy is triggered. Once triggered, the restriction cannot be lifted.
