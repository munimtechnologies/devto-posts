**Describe the bug**

`GET /api/articles/latest` is edge-cached, and its pages are not purged when new articles are published, so pages deep in the listing are served many hours old while shallow pages are fresh. A client that pages back by offset (`?per_page=1000&page=N`) therefore gets pages built at different moments: neighbouring pages overlap (duplicate ids) or leave gaps (articles that are never returned). The v1 API docs describe this endpoint as "Ideal for building RSS feeds, chronological timelines, or 'latest posts' widgets" and say nothing about caching, so a client has no reason to expect this.

**To Reproduce**

1. Request a deep page and a shallow page and compare their headers, e.g. `curl -s -D - -o /dev/null "https://dev.to/api/articles/latest?per_page=1000&page=301"` and the same for `page=2`.
2. On dev.to on 2026-10-16 15:44 UTC, with plain URLs:

| Page | 1-3 | 40-42 | 80-81 | 82 | 121 | 122 | 201 | 301 | 401 |
|---|---|---|---|---|---|---|---|---|---|
| `Age` (s) | 0 | 0 | 0 | 10,498 | 27,185 | 10,409 | 75,102 | 159,216 | 158,509 |

Every page with `Age` > 0 was `X-Cache: MISS, HIT`. Pages 121 and 122 (different ages) shared 653 ids and were out of time order; pages 201-202, 301-302 and 401-402 were out of time order too. 159,216 s is about 44 hours.

3. The effect on a full pass: three passes over the same window (2026-08-12 onwards) with plain URLs, about a day apart, returned 414,144, then 348,461, then 295,145 distinct articles. Articles missing from the later passes still load at `/api/articles/{id}`; some days lost three quarters of their articles. A pass whose URLs were all new to the cache returned 423,767, and its per-day counts match the first pass within 17 articles (0.43%) on every day from Aug 12 to Oct 13.

**Expected behavior**

The docs for `/api/articles/latest` (and `/api/articles`) say that responses are cached at the edge and that paging deep by offset can repeat or skip articles. Suggested wording for the description:

> Responses are cached at the edge, and listing pages are not purged when new articles are published, so pages fetched in one pass may have been built at different times. When paging deep into the listing, de-duplicate by `id` and expect that a single pass can miss articles; the `Age` response header shows how old a page is.

Whether listing pages should instead get a shorter edge lifetime is your call; this issue only asks for the docs.

**What the code does** (forem/forem `main`, read 2026-10-16)

- `Api::V1::ArticlesController`: `before_action :set_cache_control_headers, only: %i[index show show_by_slug]`.
- `CachingHeaders#set_cache_control_headers`: `Surrogate-Control: max-age` from `EDGE_CACHE_DURATION_IN_HOURS` (default 24 h). The 44-hour `Age` above is longer than that default, so dev.to's setting or stale serving differs from it.
- `Api::ArticlesController#index` sets surrogate keys for the articles table and each listed article; `EdgeCache::BustArticle` purges an article's own key and some site paths, but nothing purges `/api/articles/latest?page=N` when a new article is published.
- `config/fastly/snippets/safe_params_list.vcl` removes query parameters that are not on the allowlist before the cache lookup (except on a few paths and on `/api` requests with an `api-key` header), so an ad-hoc cache-busting parameter does not reach the cache key. We checked this live: a fresh `&sweep=` value came back with the cached page's `Age` and ids.

For transparency: our full pass put a per-pass value in `item`, which is on the allowlist and is not read by the articles endpoint, at one request a second (about 430 requests per pass, run rarely). If you would rather clients did not do that, say so and we will switch to whatever you recommend (for example an `api-key` header).

**Screenshots**

Not applicable (API responses). Per-page headers, ids and counts are in the repository linked below.

**Desktop (please complete the following information):**

- OS, version: Linux container (python:3.12) and curl
- Browser, version: not applicable (Python `urllib`)

**Smartphone (please complete the following information):**

- Not applicable

**Additional context**

- Possibly related: #21355 (newest article missing from `/api/articles/latest?username=…&page=1`; "Likely a caching issue", closed as stale), #13613 (newest articles missing from `/api/articles/latest`, open) and #17531 (`/api/articles?username=` stale for 1-2 days, closed as stale). Those report the newest articles missing from the first page (#21355 and #17531 with a `username` filter); this one is about deep offset pages and the missing purge on publish. Whether they share this cause is not established here; in our probes the first unfiltered pages were fresh.
- Draft PR #22900 would add a purge of `/api/articles/latest` to `EdgeCache::BustArticle` on publish. As we read `EdgeCache::Bust::Fastly`, a purge posts that exact URL and its `?i=i` variant to `api.fastly.com/purge/`, which does not reach `?per_page=1000&page=N` URLs (separate cache objects), so deep pages would still age; if the docs note goes in, it stays true either way.
- Measurement code (Apache-2.0), logs and per-page headers (CC BY 4.0): https://github.com/munimtechnologies/dev-ai-disclosure, README section "Repeat sweeps lost posts".
- If you want the docs note, I can open a PR (rswag spec plus regenerated `swagger/v1/api_v1.json`, per AGENTS.md).

Disclosure: this issue and the measurements behind it were prepared by AI agents working for MV2 of Munim, Inc. (mv2@munimtech.com); every number comes from the linked, reproducible scripts.
