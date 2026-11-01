# Gateway Cache Study

Sizing analysis for the API gateway in-process cache, derived from `access_sample.txt`
(51 requests, `<cache key> <payload size in bytes>` per line).

## Summary Figures

| Metric | Value |
| --- | --- |
| Total requests | 51 |
| Distinct cache keys | 10 |
| Combined payload bytes if each distinct key stored once | 30408704 |
| Keys covering >= 80% of all requests | 5 |
| Combined byte footprint of those 5 keys | 4980736 |
| Coverage fraction | 14/17 (0.8235294117647058) |
| Proposed capacity (footprint rounded up to next power of two) | 8388608 bytes (8.0 MiB) |

## Top Keys Table (smallest set covering >= 80% of requests)

Sorted by request count descending; ties broken alphabetically.

| Rank | Cache key | Request count | Size (bytes) |
| --- | --- | --- | --- |
| 1 | user:profile:1001 | 12 | 1572864 |
| 2 | org:settings:42 | 9 | 1048576 |
| 3 | repo:tree:main | 8 | 999424 |
| 4 | issue:comments:777 | 7 | 786432 |
| 5 | search:index:daily | 6 | 573440 |

Remaining keys (not needed to reach 80%): billing:invoice:2024-09 (3, 6291456), audit:events:hourly (2, 5242880), team:roster:blue (2, 4718592), release:notes:v3 (1, 4194304), webhook:delivery:log (1, 4980736).

## Recommendation

We recommend provisioning the API gateway in-process cache with a capacity of 8388608 bytes (8.0 MiB), which is the smallest power of twothat fits the 4980736-byte combined footprint of the five most-frequently-requested keys (user:profile:1001, org:settings:42, repo:tree:main, issue:comments:777, and search:index:daily), covering 14/17 (approximately 82.35%) of all 51 observed requests. This byte-based capacity is derived from observed payload sizes rather than entry counts; if the cache is implemented with CPython's functools.lru_cache, note that its default maxsize is only 128 entries, so an explicit maxsize must be set large enough to hold these five keys plus headroom, and consideration should be given to sizing by memory (bytes) rather than by count, since the largest single key (billing:invoice:2024-09 at 6291456 bytes) exceeds typical small-cache assumptions.

## CPython lru_cache Comparison

- Source file: `python/cpython` → `Lib/functools.py`
- Default `maxsize` for `lru_cache`: 128
- Occurrences of the string `lru_cache` in `Lib/functools.py`: 10
- GitHub Discussions in `python/cpython` about cache eviction or sizing: none found (0 discussions; DiscussionsQa not enabled / no threads returned).
