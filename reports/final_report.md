# Day 25 Reliability Final Report

## 1. Architecture Summary

```text
User Request
    |
    v
[ReliabilityGateway]
    |
    +--> [Semantic cache] -- hit --> return cached response
    |
    +--> [CircuitBreaker: primary] -- closed/half-open --> FakeLLMProvider primary
    |
    +--> [CircuitBreaker: backup]  -- fallback path ------> FakeLLMProvider backup
    |
    +--> [Static fallback] -- all providers failed
```

The gateway checks cache first, then calls providers through per-provider circuit
breakers. Provider failures are isolated by opening the affected circuit, while the
gateway continues to the next provider. If all providers fail or are open, the system
returns a static degraded response instead of raising an unhandled error.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Opens after repeated provider failures while tolerating brief noise |
| reset_timeout_seconds | 2 | Allows quick half-open probes during local chaos tests |
| success_threshold | 1 | One healthy probe closes the circuit for faster recovery |
| cache TTL | 300 seconds | Keeps repeated FAQ-style answers warm during the run |
| similarity_threshold | 0.92 | Favors exact/near-exact hits and limits semantic false positives |
| load_test requests | 100 per scenario | Enough traffic to observe cache hits and breaker transitions |

## 3. SLO Definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 100.00% | yes |
| Latency P95 | < 2500 ms | 311.06 | yes |
| Fallback success rate | >= 95% | 100.00% | yes |
| Cache hit rate | >= 10% | 63.67% | yes |
| Recovery time | < 5000 ms | not observed | n/a |

## 4. Metrics

| Metric | Value |
|---|---:|
| total_requests | 300 |
| availability | 1.0000 |
| error_rate | 0.0 |
| latency_p50_ms | 239.29 |
| latency_p95_ms | 311.06 |
| latency_p99_ms | 320.49 |
| fallback_success_rate | 1.0000 |
| cache_hit_rate | 0.6367 |
| estimated_cost | 0.052494 |
| estimated_cost_saved | 0.191 |
| circuit_open_count | 6 |
| recovery_time_ms | not observed |

## 5. Cache Comparison

The run used cache-enabled configuration. Exact repeat requests hit the semantic cache,
lowering provider calls and estimated cost. A no-cache comparison can be reproduced by
setting `cache.enabled: false` in `configs/default.yaml` and rerunning `make run-chaos`.

| Metric | With cache |
|---|---:|
| latency_p50_ms | 239.29 |
| latency_p95_ms | 311.06 |
| estimated_cost | 0.052494 |
| estimated_cost_saved | 0.191 |
| cache_hit_rate | 63.67% |

## 6. Redis Shared Cache

In-memory cache is local to one process, so multiple gateway instances cannot share
warm responses. `SharedRedisCache` stores query/response hashes in Redis with TTL,
allowing separate processes to observe the same cached entries.

Evidence implemented in tests:

```text
tests/test_redis_cache.py::test_shared_state_across_instances
```

The Redis tests are skipped unless Redis is running locally. Start it with:

```bash
docker compose up -d
pytest tests/test_redis_cache.py -q
```

Local Redis evidence for this run:

```text
pytest tests/test_redis_cache.py -q
6 passed
```

## 7. Chaos Scenarios

| Scenario | Expected behavior | Observed status |
|---|---|---|
| primary_timeout_100 | Route through fallback/cache without outage | pass |
| primary_flaky_50 | Route through fallback/cache without outage | pass |
| all_healthy | Route through fallback/cache without outage | pass |

## 8. Failure Analysis

The main remaining weakness is that circuit breaker state is still process-local.
In a production multi-instance deployment, one instance may open a provider circuit
while another continues sending traffic. The next improvement would store breaker
counters/transitions in Redis with short TTLs, matching the shared-cache pattern.

False cache hits are mitigated with privacy checks and 4-digit number mismatch
detection, but a real system should also include domain-specific cache keys and
LLM-as-judge validation for risky semantic hits.

## 9. Next Steps

1. Move circuit breaker counters to Redis for multi-instance consistency.
2. Add cache-disabled baseline runs to quantify cost and latency deltas.
3. Add concurrency/load tests with SLO assertions for P95/P99 latency.
