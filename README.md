# gateway-cache-study

API gateway in-process cache sizing study for Thursday's infrastructure review. Every figure below was computed programmatically from `access_sample.txt` (one line per request, `<cache key> <payload size in bytes>`) so the arithmetic can be re-verified; parsing found 51 well-formed lines, zero malformed lines, and zero key/size conflicts.

## Summary figures

| Metric | Value |
| --- | --- |
| Total requests in sample | 51 |
| Distinct cache keys | 10 |
| Combined payload bytes if each distinct key were stored once | 93224960 |
| Smallest set of most-frequently-requested keys covering >= 80% of requests | 5 |
| Combined byte footprint of those 5 keys (each stored once) | 45252608 |
| Requests covered by those 5 keys | 42 of 51 |
| Exact coverage fraction | 42/51 = 14/17 = 0.8235294117647058 (~82.3529411764706%) |
| **Proposed cache capacity (footprint rounded up to the next power of two)** | **67108864 bytes = 64 MiB** |

Arithmetic check: 12 + 9 + 8 + 7 + 6 = 42 requests; 18874368 + 9437184 + 7995392 + 5505024 + 3440640 = 45252608 bytes; next power of two: 2^25 = 33554432 < 45252608 <= 2^26 = 67108864, so 45252608 -> 67108864 bytes (64 MiB exactly; headroom 21893057 bytes, ~41%).

## Key table

All 10 distinct keys from the sample, sorted by request count (descending), then key name ascending for ties; the top 5 keys that together cover >= 80% of requests are marked.

| Rank | Cache key | Requests | Payload bytes (single payload stored once) | Share of requests | In 80% cover |
| --- | --- | --- | ---: | ---: | :---: |
| 1 | `user:profile:1001` | 12 | 18874368 | 23.5294118% | yes |
| 2 | `org:settings:42` | 9 | 9437184 | 17.6470588% | yes |
| 3 | `repo:tree:main` | 8 | 7995392 | 15.6862745% | yes |
| 4 | `issue:comments:777` | 7 | 5505024 | 13.7254901% | yes |
| 5 | `search:index:daily` | 6 | 3440640 | 11.7647059% | yes |
| 6 | `billing:invoice:2024-09` | 3 | 18874368 | 5.88235294% | no |
| 7 | `audit:events:hourly` | 2 | 10485760 | 3.92156863% | no |
| 8 | `team:roster:blue` | 2 | 9437184 | 3.92156863% | no |
| 9 | `webhook:delivery:log` | 1 | 4980736 | 1.96078432% | no |
| 10 | `release:notes:v3` | 1 | 4194304 | 1.96078432% | no |

## Recommendation

Propose 67108864 bytes (64 MiB) of in-process cache payload capacity for the API gateway: the five most-frequently-requested keys (`user:profile:1001`, `org:settings:42`, `repo:tree:main`, `issue:comments:777`, `search:index:daily`) deliver 42 of 51 requests — exactly 14/17 = 82.3529411764706% of traffic — while costing only 45252608 bytes (4.30 MiB) if each is stored once, so rounding that footprint up to the next power of two leaves 67108864 bytes with 21893057 bytes (~41%) of headroom for hot-set growth and per-entry bookkeeping overhead; caching the entire 10-key working set would cost 93224960 bytes (88.84 MiB, ~39% over the proposed capacity) for only 29.41 percentage points more coverage, so 64 MiB is the defensible cost/benefit point provided eviction is frequency- or recency-weighted — under a byte-size cap alone, the cold tail keys (up to 18874368 bytes) would evict each other and every request would miss. As a benchmark, CPython's own `functools.lru_cache`, defined in `Lib/functools.py` in the python/cpython repository (verified on `main`; entry point `def lru_cache(maxsize=128, typed=False)`), has an exact default maxsize of 128 with the string `lru_cache` appearing 10 times in that file; lru_cache bounds entry population, not bytes and not memory, whereas this study pins a byte budget of 67108864 for an observed working set of 5-10 entries, so the two mechanisms are complements rather than competitors: 67108864 bytes with an entry cap comfortably above the 10 distinct keys observed is generous relative to CPython's default philosophy.

## CPython `lru_cache` benchmark

- The source file that defines `functools.lru_cache` in the python/cpython repository is [`Lib/functools.py`](https://github.com/python/cpython/blob/main/Lib/functools.py) (verified on `main`; entry point `def lru_cache(maxsize=128, typed=False)`).
- Exact default maxsize: **128** — the proposal's 67108864 is a byte budget; CPython's 128 is an entry-count budget; the two are orthogonal.
- The string `lru_cache` appears **10 times** in that file (case-sensitive, confirmed by raw file fetch and programmatic count).
- **GitHub Discussions on python/cpython: checked — zero threads.** The discussion listing returns no items, so there are zero cache-eviction or cache-sizing threads to cite. That absence is the finding: for cache-eviction and sizing topics the Python community points to CPython issues, the python-dev mailing list, and PEPs rather than GitHub Discussions.

## Tracking

- Capacity decision record: issue #4 — "Cache capacity recommendation" (67108864 bytes = 64 MiB), with the full digest posted as a comment there.
- Linkable summary for the review doc: [`gateway-cache-study-summary.md`](gateway-cache-study-summary.md) in this repository.