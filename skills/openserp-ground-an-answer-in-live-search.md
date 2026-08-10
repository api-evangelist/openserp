---
name: Ground an answer in live web search
description: Retrieve current web results for a question, then pull the full cleaned text of the
  best sources so an answer can be written from primary material instead of from model memory.
api: openapi/openserp-oss-openapi.yml
operations: [megaSearch, extractBatch]
---

# Ground an answer in live web search

Use this when the question depends on facts that may have changed, or when the answer must cite
sources. Search alone gives you titles and snippets; snippets are not enough to answer from. Always
finish with an extraction pass.

## Before you start

- **Self-hosted OpenSERP** — base URL `http://localhost:7000`, no credentials.
- **OpenSERP Cloud** — base URL `https://api.openserp.org/v1`, send
  `Authorization: Bearer $OPENSERP_API_KEY`. Keys begin `osk_live_`. Never log the full key.
- Confirm the key works first with a zero-cost `GET /v1/me` (Cloud only).

## Steps

1. **Search across engines.** Call `megaSearch` (`GET /mega/search`, Cloud `GET /v1/mega/search`).
   - Set `text` to the query. Set `limit` to 10 unless you genuinely need more — on Cloud you pay
     1 credit per 10 returned web positions.
   - Leave `mode` at its default `balanced` when you want breadth and cross-engine clustering.
     Use `mode=fast` when latency matters more than coverage, and `mode=any` when you just need
     the first engine that answers. `fast` and `any` are billed flat.
   - Set `region` and `lang` when the question is locale-sensitive; a US-default search will
     mislead you on a European or Chinese question.
   - Set `date` as `YYYYMMDD..YYYYMMDD` when recency is the whole point of the question.

2. **Read the envelope, not just the results.** The response carries `query`, `meta`, `results`,
   `pagination` and — on `/mega/search` only — `clusters`.
   - `clusters` is the highest-signal field here: it groups the same `canonical_url` seen across
     engines with an `engines_count` and a `score`. A page several engines rank is a better
     citation candidate than a page one engine ranks first.
   - `meta.engine_errors` may be populated on an HTTP 200. A mega search returns partial results
     when some engines fail. Check it before you claim coverage.
   - Record `meta.request_id` (also returned as the `X-Request-ID` header) so a bad answer can be
     traced back to the exact search.

3. **Extract the sources.** Take the top 3–5 `canonical_url` values from `clusters` (or `url` from
   `results`) and call `extractBatch` (`POST /extract/batch`) **once** with all of them.
   - Do not loop `extractURL` per URL. `extractBatch` is one round-trip and, critically, a URL that
     fails comes back as an item carrying an error instead of failing the whole batch.
   - The batch cap is 20 URLs. Duplicates are dropped.
   - Leave `mode` at `auto`. It does a cheap raw fetch first and only escalates to a rendered
     browser fetch if the result falls under `min_runes`. Forcing `mode=rendered` costs 3 credits
     per URL on Cloud instead of 1.
   - Set `use_llms_txt=true` when a URL is a site root — OpenSERP will prefer that site's
     `/llms-full.txt` or `/llms.txt`, which is cleaner and cheaper than rendering a homepage.

4. **Write the answer from `markdown` or `text`**, and cite the `url` and `title` of each
   `ExtractResult` you used. Skip any batch item that came back with an error rather than silently
   dropping the citation.

## Rules

- **Retry only 408, 429, 500, 502 and 503**, with exponential backoff and jitter. On 429 honour the
  `Retry-After` header. Never retry 400, 401, 402, 404 or 422 — those need a changed request, not
  another attempt.
- **Branch on `error` and `code`**, never on `message`. On a 400 the optional `reason` tag tells you
  exactly what to fix: `EMPTY_QUERY`, `INVALID_LIMIT`, `INVALID_DATE_RANGE`, `UNKNOWN_ENGINE`,
  `UNKNOWN_LANG`, `UNKNOWN_COUNTRY`.
- **A 402 `insufficient_credits` is not retryable.** Stop and report that the account needs a top-up.
- There is **no `Idempotency-Key`** on this API and none is needed — every operation in this skill
  is a read. Retrying a search is safe apart from the credit cost on Cloud.
- Watch `X-Credits-Used` and `X-Credits-Remaining` on Cloud responses if you are running a long
  chain of searches.
