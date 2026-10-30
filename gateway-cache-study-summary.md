# gateway-cache-study — summary

Linkable summary table for Thursday's infrastructure review. Sizing study for the API gateway's in-process cache derived programmatically from `access_sample.txt` (one line per request: `<cache key> <payload size in bytes>`), the sample of last night's request log, on disk. Parsing verified: 51 non-empty lines, zero bad lines, zero key/size conflicts; each distinct key carries one consistent payload size. Every figure below was computed programmatically and is independently re-derivable from the sample.

**Proposed capacity: 8388608 bytes = 8 MiB.**

## Summary figures

| Metric | Value |
| --- | --- |
| Total requests in sample | 51 |
| Distinct cache keys | 10 |
| Combined bytes if each distinct key stored once | 30408704 (29 MiB) |
| Keys needed for >= 80% request coverage | 5 |
| Combined footprint of those 5 keys (stored once) | 4980736 bytes (4.75 MiB) |
| Requests covered by those 5 keys | 42 of 51 |
| Exact coverage fraction | 42/51 = 14/17 = 82.3529411764706% |

## Key table (all 10 distinct keys, sorted by request count)

| Rank | Cache key | Requests | Payload bytes (stored once) | In 80% cover |
| --- | --- | --- | ---: | :---: |
| 1 | `user:profile:1001` | 12 | 1572864 | yes |
| 2 | `org:settings:42` | 9 | 1048576 | yes |
| 3 | `repo:tree:main` | 8 | 999424 | yes |
| 4 | `issue:comments:777` | 7 | 786432 | yes |
| 5 | `search:index:daily` | 6 | 573440 | yes |
| 6 | `billing:invoice:2024-09` | 3 | 6291456 | no |
| 7 | `audit:events:hourly` | 2 | 5242880 | no |
| 8 | `team:roster:blue` | 2 | 4718592 | no |
| 9 | `webhook:delivery:log` | 1 | 4980736 | no |
| 10 | `release:notes:v3` | 1 | 4194304 | no |

| Key | Requests | Cumulative | Bytes stored | Cumulative bytes |
| --- | --- | --- | ---: | ---: |
| `user:profile:1001` | 12 | 12 | 1572864 | 1572864 |
| `org:settings:42` | 9 | 21 | 1048576 | 2621440 |
| `repo:tree:main` | 8 | 29 | 999424 | 3620864 |
| `issue:comments:777` | 7 | 36 | 786432 | 4407296 |
| `search:index:daily` | 6 | 42 | 573440 | 4980736 |

Top 4 cover 36/51 = 12/17 = 70.5882352941%; top 5 deliver 42/51 = 14/17 = 82.3529411764706%. Byte budget 4980736 -> next power of two 8388608 (8 MiB). CPython lru_cache (`Lib/functools.py`), default maxsize 128, string `lru_cache` appears 10 times; Discussions on python/cpython: 0 threads.
