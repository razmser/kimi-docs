> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# BatchJob Pricing

> Review Kimi BatchJob pricing for input, output, and cache-hit tokens, along with billing notes.

## Product Pricing

**Explanation: Prices exclude applicable taxes. Specific tax obligations are subject to local tax regulations and will be calculated at checkout based on your jurisdiction.**

Batch API inference costs **60%** of the standard model price, making it well suited for large-scale tasks that do not require real-time responses.

See the [Model Parameter Reference](../api/models-overview.md) for each model's context window.

<table>
  <thead>
    <tr />
  </thead>

  <tbody />
</table>

1 MTok = 1,000,000 tokens. All prices above are per MTok consumed.

## Notes

* See the table above for the models supported by Batch API
* See the table above for Cache Write fees and support
* Because batch requests are processed asynchronously and concurrently, cache hits are provided on a best-effort basis and are not guaranteed
* Batch API is not subject to real-time concurrency limits, making it suitable for bulk tasks
* Tasks must complete within the specified `completion_window`, or they expire
* See the [Batch API Guide](../guide/use-batch-api.md) for usage details
