# Circuit Breaker Pattern

## What It Is

A circuit breaker wraps a remote call and monitors its failure rate. When failures exceed a threshold, it "opens" the circuit: subsequent calls fail immediately without hitting the downstream service, giving it time to recover. After a cooldown, it allows a small number of probe requests through. If they succeed, the circuit closes. If not, it stays open.

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
- **HALF-OPEN**: limited probe requests allowed through to test recovery

---

## Configuration Parameters

### Hystrix (Netflix, maintenance-mode) Defaults

| Parameter | Default | What It Controls |
|-----------|---------|------------------|
| `circuitBreakerRequestVolumeThreshold` | 20 requests | Minimum requests before circuit can trip |
| `circuitBreakerErrorThresholdPercentage` | 50% | Failure rate that trips the circuit |
| `metrics.rollingStats.timeInMilliseconds` | 10,000 ms | Rolling window for failure rate |
| `circuitBreakerSleepWindowInMilliseconds` | 5,000 ms | Time OPEN before HALF-OPEN probe |
| `execution.isolation.thread.timeoutInMilliseconds` | 1,000 ms | Per-call timeout |

Netflix ran 40+ thread pools per instance, 10+ billion command executions per day.

### Resilience4j (Current Standard) Defaults

| Parameter | Default | Notes |
|-----------|---------|-------|
| `failureRateThreshold` | 50% | % of calls that must fail to open circuit |
| `slowCallRateThreshold` | 100% | % of slow calls that also trips circuit |
| `slidingWindowType` | COUNT_BASED | Alternative: TIME_BASED |
| `slidingWindowSize` | 100 | Last N calls (count-based) |
| `minimumNumberOfCalls` | 100 | Must have this many calls before rate is calculated |
| `waitDurationInOpenState` | 60,000 ms | How long circuit stays OPEN |
| `permittedNumberOfCallsInHalfOpenState` | 10 | Probe requests in HALF-OPEN |

### Production Tuning

- **Failure rate**: 50% typical. Lower (25–30%) for critical paths like payments. Higher (60–70%) for flaky non-critical services.
- **Window size**: 20–50 for low-traffic services. 100–200 for high-traffic.
- **Wait duration**: 5–30s in practice — 60s default is usually too long.
- **Timeout**: set at p99.9 latency of the downstream, not intuition.

---

## Merchant System Example

Notification Service is down.

**Without circuit breaker:**
```
Order Service → Notification Service (failing)
              → keeps retrying
              → threads pile up
              → system crashes
```

**With circuit breaker:**
```
Order Service → detects 50% failure rate
              → opens circuit
              → writes event to Kafka for retry later
              → returns success to customer
              → system stays healthy
```

---

## The Resilience Trinity: Circuit Breakers + Retries + Timeouts

### Correct Composition Order

```
Request
  → Timeout wrapper (outermost)
    → Circuit Breaker check (fail fast if OPEN)
      → Retry loop
        → Individual call with per-attempt timeout
```

The circuit breaker sits **outside** the retry loop. If the circuit is open, skip the retry entirely.

### How They Interact

- **Timeout → Circuit Breaker**: timeouts count as failures. If your timeout is below the downstream's p99.9 latency, you'll falsely trip the circuit.
- **Retry amplification danger**: 5 services each retrying 3 times = 3^5 = 243 requests from one user request. Retry at one layer only.
- **Bulkheads**: isolate thread pools per downstream so one slow service can't exhaust all threads. Complementary to circuit breakers — bulkhead prevents resource exhaustion while the circuit is closing.

---

## Where Circuit Breakers Live

| Location | Mechanism | Trade-off |
|----------|-----------|-----------|
| In-process library | Hystrix, Resilience4j | Language-specific, fine-grained control |
| Sidecar proxy | Envoy, Linkerd | Language agnostic, infra overhead |
| API Gateway | Kong, AWS API GW | Protects backend from client spikes |
| Client-side | Mobile apps | Prevents battery drain, no server relief |

---

## Client-Side Example (Android)

If WebSocket keeps failing:
- Stop reconnecting aggressively
- Backoff for 30s
- Switch to polling
- Avoid draining battery

That's client-side circuit breaking.

---

## Common Interview Follow-Ups

**"What do you return when the circuit opens?"**
Fallback options in order of preference: cached/stale data → degraded default response → async queue (Kafka) → fast error. The fallback must not itself make a network call without its own circuit breaker.

**"Isn't 50% failure rate obvious? What's subtle?"**
The minimum request volume matters more. 2 failures out of 4 requests is 50% but statistically meaningless. Also: HTTP 400 (bad client input) should not count toward failures. HTTP 503 and timeouts should.

**"What's the thundering herd problem on recovery?"**
When 50 instances all see the cooldown expire simultaneously, all 50 probe the recovering service at once. Solutions: jitter the cooldown expiry, use a distributed lock for probes, or limit probe count with `permittedNumberOfCallsInHalfOpenState`.

**"Per-instance or shared state?"**
Library-based breakers are per-instance. This is usually fine — the LB routes away from broken instances. True global state requires Redis, which adds a new failure mode.

**"How is this different from rate limiting?"**
Rate limiting protects your service from inbound overload. Circuit breaking protects your service from outbound dependency failures. Opposite directions.

---

## Gotchas

1. **Minimum call volume trap**: low-traffic services at night may never collect enough samples. Use time-based windows and set `minimumNumberOfCalls` to 5–10 for low-traffic paths.
2. **Cold start**: circuit starts CLOSED with no history. First requests hit a potentially unhealthy downstream with no protection.
3. **Retry + circuit breaker double-counting**: if retries are inside the circuit breaker scope, one user request retrying 3 times counts as 3 failures. Put the circuit breaker outside the retry loop.
4. **Wrong timeout values**: timeout below downstream p99 causes false positives — circuit stays open even when downstream is healthy.
5. **Half-open race condition**: multiple threads can simultaneously pass the "is circuit open?" check. Resilience4j handles this atomically; hand-rolled implementations often don't.

---

## Interview One-Liners

- "Circuit breakers prevent cascading failure by failing fast instead of queuing indefinitely."
- "I'd set the failure rate threshold at 50% with a minimum of 20 requests in a 10-second rolling window, then tune from real traffic data."
- "Retries handle transient failures; circuit breakers handle persistent ones. Without a circuit breaker, retries amplify load on a failing service."
- "The thundering herd on recovery is the half-open state's main gotcha — serialize probe requests, don't let all instances probe simultaneously."
- "Fallback responses must not make network calls. If they do, that fallback also needs its own circuit breaker."
