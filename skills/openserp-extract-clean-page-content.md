---
name: Extract clean page content from URLs
description: Turn a list of web URLs into clean markdown or text suitable for a RAG index or an
  LLM prompt, choosing the cheapest extraction strategy that still returns usable content.
api: openapi/openserp-oss-openapi.yml
operations: [extractURL, extractURLPost, extractBatch]
---

# Extract clean page content from URLs

OpenSERP's extractor fetches a page and returns it as structured, LLM-ready content — markdown,
plain text, headings, links, canonical URL, language, schema.org blocks and Open Graph tags. Use it
instead of writing your own fetch-and-strip step.

## Before you start

- Self-hosted: `http://localhost:7000`, no credentials. Cloud: `https://api.openserp.org/v1` with
  `Authorization: Bearer $OPENSERP_API_KEY`.
- URLs must be absolute and `http`/`https`.

## Choosing the operation

- **One URL, simple** — `extractURL` (`GET /extract?url=...`).
- **One URL, long or awkward to encode** — `extractURLPost` (`POST /extract`), same capability with
  the URL in a body.
- **Two or more URLs** — always `extractBatch` (`POST /extract/batch`). One round-trip, up to 20
  URLs, duplicates dropped, and a failing URL returns a `BatchExtractItem` carrying an error rather
  than failing the batch. Never loop `extractURL`.

## Steps

1. **Start with `mode=auto`.** Auto does a cheap raw HTTP fetch and only escalates to a rendered
   browser fetch when the raw pass yields fewer than `min_runes` characters of content. That is the
   right default and the right price.
   - `mode=fast` forces the raw fetch — use it for sites you know are server-rendered.
   - `mode=rendered` forces the browser — use it only for known JavaScript-only pages. On Cloud this
     is 3 credits per URL instead of 1.

2. **Tune `min_runes` rather than forcing `rendered`.** Raise it when auto is handing back thin
   nav-only content; lower it when auto is escalating on pages that were fine.

3. **Set `use_llms_txt=true` when a URL is a site root.** OpenSERP will probe that site's
   `/llms-full.txt` then `/llms.txt` and prefer them — cleaner, cheaper, and already written for
   machine consumption.

4. **Use `clean` deliberately.** The default is article-only extraction. Set `clean=false` when you
   need the whole readable body, for example on documentation or reference pages where the
   article-detection heuristic will discard the part you actually want.

5. **Set `region` only when a page is geo-fenced or localized.** It takes a two-letter country code.
   On Cloud it adds 1 credit per extracted URL, so do not set it by habit.

6. **Read the result.** An `ExtractResult` gives you `url`, `title`, `description`, `markdown`,
   `text`, `headings`, `links`, `canonical`, `lang`, `schema_org`, `og_tags` and `meta`.
   - Index `markdown` for RAG; use `text` when markdown syntax would confuse a downstream parser.
   - Prefer `canonical` over the requested `url` when de-duplicating an index — the same page is
     often reachable at several URLs.
   - `lang` lets you route or filter before embedding.

7. **Handle per-item failures in a batch.** Walk every `BatchExtractItem` and check for an error
   before using it. Report which URLs failed; do not quietly return a shorter list than you were
   asked for.

## Rules

- `format` accepts `json`, `markdown`, `text` and `ndjson`. Keep `json` when a program consumes the
  result — you lose `headings`, `links` and `meta` in the flattened formats.
- Retry only 408, 429, 500, 502 and 503, with exponential backoff and jitter; honour `Retry-After`
  on 429. A 502 on extraction usually means the target site refused the fetch, not that OpenSERP is
  down — retrying the same URL repeatedly will not fix it.
- Extraction is a read. There is no idempotency key and none is needed; a repeat call is safe apart
  from the credit cost on Cloud.
- If you are extracting the results of a search, do not run this skill standalone — use
  `openserp-ground-an-answer-in-live-search.md`, which combines the search and the batch extraction
  into one flow.
