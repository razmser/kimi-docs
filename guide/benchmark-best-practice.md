> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Best Practices for Benchmarking

> Run reproducible Kimi model benchmarks with streaming requests, appropriate sampling, and retry strategies.

Benchmarking is an engineering task that requires stability and reproducibility. You'll be calling the model many times, and even small changes in the system environment or network conditions can affect accuracy. Use the following practices to make evaluations reproducible and trustworthy.

## API Recommendations & Notes

* Use the official API whenever possible; third-party endpoints may introduce accuracy drift.
* Use currently available models and do not reuse parameter tables from retired models.
* Set `stream = true` to reduce connection interruptions during long outputs.
* Configure `temperature`, `top_p`, `reasoning_effort`, and related parameters according to the current [Model Parameter Reference](../api/models-overview.md), rather than inferring them from older models.
* Determine max tokens, sample counts, and evaluation-set sizes for Reasoning, Coding, and Agentic Task benchmarks through dedicated evaluations of the current model.
* Keep concurrency low and retry transient failures caused by network jitter, server overload, or rate limiting.

## FAQ

**Q1. Is the temperature setting consistent across models?**

**A.** Follow the current [Model Parameter Reference](../api/models-overview.md). `kimi-k3.1-pro` and `kimi-k3` currently fix `temperature` at `1.0`; do not infer the setting from older models.

**Q2. Why use stream = true?**

**A.** Long outputs can take minutes. Streaming keeps the connection active and reduces the risk that intermediate network devices terminate the connection early.

**Q3. How much concurrency should I use?**

**A.** Start low and adjust based on your [account rate limits](../pricing/limits.md) and HTTP 429 responses. Evaluation accuracy matters more than speed.

**Q4. Why should I add retries?**

**A.** Requests can fail even with streaming. Retry transient failures caused by network jitter, server overload, or rate limiting so they do not distort the evaluation results.

**Q5. Why should multi-turn or multi-step tasks include full context and reasoning?**

**A.** Full context helps the model stay logically consistent, especially during tool calls. Omitting previous reasoning can lead to inconsistent or lower-quality answers.

## Contact Us

If you encounter any issues, email [api-service@moonshot.ai](mailto:api-service@moonshot.ai) for technical support.
