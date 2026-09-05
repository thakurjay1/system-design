# 7. Retries Are Creating a Traffic Spike. How Would You Prevent a Retry Storm?

A **retry storm** happens when a service or downstream dependency starts failing or becoming slow, and clients automatically retry their requests. Those retries create additional traffic, which puts even more pressure on the already unhealthy service.

Instead of helping the system recover, retries make the situation worse.

The basic pattern looks like this:

```text
                    Request
                       |
                       v
                 +-----------+
                 | Service A  |
                 +-----------+
                       |
                       | Request
                       v
                 +-----------+
                 | Service B |
                 +-----------+
                       |
                  B is slow/down
                       |
                       v
                 Request fails
                       |
                       v
                 Retry immediately
                       |
                       v
                 Request fails
                       |
                       v
                 Retry again
                       |
                       v
                 More traffic
                       |
                       v
                 Service B overloaded
                       |
                       v
                 More failures
                       |
                       +------------------+
                                          |
                                          v
                                  RETRY STORM
```

The most important principle is:

> **A retry should help the system recover, not increase the load on a system that is already struggling.**

---

# 1. First Understand Why the Retry Storm Is Happening

Before changing retry configuration, I would first identify:

* Which service is retrying?
* Which downstream service is failing?
* What errors are triggering retries?
* How many retries are happening?
* How quickly are retries being sent?
* How many clients/instances are retrying simultaneously?
* Is the downstream service slow, unavailable, rate-limited, or returning errors?
* Did traffic increase?
* Did a recent deployment change timeout/retry behavior?
* Are multiple layers retrying the same request?

That last point is particularly dangerous.

For example:

```text
Frontend
   |
   v
API Gateway
   | retry x3
   v
Service A
   | retry x3
   v
Service B
   | retry x3
   v
Database
```

A single original request could potentially result in many attempts.

If each layer independently retries three times, the number of downstream attempts can grow dramatically.

Therefore:

> **Retries should be deliberate and ideally controlled at the appropriate layer, rather than blindly enabled everywhere.**

---

# 2. Use Exponential Backoff

One of the most important techniques for preventing retry storms is **exponential backoff**.

Instead of retrying immediately:

```text
Request
  ↓
Fail
  ↓
Retry immediately
  ↓
Fail
  ↓
Retry immediately
  ↓
Fail
```

we progressively increase the waiting time:

```text
Attempt 1
   ↓
Failure
   ↓
Wait 100 ms
   ↓
Attempt 2
   ↓
Failure
   ↓
Wait 200 ms
   ↓
Attempt 3
   ↓
Failure
   ↓
Wait 400 ms
```

A typical formula is:

```text
delay = initialDelay × 2^attempt
```

For example:

```text
Initial delay = 100 ms

Attempt 1 → 100 ms
Attempt 2 → 200 ms
Attempt 3 → 400 ms
Attempt 4 → 800 ms
```

The exact values depend on the system.

The important point is that clients **do not immediately hammer the unhealthy service again**.

---

# 3. Add Jitter

Exponential backoff alone has another problem.

Imagine 10,000 clients all receive an error at exactly the same time.

If they all calculate:

```text
100 ms
200 ms
400 ms
800 ms
```

they may all retry at approximately the same moments.

You could end up with:

```text
10:00:00 → 10,000 requests
10:00:01 → failure

10:00:01.1 → 10,000 retries
10:00:01.3 → 10,000 retries
10:00:01.7 → 10,000 retries
```

This creates synchronized traffic bursts.

That's why we add **jitter**, which introduces some randomness into the retry delay.

For example:

```text
Client A → retry after 137 ms
Client B → retry after 182 ms
Client C → retry after 119 ms
Client D → retry after 163 ms
```

Now the requests are distributed over time.

A common approach is:

```text
retryDelay = exponentialBackoff + randomJitter
```

Or use a capped randomized strategy such as:

```text
delay = random(0, min(cap, base × 2^attempt))
```

The exact backoff algorithm matters less than the principle:

> **Don't allow thousands of clients to retry at exactly the same time.**

---

# 4. Put a Maximum Retry Limit

Retries should never continue indefinitely.

For example:

```text
Maximum attempts = 3
```

Then:

```text
Attempt 1
   ↓
Failure
   ↓
Attempt 2
   ↓
Failure
   ↓
Attempt 3
   ↓
Failure
   ↓
Stop
```

At that point, the application should:

* return an appropriate error,
* use a fallback,
* queue the operation,
* mark it for later processing,
* or allow the user/system to try again later.

For example:

```text
maxAttempts = 3
```

is much safer than:

```text
while request fails:
    retry()
```

---

# 5. Use a Maximum Retry Time / Retry Budget

Limiting attempts is good, but sometimes the number of attempts isn't enough.

Suppose:

```text
Attempt 1 → 100 ms
Attempt 2 → 500 ms
Attempt 3 → 2 sec
Attempt 4 → 5 sec
```

You could technically have only four attempts but still keep the original request alive for a long time.

Therefore, I would also define a **maximum retry duration** or deadline.

For example:

```text
Total request deadline = 3 seconds
```

Once the deadline is reached:

```text
Stop retrying
```

This prevents retries from consuming application resources for too long.

---

# 6. Use Circuit Breakers

A **circuit breaker** is extremely useful when a downstream service is consistently failing.

Instead of every request continuing to call the unhealthy service:

```text
Service A
   |
   +----> Service B ❌
   |
   +----> Service B ❌
   |
   +----> Service B ❌
   |
   +----> Service B ❌
```

the circuit breaker detects repeated failures and opens the circuit.

```text
Service A
   |
   v
Circuit Breaker
   |
   X
   |
Service B
```

Now calls fail fast instead of reaching Service B.

The circuit breaker generally has three states:

```text
             failures
        +----------------+
        |                v
     CLOSED ---------> OPEN
        ^                |
        |                |
        |          wait / timeout
        |                |
        |                v
        +----------- HALF-OPEN
                    |
              test request
```

### CLOSED

Normal traffic flows through.

```text
A → B
```

### OPEN

Service B is considered unhealthy.

```text
A → Circuit Breaker → Fail Fast
```

No unnecessary request is sent to B.

### HALF-OPEN

After some recovery period, allow a small number of test requests.

If successful:

```text
HALF-OPEN → CLOSED
```

If they fail:

```text
HALF-OPEN → OPEN
```

This prevents a recovering service from suddenly receiving the entire traffic load.

---

# 7. Retry Only Errors That Are Actually Retryable

This is one of the most commonly overlooked points.

Not every failure should trigger a retry.

For example:

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
```

usually should not be blindly retried.

If the request itself is invalid, retrying won't fix it.

Similarly, some business errors should not be retried.

Retries are generally more appropriate for **transient failures**, such as:

```text
Timeout
Connection reset
Temporary network failure
503 Service Unavailable
429 Too Many Requests
```

Even here, the exact policy depends on the API semantics.

For example:

```text
Payment request
```

requires much more careful retry handling than:

```text
GET product details
```

because repeating a payment operation can have financial consequences.

Therefore:

> **Retryability should be based on error type and operation semantics, not simply "if anything fails, retry."**

---

# 8. Be Careful With HTTP 429

If a downstream service responds with:

```text
429 Too Many Requests
```

it is explicitly telling us:

> "You are sending too much traffic."

Immediately retrying is obviously counterproductive.

If the service provides:

```text
Retry-After
```

the client should respect it where appropriate.

For example:

```text
Service B
   |
   | 429
   | Retry-After: 2 seconds
   v

Service A waits

   |
   | after 2 sec
   v

Retry
```

This is much better than:

```text
429
 ↓
retry immediately
 ↓
429
 ↓
retry immediately
```

---

# 9. Use Rate Limiting

Retry control should also exist at the infrastructure/service level.

Suppose Service B can safely process:

```text
5,000 requests/sec
```

but clients are generating:

```text
20,000 requests/sec
```

We need some form of traffic control.

Rate limiting can prevent excessive requests from reaching the service.

Conceptually:

```text
Clients
   |
   v
Rate Limiter
   |
   +---- Allowed ----> Service
   |
   +---- Rejected ----> 429
```

This protects the downstream service from being overwhelmed.

---

# 10. Use Load Shedding

Sometimes the system simply cannot process all incoming traffic.

Instead of allowing everything into the system and causing total failure, we deliberately reject or defer some work.

This is called **load shedding**.

For example:

```text
Incoming traffic = 100,000 RPS

System capacity = 20,000 RPS
```

Trying to process all 100,000 requests may cause:

```text
CPU ↑
Memory ↑
Threads ↑
Connections ↑
Latency ↑
Timeouts ↑
Retries ↑
```

Eventually the entire system may collapse.

Instead:

```text
100,000 RPS
      |
      v
Load Shedding
      |
      +----> 20,000 → Process
      |
      +----> 80,000 → Reject / defer / queue
```

This allows the service to remain healthy.

---

# 11. Use Bulkheads

Another useful technique is the **bulkhead pattern**.

The idea is to isolate resources so that one failing dependency doesn't consume everything.

For example:

```text
Service A
 |
 +---- Payment calls
 |      Thread Pool: 20
 |
 +---- Product calls
 |      Thread Pool: 50
 |
 +---- Reporting calls
        Thread Pool: 10
```

Suppose reporting becomes extremely slow.

Without isolation:

```text
Reporting requests
       ↓
Consume all threads
       ↓
Payment requests also blocked
       ↓
Entire Service A becomes unhealthy
```

With bulkheads:

```text
Reporting → only consumes its allocated resources
Payment   → continues operating
Product   → continues operating
```

This is especially useful when retries can consume threads, connection pools, or other limited resources.

---

# 12. Avoid Retrying at Every Layer

Consider this architecture:

```text
Client
  ↓ retry ×3
API Gateway
  ↓ retry ×3
Order Service
  ↓ retry ×3
Payment Service
  ↓
Database
```

This can create an amplification problem.

One original request can result in many downstream attempts.

A better design is to establish a clear retry ownership strategy.

For example:

```text
Client
  |
  v
API Gateway
  |
  v
Order Service
  |
  | controlled retry
  v
Payment Service
```

Not every layer needs its own aggressive retry policy.

The exact location depends on the architecture and responsibility of each layer.

---

# 13. Use Deadlines Instead of Independent Timeouts Everywhere

Consider:

```text
Client timeout = 5 sec

Service A timeout = 5 sec

Service B timeout = 5 sec

Service C timeout = 5 sec
```

This can become problematic because the total execution time can exceed what the original caller is willing to wait.

Instead, propagate a request deadline.

For example:

```text
Client
Deadline = 5 sec
    |
    v
Service A
Remaining = 4 sec
    |
    v
Service B
Remaining = 2 sec
    |
    v
Service C
Remaining = 1 sec
```

Once the deadline expires:

```text
STOP
```

There is little value in continuing expensive work for a request whose caller has already given up.

---

# 14. Use Asynchronous Processing Where Appropriate

Some operations don't need to be completed synchronously.

Suppose a user requests:

```text
Generate monthly report
```

Instead of:

```text
HTTP Request
    ↓
Generate report
    ↓
Wait 30 seconds
    ↓
Response
```

we can use:

```text
Client
  |
  v
API
  |
  +----> Queue
           |
           v
       Worker
           |
           v
       Generate Report
```

The API can respond:

```text
202 Accepted
```

and the client can later check the status or receive a notification.

This reduces pressure on synchronous request threads and makes retry behavior easier to control.

---

# 15. Make Retries Idempotent

This becomes extremely important for operations that change state.

Consider:

```text
POST /payment
```

If the request succeeds but the response is lost:

```text
Client
   |
   | Payment
   v
Payment Service
   |
   | SUCCESS
   |
   X response lost
```

The client doesn't know whether payment succeeded.

If it blindly retries:

```text
Payment
Payment
```

you could charge the customer twice.

Therefore, retries for important operations should use **idempotency**.

For example:

```text
Idempotency-Key: PAYMENT-12345
```

The server stores the result associated with that key.

Then:

```text
Request 1
Key = PAYMENT-12345
→ Payment processed

Request 2
Key = PAYMENT-12345
→ Existing result returned
```

The retry does not create another business operation.

---

# 16. Prefer Queue-Based Retry for Background Processing

For asynchronous workloads, retrying directly from the application can create a huge spike.

Instead:

```text
Application
    |
    v
Queue
    |
    v
Worker
    |
    X Downstream failure
    |
    v
Retry Queue / Delayed Queue
    |
    v
Worker
```

This allows us to control the retry rate.

For example:

```text
Retry after 1 minute
Retry after 5 minutes
Retry after 15 minutes
```

rather than:

```text
retry immediately
retry immediately
retry immediately
```

Dead-letter queues can also be used when a message repeatedly fails.

---

# 17. Monitor Retry Traffic Separately

Retries should be observable.

I would monitor metrics such as:

```text
original_requests
retry_requests
retry_rate
retry_attempt_number
retry_success_rate
retry_failure_rate
circuit_breaker_state
429_rate
timeout_rate
downstream_latency
downstream_error_rate
```

One particularly useful metric is:

```text
Retry Ratio = Retry Requests / Total Requests
```

For example:

```text
Normal:

Original = 100,000
Retries  = 2,000

Retry Ratio = 2%
```

During an incident:

```text
Original = 100,000
Retries  = 300,000

Retry Ratio = 300%
```

That is a strong indication that the system is amplifying its own traffic.

---

# 18. Distributed Tracing Helps Identify Retry Amplification

Suppose one user request produces:

```text
Trace ID: abc123

Service A
   |
   +---- Service B attempt #1 → timeout
   |
   +---- Service B attempt #2 → timeout
   |
   +---- Service B attempt #3 → success
```

Distributed tracing makes this visible.

You can determine:

```text
Original request
     ↓
Attempt 1
     ↓
Attempt 2
     ↓
Attempt 3
```

Without tracing, the downstream service may simply appear to be receiving unusually high traffic.

---

# 19. Retry Storm Prevention Strategy

A production-grade retry strategy usually looks something like this:

```text
                 Request
                    |
                    v
             Is error retryable?
                 /       \
               No         Yes
               |           |
               v           v
          Return error   Retry budget?
                            |
                       +----+----+
                       |         |
                      No        Yes
                       |         |
                       v         v
                  Stop retry   Backoff
                                  |
                                  v
                               Jitter
                                  |
                                  v
                           Circuit Breaker
                                  |
                           +------+------+
                           |             |
                         OPEN         CLOSED
                           |             |
                       Fail fast       Retry
                                         |
                                         v
                                  Rate Limiting
                                         |
                                         v
                                    Downstream
```

This gives us multiple layers of protection.

---

# 20. What I Would Change During a Production Incident

Suppose I discover:

```text
Service B latency increased
       ↓
Service A timeout rate increased
       ↓
Service A started retrying
       ↓
Traffic to B increased 5x
       ↓
B became even slower
```

I would not simply increase the timeout.

I would first stabilize the system.

Depending on the situation, I could:

1. Reduce or temporarily disable aggressive retries.
2. Enable/open circuit breakers for the unhealthy dependency.
3. Increase backoff and jitter.
4. Respect downstream rate limits.
5. Reduce concurrency toward the failing dependency.
6. Enable load shedding if necessary.
7. Protect critical traffic using bulkheads.
8. Move non-critical work to asynchronous processing.
9. Investigate why the downstream service is unhealthy.
10. Once the dependency recovers, allow traffic to ramp up gradually.

The priority during an incident is:

> **Stabilize first, diagnose second, optimize third.**

---

# Real-World Example 1: Product Catalog Service

Imagine an e-commerce system:

```text
User
 |
 v
Product API
 |
 v
Catalog Service
```

Catalog Service suddenly becomes slow.

Without retry protection:

```text
100,000 requests
      ↓
Catalog timeout
      ↓
100,000 retries
      ↓
Catalog gets 200,000 requests
      ↓
More timeouts
      ↓
200,000 retries
      ↓
System collapses
```

With proper controls:

```text
100,000 requests
      ↓
Catalog timeout
      ↓
Circuit Breaker
      ↓
Fail fast / cached response
```

For retryable requests:

```text
Exponential Backoff
        +
Jitter
        +
Retry Limit
        +
Rate Limit
```

The catalog service gets a chance to recover.

---

# Real-World Example 2: Payment Service

Suppose:

```text
Order Service → Payment Service
```

Payment Service becomes temporarily unavailable.

A dangerous strategy would be:

```text
Payment failed
   ↓
Retry immediately
   ↓
Retry immediately
   ↓
Retry immediately
```

Instead:

```text
Payment request
      |
      v
Idempotency Key
      |
      v
Payment Service
      |
      X timeout
      |
      v
Controlled retry
      |
      +---- Exponential backoff
      +---- Jitter
      +---- Maximum attempts
      +---- Circuit breaker
```

And importantly, if the payment provider's outcome is uncertain:

```text
UNKNOWN
```

should not automatically be interpreted as:

```text
FAILED
```

The system may need reconciliation before attempting another charge.

---

# Real-World Example 3: Kafka/Queue Consumer

Suppose a worker receives a message:

```text
InvoiceCreated
```

and downstream processing fails.

A bad implementation might continuously retry:

```text
Message
 ↓
Fail
 ↓
Retry immediately
 ↓
Fail
 ↓
Retry immediately
 ↓
Fail
 ↓
...
```

This can consume worker capacity and prevent healthy messages from being processed.

A better approach:

```text
Main Queue
    |
    v
Worker
    |
    X Failure
    |
    v
Retry Queue
    |
    | delay
    v
Worker
    |
    X Failure
    |
    v
Retry Queue
    |
    v
Dead Letter Queue
```

This gives the system controlled recovery instead of an infinite retry loop.

---

# A Practical Retry Policy

A reasonable starting point for a transient downstream failure might look conceptually like:

```text
Maximum Attempts:       3
Initial Backoff:        100 ms
Backoff Strategy:       Exponential
Jitter:                 Enabled
Maximum Backoff:        2 seconds
Total Request Deadline: 3 seconds
Circuit Breaker:        Enabled
Rate Limiting:          Enabled
```

These are **example values**, not universal production settings.

The correct values depend on:

* downstream SLA,
* request latency,
* traffic volume,
* business criticality,
* dependency capacity,
* failure characteristics,
* and whether the operation is synchronous or asynchronous.

---

# How I Would Approach This in Production

My debugging sequence would be:

```text
1. Detect traffic increase
        ↓
2. Compare original vs retry traffic
        ↓
3. Identify failing downstream
        ↓
4. Check retry configuration
        ↓
5. Check retryable status codes/errors
        ↓
6. Check retry count and timing
        ↓
7. Check whether multiple layers retry
        ↓
8. Check circuit breaker state
        ↓
9. Check rate limiting / 429 responses
        ↓
10. Stabilize with backoff/circuit breaking/load shedding
        ↓
11. Investigate downstream root cause
        ↓
12. Gradually restore traffic
```

The key metric I would immediately look for is:

```text
Original Traffic vs Retry Traffic
```

because it tells me whether the application is **amplifying the incident itself**.

---

# Interview-Specific Answer

If an interviewer asks:

> **"Retries are creating a traffic spike. How would you prevent a retry storm?"**

I would answer:

> "First, I would confirm whether the traffic increase is coming from retries and identify which downstream dependency is failing. I wouldn't just increase the timeout or allow unlimited retries. I would make the retry policy bounded and controlled — use exponential backoff with jitter, limit the maximum number of attempts and total retry time, and retry only transient and genuinely retryable failures.
>
> I would also use a circuit breaker so that when the downstream service is consistently unhealthy, we fail fast instead of continuously sending requests to it. Rate limiting, concurrency limits, and bulkheads can prevent retries from consuming all available resources. If the downstream returns 429, I would respect its rate limit or Retry-After guidance rather than immediately retrying.
>
> I'd also check whether multiple layers are independently retrying the same request because that can amplify traffic significantly. For asynchronous workloads, I would prefer delayed queue-based retries rather than immediate application-level retries.
>
> Finally, for state-changing operations such as payments or order creation, retries must be idempotent so that retrying doesn't create duplicate business operations. I would monitor retry rate, downstream latency, error rate, circuit-breaker state, and original-versus-retry traffic throughout the incident.
>
> So my overall approach is: **bounded retries + exponential backoff + jitter + circuit breaker + rate limiting + bulkheads + idempotency + strong observability.**"

---

# Key Interview Takeaway

The biggest mistake is thinking:

```text
Failure → Retry harder
```

A resilient system thinks:

```text
Failure
   ↓
Is it transient?
   ↓
Should I retry?
   ↓
Do I have retry budget?
   ↓
Wait using backoff
   ↓
Add jitter
   ↓
Check circuit breaker
   ↓
Respect rate limits
   ↓
Retry safely
   ↓
Stop after bounded attempts
```

The fundamental principle is:

> **Retries are a form of additional traffic. During a failure, uncontrolled retries can become more dangerous than the original failure.**

Therefore, retries should always be **bounded, delayed, randomized, selective, observable, and protected by circuit breakers/rate limits where appropriate.**
