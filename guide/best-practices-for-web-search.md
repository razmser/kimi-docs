> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# Best Practices for Web Search

> Choose the right search API for the job, control costs, write effective queries, and configure parameters to improve web search result quality.

When you add search to an AI app, a Q\&A agent, or a RAG pipeline, finding relevant pages is only half the job. Full pages are packed with navigation, ads, and other noise, and feeding them straight to a model wastes tokens and dilutes the key information.
Kimi Open Platform now offers three standalone REST APIs — Web Search Basic, Web Search Pro, and URL Fetch — that turn pages into ready-to-use information: structured search results, relevant passages, or the full body of a given page.

Web Search Pro is tuned for in-depth knowledge work and factual queries: it reads each page, extracts the passages most relevant to your query, and returns them ranked — along with an `authority` field (the source's authority level) so you can evaluate and filter the results.

Compared with feeding full pages to a model, the Web Search Pro API has four benefits:

* **More accurate answers**: key information stands out instead of getting buried in navigation and ads.
* **Lower token cost**: only the most relevant passages enter the context, so you spend fewer tokens and get faster responses.
* **Less engineering**: chunking and relevance ranking happen on the server, so results are ready for the model.
* **Traceable sources**: every passage keeps its title, URL, and site, so answers are easy to cite and verify.

This page covers how to pick the right API, control costs, write effective queries, and configure parameters, so you get more accurate results with fewer calls.

## Choose the right API

| API | Endpoint | Returns | Best for |
| - | - | - | - |
| Web Search Basic | `POST /v1/tools/search` | Titles, URLs, and snippets of relevant pages; page content included when `include_content=true` | Showing search results to users, collecting pages, or when you want to process page content yourself |
| Web Search Pro | `POST /v1/tools/search_pro` | The passages of page content most relevant to the query, ranked by relevance | Web-grounded Q\&A, RAG, agents, reports and summaries |
| URL Fetch | `POST /v1/tools/fetch` | The title and Markdown body of a given page | You already know the target URL and need its content |

The three APIs are billed per call; see [Pricing](../pricing/websearch.md).

Common tasks and the recommended API:

| Task | Recommended API | Why |
| - | - | - |
| Showing search results | Web Search Basic | Titles, snippets, URLs, and page content work directly as source cards or a result list |
| Collecting pages for your own processing | Web Search Basic | Take the full page content into your own chunking and filtering pipeline |
| Web-grounded Q\&A | Web Search Pro | Returns passages relevant to the question; the model composes answers from them with source links preserved |
| Real-time Web RAG | Web Search Pro | Search external pages first, then hand the relevant passages to the model as ad hoc retrieval results |
| Reports and research | Web Search Pro | Search multiple sources around a topic, extract the key passages first, then let the model compile them into competitive analyses, policy briefings, or industry reports |
| Searching within specific sites or a time range | Web Search Pro | Use the `sites` and `time_window` parameters to constrain the scope and improve relevance |
| You already have the page URL | URL Fetch | Get the title and Markdown body of the given page directly |

After Web Search Basic finds the pages, Web Search Pro takes one more step: it reads the content, picks the passages most relevant to your question, and ranks them by relevance.

If the results go straight to users instead of a model (search result pages, source cards), or you want to process the content yourself (chunking, filtering), Web Search Pro's extra processing is wasted, so choose Web Search Basic.

## Control costs

| Scenario | What to do |
| - | - |
| You only need snippets to judge relevance | Keep `include_content=false` (the default) on Web Search Basic to get just titles, URLs, and snippets. This is the fastest and cheapest option |
| The model needs to read the content closely | Use Web Search Pro. The returned passages (chunks) are already ranked by relevance. Compared with feeding whole pages to the model, token usage usually drops by about an order of magnitude (actual savings depend on the query and the page content). Each chunk keeps its page's title, URL, and site, so sources stay traceable |
| You are sure you need the full page (for example, full-text ingestion) | Set `include_content=true` on Web Search Basic, or call URL Fetch directly on a known URL |

Call count matters as much as per-call content. If you have follow-up questions on the same topic, merge them into one complete query instead of issuing several calls. Similar queries return heavily overlapping pages, so splitting the work means paying for the same pages multiple times; one merged call covers more distinct sources for the same price.

For example, to research the 2025 Nobel Prize in Physics:

* ✗ Split it into three calls: `2025 Nobel Prize in Physics`, `2025 Nobel Prize in Physics awarded work`, `2025 Nobel Prize in Physics experimental verification`
* ✓ Merge them into one call: `2025 Nobel Prize in Physics awarded work experimental verification`

For the pricing of search and page-fetching calls, see [Pricing](../pricing/websearch.md).

If a single search does not give you enough detail, make the query more specific.

## Write effective queries

The APIs search exactly what you submit in `text_query`, with no automatic rewriting, completion, or expansion. The query decides what gets searched. A vague query wastes calls on irrelevant pages, while a specific one makes every result count.

There are three dimensions to making a query specific:

| Dimension | How to write it | Bad example | Good example |
| - | - | - | - |
| Entity | Give concrete proper nouns (people, companies, products, event names), not category descriptions. Without an entity, the search engine can only guess what you are looking for | `the AI model that is trending lately` | `Kimi K3 new features review` |
| Time | For time-sensitive questions, include the year, quarter, or version number. Time terms affect both the freshness of results and relevance ranking | `latest earnings report` | `2026 Q2 earnings revenue` |
| Qualifier | Constrain the angle or source type, which decides whether you get official sites, news, original announcements, or forum discussions | `Kimi K3 API pricing` | `Kimi K3 API pricing official documentation` |

With all three dimensions in place, one call returns material you can use directly:

* ✗ `Nobel Prize`
* ✓ `2025 Nobel Prize in Physics awarded work experimental verification`

Two more points to note:

* **One information need per call**. Stacking unrelated questions into a single query just produces a long, scattered query, and result quality drops.
* **Don't retry with synonym rewrites**. Two near-identical queries return heavily overlapping results, so you pay twice for the same results. When the first result is poor, add an entity, a time term, or a qualifier, or come at the question from a different angle (a different dimension, or a different language).

## Filter results by site and time

Web Search Pro offers two constraint parameters that directly limit the search scope:

* `sites`: restrict sources to specific sites, up to 5; multiple sites are treated as OR. For example, `["nobelprize.org", "nature.com"]` limits results to those sites.
* `time_window`: restrict the publication date range. Dates accept three formats: `YYYY`, `YYYY-MM`, and `YYYY-MM-DD`. Time terms in the query influence searching and ranking, while `time_window` filters results directly by date; the two can be combined.

## Set result count and timeout

Web Search Basic and Web Search Pro also accept `limit` and `timeout_seconds`; set them explicitly:

* `limit`: the maximum number of results, default 5, range 1 to 20. Start with a few results to gauge direction, then decide whether to go deeper.
* `timeout_seconds`: the timeout, range 1 to 60 seconds. Returning page content (`include_content=true`) or using Web Search Pro takes longer, so allow enough time.

## Handle edge cases

* Whether a page's content can be fetched depends on site accessibility, login requirements, anti-scraping policies, page structure, and network conditions. When the search APIs cannot fetch the content, they still return the page's title, URL, and snippet, so you can judge relevance from the snippet first and then decide whether to switch sources.
* The Markdown returned by URL Fetch is an extraction of the page body and does not include images, videos, or other multimedia resources.

## Full example

The following example uses Web Search Pro to search for the 2025 Nobel Prize in Physics: the query carries a time term and an entity, `sites` restricts sources to the official Nobel Prize site, and `time_window` bounds the publication date range:

```bash theme={null}
curl -X POST "https://api.moonshot.ai/v1/tools/search_pro" \
  --header "Authorization: Bearer $MOONSHOT_API_KEY" \
  --header "Content-Type: application/json" \
  --data '{
    "text_query": "2025 Nobel Prize in Physics awarded work experimental verification",
    "limit": 5,
    "timeout_seconds": 30,
    "sites": ["nobelprize.org"],
    "time_window": {
      "start": "2025-10",
      "end": "2025-12"
    }
  }'
```

Each result in the response contains the page's title, URL, site, date, and several content passages, ranked by relevance and ready to hand to a model:

```json theme={null}
{
  "search_results": [
    {
      "title": "The Nobel Prize in Physics 2025 - Press release",
      "url": "https://www.nobelprize.org/prizes/physics/2025/press-release/",
      "site_name": "NobelPrize.org",
      "date": "2025-10-07",
      "snippet": "...",
      "chunks": [
        { "text": "...the passage most relevant to the query...", "score": 1.2432842 }
      ]
    }
  ]
}
```

`chunks[].score` is the relevance score of a passage to the query. It is only meaningful relative to other chunks from the same query; do not compare absolute scores across queries.
