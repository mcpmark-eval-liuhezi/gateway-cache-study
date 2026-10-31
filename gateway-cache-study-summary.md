# gateway-cache-study summary table

API gateway in-process cache sizing study, computed programmatically from `access_sample.txt` (one line per request, `<cache key> <payload size in bytes>`). Parsing: 51 well-formed lines, zero malformed lines, zero key/size conflicts.

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
| Proposed cache capacity (footprint rounded up to next power of two) | 67108864 bytes = 64 MiB |

Arithmetic check: 12 + 9 + 8 + 7 + 6 = 42 requests; 18874368 + 9437184 + 7995392 + 5505024 + 3440640 = 45252608 bytes; next power of two: 2^25 = 33554432 < 45252608 <= 2^26 = 67108864, so 45252608 -> 67108864 bytes (64 MiB exactly; headroom 21893057 bytes, ~41%).

## Key table (all 10 distinct keys, sorted by request count desc, then name asc)

| Rank | Cache key | Requests | Payload bytes (single payload stored once) | Share of requests | In 80% cover |
| --- | --- | --- | ---: | ---: | :---: |
| 1 | user:profile:1001 | 12 | 18874368 | 23.5294118% | yes |
| 2 | org:settings:42 | 9 | 9437184 | 17.6470588% | yes |
| 3 | repo:tree:main | 8 | 7995392 | 15.6862745% | yes |
| 4 | issue:comments:777 | 7 | 5505024 | 13.7254901% | yes |
| 5 | search:index:daily | 6 | 3440640 | 11.7647059% | yes |
| 6 | billing:invoice:2024-09 | 3 | 18874368 | 5.88235294% | no |
| 7 | audit:events:hourly | 2 | 10485760 | 3.92156863% | no |
| 8 | team:roster:blue | 2 | 9437184 | 3.92156863% | no |
| 9 | webhook:delivery:log | 1 | 4980736 | 1.96078432% | no |
| 10 | release:notes:v3 | 1 | 4194304 | 1.96078432% | no |

## CPython lru_cache benchmark

- File: python/cpython `Lib/functools.py` (main); entry point `def lru_cache(maxsize=128, typed=False)`; exact default maxsize = 128; string `lru_cache` appears 10 times.
- GitHub Discussions on python/cpython: checked, zero threads found.

## Recommendation

Propose 67108864 bytes (64 MiB) of in-process cache payload capacity: the five most-frequently-requested keys cover 42 of 51 requests (82.3529411764706%) at a combined footprint of 45252608 bytes; rounding up to the next power of two gives 67108864 bytes (64 MiB exactly) with ~41% headroom, vs. 93224960 bytes (88.84 MiB) to cache all 10 distinct keys. lru_cache's default maxsize of 128 bounds entry population, not bytes, so it complements rather than competes with this byte budget.