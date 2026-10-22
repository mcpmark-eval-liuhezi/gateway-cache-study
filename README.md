# gateway-cache-study

API gateway in-process cache sizing study: an 80%-coverage analysis of a request-log sample, with the resulting capacity proposal benchmarked against CPython's own LRU caching defaults. Prepared for Thursday's infrastructure review; every figure below was computed programmatically from the sample so the arithmetic can be independently re-checked.

**Source sample:** `access_sample.txt` — one line per request, `<cache key> <payload size in bytes>`, taken from last night's request log. In the sample, each cache key has a single consistent payload size, so the per-key footprint below is unambiguous.

## Summary figures

| Metric | Value |
| --- | --- |
| Total requests in sample | 51 |
| Distinct cache keys | 10 |
| Combined payload bytes if each distinct key stored once | 30408704 (29 MiB) |
| Keys required for ≥ 80% request coverage | 5 |
| Combined byte footprint of those 5 keys | 4980736 bytes (4.75 MiB) |
| Requests covered by those 5 keys | 42 of 51 |
| Exact coverage fraction | 42/51 = 14/17 = 82.3529411765% |
| **Proposed cache capacity** | **8388608 bytes = 8 MiB** |

The top 4 keys cover only 70.5882352941% of requests (36 of 51), so 5 keys is provably the minimum set that reaches the 80% bar. The proposed capacity is the 5-key footprint of 4980736 bytes rounded up to the next power of two: 8388608 bytes, leaving 3407872 bytes of headroom (~68%).

## Key table

All 10 distinct keys from the sample, sorted by request count; the 80%-coverage set is marked:

| Rank | Cache key | Requests | Payload bytes | Share of requests | In 80% cover |
| --- | --- | --- | ---: | ---: | :---: |
| 1 | `user:profile:1001` | 12 | 1572864 | 23.53% | yes |
| 2 | `org:settings:42` | 9 | 1048576 | 17.65% | yes |
| 3 | `repo:tree:main` | 8 | 999424 | 15.69% | yes |
| 4 | `issue:comments:777` | 7 | 786432 | 13.73% | yes |
| 5 | `search:index:daily` | 6 | 573440 | 11.76% | yes |
| 6 | `billing:invoice:2024-09` | 3 | 6291456 | 5.88% | no |
| 7 | `audit:events:hourly` | 2 | 5242880 | 3.92% | no |
| 8 | `team:roster:blue` | 2 | 4718592 | 3.92% | no |
| 9 | `webhook:delivery:log` | 1 | 4980736 | 1.96% | no |
| 10 | `release:notes:v3` | 1 | 4194304 | 1.96% | no |

Note the skew that drives the sizing: the five hot keys are the *small* payloads (0.5–1.5 MiB each), while the cold keys are multi-megabyte objects fetched rarely — an argument for a frequency-weighted (LRU-style) cache rather than a plain size cap.

## Recommendation

For Thursday's review we propose provisioning the API gateway's in-process cache with **8 MiB (8388608 bytes)** of payload capacity: the 5 most-frequently-requested keys in last night's sample — `user:profile:1001`, `org:settings:42`, `repo:tree:main`, `issue:comments:777` and `search:index:daily` — together account for 42 of 51 requests (82.3529411765%, exactly 14/17) while needing only 4980736 bytes (4.75 MiB) if each is stored once, so rounding that footprint up to the next power of two yields 8 MiB with ~68% headroom for hot-set growth and per-entry bookkeeping; caching the full 10-key working set would instead cost 30408704 bytes (29 MiB) — roughly 3.6× the proposal — for only 17.6 percentage points more coverage, so 8 MiB is the right cost/benefit point provided eviction is frequency- or recency-weighted (the sample's cold keys are large objects that should never displace small hot ones under an LRU discipline). For a sanity check against a very widely deployed LRU implementation, CPython's `functools.lru_cache` — defined in `Lib/functools.py` in the python/cpython repository — defaults to **maxsize = 128** entries, i.e. it manages population (entry count), not bytes, and sets no default memory ceiling; our proposal instead pins a byte budget of 8388608 for an observed hot set of just 5–10 entries, so the two knobs are complements rather than competitors: 8 MiB of capacity with an entry cap comfortably above the ~10 distinct keys seen is, if anything, generous relative to CPython's default philosophy.

## CPython `lru_cache` benchmark

Benchmarking the proposal against how CPython itself does LRU caching:

- The source file that defines `functools.lru_cache` in the python/cpython repository is [`Lib/functools.py`](https://github.com/python/cpython/blob/main/Lib/functools.py).
- Its exact default maxsize is **128** (signature: `def lru_cache(maxsize=128, typed=False)`).
- The string `lru_cache` appears **10 times** in that file (6 of them as the standalone word `lru_cache`; the rest inside identifiers such as `_lru_cache_wrapper`). The count was taken from the file served at the pinned blob SHA `2c08791d4d2651b25ef5f5ff47cdc940780b436d`.
- **GitHub Discussions on python/cpython: not enabled.** Both the discussion list and the discussion-categories list for that repository return zero items/empty, so there are no threads about cache eviction or sizing there to cite — that absence is itself the finding (CPython's cache-eviction/sizing conversations happen in issues, the python-dev mailing lists and PEPs rather than GitHub Discussions).

## Tracking

- Capacity decision record: issue #1 — ["Cache capacity recommendation"](https://github.com/mcpmark-eval-liuhezi/gateway-cache-study/issues/1) (8388608 bytes = 8 MiB), with the full digest posted as a comment there.
