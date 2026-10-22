# gateway-cache-study — summary

This file is the linkable summary for the review doc (published in-repo because the current API token cannot create gists). Source: `access_sample.txt` — one line per request, `<cache key> <payload size in bytes>`, sampled from last night's API gateway request log. All figures computed programmatically and independently re-derivable from the sample.

## Summary figures

| Metric | Value |
| --- | --- |
| Total requests in sample | 51 |
| Distinct cache keys | 10 |
| Combined bytes if each distinct key stored once | 30408704 (29 MiB) |
| Keys needed for >= 80% request coverage | 5 |
| Combined footprint of those 5 keys | 4980736 bytes (4.75 MiB) |
| Requests covered by those 5 keys | 42 of 51 |
| Exact coverage fraction | 42/51 = 14/17 = 82.3529411765% |
| **Proposed capacity (footprint rounded up to next power of two)** | **8388608 bytes = 8 MiB** |

## Key table (all 10 distinct keys, sorted by request count)

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

The top 4 keys cover only 70.5882352941% of requests, so 5 keys is the minimum that reaches 80%.

## CPython lru_cache reference point

`functools.lru_cache` is defined in [`Lib/functools.py`](https://github.com/python/cpython/blob/main/Lib/functools.py) (python/cpython): default **maxsize = 128**; the string `lru_cache` appears 10 times in that file. GitHub Discussions are not enabled on python/cpython, so there are no cache eviction/sizing discussion threads there to cite.

## Recommendation

Provision the API gateway's in-process cache with **8 MiB (8388608 bytes)** of payload capacity; see the [README](https://github.com/mcpmark-eval-liuhezi/gateway-cache-study/blob/main/README.md) for the full reasoning and [issue #1](https://github.com/mcpmark-eval-liuhezi/gateway-cache-study/issues/1) for the decision record.