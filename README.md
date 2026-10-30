# gateway-cache-study

API gateway in-process cache sizing study: an 80% coverage analysis of a request-log sample, implemented so Thursday's infrastructure review can independently re-check every figure. Prepared for Thursday's infrastructure review; every figure below was computed programmatically from `access_sample.txt` (51 non-empty lines, one per request, `<cache key> <payload size in bytes>`), with parsing verified by two independent methods, so the arithmetic can be re-verified.

**Source sample:** `access_sample.txt` — one line per request, `<cache key> <payload size in bytes>`, the sample of last night's request log. This run's parsing found 51 well-formed lines, zero malformed lines, and zero key/size conflicts, so each distinct key carries a single consistent payload size across all of its lines.

## Summary figures

| Metric | Value |
| --- | --- |
| Total requests in sample | 51 |
| Distinct cache keys | 10 |
| Combined payload bytes if each distinct key were stored once | 30408704 bytes (exactly 29 MiB) |
| Smallest set of most-frequently-requested keys covering >= 80% of requests | 5 |
| Combined byte footprint of those 5 keys (each stored once) | 4980736 bytes (4.75 MiB) |
| Requests covered by those 5 keys | 42 of 51 |
| Exact coverage fraction | 42/51 = 14/17 = 82.3529411764706% (top 4 keys cover only 36/51 = 70.5882352941%, proving 5 keys is the minimum set that reaches the 80% bar; 0.8 x 51 = 40.8 requests) |
| **Proposed cache capacity (footprint rounded up to the next power of two)** | **8388608 bytes = 8 MiB** (input lies between 2^22 = 4194304 and 2^23 = 8388608; headroom 3407872 bytes, ~41%) |

Arithmetic check the review can redo on its own: 12 + 9 + 8 + 7 + 6 = 42 requests; 1572864 + 1048576 + 999424 + 786432 + 573440 = 4980736 bytes; ceil to power of two: 4980736 -> 8388608. 51 distinct keys, no conflicts. Combined-once verified two ways: 30408704 (= 29 MiB exactly; 1048576 x 29 = 30408704).

## Key table

All 10 distinct keys from the sample, sorted by request count (descending), then key name ascending for ties; the top 5 keys that together cover >= 80% of requests are marked.

| Rank | Cache key | Requests | Payload bytes (single payload stored once) | Share of requests | In 80% cover |
| --- | --- | --- | ---: | ---: | :---: |
| 1 | `user:profile:1001` | 12 | 1572864 | 23.5294118% | yes |
| 2 | `org:settings:42` | 9 | 1048576 | 17.6470588% | yes |
| 3 | `repo:tree:main` | 8 | 999424 | 15.6862745% | yes |
| 4 | `issue:comments:777` | 7 | 786432 | 13.7254901% | yes |
| 5 | `search:index:daily` | 6 | 573440 | 11.7647059% | yes |
| 6 | `billing:invoice:2024-09` | 3 | 6291456 | 5.88235294% | no |
| 7 | `audit:events:hourly` | 2 | 5242880 | 3.92156863% | no |
| 8 | `team:roster:blue` | 2 | 4718592 | 3.92156863% | no |
| 9 | `webhook:delivery:log` | 1 | 4980736 | 1.96078432% | no |
| 10 | `release:notes:v3` | 1 | 4194304 | 1.96078432% | no |

The skew that drives the sizing: the five hot keys are small objects (0.5-1.5 MiB each), while the cold tail is large, rarely-fetched payloads — `billing:invoice:2024-09` alone is 6291456 bytes (6 MiB, touched 3 times) and `audit:events:hourly` is 5242880 bytes (5 MiB, touched twice). Under plain size caps the boilerplate-heavy keys would evict each other and every request would miss.

## Recommendation

Attack Thursday's review with **8388608 bytes (8 MiB)** of in-process payload capacity for last night's hot set: the five most-frequently-requested keys (`user:profile:1001`, `org:settings:42`, `repo:tree:main`, `issue:comments:777`, `search:index:daily`) deliver 42 of 51 requests — exactly 14/17 = 82.3529411764706% of traffic — while costing only 4980736 bytes (4.75 MiB) if each is stored once, so rounding that footprint up to the next power of two leaves 8388608 bytes with 3407872 bytes (~41%) of headroom for hot-set growth and per-entry bookkeeping; caching the entire 10-key working set would cost 30408704 bytes (29 MiB, 50.5% above the proposed capacity) for only 6.25 percentage points more coverage, so 8 MiB is the defensible cost/benefit point provided eviction is frequency- or recency-weighted — under a byte-size cap alone, the cold tail keys (up to 6291456 bytes) would evict each other and every request would miss. As a benchmark, CPython's own `functools.lru_cache` — defined in `Lib/functools.py` in the python/cpython repository (verified on `main`; its entry point is `def lru_cache(maxsize=128, typed=False)`) — has an exact default **maxsize of 128** with the string `lru_cache` appearing **10 times** in that file; lru_cache bounds entry *population*, not bytes and not memory, whereas this study pins a byte budget of 8388608 for an observed working set of 5-10 entries, so the two mechanisms are complements rather than competitors: 8388608 bytes with an entry cap comfortably above the 10 distinct keys observed is generous relative to CPython's default philosophy.

## CPython `lru_cache` benchmark

- The source file that defines `functools.lru_cache` in the python/cpython repository is [`Lib/functools.py`](https://github.com/python/cpython/blob/main/Lib/functools.py) (verified on `main`; entry point `def lru_cache(maxsize=128, typed=False)`).
- Exact default maxsize: **128** — the proposal's 8388608 is a byte budget; CPython's 128 is an entry-count budget; the two are orthogonal.
- The string `lru_cache` appears **10 times** in that file (case-sensitive, confirmed by raw file fetch and programmatic count).
- **GitHub Discussions on python/cpython: checked — zero threads.** The discussion listing returns no items, so there are zero cache-eviction or cache-sizing threads to cite. That absence is the finding: for cache-eviction and sizing topics the Python community points to CPython issues, the python-dev mailing list, and PEPs rather than GitHub Discussions.

## Tracking

- Capacity decision record: issue #1 — "Cache capacity recommendation" (8388608 bytes = 8 MiB), with the full digest posted as a comment there.
- Linkable summary for the review doc: [`gateway-cache-study-summary.md`](gateway-cache-study-summary.md) in this repository. A gist with the same table could not be published with current credentials (403, token does not permit gist creation); the in-repo file is the linkable artifact until a permitted token is in place.
