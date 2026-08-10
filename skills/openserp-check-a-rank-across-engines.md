---
name: Check where a page ranks across search engines
description: Measure a domain's position for a keyword on several search engines in one pass, and
  read the cross-engine differences without mistaking a partial failure for a lost ranking.
api: openapi/openserp-oss-openapi.yml
operations: [megaSearch, searchWeb, listMegaEngines]
---

# Check where a page ranks across search engines

Rank differs by engine, region and language. A single Google check is not a ranking picture, and —
more importantly — an engine that failed looks exactly like an engine where you do not rank unless
you check for it.

## Before you start

- Self-hosted: `http://localhost:7000`, no credentials. Cloud: `https://api.openserp.org/v1` with
  `Authorization: Bearer $OPENSERP_API_KEY`.
- Call `listMegaEngines` (`GET /mega/engines`) first on a self-hosted deployment to see which
  engines are actually up. On Cloud the equivalent is `GET /v1/engines/capabilities`.

## Steps

1. **Run the keyword across engines.** Call `megaSearch` (`GET /mega/search`).
   - `text` = the keyword exactly as a searcher would type it.
   - `engines` = the comma-separated list you care about, e.g. `google,bing,duckduckgo`. Omit it to
     query every available engine.
   - `mode=balanced` — this is the only mode that queries all selected engines and returns
     `clusters`. `fast` and `any` return the first engine that answers, which is useless for rank
     comparison.
   - `limit` = 100 if you need to find a page that ranks deep. Remember Cloud bills 1 credit per 10
     returned positions, so a 100-result check costs 10 credits per call.
   - Set `region` and `lang` deliberately, and keep them identical across runs. A rank measured at
     `region=US` is not comparable to one measured at `region=DE`.
   - Set `merge=false` if you want to keep each engine's result list separate rather than flattened.

2. **Check `meta.engine_errors` before reading any rank.** This is the step people skip. A mega
   search returns HTTP 200 with partial results, so an engine listed in `meta.engines_failed` or
   `meta.engine_errors` produced *no* data — it did not rank you at zero. Treat those engines as
   "not measured", never as "not ranking". The error class tells you why:
   `captcha_detected`, `blocked`, `search_timeout`, `proxy_*`, `parser_failure`, `engine_internal`,
   `circuit_open` or `all_engines_failed`.

3. **Find your page.** Scan `results` for entries whose `domain` (or `url`) matches your target.
   - `rank` is the position within that engine's result list.
   - `position.absolute` is the absolute placement on the SERP including non-organic elements.
   - `engine` on each result tells you which engine produced it.
   - `domain_info` gives you `tld` and `sld` if you need to match a domain family rather than an
     exact host.

4. **Use `clusters` for the cross-engine view.** Each cluster keys on `canonical_url` and carries
   `engines_count`, `best_rank`, `score`, and an `occurrences` array recording where that URL
   landed on each engine. This is the ranking-diff view: one row per page, one occurrence per
   engine, so you can see a competitor that is strong on Bing and absent on Google without joining
   result sets by hand.

5. **Deepen only when needed.** If the page is not in the first `limit` results, page with `start`
   using `pagination.next_start`, or fall back to a single-engine `searchWeb`
   (`GET /{engine}/search`) against the one engine you care about, which is cheaper than another
   full mega pass.

## Rules

- Hold `region`, `lang`, `limit` and `engines` constant between runs, or your time series measures
  your own parameters rather than the market.
- Set `features=true` only when you need the `serp_features` array (AI summaries, answer boxes). It
  changes what occupies the top of the SERP and is worth capturing when explaining a rank drop that
  is really a layout change.
- Retry only 408, 429, 500, 502, 503, with backoff; honour `Retry-After` on 429. A `circuit_open`
  error on one engine means the server has tripped a breaker for it — back off that engine rather
  than hammering it.
- For scheduled, recurring rank tracking, OpenSERP Cloud has a first-party Search Monitor product
  that fires a `monitor.run.completed` webhook; see `asyncapi/openserp-monitor-webhooks.yml`. Prefer
  it over building your own cron loop against this API.
