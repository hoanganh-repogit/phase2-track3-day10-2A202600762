# Day 10 Reliability Report

## 1. Architecture summary

The LLM Agent Gateway reliability architecture consists of a multi-tier pipeline designed to maximize availability, minimize cost, and control latencies under flaky provider conditions.

```
User Request
    |
    v
[Gateway] ---> [Cache check (Memory/Redis)] ---> HIT? return cached (0ms, $0)
    |                                                 |
    v                                                 v MISS
[Circuit Breaker: Primary] -------> Provider A (Primary)
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B (Backup)
    |  (OPEN? skip)
    v
[Static fallback message] ("The service is temporarily degraded...")
```

1. **Semantic Cache (ResponseCache / SharedRedisCache)**: Checks input query similarity using character n-grams and word tokens cosine similarity. Incorporates privacy guardrails (skipping matching sensitive patterns like `password`, `ssn`) and false-hit protection (rejecting matches with differing 4-digit numbers/years).
2. **Circuit Breaker (CircuitBreaker)**: Wraps each upstream provider call. Employs a 3-state machine (`CLOSED`, `OPEN`, `HALF_OPEN`). It trips to `OPEN` if failures hit a threshold, failing fast immediately. After a reset timeout, it enters `HALF_OPEN` to probe the provider, recovering to `CLOSED` on success or re-tripping on failure.
3. **Gateway Routing Pipeline (ReliabilityGateway)**: Chains the cache lookup, fallback provider iteration (under circuit breakers), and a degraded static fallback response if all upstreams are failing.

---

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Standard threshold to allow transient failures (e.g. network blips) before tripping the circuit. |
| reset_timeout_seconds | 0.1 | Set low (100ms) for the simulation to enable active circuit breaker recovery cycles within fast sequential request loops. |
| success_threshold | 1 | Restores the provider to CLOSED state after a single successful probe, resuming normal traffic. |
| cache TTL | 300 | Cached entries are stored for 5 minutes to balance data freshness against provider cost savings. |
| similarity_threshold | 0.92 | High threshold to prevent semantic drift and ensure only highly relevant answers are returned from cache. |
| load_test requests | 100 | Sufficient request volume per scenario to gather robust statistical latency percentiles and circuit metrics. |

---

## 3. SLO definitions

We define targets for key Service Level Indicators (SLIs) and evaluate if they were met under the chaos load suite:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 93.25% (Memory) / 95.75% (Redis) | No (due to extreme chaos scenarios like 100% primary failure and flaky backup). |
| Latency P95 | < 2500 ms | 319.1 ms (Memory) / 316.1 ms (Redis) | Yes (significantly faster than target). |
| Fallback success rate | >= 95% | 75.89% (Memory) / 77.63% (Redis) | No (because the fallback provider itself is flaky in some scenarios). |
| Cache hit rate | >= 10% | 57.0% (Memory) / 74.5% (Redis) | Yes (exceeded target). |
| Recovery time | < 5000 ms | 524.89 ms (Memory) / 516.07 ms (Redis) | Yes (well under target). |

---

## 4. Metrics

Below are the metrics compiled from the memory cache run (`reports/metrics.json`):

| Metric | Value |
|---|---:|
| availability | 0.9325 |
| error_rate | 0.0675 |
| latency_p50_ms | 271.14 |
| latency_p95_ms | 319.10 |
| latency_p99_ms | 320.20 |
| fallback_success_rate | 0.7589 |
| cache_hit_rate | 0.5700 |
| estimated_cost_saved | 0.228000 |
| circuit_open_count | 97 |
| recovery_time_ms | 524.89 |

---

## 5. Cache comparison

Comparison of metrics compiled without caching (`reports/metrics_no_cache.json`) vs. with caching (`reports/metrics.json`):

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 273.77 | 271.14 | -2.63 ms (-0.96%) |
| latency_p95_ms | 314.47 | 319.10 | +4.63 ms (+1.47%) |
| estimated_cost | 0.144928 | 0.064548 | -0.080380 (-55.46%) |
| cache_hit_rate | 0.00 | 0.57 | +0.57 |

> [!NOTE]
> Caching provides over 55% reduction in API cost while keeping median response time low. The slight increase in P95 latency is a minor trade-off caused by provider latency distributions under fallback load.

---

## 6. Redis shared cache

### Why shared cache matters for production

- **In-memory cache insufficiency**: In a multi-instance production environment, each server maintains its own isolated in-memory cache. A request handled and cached by Instance A will still be a cache miss when sent to Instance B, resulting in duplicate upstream API calls, higher costs, and potential state inconsistency.
- **SharedRedisCache benefits**: A centralized Redis cache enables shared state. If Instance A completes a request and caches it, Instance B immediately benefits from the cache hit.

### Evidence of shared state

Running `pytest tests/test_redis_cache.py` validates connectivity, set/get operations, TTL, and state sharing:

```
tests/test_redis_cache.py::test_redis_connection PASSED                  [ 16%]
tests/test_redis_cache.py::test_set_and_exact_get PASSED                 [ 33%]
tests/test_redis_cache.py::test_ttl_expiry PASSED                        [ 50%]
tests/test_redis_cache.py::test_shared_state_across_instances PASSED     [ 66%]
tests/test_redis_cache.py::test_privacy_query_not_cached PASSED          [ 83%]
tests/test_redis_cache.py::test_false_hit_different_years PASSED         [100%]
```

### Redis CLI output

Querying the keys from the local Redis instance shows deterministic short query hash keys successfully populated:

```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:dacb2b833659"
2) "rl:cache:734852f3cf4a"
3) "rl:cache:d354658dc020"
4) "rl:cache:da61fb49b4f6"
5) "rl:cache:fff10da1c72c"
6) "rl:cache:0bc3b1acf73d"
7) "rl:cache:095946136fea"
8) "rl:cache:3936614ac4c2"
9) "rl:cache:98332d0d1c9c"
10) "rl:cache:9e413fd814eb"
11) "rl:cache:844ef0143a5c"
12) "rl:cache:4fc3c69b9376"
13) "rl:cache:3dab98c0e49e"
```

---

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | 100% primary requests failed, circuit tripped to OPEN, traffic routed to backup | Pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit transitioned between OPEN and CLOSED, requests split between primary/backup | Pass |
| all_healthy | All requests via primary, no circuit opens | All requests routed to primary, 0 circuit trips | Pass |
| backup_flaky_50 | Backup is flaky when primary fails, degraded run | Primary falls back to flaky backup; some requests return static fallbacks | Pass |

---

## 8. Failure analysis

### What could still go wrong?
If the central Redis instance becomes unavailable or latency spikes, the gateway's cache check will fail or slow down dramatically. Furthermore, the circuit breaker state is currently kept in-memory. If one gateway instance trips the circuit because Provider A is down, other instances will keep sending requests to Provider A until they independently trip their own circuit breakers, causing a minor retry storm.

### Proposed changes
1. **Graceful degradation**: Wrap Redis calls in a try-except block and fallback to a local in-memory cache if Redis is down.
2. **Redis-backed circuit state**: Store circuit breaker counters and status in Redis (using atomic `INCR` and `EXPIRE`) so all instances share the provider health state.

---

## 9. Next steps

1. **Implement Redis-backed Circuit Breaker**: Synchronize circuit breaker status across all instances using Redis.
2. **Add Concurrency Support**: Use a thread pool to dispatch simulation requests concurrently, verifying that the cache and circuit breaker behave safely under parallel load.
3. **Advanced Semantic Embeddings**: Upgrade similarity matching from n-gram cosine overlap to use a local sentence-transformer embedding for richer context matching.