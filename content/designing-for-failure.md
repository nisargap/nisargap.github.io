+++
title = "Designing for Failure: Building Resilient Distributed Systems"
date = 2026-01-12
description = "In distributed systems, failure isn't exceptional—it's expected. Here's how to design for it."
[taxonomies]
tags = ["system-design", "distributed-systems", "reliability"]
categories = ["System Design"]
+++

The first lesson of distributed systems: everything fails. Networks partition. Disks corrupt. Services crash. The question isn't whether failures will happen, but how your system behaves when they do.

<!-- more -->

## The Fallacies We Believe

Peter Deutsch's "Fallacies of Distributed Computing" remain as relevant today as when they were written:

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

Every distributed system bug I've debugged traced back to violating one of these assumptions.

## Timeouts: Your First Line of Defense

A missing timeout is a system waiting forever. Every network call needs a timeout, and choosing the right value is harder than it seems.

**Too short**: Legitimate slow responses get treated as failures, increasing load as clients retry.

**Too long**: Resources stay tied up waiting for responses that will never come.

A common pattern is adaptive timeouts that adjust based on observed latency:

```python
class AdaptiveTimeout:
    def __init__(self, initial_ms=100):
        self.p99_latency = initial_ms
        self.samples = []

    def get_timeout(self):
        # Timeout at 2x the p99 latency
        return self.p99_latency * 2

    def record_latency(self, ms):
        self.samples.append(ms)
        if len(self.samples) > 1000:
            self.samples.pop(0)
        self.p99_latency = sorted(self.samples)[int(len(self.samples) * 0.99)]
```

## Circuit Breakers: Failing Fast

When a downstream service is struggling, continuing to send requests makes things worse. Circuit breakers detect failures and stop calling the failing service:

**Closed**: Requests flow normally. Failures are counted.

**Open**: Requests fail immediately without calling the downstream service.

**Half-Open**: After a timeout, allow a few test requests. If they succeed, close the circuit. If they fail, reopen it.

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=30):
        self.failures = 0
        self.state = "closed"
        self.last_failure_time = None
        self.failure_threshold = failure_threshold
        self.reset_timeout = reset_timeout

    def call(self, func):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()

        try:
            result = func()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise

    def on_success(self):
        self.failures = 0
        self.state = "closed"

    def on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        if self.failures >= self.failure_threshold:
            self.state = "open"
```

## Retries: Harder Than They Look

Naive retry logic causes thundering herds. When a service recovers from an outage, all clients retry simultaneously, potentially causing another outage.

**Exponential backoff**: Each retry waits longer than the last.

**Jitter**: Add randomness to spread out retry attempts.

```python
def retry_with_backoff(func, max_attempts=5):
    for attempt in range(max_attempts):
        try:
            return func()
        except RetryableError:
            if attempt == max_attempts - 1:
                raise
            # Exponential backoff with jitter
            delay = min(100 * (2 ** attempt), 10000)  # Cap at 10 seconds
            jitter = random.uniform(0, delay * 0.1)
            time.sleep((delay + jitter) / 1000)
```

**Idempotency**: Retries only work safely if the operation can be repeated without side effects. Design your APIs to be idempotent—use idempotency keys for operations that create resources.

## Bulkheads: Isolating Failures

Ships have bulkheads so that a breach in one compartment doesn't sink the entire vessel. Software systems need the same isolation.

Techniques include:

**Thread pool isolation**: Each downstream service gets its own thread pool. If one service is slow, it exhausts its pool without affecting others.

**Process isolation**: Critical components run in separate processes, so memory leaks or crashes don't cascade.

**Regional isolation**: Deploy in multiple regions so a regional outage doesn't take down your entire system.

## Graceful Degradation

When parts of your system fail, the whole system shouldn't fail. Design for partial functionality:

- If recommendations are unavailable, show popular items instead of an error
- If real-time data is stale, display it with a "last updated" timestamp
- If a feature flag service is down, fall back to conservative defaults

The user experience of degraded functionality is almost always better than an error page.

## Health Checks and Load Shedding

**Health checks** let load balancers route around failing instances. But shallow health checks (just returning 200) miss real problems. Deep health checks verify database connectivity, dependency availability, and resource headroom.

**Load shedding** means intentionally dropping requests when the system is overloaded. It's better to serve 80% of requests successfully than to have 100% of requests timeout.

```python
class LoadShedder:
    def __init__(self, max_concurrent=1000):
        self.current = 0
        self.max = max_concurrent

    def try_acquire(self):
        if self.current >= self.max:
            raise OverloadedError("Service at capacity")
        self.current += 1

    def release(self):
        self.current -= 1
```

## Observability: Knowing What Failed

You can't fix what you can't see. Resilient systems have:

**Metrics**: Request rates, error rates, latency percentiles. Alert on deviations from normal.

**Distributed tracing**: Follow a request through multiple services to identify bottlenecks and failures.

**Structured logging**: Logs with correlation IDs that can be searched and aggregated.

## Chaos Engineering

The ultimate test of resilience is intentionally causing failures in production. Netflix's Chaos Monkey randomly terminates instances. Their Chaos Kong simulates entire region failures.

Start small: Kill one instance and verify the system handles it. Gradually escalate to larger failures as confidence grows.

## The Cost of Resilience

Resilience isn't free. It requires:

- More complex code
- More infrastructure (multiple regions, redundant services)
- More testing (failure scenarios, chaos experiments)
- More operational effort (runbooks, on-call rotations)

Not every system needs Netflix-level resilience. Match your investment to your requirements. But never assume failures won't happen.

They will. The question is whether you'll be ready.
