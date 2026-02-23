# Circuit Breaker Pattern

## What It Is (10-Second Definition)

A circuit breaker wraps a remote call and monitors its failure rate. When failures exceed a threshold, it "opens" the circuit: subsequent calls fail immediately without hitting the downstream service, giving it time to recover. After a cooldown period, it allows a small number of probe requests through. If they succeed, the circuit closes. If not, it stays open.

Prevents **cascading failure**: one sick service taking down everything upstream.

---

## The 3 States

```
CLOSED ──(failure rate exceeds threshold)──> OPEN
  ^                                            |
  |                                    (wait duration elapses)
  |                                            |
  └──(probe requests succeed)──── HALF-OPEN <──┘
                                       |
                             (probe requests fail)
                                       |
                                     OPEN
```

- **CLOSED**: normal operation, all requests pass through, failures counted
- **OPEN**: all requests fail immediately (fail fast), no calls to downstream
- **HALF-OPEN**: a limited number of probe requests allowed through to test recovery

---

## Configuration Parameters and Real-World Values

These are the knobs you tune. Know all of them for interviews.

### Hystrix (Netflix, now maintenance-mode) Defaults

| Parameter | Default | What It Controls |
|-----------|---------|------------------|
| `circuitBreakerRequestVolumeThreshold` | 20 requests | Minimum requests in window before circuit can trip |
| `circuitBreakerErrorThresholdPercentage` | 50% | Failure rate that trips the circuit |
| `metrics.rollingStats.timeInMilliseconds` | 10,000 ms | Rolling window for failure rate calculation |
| `circuitBreakerSleepWindowInMilliseconds` | 5,000 ms | Time circuit stays OPEN before HALF-OPEN probe |
| `execution.isolation.thread.timeoutInMilliseconds` | 1,000 ms | Per-call timeout before counting as failure |

Netflix API ran 40+ thread pools per instance, 5–20 threads each (most at 10), processing 10+ billion command executions per day.

### Resilience4j (Current Standard) Defaults

| Parameter | Default | Notes |
|-----------|---------|-------|
| `failureRateThreshold` | 50% | % of calls that must fail to open circuit |
| `slowCallRateThreshold` | 100% | % of slow calls that also trips circuit |
| `slowCallDurationThreshold` | 60,000 ms | What counts as a "slow call" |
| `slidingWindowType` | COUNT_BASED | Alternative: TIME_BASED |
| `slidingWindowSize` | 100 | Last N calls (count-based) or last N seconds (time-based) |
| `minimumNumberOfCalls` | 100 | Must have this many calls before rate is calculated |
| `waitDurationInOpenState` | 60,000 ms | How long circuit stays OPEN |
| `permittedNumberOfCallsInHalfOpenState` | 10 | Probe requests allowed in HALF-OPEN |
| `automaticTransitionFromOpenToHalfOpenEnabled` | false | Requires a call to trigger HALF-OPEN transition |

### Practical Production Tuning

These defaults are too loose for most production use. Real tuning:

- **Failure rate threshold**: 50% is typical. Lower (25–30%) for critical paths like payments. Higher (60–70%) for flaky non-critical services.
- **Window size**: 20–50 for low-traffic services (avoids the minimum-call problem). 100–200 for high-traffic.
- **Wait duration**: 5–30 seconds, not 60. 60 seconds is too long in most cases. Start at 5s, double on repeated failures.
- **Timeout**: Set to the p99.9 latency of the downstream service. If p50 is 50ms and p99.9 is 200ms, set timeout to 200–250ms.

---

## Real Production Examples

### Netflix (Hystrix)
Netflix uses circuit breakers on every service call in their API layer. When a microservice (e.g., recommendations, ratings) degrades, Hystrix opens the circuit and returns cached or empty fallback data. A movie detail page might degrade gracefully: show the film without ratings rather than failing the entire page. The pattern allowed Netflix to survive partial failures without full outages.

### AWS SDK
The AWS SDK v3 implements retry with exponential backoff and jitter automatically. The circuit breaker logic is layered above this at the application level. AWS Prescriptive Guidance explicitly recommends circuit breakers for service-to-service calls in microservices on AWS. AWS App Mesh (Envoy-based) provides circuit breaking at the infrastructure layer via `outlierDetection` and connection pool limits.

### Envoy / Istio (Service Mesh Approach)
Envoy implements circuit breaking at the network layer, not application layer. No code changes needed.

Two mechanisms:
1. **Connection pool limits**: `maxConnections`, `http1MaxPendingRequests`, `http2MaxRequests` — requests that exceed these limits are rejected (circuit broken for capacity).
2. **Outlier detection (passive circuit breaker)**: Envoy ejects hosts from the load balancing pool when they return 5xx responses. Example: eject a pod for 3 minutes after it throws errors.

Key distinction: Envoy's state is **per-instance per sidecar**, not globally shared. Each pod's Envoy makes its own circuit-open decisions independently.

### Google SRE / Spanner
Large-scale Google services use adaptive throttling: clients track the ratio of accepted vs. attempted requests and self-throttle based on that ratio. Functionally equivalent to a circuit breaker but distributed without shared state.

---

## The Resilience Trinity: Circuit Breakers + Retries + Timeouts

These three patterns must be composed carefully. Used wrong, they interact badly.

### Correct Composition Order (per request)

```
Request
  → Timeout wrapper (outermost)
    → Circuit Breaker check (fail fast if OPEN)
      → Retry loop
        → Individual call with per-attempt timeout
```

The circuit breaker sits **outside** the retry loop. If the circuit is open, don't even enter the retry loop.

### How They Interact

**Timeout → Circuit Breaker**: Timeouts count as failures. If p99.9 latency of your downstream is 200ms and your timeout is 150ms, you will falsely trip the circuit. Set timeouts at or above p99.9.

**Retry → Circuit Breaker**: Retries without a circuit breaker cause retry storms. If 100 clients each retry 3 times against a failing service, you go from 100 RPS to 300 RPS exactly when the service is most fragile. The circuit breaker short-circuits the retry: once the circuit opens, retries return immediately without hitting downstream.

**Retry amplification is the biggest danger**: If you have 5 microservices each retrying 3 times, a request failure at the leaf causes 3^5 = 243 retries total. Retry at one layer only, or use circuit breakers to cut off retry cascades.

**Timeouts must be set relative to the full retry budget**: If you allow 3 retries with 2s timeout each, your caller's timeout must be at least 6s (plus backoff time).

### Bulkheads: The Fourth Leg

A bulkhead isolates resource pools so one slow downstream cannot exhaust all threads.

- **Thread pool bulkhead**: Each downstream service gets its own thread pool. If Notification Service is slow, its pool fills up, but Payment Service's pool is unaffected.
- **Semaphore bulkhead**: Limits concurrent calls via a counter. Lower overhead than thread pools but blocks the calling thread.
- **Connection pool isolation**: Separate DB connection pools for read vs. write paths.

Hystrix used per-dependency thread pools as the primary bulkhead mechanism. Netflix ran 40+ pools per API instance. Resilience4j offers both thread pool and semaphore bulkheads.

**Circuit Breaker + Bulkhead together**: The bulkhead prevents resource exhaustion while the circuit is closing. The circuit breaker stops calls once failure rate is detected. They are complementary, not redundant.

```
Incoming request
  → Bulkhead (am I allowed to use resources?)
    → Circuit Breaker (is downstream healthy?)
      → Execute call
```

---

## Common Interview Follow-Up Questions and Good Answers

### "What happens when the circuit opens? What do you return to the caller?"

You need a **fallback strategy**. Options in order of preference:
1. Cached/stale data (last known good response)
2. Default/degraded response (empty list, placeholder content)
3. Async queue (write to Kafka, process when service recovers)
4. Immediate error with clear error code (fail fast rather than hang)

The fallback must not itself make a network call. If the fallback calls another service, wrap that in its own circuit breaker.

### "How do you decide when to trip the circuit? Isn't 50% failure rate obvious?"

The minimum request volume threshold matters more than people think. If you only had 4 requests and 2 failed, that's 50% failure rate but statistically meaningless. Hystrix defaults to requiring 20 requests in the window before tripping. For low-traffic services at night, the circuit might never have enough data to make a decision. You need to think about floor volume separately from rate.

Also: distinguish **failure types**. Not all failures should count. HTTP 400 (bad request) is a client bug, not a service failure. HTTP 429 (rate limited) is real pressure. HTTP 503 is a service failure. Circuit breakers should only count errors that indicate downstream health problems.

### "What's the thundering herd problem on recovery?"

When a circuit transitions from OPEN to HALF-OPEN, if you have 50 instances all checking at the same time, all 50 may simultaneously decide the cooldown has expired and all probe the downstream service at once. A service trying to recover from overload now gets hammered with 50 concurrent probe requests.

Solutions:
- Serialize probe requests using a distributed lock
- Use Resilience4j's `permittedNumberOfCallsInHalfOpenState` (allows only N probes) combined with per-instance coordination
- Jitter the cooldown expiry so instances don't all transition simultaneously
- Have one designated probe instance (leader election)

### "Per-instance or shared circuit breaker state?"

Most library implementations (Hystrix, Resilience4j) are **per-instance**. Each instance of your service maintains its own circuit state independently.

This has a consequence: if you have 10 instances and one starts seeing failures, only that instance's circuit trips. The other 9 keep sending traffic. This is actually fine for load balancer scenarios (the LB routes away from the failing instance), but can be a problem if the failures are global (downstream is completely down).

Service mesh (Envoy/Istio) maintains state per-sidecar-proxy, which is also per-instance. True globally shared circuit state requires a distributed cache (Redis) which adds complexity and a new failure mode.

### "How is a circuit breaker different from rate limiting?"

- Rate limiting protects **your service** from inbound traffic overload
- Circuit breaker protects **your service** from outbound dependency failures

They protect in opposite directions. Rate limiting is egress throttling for the caller. Circuit breaking is ingress protection for the callee.

### "What's slow call rate threshold?"

Resilience4j can open the circuit not just on errors but on **slow calls**. If 80% of calls are taking longer than 2 seconds, that's a health signal even if they're all returning 200 OK. Slow call rate threshold catches latency degradation that doesn't manifest as errors. Hystrix handles this via the timeout counting as a failure.

### "How do you test circuit breakers?"

- Chaos engineering: deliberately kill a downstream service and verify the circuit opens
- Fault injection: return 500s or inject latency via a proxy (Toxiproxy, Chaos Monkey)
- Verify fallback paths actually execute correct logic
- Verify circuit closes after recovery (HALF-OPEN probes succeed)
- Verify monitoring/alerting fires when circuit opens

Circuits that trip silently in production without alerting are a hidden operational risk.

---

## Subtle Gotchas and Tradeoffs

### 1. The Minimum Call Volume Trap

At low traffic, the circuit never has enough samples to compute a meaningful failure rate. A service that's completely dead might not trip the circuit if it only gets 5 requests in the measurement window. Fix: use time-based sliding windows for low-traffic services, and set `minimumNumberOfCalls` to a low value (5–10) for non-critical paths.

### 2. The Cold Start Problem

On service startup, the circuit is CLOSED and has no history. The first batch of requests hits a potentially unhealthy downstream. You have no protection until the sliding window fills. Mitigation: warm up slowly, use readiness probes, or pre-seed state from a shared store.

### 3. Cascading Fallbacks

If the fallback for Service A is to call Service B, and Service B is also down, you now need a circuit breaker on that fallback call. Chains of fallbacks must each be independently protected. Hystrix supports chained Hystrix commands for this reason.

### 4. State Inconsistency Across Instances

Per-instance state means different instances will have different circuit states at the same time. This is usually acceptable but can cause confusion in debugging: instance 1 is happily serving, instance 2 is returning fallbacks, and your logs show a mix of both. Centralized dashboards (Hystrix Dashboard, Resilience4j + Micrometer) are essential for visibility.

### 5. Half-Open Race Condition

In concurrent environments, multiple threads can simultaneously pass the "is the circuit open?" check and all enter HALF-OPEN probing. Resilience4j handles this with atomic operations. Implementations without proper synchronization can have two threads simultaneously evaluate conflicting probe outcomes and make contradictory state transition decisions.

### 6. Retry + Circuit Breaker Double-Counting

If retries are inside the circuit breaker's scope, one user request that retries 3 times counts as 3 failures toward the circuit breaker's threshold. This makes the circuit trip faster under load, which may or may not be what you want. Put the circuit breaker outside the retry loop to count user requests, not attempts.

### 7. Wrong Timeout Values Trip Circuits Prematurely

If your downstream's p99 latency is 800ms but you set a timeout of 500ms, you'll regularly timeout on healthy requests and the circuit may stay open even when the downstream is healthy. Always calibrate timeouts from actual latency percentile data, not intuition.

### 8. Health Check ≠ Circuit Breaker Health

A service can pass its `/health` check while still being degraded in a way the circuit breaker detects. For example, a database running slowly: health check passes, but actual query latency causes 80% slow calls. Circuit breakers on actual traffic paths are more accurate than health endpoints.

---

## Where Circuit Breakers Live in Architecture

| Location | Mechanism | Trade-off |
|----------|-----------|-----------|
| In-process library | Hystrix, Resilience4j | App-language specific, fine-grained control |
| Sidecar proxy | Envoy, Linkerd | Language agnostic, per-instance, infra overhead |
| API Gateway | Kong, AWS API GW | Protects backend from client spikes, less granular |
| Client-side | Mobile apps, browsers | Prevents battery drain and stuck UIs, no server relief |

---

## Interview One-Liners Worth Having Ready

- "Circuit breakers prevent cascading failure by failing fast instead of queuing indefinitely."
- "The three states—closed, open, half-open—implement the probe-recover-confirm pattern."
- "I'd set the failure rate threshold at 50% with a minimum of 20 requests in a 10-second rolling window, then tune from real traffic data."
- "Retries and circuit breakers are complementary: retries handle transient failures, circuit breakers handle persistent ones. Without a circuit breaker, retries amplify load on a failing service."
- "The thundering herd on recovery is the half-open state's main gotcha—you need to serialize probe requests, not let all instances probe simultaneously."
- "In service mesh deployments, the circuit breaker state is per-sidecar, not global. A globally failing downstream needs all instances to independently trip."
- "Fallback responses must not make network calls. If they do, that fallback also needs its own circuit breaker."

---

## Sources

- [Netflix Hystrix: How It Works](https://github.com/netflix/hystrix/wiki/how-it-works)
- [Resilience4j Circuit Breaker Configuration](https://resilience4j.readme.io/docs/circuitbreaker)
- [AWS Builders' Library: Timeouts, Retries, and Backoff with Jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [AWS Prescriptive Guidance: Circuit Breaker Pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html)
- [Why Circuit Breaker Recovery Needs Coordination](https://blog.bolshakov.dev/2025/12/06/why-circuit-breaker-recovery-needs-coordination.html)
- [Comparing Envoy/Istio Circuit Breaking with Netflix Hystrix](https://blog.christianposta.com/microservices/comparing-envoy-and-istio-circuit-breaking-with-netflix-hystrix/)
- [Resilience4j vs Hystrix](https://www.tutorialpedia.org/blog/resilience4j-vs-hystrix-what-would-be-the-best-for-fault-tolerance/)
- [Building Resilient Systems: Circuit Breakers and Retry Patterns](https://dasroot.net/posts/2026/01/building-resilient-systems-circuit-breakers-retry-patterns/)
