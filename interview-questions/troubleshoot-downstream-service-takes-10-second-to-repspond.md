# A Downstream Service Takes 10 Seconds to Respond. What Would You Do?

If a downstream service is taking **10 seconds to respond**, I would not simply increase my API timeout to 10 or 15 seconds.

My first question would be:

> **"Does my business operation actually need to wait for this downstream response?"**

A slow downstream service can quickly become a problem for my service because every request that is waiting for the downstream response is also consuming application resources.

For example:

```text
Client
   |
   v
My Service
   |
   | Request
   v
Downstream Service
   |
   |---- 10 seconds ----|
   |
   v
Response
```

If 1,000 users make requests simultaneously, I could potentially have 1,000 requests waiting for the same slow dependency.

That can eventually cause:

```text
Slow Downstream Service
        ↓
My requests wait
        ↓
Threads / connections remain occupied
        ↓
Resource utilization increases
        ↓
Request queue grows
        ↓
Latency increases
        ↓
Timeouts
        ↓
My service becomes unhealthy
```

So I would approach the problem systematically.

---

# 1. First, I Would Understand Why It Is Taking 10 Seconds

Before changing the architecture, I would find out whether the 10 seconds is actually expected.

I would check:

* Is the downstream service normally this slow?
* Did the latency suddenly increase?
* Is the problem affecting every request or only certain requests?
* Is the downstream service overloaded?
* Are only particular endpoints slow?
* Are there slow database queries behind that service?
* Is there network latency?
* Is the downstream service waiting on another dependency?

I would compare the current latency with the normal baseline.

For example:

```text
Normal:
Downstream latency = 200 ms

Current:
Downstream latency = 10 seconds
```

That strongly suggests something has changed.

But if:

```text
Normal:
Downstream latency = 8–12 seconds
```

then the question becomes more architectural:

> **"Why are we making a synchronous call to a service that naturally takes 10 seconds?"**

---

# 2. I Would Check Whether the Call Needs to Be Synchronous

This would be one of my first design decisions.

Suppose my API is:

```text
POST /generate-report
```

and generating the report takes 10 seconds.

There may be no reason to keep the user's HTTP request open for 10 seconds.

Instead, I could make the operation asynchronous:

```text
Client
   |
   | POST /generate-report
   v
My Service
   |
   | Create Job
   |
   v
Message Queue
   |
   v
Worker
   |
   v
Downstream Service
```

My API could immediately return:

```text
202 Accepted

Job ID: JOB123
```

The client can then check:

```text
GET /jobs/JOB123
```

or receive a notification when the job is completed.

This is often a much better design for long-running operations.

---

# 3. If It Must Be Synchronous, I Would Set a Timeout

If the request genuinely requires the downstream response, I would still avoid waiting indefinitely.

I would define a reasonable timeout based on the business requirement.

For example:

```text
Connect Timeout = 1 second
Read Timeout    = 2 seconds
```

The exact values depend on the service and expected latency.

The important point is:

> **My service should have control over how long it is willing to wait.**

Without a timeout, a dependency that stops responding can keep my requests waiting indefinitely.

For example:

```text
Request 1 → waiting
Request 2 → waiting
Request 3 → waiting
...
Request 10,000 → waiting
```

Eventually my own service can run out of resources.

---

# 4. I Would Use a Circuit Breaker

If the downstream service is consistently slow or failing, I would consider a **circuit breaker**.

The idea is that after enough failures or timeouts, my service temporarily stops sending requests to the unhealthy dependency.

Conceptually:

```text
             Normal
               |
               v
        Downstream Service
               |
          Repeated failures
               |
               v
          Circuit Opens
               |
               v
     Don't call downstream
               |
               v
      Return fallback/error
```

The circuit can later move to a half-open state and test whether the downstream service has recovered.

This prevents my service from continuously hammering an already unhealthy dependency.

---

# 5. I Would Add Retry Carefully

If the downstream request fails because of a temporary problem, a retry might help.

But I would **not automatically retry a request just because it took 10 seconds**.

Suppose the downstream service is overloaded.

If 10,000 requests timeout and my service retries all of them:

```text
10,000 original requests
        +
10,000 retries
        +
more retries
        ↓
20,000+ requests
        ↓
Downstream becomes even more overloaded
```

This can create a **retry storm**.

If retries are appropriate, I would use:

* Limited retry count
* Exponential backoff
* Jitter
* Retry only transient failures
* Idempotency for operations where duplicates could cause side effects

For example:

```text
Attempt 1 → fail
     ↓
Wait 200 ms
     ↓
Attempt 2 → fail
     ↓
Wait 500 ms
     ↓
Attempt 3 → fail
```

Not:

```text
Fail → immediately retry → fail → immediately retry → ...
```

---

# 6. I Would Check Whether the Operation Is Idempotent

This becomes especially important for writes.

Suppose I call:

```text
POST /payments
```

and the request takes 10 seconds.

If I don't receive a response, I cannot automatically assume the operation failed.

The downstream service might have successfully processed it, but the response might have been lost.

If I blindly retry:

```text
Request 1 → Payment processed
Request 1 → Timeout
Retry → Payment processed again
```

I could create a duplicate payment.

So for operations with side effects, I would make sure retries are safe using mechanisms such as:

```text
Idempotency Key
```

This is why timeout, retry and idempotency are closely related in distributed systems.

---

# 7. I Would Check Resource Consumption in My Service

A 10-second downstream call doesn't just increase latency.

It can consume resources in my application.

For example:

```text
100 requests
×
10 seconds waiting
```

means many requests can remain in-flight simultaneously.

I would check:

* Thread pool utilization
* Connection pool utilization
* HTTP client connection pool
* Request queue size
* CPU
* Memory
* Number of active requests
* Number of blocked/waiting threads

This is especially important in traditional thread-per-request architectures.

For example:

```text
100 worker threads
       |
       +---- Request waiting
       +---- Request waiting
       +---- Request waiting
       ...
       +---- Request waiting
```

If enough threads are blocked waiting for the downstream service, my service may stop accepting useful work even though CPU usage is relatively low.

---

# 8. I Would Look at Connection Pool Exhaustion

This is closely related to the previous system-design question.

Suppose my HTTP client has:

```text
Maximum connections = 100
```

and the downstream service takes 10 seconds.

If 100 requests are waiting for responses:

```text
100 connections occupied
       ↓
No connection available
       ↓
New requests wait
```

Now the downstream problem has become **my service's problem**.

So I would check both:

```text
Application Thread Pool
        +
HTTP Connection Pool
```

A slow dependency can exhaust either one depending on the application architecture.

---

# 9. I Would Check Whether I Can Cache the Response

If the downstream response doesn't change frequently, caching may eliminate unnecessary calls.

For example:

```text
My Service
   |
   v
Redis / Cache
   |
   +---- HIT → Return immediately
   |
   +---- MISS
            |
            v
      Downstream Service
```

Suppose the downstream service takes 10 seconds but the data only changes every few minutes.

Instead of making every request wait 10 seconds, I could cache the result.

For example:

```text
First request → 10 seconds
Next 1,000 requests → Cache response
```

Of course, caching is only appropriate when stale data is acceptable and the business requirement allows it.

---

# 10. I Would Check Whether Requests Can Be Batched

Sometimes the problem is caused by making many downstream calls.

For example:

```text
My Service
   |
   +---- Request A → Downstream
   +---- Request B → Downstream
   +---- Request C → Downstream
   +---- Request D → Downstream
```

If the downstream service supports batching, I might instead do:

```text
My Service
   |
   | One batch request
   v
Downstream Service
```

This can significantly reduce:

* Network overhead
* Number of connections
* Number of downstream requests
* Overall processing overhead

Again, this depends on whether the downstream API supports it.

---

# 11. I Would Check Whether I Can Parallelize Independent Calls

Suppose my service needs data from three independent downstream services.

If I call them sequentially:

```text
Service A → 2 sec
      ↓
Service B → 3 sec
      ↓
Service C → 5 sec

Total ≈ 10 sec
```

If they are independent, I may be able to call them concurrently:

```text
             +---- Service A → 2 sec
             |
My Service --+---- Service B → 3 sec
             |
             +---- Service C → 5 sec
```

Then the overall time can approach the slowest dependency rather than the sum of all three.

```text
Sequential ≈ 2 + 3 + 5 = 10 sec

Parallel ≈ max(2, 3, 5) = 5 sec
```

But I would only do this when the calls are genuinely independent and the additional concurrency is safe.

---

# 12. I Would Look for a Better API or Architecture

Sometimes the problem isn't performance.

The API itself may be poorly designed for the use case.

For example:

```text
GET /customer
```

might internally make:

```text
Customer Service
    ↓
Order Service
    ↓
Payment Service
    ↓
Recommendation Service
    ↓
Analytics Service
```

Now one slow dependency can make the entire request slow.

I would ask:

> **"Does this request really need all of this information before responding?"**

Maybe some information can be:

* Loaded asynchronously
* Cached
* Precomputed
* Fetched later
* Returned separately

This is where system design becomes more important than simply tuning timeout values.

---

# 13. I Would Consider a Fallback

If the downstream service isn't critical to the main operation, I might return a fallback.

For example:

```text
Product Service
      |
      +---- Inventory Service
      |
      +---- Recommendation Service
```

Suppose recommendations take 10 seconds.

I wouldn't necessarily make the entire product page wait for recommendations.

Instead:

```text
Product data → Return immediately
Recommendations → Load separately
```

Or:

```text
Recommendation Service unavailable
        ↓
Return cached recommendations
        ↓
Continue serving product page
```

This is an example of **graceful degradation**.

The key question is:

> **"What functionality can I safely sacrifice when a dependency is unhealthy?"**

---

# 14. I Would Use Bulkheads

Another useful resilience pattern is the **bulkhead pattern**.

The idea is to prevent one slow dependency from consuming all of my application's resources.

For example:

```text
My Service
   |
   +---- Dependency A → Pool A
   |
   +---- Dependency B → Pool B
   |
   +---- Dependency C → Pool C
```

Suppose Dependency B becomes extremely slow.

Without isolation:

```text
Dependency B
      ↓
Consumes all threads
      ↓
Other requests affected
```

With bulkheads:

```text
Dependency B
      ↓
Consumes only its allocated resources
      ↓
Other operations continue
```

This is especially useful when one service depends on many external systems.

---

# 15. I Would Monitor the Dependency

I would make sure we have enough observability to understand what is happening.

I would look at:

```text
Latency
Error rate
Timeout rate
Request rate
Throughput
Connection pool usage
```

And ideally distributed tracing:

```text
Client
  |
  v
Service A
  |
  | 120 ms
  v
Service B
  |
  | 9.7 sec
  v
Service C
```

Tracing makes it much easier to identify exactly where the 10 seconds is being spent.

I would also compare:

```text
p50
p95
p99
```

rather than relying only on average latency.

For example:

```text
Average = 500 ms
p99     = 10 sec
```

The average looks fine, but a significant tail-latency problem may still exist.

---

# 16. I Would Check Dependency Health Before Scaling My Service

A common reaction is:

> "Let's add more application instances."

But if the bottleneck is the downstream service, scaling my service might increase the number of requests hitting it.

For example:

```text
10 pods
    ↓
1,000 requests/sec
```

Scale to:

```text
50 pods
    ↓
Potentially more concurrent downstream calls
```

Now the downstream service could become even more overloaded.

So I would identify the bottleneck before scaling.

---

# Real-World Example 1: Report Generation

Suppose I have:

```text
POST /reports
```

The report generation service takes 10 seconds.

I would not keep the HTTP connection open for the entire operation if the business doesn't require synchronous behavior.

Instead:

```text
Client
   |
   | POST /reports
   v
Report API
   |
   | Create Job
   v
Queue
   |
   v
Worker
   |
   v
Report Service
```

API response:

```text
202 Accepted
Job ID = JOB123
```

The client can then query:

```text
GET /reports/jobs/JOB123
```

This makes the API much more resilient to long-running operations.

---

# Real-World Example 2: Payment Service

Suppose:

```text
Order Service
      |
      v
Payment Service
      |
      | 10 seconds
      v
Payment Provider
```

I wouldn't simply set:

```text
Timeout = 15 seconds
```

and call it solved.

I'd first understand why the provider is taking 10 seconds.

If the payment request times out, I also need to consider that the payment might already have succeeded.

So I would use:

```text
Idempotency Key
+
Payment State
+
Provider Reconciliation
+
Timeout
+
Controlled Retry
```

For example:

```text
PROCESSING
    |
    v
Provider call
    |
    X Timeout
    |
    v
UNKNOWN
    |
    v
Reconcile with provider
    |
    +---- SUCCESS
    |
    +---- FAILED
```

This avoids accidentally charging the customer twice.

---

# Real-World Example 3: Product Page

Suppose a product page requires:

```text
Product Service → 100 ms
Pricing Service → 200 ms
Recommendation Service → 10 sec
```

If recommendations aren't critical, I wouldn't make the entire page wait 10 seconds.

Instead:

```text
Product + Price
      |
      v
Return response
      |
      +---- Recommendations loaded separately
```

Or use:

```text
Cached recommendations
```

This provides a much better user experience.

---

# Real-World Example 4: Downstream Service Is Temporarily Down

Suppose:

```text
Service A
   |
   v
Service B
```

Service B starts timing out.

Without resilience:

```text
Service B slow
     ↓
Service A waits
     ↓
Threads exhausted
     ↓
Service A becomes slow
     ↓
Service A starts failing
```

With:

```text
Timeout
+
Circuit Breaker
+
Bulkhead
+
Fallback
```

we can contain the failure:

```text
Service B unhealthy
       ↓
Timeout
       ↓
Circuit opens
       ↓
Stop calling B temporarily
       ↓
Fallback / controlled error
       ↓
Service A remains available
```

This is the idea of **failure isolation**.

---

# My Production Debugging Approach

If this happened in production, I would roughly follow this sequence:

```text
Downstream latency = 10 sec
          |
          v
Is 10 sec normal?
          |
      +---+---+
      |       |
     NO      YES
      |       |
      |       +---- Reconsider synchronous design
      |
      v
Check downstream health
      |
      +---- CPU
      +---- Memory
      +---- DB
      +---- Locks
      +---- Network
      +---- Traffic
      |
      v
Check my service
      |
      +---- Thread pool
      +---- HTTP connection pool
      +---- Request queue
      +---- Memory
      |
      v
Check traces
      |
      v
Choose mitigation
      |
      +---- Timeout
      +---- Circuit Breaker
      +---- Controlled Retry
      +---- Cache
      +---- Async Processing
      +---- Fallback
      +---- Bulkhead
      +---- Parallelization
      +---- Batching
```

---

# Interview-Specific Answer

If the interviewer asks:

**"A downstream service takes 10 seconds to respond. What would you do?"**

I would answer:

> "First, I would find out whether the 10-second latency is expected or whether it has recently increased. I'd use metrics and distributed tracing to identify whether the delay is actually coming from the downstream service, its database, the network, or something else.
>
> Then I'd ask whether my API really needs to wait for that response. If it's a long-running operation such as report generation, I would prefer an asynchronous design using a queue or job-based approach instead of keeping the HTTP request open for 10 seconds.
>
> If the call has to remain synchronous, I'd configure a reasonable timeout rather than waiting indefinitely. I'd also consider a circuit breaker so that if the downstream service is consistently unhealthy, my service stops sending requests temporarily instead of exhausting its own resources.
>
> I would be careful with retries because retries against an already overloaded service can make the situation worse. If retries are required, I'd use limited retries with exponential backoff and jitter, and I'd make sure operations with side effects are idempotent.
>
> I'd also check my own thread pool and HTTP connection pool because a slow dependency can exhaust resources in my service even when CPU usage is normal. Depending on the use case, I could use caching, batching, parallel calls, bulkheads, or graceful degradation.
>
> So I wouldn't treat this simply as a timeout problem. I'd first identify the bottleneck, then decide whether the operation should be synchronous, and finally add appropriate resilience mechanisms so that one slow dependency doesn't bring down my entire service."

---

# Key Interview Takeaway

When you hear:

> **"Downstream service takes 10 seconds."**

Don't immediately say:

> "Increase the timeout."

Think:

```text
Slow Dependency
      |
      v
Do I really need to wait?
      |
   +--+--+
   |     |
  NO    YES
   |     |
   v     v
Async   Timeout
   |      |
Queue    Circuit Breaker
   |      |
Worker   Controlled Retry
          |
       Fallback
          |
       Bulkhead
```

And always remember:

```text
Slow downstream service
        ↓
Can consume my resources
        ↓
Can increase my latency
        ↓
Can exhaust threads/connections
        ↓
Can cause cascading failures
```

The goal is not just to make **one request** survive a 10-second dependency.

The goal is to make sure:

> **"One slow or failing dependency does not take down my entire system."**
