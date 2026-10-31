# Gateway Cache Study

Sizing analysis for the API gateway's in-process cache, derived programmatically from the request log sample `access_sample.txt` (51 requests, one per line as `<cache key> <payload size in bytes>`).

## Summary figures

| Metric | Value |
| --- | --- |
| Total request count | 51 |
| Distinct cache keys | 10 |
| Combined payload bytes if each distinct key were stored once | 30408704 |
| Smallest set of most-frequently-requested keys covering >= 80% of requests | 5 keys |
| Exact coverage fraction | 14/17 = 0.8235294117647058 (82.352941%) |
| Combined byte footprint of the selected keys | 4980736 |
| Proposed capacity (footprint rounded up to next power of two) | 8388608 bytes = 8 MiB |

## Key table

| Cache key | Request count | Payload size (bytes) |
| --- | --- | --- |
| user:profile:1001 | 12 | 1572864 |
| org:settings:42 | 9 | 1048576 |
| repo:tree:main | 8 | 999424 |
| issue:comments:777 | 7 | 786432 |
| search:index:daily | 6 | 573440 |
| billing:invoice:2024-09 | 3 | 6291456 |
| audit:events:hourly | 2 | 5242880 |
| team:roster:blue | 2 | 4718592 |
| release:notes:v3 | 1 | 4194304 |
| webhook:delivery:log | 1 | 4980736 |

## Recommendation

Provision 8 MiB (8388608 bytes) for the in-process cache: the five hottest keys account for 42 of 51 requests (14/17 = 0.8235294117647058, i.e. 82.352941%) at a combined footprint of 4980736 bytes, which rounds up cleanly to the next power of two, 8388608 bytes = 8 MiB, avoiding fragmentation while guaranteeing the 80% coverage target; the remaining five cold keys (~25.9 MiB combined) will inevitably evict under LRU and only contribute 9 of 51 requests, so excluding them from the capacity calculation is sound, and the comparison point is CPython's `functools.lru_cache` default of `maxsize=128` entries, which is an entry count rather than a byte budget and orders of magnitude smaller than this payload-dominated workload, confirming that an explicit byte-budgeted capacity is the right lever here.

## CPython lru_cache benchmark

- Source file: `python/cpython` repository, `Lib/functools.py`
- Definition: `def lru_cache(maxsize=128, typed=False)`
- Default `maxsize`: 128
- Occurrences of `lru_cache` in that file: 10 (6 as a whole word)

## Discussions check

GitHub Discussions for `python/cpython` returned totalCount 0 — no discussions about cache eviction or sizing exist there (Discussions are effectively Not Enabled/empty for that repository).
