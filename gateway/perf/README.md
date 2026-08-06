# API Platform Gateway Performance Results

<!-- PERF_RESULTS_START -->
### Test ID `29-20260806-165025`

- **UTC:** `2026-08-06T17:54:49Z`
- **Chart:** `1.2.0-rc`
- **Gateway node:** `c5.4xlarge`
- **Tuning:** `ROUTER_CONCURRENCY=3`, `GOMAXPROCS=1`
- **RUN_PERF_OPTS:** `-u 100 -u 1000 -b 1 -s 0 -d 900 -w 180 -i api_api_plain_get`

| Scenario | Users | TPS | Avg (ms) | Err % | p90 (ms) | p99 (ms) | Samples |
| --- | --- | --- | --- | --- | --- | --- | --- |
| API Gateway Plain GET | 100 | 10705.52 | 9.18 | 0.00 | 13.00 | 18.00 | 7697036 |
| API Gateway Plain GET | 1000 | 10383.15 | 95.78 | 0.00 | 117.00 | 139.00 | 7469562 |

---
<!-- PERF_RESULTS_END -->
