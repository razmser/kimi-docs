> ## Documentation Index
> Fetch the complete documentation index at: https://platform.kimi.ai/docs/llms.txt
> Use this file to discover all available pages before exploring further.

# AI-Readable Docs

> Provide the Kimi API Platform documentation to AI coding assistants, enterprise bots, or RAG systems through llms.txt, llms-full.txt, the OpenAPI schema, and per-page Markdown.

The Kimi API Platform documentation site provides several machine-readable endpoints. You can hand the entire documentation directly to AI coding assistants, enterprise bots, or RAG systems without crawling pages one by one.

## Site-wide endpoints

| Endpoint | URL | Description |
| :- | :- | :- |
| llms.txt | [https://platform.kimi.ai/docs/llms.txt](https://platform.kimi.ai/docs/llms.txt) | An index of all documentation pages in a directory-style layout, about 9 KB. Start here to give a model the site structure, then let it fetch specific pages on demand |
| llms-full.txt | [https://platform.kimi.ai/docs/llms-full.txt](https://platform.kimi.ai/docs/llms-full.txt) | The complete documentation in Markdown, about 700 KB. Suitable for loading as full context or as RAG source material |
| OpenAPI Schema | [https://platform.kimi.ai/docs/openapi.json](https://platform.kimi.ai/docs/openapi.json) | The raw API specification (OpenAPI 3.1). Suitable for API integration and generating client code |

## Per-page Markdown

Append `.md` to any documentation page URL to get that page in Markdown, for example [https://platform.kimi.ai/docs/get-your-api-key.md](../get-your-api-key.md). The menu at the top right of every page also offers Copy Page (copy this page as Markdown) and an option to open the page in an AI assistant, which works well when you only need specific pages.

![Copy page menu: Copy page, View as Markdown, and Open in Kimi](https://mintcdn.com/moonshotai/71x3XMKsoVXdlAh9/assets/pics/ai-readable-docs/copy-page-menu.png?fit=max\&auto=format\&n=71x3XMKsoVXdlAh9\&q=85\&s=a10b63913bca776fd51887692594d5c1)

## Recommended usage

* **AI coding assistants** (Claude Code, Cursor, and similar tools): add the llms.txt URL to your rules or context file. The model learns the site structure first, then fetches specific pages on demand;
* **RAG indexing and enterprise bot knowledge bases**: crawl llms-full.txt to obtain the entire site in one pass, split it by page, and build your index;
* **API integration and client code generation**: use openapi.json directly instead of extracting endpoint details from documentation pages.

The Chinese documentation site offers the same endpoints.
