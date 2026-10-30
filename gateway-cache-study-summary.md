# gateway-cache-study — summary

Linkable summary table for Thursday's infrastructure review. Sizing study for the API gateway's in-process cache derived programmatically from `access_sample.txt` (one line per request: `<cache key> <payload size in bytes>`), the sample of last night's request log, on disk. Every figure below was computed programmatically and is independently re-derivable from the sample.

**Proposed capacity: 8388608 bytes = 8 MiB.**

## Summary figures

| Metric | Value |
| --- | --- |
| Total requests in sample | 51 |
| Distinct cache keys | 10 |
| Combined bytes if each distinct key stored once | 52953088 (50.5 MiB) |
| Keys needed for >= 80% request coverage | 5 |
| Combined footprint of those 5 keys (stored once) | 4980736 bytes (4.75 MiB) |
| Requests covered by those 5 keys | 42 of 51 |
| Exact coverage fraction | 42/51 = 14/17 = 82.3529411765% |

## Key table (all 10 distinct keys, sorted by request count)

| Rank | Cache key | Requests | Payload bytes (stored once) | Share of requests | In 80% cover |
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

The top 4 keys cover only 36 of 51 (70.5882352941%), so 5 keys is the minimum reach-80% set; the cold tail carries disproportionately large payloads (up to 18874368 bytes), so a frequency-weighted eviction discipline is essential. CPython's `functools.lru_cache` (lib `Lib/functools.py`, default `maxsize=128`, the string `lru_cache` appearing 10 times) bounds entry count, not bytes; Discussions on python/cpython returned 0 threads. Full decision record in README.md.
