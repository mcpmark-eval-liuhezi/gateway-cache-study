# gateway-cache-study

API gateway in-process cache sizing study: an 80%-coverage analysis of a request-log sample, implemented so Thursday's infrastructure review can re-check every figure with its own arithmetic. Prepared for Thursday's infrastructure review; every figure below was computed programmatically from `access_sample.txt`, the sample of last night's request log, so the arithmetic can be independently re-verified.

**Source sample:** `access_sample.txt` — one line per request, `<cache key> <payload size in bytes>`, taken from last night's request log. Parsing verified: 51 non-empty lines, all well-formed; each distinct cache key carries a single, consistent payload size across its lines.

## Summary figures

| Metric | Value |
| --- | --- |
| Total requests in sample | 51 |
| Distinct cache keys | 10 |
| Combined payload bytes if each distinct key stored once | 52953088 bytes (50.5 MiB) |
| Smallest set of most-frequently-requested keys covering >= 80% of requests | 5 keys |
| Combined byte footprint of those 5 keys (each stored once) | 4980736 bytes (4.75 MiB) |
| Requests covered by those 5 keys | 42 of 51 |
| Exact coverage fraction | 42/51 = 14/17 = 82.3529411765% |
| **Proposed cache capacity (footprint rounded up to next power of two)** | **8388608 bytes = 8 MiB** |

The exact threshold is `0.8 x 51 = 40.8` requests; the frequency-ranked 5 keys deliver 42 of 51, and 4 keys deliver only 36 of 51 (70.5882352941%), so 5 keys is provably the minimum set that reaches the 80% bar. The footprint 4980736 bytes lies between 2^22 (4194304) and 2^23 (8388608), so rounding up to the next power of two gives 8388608 bytes = 8 MiB, leaving 3407872 bytes (~41%) of headroom.

## Key table

All 10 distinct keys from the sample, sorted by request count (descending), then key name ascending for ties; the 80%-coverage set is marked:

| Rank | Cache key | Requests | Payload bytes (single payload stored once) | Share of requests | In 80% cover |
| --- | --- | --- | ---: | ---: | :---: |
| 1 | `user:profile:1001` | 12 | 1572864 | 23.5294118% | yes |
| 2 | `org:settings:42` | 9 | 1048576 | 17.6470588% | yes |
| 3 | `repo:tree:main` | 8 | 999424 | 15.6862745% | yes |
| 4 | `issue:comments:777` | 7 | 786432 | 13.7254901% | yes |
| 5 | `search:index:daily` | 6 | 573440 | 11.7647059% | yes |
| 6 | `billing:invoice:2024-09` | 3 | 18874368 | 5.88235294% | no |
| 7 | `audit:events:hourly` | 2 | 10485760 | 3.92156863% | no |
| 8 | `team:roster:blue` | 2 | 9437184 | 3.92156863% | no |
| 9 | `webhook:delivery:log` | 1 | 4980736 | 1.96078432% | no |
| 10 | `release:notes:v3` | 1 | 4194304 | 1.96078432% | no |

The skew that drives the sizing: the five hot keys are small objects (0.5–1.5 MiB each), while the cold tail is large, rarely-fetched payloads — `billing:invoice:2024-09` alone is 18874368 bytes (18 MiB) and is touched twice. That argues for a frequency-weighted (LRU-style) eviction discipline rather than a plain size cap: under plain size caps the cold keys would evict each other and every request miss.

## Recommendation

Attack Thursday's review with **8 MiB (8388608 bytes)** of in-process payload capacity for last night's hot set: the five most-frequently-requested keys (`user:profile:1001`, `org:settings:42`, `repo:tree:main`, `issue:comments:777`, `search:index:daily`) deliver 42 of 51 requests — exactly 82.3529411765% (14/17) — while costing only 4980736 bytes (4.75 MiB) if each is stored once, so rounding that footprint up to the next power of two leaves 8388608 bytes with 3407872 bytes (~41%) of headroom for hot-set growth and per-entry bookkeeping; caching the entire 10-key working set would cost 52953088 bytes (50.5 MiB) for only 17.6470588 percentage points more coverage, so 8 MiB is the defensible cost/benefit point provided eviction is frequency- or recency-weighted. As a benchmark, CPython's own `functools.lru_cache` — defined in `Lib/functools.py` in the python/cpython repository, default `maxsize=128` with the string `lru_cache` appearing 10 times in that file — manages entry *population*, not bytes, and imposes no default memory ceiling; our proposal inverts that axis by pinning a byte budget of 8388608 for an observed working set of just 5–10 entries, so the two mechanisms are complements rather than competitors: 8 MiB with an entry cap comfortably above the 10 distinct keys seen in the sample is, if anything, generous relative to CPython's default philosophy. Note: the massively disproportionate numbers above (`billing:invoice:2024-09` 18874368, `audit:events:hourly` 10485760, `webhook:delivery:log` 4980736, `release:notes:v3` 4194304) are the actual per-request payloads those keys carry in last night's log, not request counts — an earlier draft of this repo's files had mislabeled them, and the table uses the true per-key sizes.

## CPython `lru_cache` benchmark

- The source file that defines `functools.lru_cache` in the python/cpython repository is [`Lib/functools.py`](https://github.com/python/cpython/blob/main/Lib/functools.py) (verified on `main`; the `lru_cache` entry point sits on line 560: `def lru_cache(maxsize=128, typed=False)`).
- Its exact default maxsize is **128** — the proposal's 8 MiB is a byte budget, CPython's 128 is an entry-count budget; the two are orthogonal and complementary.
- The string `lru_cache` appears **10 times** in that file (6 as the direct identifier/string tokens such as in `__all__` and the decorator signature, plus `_lru_cache_wrapper` and `from _functools import _lru_cache_wrapper`).
- **GitHub Discussions on python/cpython: checked — zero threads.** The discussion listing for that repository returns no items (no eviction, cache sizing, or `lru_cache` threads to cite). That absence is the finding: for cache-eviction/sizing topics the community points to CPython issues, the python-dev mailing list and PEPs rather than GitHub Discussions.

## Tracking

- Capacity decision record: issue #1 — ["Cache capacity recommendation"](https://github.com/mcpmark-eval-liuhezi/gateway-cache-study/issues/1) (8388608 bytes = 8 MiB), with the full digest posted as a comment there.
- Linkable summary for the review doc: `gateway-cache-study-summary.md` in this repository (note: the `gateway-cache-study` gist delivery was attempted separately; the token in use cannot create gists, so this summary table lives in-repo).
