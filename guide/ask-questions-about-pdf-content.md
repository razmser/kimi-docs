> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Ask questions about PDF content

> Ask questions about PDF content with the Kimi API, with real run outputs.

With the Kimi API, file-based Q\&A takes only two steps: upload a file through the Files API to extract its text, then put the extracted text into messages. When you query the same document repeatedly, keeping the document prefix stable lets the built-in Context Caching feature cut your costs.

This cookbook walks through:

* **Upload and read**: get model-readable text with the file-extract flow;
* **Ask**: put the file content into messages as a system message;
* **Multi-turn chat**: keep the fixed prefix in place and append turns to the end of messages;
* **Multi-file Q\&A**: put each file in its own system message at the head of messages;
* **Constrain the output**: add a fixed response rule to control format and length;
* **Cut costs**: reuse a stable document prefix so Context Caching kicks in automatically;
* **Clean up uploaded files**: keep the extracted text locally and delete uploaded files on a schedule.

## 1. Setup

The Kimi API is compatible with the OpenAI SDK, so you only need the openai package:

```bash theme={null}
pip install -U openai
```

When creating the client, point base\_url at the Kimi API endpoint and read the API key from an environment variable:

```python theme={null}
import os
from pathlib import Path
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["MOONSHOT_API_KEY"],  # set the MOONSHOT_API_KEY environment variable before running
    base_url="https://api.moonshot.ai/v1",
)
```

The examples use the **kimi-k3.1-pro** model. To use another model, replace the `model` field, but note that parameter support differs between models. See the [model parameter reference](../api/models-overview.md).

The sample file is the PDF of the [Kimi K3 technical report](https://github.com/MoonshotAI/Kimi-K3/blob/main/k3_tech_report.pdf) from the official MoonshotAI GitHub repository. Download it with the code below (or use any PDF of your own; the Files API supports .pdf, .txt, .csv, .doc, and .docx, with a 100MB limit per file):

```bash theme={null}
curl -sSL -O https://raw.githubusercontent.com/MoonshotAI/Kimi-K3/main/k3_tech_report.pdf
ls -lh k3_tech_report.pdf
```

> [accordion] Example output
>
> ```text theme={null}
> -rw-r--r--  1 user  staff   1.7M  8月 16 22:23 k3_tech_report.pdf
> ```

## 2. Upload the file

To get started, we'll upload the PDF through the Files API with purpose set to file-extract, which marks the file for text extraction. The Files API also accepts purpose values such as image and video, used for the model's native understanding of those files:

```python theme={null}
file_object = client.files.create(file=Path("k3_tech_report.pdf"), purpose="file-extract")
print(file_object.id)
```

> [accordion] Example output
>
> ```text theme={null}
> fc1wi3cne4z111bgipq1
> ```

## 3. Read the extracted content

After uploading, fetch the extracted content through the file content endpoint. The result is Markdown-formatted text (bold, headings, lists, etc. are represented with Markdown syntax):

```python theme={null}
# retrieve_content in older examples is deprecated; files.content is the replacement
file_content = client.files.content(file_id=file_object.id).text
print(file_content[:200])  # take a look at the first 200 characters
```

> [accordion] Example output
>
> ```text theme={null}
> {"content":"# KIMI K3:OPEN FRONTIER INTELLIGENCE\n\n\nTECHNICAL REPORT OF KIMI K3\n\n\nKimi Team\n\n\n# ABSTRACT\n\n\nWe introduce Kimi K3,a 2.8T parameter Mixture-of-Experts model with 104billion act
> ```

If you will query the same document repeatedly, save the extracted text locally and read it back next time, instead of uploading and extracting again:

```python theme={null}
# save the extracted text locally
Path("k3_tech_report_extract.txt").write_text(file_content)

# next time: read the local file instead of uploading and extracting again
file_content = Path("k3_tech_report_extract.txt").read_text()
```

> **Two common mistakes:**
>
> 1. **Do not put the file\_id into messages.** The file\_id is only a handle; the model cannot see any content through it. You must read the extracted content first and put the text into messages.
> 2. **Do not inline a base64-encoded PDF into messages.** Base64-encoded files lead to very high token consumption. If the file type is supported by the /v1/files API, upload the file and extract its content instead.

## 4. Ask a question

Put the extracted file content into messages as a system message, then ask your question in a user message:

```python theme={null}
persona = "You are Kimi, an AI assistant provided by Moonshot AI. You are especially good at conversations in Chinese and English. You give safe, helpful, and accurate answers, and you refuse to answer questions involving terrorism, racial discrimination, or pornographic violence. Moonshot AI is a proper noun and must not be translated into other languages."

messages = [
    {"role": "system", "content": persona},
    {"role": "system", "content": file_content},  # extracted file content (the content itself, not the file ID)
    {"role": "user", "content": "What capabilities of K3 does this technical report introduce? Please summarize in bullet points."},
]

completion = client.chat.completions.create(
    model="kimi-k3.1-pro",
    messages=messages,
)
print(completion.choices[0].message.content)
```

> [accordion] Example output
>
> ```text theme={null}
> # Kimi K3 Capabilities Summary
>
> ## Core Model Specifications
> - **2.8T-parameter MoE model** with 104B activated parameters — the first open 3T-class model
> - **Native multimodality**: text, images, and videos processed jointly by a single shared backbone from the start of training, with no post-hoc alignment stage
> - **1-million-token context window**, achieved via hybrid attention (KDA + Gated MLA) with NoPE, enabling direct extrapolation without positional-encoding modification
> - **~2.5× scaling efficiency improvement** over Kimi K2 through architectural and training innovations
>
> ## Long-Horizon Agentic Capabilities
> - **Sustained multi-step execution**: hundreds to thousands of tool calls and millions of accumulated context tokens per trajectory
> - **Multi-day persistent assistant workflows** across evolving environments (e.g., mock Gmail, Notion, Slack) with interdependent events
> - **Autonomous execution**: task decomposition, planning, error recovery, and termination without reference trajectories, trained via verify-in-the-loop optimization
> - **Agent swarm orchestration**: coordinating parallel subagents on complex tasks
> - **Diverse harness generalization**: trained across configurable agent harnesses rather than overfitting to one tool schema
>
> ## Coding & Engineering Capabilities
> - **Long-horizon software engineering** (SWE tasks, terminal work, web development)
> - **GPU kernel optimization**: CUDA, Triton, CuTe DSL, etc., with correctness-and-performance rewards
> - **Full compiler development**: built MiniTriton, a working Triton-like compiler with DSL frontend, MLIR passes, PTX codegen, and autograd
> - **Chip design**: designed and verified an inference-chip prototype (RTL, timing closure) autonomously in 48 hours
>
> ## Reasoning, Knowledge & Vision
> - **Multi-effort reasoning**: low/high/max effort levels trained via per-problem budget control RL
> - **Vision-in-the-loop reasoning**: writes and executes code to crop, zoom, and transform images iteratively to solve visual problems
> - **Research-level tasks**: literature review, numerical pipeline implementation, and scientific analysis (e.g., reproducing astrophysics relations in ~2 hours vs. weeks)
> - **Strong benchmark results**: frontier-competitive on GPQA, BrowseComp (91.2%), Math-Vision (97.8% w/ tools), and top open-model scores across most agentic and coding suites
>
> ## Deployment & Cost Efficiency
> - **MXFP4 quantization-aware post-training** for reduced serving cost with no train–inference mismatch
> - **EAGLE-3-style speculative decoding** via a fine-tuned MTP draft model
> - **On or near the cost-efficiency frontier**: near-top scores at a fraction of proprietary model costs
> - **Full open-weight release** for research and deployment
>
> ## Security-Related Capability (Evaluated)
> - **Vulnerability discovery**: found genuine bugs including 16 previously unknown vulnerabilities
> - **End-to-end exploit development**: solved 14/36 expert-verified exploitation tasks (though still trailing human experts on hardened targets)
> ```

> Note: all outputs in this cookbook come from real runs. Model output is not deterministic, so your results may differ slightly.

## 5. Multi-turn chat

To follow up, append the question and answer to the end of messages. The file content and instructions stay at the front, unchanged. This "fixed content first, conversation appended at the end" structure is not just a convention; it determines whether the cache described later can be hit:

```python theme={null}
# append to the messages from section 4: prefix untouched, turns go at the end
messages.append({"role": "assistant", "content": completion.choices[0].message.content})
messages.append({"role": "user", "content": "Which of these is most useful for developers?"})

completion = client.chat.completions.create(
    model="kimi-k3.1-pro",
    messages=messages,
)
print(completion.choices[0].message.content)
```

> [accordion] Example output
>
> ```text theme={null}
> It depends on what kind of developer you are, but here's how I'd rank them by practical impact:
>
> ## Most broadly useful
>
> **1. Long-horizon agentic coding** — This is the headline capability for most developers. Sustained execution over hundreds/thousands of tool calls means K3 can handle realistic end-to-end work (multi-file refactors, debugging sessions, feature implementation) rather than just single-shot code completion. Its strong Terminal-Bench, SWE, and webdev scores suggest it performs well under real harnesses like Claude Code and Codex, not just in controlled evals.
>
> **2. Open weights + deployment efficiency** — For developers building products, this may matter even more than raw capability:
> - Full weight release means you can self-host, fine-tune, and avoid API lock-in
> - MXFP4 quantization-aware training means quantized deployment without the usual quality cliff
> - Speculative decoding support for lower latency
> - Cost efficiency near proprietary frontier models at a fraction of the price
>
> **3. 1M-token context** — Whole-codebase reasoning without RAG plumbing: you can feed entire repos, long logs, or extensive documentation directly. This simplifies a lot of tooling developers currently build around context limits.
>
> ## Most useful for specific niches
>
> - **ML/infra engineers**: GPU kernel optimization (CUDA/Triton) — the report shows it doing real profiling-and-rewriting work, and Moonshot apparently used it for their own kernel work internally
> - **Frontend/creative developers**: Native vision + webdev strength (topped WebDev Arena, +31% win margin over Opus 4.8 on webdev tasks) — it can iterate on UI by *looking* at rendered output
> - **Security engineers**: Vulnerability discovery for defensive auditing of codebases
>
> ## Bottom line
>
> For the **average application developer**: agentic coding + cost efficiency + open weights is the winning combination — near-frontier coding ability you can actually afford to run and control.
>
> For **infrastructure/ML developers**: the kernel optimization and compiler-building capabilities are the differentiators, since few models operate at that level of systems depth.
> ```

## 6. Multi-file Q\&A

Multi-file Q\&A is the same pattern scaled out: **put each file in its own system message** and place these messages at the head of the messages list:

```bash theme={null}
# the multi-file example also needs the K2 technical report (also from the official repository, renamed for consistency)
curl -sSL -o k2_tech_report.pdf https://raw.githubusercontent.com/MoonshotAI/Kimi-K2/main/tech_report.pdf
ls -lh k2_tech_report.pdf
```

> [accordion] Example output
>
> ```text theme={null}
> -rw-r--r--  1 user  staff   4.9M  8月 16 22:26 k2_tech_report.pdf
> ```

```python theme={null}
def upload_files(files):
    """Upload all files and extract their content; each file becomes one system message."""
    messages = []
    for file in files:
        file_object = client.files.create(file=Path(file), purpose="file-extract")
        file_content = client.files.content(file_id=file_object.id).text
        messages.append({"role": "system", "content": file_content})
    return messages

file_messages = upload_files(["k3_tech_report.pdf", "k2_tech_report.pdf"])  # both reports are on the MoonshotAI GitHub repository

messages = [
    *file_messages,  # file messages first
    {"role": "system", "content": persona},
    {"role": "user", "content": "Compare these two technical reports. What improvements does K3 have over K2?"},
]

completion = client.chat.completions.create(
    model="kimi-k3.1-pro",
    messages=messages,
)
print(completion.choices[0].message.content)
```

> [accordion] Example output
>
> ```text theme={null}
> # Kimi K3 vs. Kimi K2: Key Improvements
>
> ## Scale and Architecture
>
> | Aspect | K2 | K3 | Change |
> |---|---|---|---|
> | Total params | 1.04T | 2.78T | +167% |
> | Activated params | 32.6B | 104.2B | +220% |
> | Layers | 61 | 93 | +52% |
> | Routed experts | 384 (8 active) | 896 (16 active) | sparsity 48→56 |
> | Shared experts | 1 | 2 | — |
> | Attention heads | 64 | 96 | +50% |
> | Attention | All MLA | Hybrid: 69 KDA + 24 Gated MLA | new |
> | Activation | SwiGLU | SiTU-GLU | new |
> | Context | 128K | 1M | 8× |
> | Vision | None (text-only) | Native (401M MoonViT-V2) | new |
>
> ## Architectural Innovations
>
> 1. **Kimi Delta Attention (KDA)** — A linear attention mechanism with channel-wise forget gates replaces most MLA layers (3:1 KDA:MLA ratio), giving fixed-size recurrent state instead of a growing KV cache. K3 adds a lower-bounded decay (scaled sigmoid instead of negative Softplus) that keeps values in BF16 range and enables full Tensor Core computation, plus a full-rank output gate.
>
> 2. **Gated MLA with NoPE** — The periodic global-attention layers drop positional encoding entirely (no RoPE/YaRN retuning needed for context extension) and add input-dependent output gating.
>
> 3. **Attention Residuals (AttnRes)** — Replaces uniform residual accumulation with learned attention over all preceding block outputs, letting each layer selectively retrieve information across depth (8 blocks of 12 layers).
>
> 4. **Stable LatentMoE** — Routed experts operate in a compressed 3584-dim latent space, making 896 experts affordable. Stabilized by RMSNorm before up-projection, SiTU-GLU (smoothly bounded activation, output ≤ β₁β₂), and **Quantile Balancing** — a closed-form, learning-rate-free load-balancing rule replacing K2-style fixed-step bias updates.
>
> 5. **Native vision** — MoonViT-V2 is trained from scratch with next-token prediction rather than initialized from SigLIP, yielding more stable optimization and no contrastive pre-training dependency.
>
> 6. **Per-Head Muon** — Newton–Schulz orthogonalization applied per attention head rather than per full projection matrix, for more balanced updates.
>
> ## Training Recipe
>
> - **~2.5× better scaling efficiency** over K2 (per fitted scaling-law curves on held-out data).
> - **Cosine LR schedule** replaces WSD, after a fair per-schedule hyperparameter search showed cosine wins.
> - **Native multimodal pre-training** from the start (no post-hoc alignment stage).
> - **Progressive context extension** 8K→64K→256K→1M (vs. K2's 4K→128K via YaRN), with synthesized long-range tasks to prevent attention degenerating into local patterns.
>
> ## Post-Training
>
> K2 was a **non-thinking** model trained with SFT + RLVR + self-critique rubrics. K3 is a full **reasoning model**:
>
> - RL across 3 domains (general, general agents, coding agents) × 3 effort levels (low/high/max) = 9 expert policies, consolidated into one model via **Multi-Teacher On-Policy Distillation**.
> - Per-problem **budget control** for token-efficient reasoning; agentic generative reward model with anti-verbosity control.
> - Rich verifiable environments: kernel optimization (CUDA/Triton/etc.), personal assistant workflows spanning days, autonomous execution tasks with verify-in-the-loop, web development with deterministic + judged rewards.
> - **MXFP4/MXFP8 quantization-aware training** throughout post-training, and EAGLE-3 draft-model fine-tuning with LK loss for speculative decoding.
> - New XTML chat template for extensibility and low alignment tax.
>
> ## Infrastructure
>
> - **KDA systems co-design**: FlashKDA fused kernels, intra-device context-parallel planner, and KDA Context Parallelism (fixed-size all-gather vs. growing KV exchange).
> - **MoonEP**: provably perfectly balanced expert parallelism (≤ E/R redundant experts per rank), zero-copy communication, static shapes eliminating per-layer host sync.
> - Memory-efficient training: unified activation manager, Pipeline ZeRO-2, P2P-based Muon orthogonalization.
> - Million-token agentic RL: external KV cache pool, auto-throttling rollout scheduler, resumable microVM sandboxes (AgentENV, 51M+ sandboxes used).
> - Serving: KDA-aware prefix cache (fine-grained 512-token reuse in coarse blocks), dedicated decode kernels, cache-affinity and budget-aware fleet scheduling.
>
> ## Performance Positioning
>
> - **K2**: SOTA among open-source *non-thinking* models (e.g., 65.8 SWE-bench Verified, 75.1 GPQA-Diamond), competitive with Claude 4/GPT-4.1-era models.
> - **K3**: Operates at the reasoning frontier — trails only Claude Fable 5 and GPT-5.6 Sol while beating all other evaluated models. Headline results: 93.5 GPQA-Diamond, 91.2 BrowseComp, 88.3 Terminal-Bench 2.1, 42.0 SWE-Marathon (7 pts ahead of Fable 5), #1 on WebDev Arena (first open model ever), #4/580 on Artificial Analysis Intelligence Index. It also reaches these scores at a fraction of the cost (e.g., best BrowseComp score at half the cost of GPT-5.6 Sol).
>
> In short: K3 moves from K2's 1T-scale non-thinking agentic model to a 2.8T natively multimodal reasoning model with a redesigned attention/residual/MoE architecture, 1M-token context, and matching systems innovations — while remaining fully open-weight.
> ```

## 7. Constrain the response

The system message in section 4 is an identity setting: it tells the model who it is. In practice you often add a **response rule** that tells the model what the answer should look like. Format, length, and style can all be pinned down with a fixed instruction. The rule is part of the stable prefix, just like the identity setting and the file content, and does not affect cache hits in the next section:

```python theme={null}
answer_rules = """Response rules:
1. Summarize in at most 5 bullet points;
2. One sentence per point, each with the corresponding quote from the document as evidence;
3. No more than 200 words in total."""

question = "What capabilities of K3 does this technical report introduce?"

base_messages = [
    {"role": "system", "content": persona},
    {"role": "system", "content": file_content},
    {"role": "user", "content": question},
]
messages_with_rules = [
    {"role": "system", "content": persona},
    {"role": "system", "content": file_content},
    {"role": "system", "content": answer_rules},  # the added response rule
    {"role": "user", "content": question},
]
```

We run the same question twice, with and without the rule: without it the model answers freely with a long, sectioned piece; with it the output fits the defined shape, with bullet points, a quote per point, and a much tighter length:

```python theme={null}
# without the rule: the model answers freely
completion = client.chat.completions.create(
    model="kimi-k3.1-pro",
    messages=base_messages,
)
print(completion.choices[0].message.content)
```

> [accordion] Example output
>
> ```text theme={null}
> # Kimi K3 Capabilities Overview
>
> The report introduces Kimi K3, an open 2.8T-parameter Mixture-of-Experts model (104B activated parameters), with capabilities spanning five core areas:
>
> ## 1. Long-Context Processing
> - **1-million-token context window**, achieved through progressive context extension (8K → 64K → 256K → 1M)
> - **NoPE design**: No explicit positional encoding—Kimi Delta Attention (KDA) implicitly encodes position, allowing direct extrapolation to 1M tokens without RoPE rescaling or interpolation
> - Strong long-context reasoning (74.7% on AA-LCR, best among evaluated models)
>
> ## 2. Native Multimodal / Vision
> - Processes **text, images, and video in a single shared backbone** with no post-hoc alignment stage
> - MoonViT-V2 vision encoder trained from scratch with next-token prediction (no contrastive pre-training)
> - **Vision-in-the-loop behavior**: can write code, inspect rendered outputs (screenshots/video frames), and iteratively refine visual artifacts
> - Strong results: 91.1% on OmniDocBench (best), 97.8% on Math-Vision with Python tools, ties Claude Fable 5 on ZeroBench-main
>
> ## 3. Agentic & Long-Horizon Execution
> - Executes tasks over **hundreds to thousands of tool calls and millions of accumulated context tokens**
> - State-of-the-art on BrowseComp (91.2%), DeepSearchQA (95.0% F1), MCPMark-Verified (94.5%), Harvey Lab-AA (94.6%)
> - Trained via a unified white-box RL environment spanning multiple agent harnesses, plus environments for professional work (investment banking, legal, data analysis), persistent personal-assistant workflows, and autonomous execution with verify-in-the-loop feedback
>
> ## 4. Coding
> - Best score on ProgramBench (77.8%) and SWE-Marathon (42.0%, GPU-kernel-oriented, 7 points ahead of Claude Fable 5)
> - Near-frontier on Terminal-Bench 2.1 (88.3%) and DeepSWE (67.5%)
> - **First open model to top WebDev Arena** (1,678 Elo)
> - Case studies demonstrate: GPU kernel optimization (e.g., 73.6% runtime reduction on KDA), building a full Triton-like compiler (MiniTriton), and designing a working inference chip (nano-KPU) in a 48-hour autonomous run
>
> ## 5. Reasoning & Knowledge
> - Competitive graduate-level reasoning (93.5% GPQA Diamond)
> - **Multiple reasoning effort levels** (low, high, max) trained via budget-controlled RL, consolidated into one model via multi-teacher on-policy distillation
> - Acknowledged gap to frontier on research-level tasks (HLE-Full, CritPt)
>
> ## Additional Notable Capabilities
> - **Cyber security**: discovered 16 previously unknown vulnerabilities (including Linux kernel bugs); solves 14/36 end-to-end exploit tasks, though trailing human experts on hardened targets
> - **Cost efficiency**: sits on or near the cost-efficiency frontier—e.g., best BrowseComp score at half the cost of GPT-5.6 Sol and ~10× cheaper than Claude models at max effort
> - **Agent orchestration**: leads in-house Swarm Bench and Deep Research Bench, coordinating parallel subagents
>
> Overall, the report positions K3 as the **first open 3T-class model**, trailing only Claude Fable 5 and GPT-5.6 Sol while consistently outperforming all other open and proprietary models evaluated.
> ```

```python theme={null}
# with the rule: the output fits the defined shape
completion = client.chat.completions.create(
    model="kimi-k3.1-pro",
    messages=messages_with_rules,
)
print(completion.choices[0].message.content)
```

> [accordion] Example output
>
> ```text theme={null}
> **Kimi K3's key capabilities per the report:**
>
> - **Massive-scale open model**: It is "a 2.8T parameter Mixture-of-Experts model with 104billion activated parameters, native vision capabilities, and a 1-million-token context window," with full weights released.
>
> - **Frontier-level general performance**: "Kimi K3 achieves frontier-level performance across long-horizon coding, agentic, knowledge, reasoning, and vision tasks," trailing only Claude Fable 5 and GPT-5.6 Sol while "consistently outperform[ing] other open and proprietary models."
>
> - **Multi-effort reasoning via RL**: "Post-training highlights reinforcement learning across general, agentic, and coding domains and multiple reasoning effort levels, enabling compositional generalization and robust long-horizon execution."
>
> - **Long-horizon agentic execution**: Training environments "train a general loop of reasoning, acting, observing, verifying, and adapting, often over hundreds or thousands of tool calls and millions of accumulated context tokens."
>
> - **Advanced technical & cost-efficient capabilities**: Case studies show it optimizing GPU kernels, building a compiler (MiniTriton), and designing a chip, while "delivering near-top scores at a fraction of the cost of Claude Fable 5."
>
> (148 words)
> ```

## 8. Cut costs: Context Caching

Billing for file Q\&A has a structural property: the document content appears in every request as a fixed prefix, so the more questions you ask about the same document, the more times that prefix is billed. Context Caching removes this repeated cost. The Kimi API enables it automatically for all requests: when the system detects a repeated leading context (system prompt, file content, tool definitions, and so on), it reuses the cached prefix and bills it as a cache hit instead of charging the full price again.

**No extra code is needed by default.** You do not create a cache or reference a cache ID. Just call /v1/chat/completions as usual. There is only one thing to do: keep the fixed parts (file content, system prompt, tool definitions) stable and at the front of the messages array.

Two details from the official documentation:

* With no extra parameters, eligible prefixes are written at the default `5m` TTL and incur a one-time cache write charge; if requests may be more than 5 minutes apart, pass `prompt_cache_options.ttl` to select `1h`. Follow-up questions in file Q\&A usually arrive within 5 minutes, so the default `5m` is enough. See [Context Caching](context-caching.md) for help choosing a TTL and improving hit rates.
* The cache is stored in blocks; any partial block that does not fill a whole block cannot be written to the cache and is billed as a cache miss. In file Q\&A the file content forms a long prefix, which is naturally well suited to caching.

We can send the exact same request as in section 7 again and look at the cache hit in usage:

```python theme={null}
completion = client.chat.completions.create(
    model="kimi-k3.1-pro",
    messages=messages_with_rules,  # the exact same request as in the previous section
)

usage = completion.usage
cached = usage.prompt_tokens_details.cached_tokens or 0
print(f"Prompt tokens this request: {usage.prompt_tokens:,}")
print(f"Cached: {cached:,} ({cached / usage.prompt_tokens:.1%})")
```

> [accordion] Example output
>
> ```text theme={null}
> Prompt tokens this request: 60,260
> Cached: 60,160 (99.8%)
> ```

60,160 of the 60,260 prompt tokens in that request came from cache. Cached tokens are billed at a discount, so this is where the cost figures from the official documentation show up on a real bill.

Compared with RAG, the official advice is: for frequent queries over fixed content (such as FAQ or document Q\&A), prefer Context Caching; when the content is extremely long and the query direction is not fixed, consider RAG. The two compare as follows:

| Dimension | Context Caching | RAG |
| - | - | - |
| Cost | Extremely high compression in specific scenarios, up to 90% savings | Works for any business, but recall quality problems can hurt answer accuracy |
| Engineering effort | Relatively low; the system handles caching with no extra integration or tuning | Relatively high; requires combining RAG with Embedding and continuous business-specific tuning |
| Extra benefit | Average time to first token under 5s for long texts | Source text can scale to millions of words in a single pass |

## 9. Clean up uploaded files

Uploaded files count toward the organization's total storage quota, which is 10 GiB by default. After extraction, delete files you no longer need to free up quota. For a periodic full cleanup, list all files with `files.list` and delete them one by one with `files.delete`:

```python theme={null}
# delete the file uploaded in this run to free up quota
client.files.delete(file_id=file_object.id)
```

> [accordion] Example output
>
> ```text theme={null}
> FileDeleted(id='file_fc1wi3cne4z111bgipq1', deleted=True, object='file')
> ```

```python theme={null}
# full cleanup: list all files and delete them one by one
file_list = client.files.list()

for file in file_list.data:
    client.files.delete(file_id=file.id)
```

## 10. FAQ

> [accordion] What if extraction fails
>
> Extraction can fail for several reasons: unsupported format, corrupted file, or exceeding the 100MB limit. Formats the API does not support cannot be parsed by the model, so do not put them into the context. Wrapping upload and extraction in defensive handling is a good idea:
>
> ```python theme={null}
> def extract_file(path):
>     try:
>         file_object = client.files.create(file=Path(path), purpose="file-extract")
>         content = client.files.content(file_id=file_object.id).text
>     except Exception as e:
>         print(f"{path} extraction failed: {e}")
>         return None
>     if not content or not content.strip():
>         print(f"{path} extracted empty content; check whether the file is corrupted or the format is supported")
>         return None
>     return content
>
> extract_file("missing-file.pdf")  # demonstrate the failure branch
> ```
>
> ```text Example output theme={null}
> missing-file.pdf extraction failed: [Errno 2] No such file or directory: 'missing-file.pdf'
> ```

> [accordion] What if the document is too long
>
> With kimi-k3.1-pro's 1M-token context, most single documents fit in one piece. If a document is really too long, split it along its own structure (chapters, headings) and put each part in a separate system message.
>
> If your scenario is "a huge document collection with unpredictable queries", for example asking anything across an entire knowledge base, that is beyond what single-document Q\&A covers. See the Context Caching versus RAG guidance in section 8.

***

Reference docs: [File-based Q\&A with the Kimi API](use-kimi-api-for-file-based-qa.md), [File upload API reference](../api/files-upload.md), [Context Caching with the Kimi API](context-caching.md).
