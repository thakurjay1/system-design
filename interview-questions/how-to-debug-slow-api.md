# Your API Suddenly Becomes 5× Slower. How Do You Debug It?

If an API suddenly becomes 5× slower, I would not immediately assume that there is a problem with the application code or start increasing the number of servers. My first goal would be to understand **where the additional latency is coming from**.

For example, if the API normally responds in around 100 ms and suddenly starts taking 500 ms, somewhere in the request path an additional 400 ms is being introduced. My job is to find out where that 400 ms is being spent.

I would approach the problem step by step.

## 1. First, I would confirm that the problem is actually happening

I would first look at the monitoring and application metrics rather than relying only on someone saying that "the API is slow."

I would check the API's latency, especially **P50, P95 and P99**.

Suppose I see something like:

* P50 increased from 80 ms to 100 ms
* P95 increased from 150 ms to 400 ms
* P99 increased from 200 ms to 800 ms

That already gives me some information.

If only P99 is significantly affected, it may be a problem with a particular type of request or a small percentage of traffic. But if P50, P95 and P99 have all increased, then the problem is probably affecting the majority of requests.

I would also check the error rate because sometimes increased latency and increased failures are related.

## 2. Then I would understand the scope of the problem

I would ask myself whether **all APIs are slow or only one particular endpoint**.

For example:

```text
GET /users       → Slow
GET /products    → Normal
POST /orders     → Normal
```

If only `/users` is slow, I would focus on everything specific to that endpoint.

But if I see:

```text
/users       → Slow
/products    → Slow
/orders      → Slow
/payments    → Slow
```

then I would start looking at something shared by all those APIs.

That could be:

* Database
* Redis
* API Gateway
* Load Balancer
* Network
* Kubernetes infrastructure
* Shared microservice
* Connection pools
* External dependencies

This distinction is important because I don't want to spend an hour debugging one controller when the actual problem is a shared database.

## 3. I would check when the problem started

Next, I would look at the timeline.

If the API was healthy for several hours and suddenly became slow at 10:30 AM, I would check what changed around 10:30.

For example:

* Was there a new deployment?
* Was there a configuration change?
* Was a database migration performed?
* Did traffic suddenly increase?
* Was a new feature enabled?
* Did a downstream service become slow?
* Did Kubernetes scale up or down?
* Was a cache flushed?
* Did a database index change?

In production, knowing **what changed immediately before the incident** can significantly narrow down the investigation.

If a new version was deployed at 10:25 and latency increased at 10:30, that would be one of the first things I would investigate.

## 4. Then I would break down the request

At this point, I want to understand where the request is spending its time.

For a typical API, the request might look like:

```text
Client
   |
   v
Load Balancer
   |
   v
API Gateway
   |
   v
Application
   |
   +------> Redis
   |
   +------> Database
   |
   +------> Other Microservice
   |
   +------> External API
   |
   v
Response
```

The application might appear to be slow, but the actual problem could be somewhere downstream.

This is where **distributed tracing** becomes very useful.

For example, suppose a trace shows:

```text
API Gateway          10 ms
Order Service        30 ms
User Service         20 ms
Inventory Service   350 ms
Database             40 ms
Network               20 ms
--------------------------------
Total                ~470 ms
```

Now I have a strong indication that the Inventory Service is responsible for most of the additional latency.

Instead of randomly debugging everything, I can focus my investigation there.

## 5. I would check application health

Once I know the general area, I would check the application itself.

I would look at CPU, memory, garbage collection, thread pools and request queues.

For example, if CPU has suddenly gone from 40% to 95%, I would investigate why.

It could be because:

* Traffic increased
* A new piece of code is doing expensive processing
* A query is returning much more data
* Serialization has become expensive
* There is excessive logging
* Garbage collection is consuming CPU
* Some code is stuck in an inefficient loop

Similarly, if memory usage has increased significantly, I would check whether the application is experiencing excessive object creation, memory pressure or frequent GC pauses.

For a Java application, for example, I would specifically look at GC activity because frequent long GC pauses can directly contribute to increased request latency.

## 6. I would check thread pools and connection pools

This is something that can easily be missed.

An application can have plenty of CPU available and still be slow because requests are waiting for resources.

For example, suppose my application has a database connection pool of 100 connections, and all 100 are currently being used.

Now another 1,000 requests arrive.

Those requests may not even be executing database queries yet. They may simply be **waiting for a database connection**.

The same concept applies to:

* HTTP connection pools
* Database connection pools
* Thread pools
* Worker pools

So I would check whether requests are spending time waiting for resources rather than actually processing.

## 7. I would investigate the database

The database would be one of my major areas of investigation because database performance problems frequently appear as API latency problems.

I would check:

* Database CPU and memory
* Slow queries
* Query execution time
* Lock contention
* Deadlocks
* Connection pool usage
* Replication lag
* I/O
* Query execution plans
* Recent schema or index changes

For example, imagine an API normally executes a query in 40 ms.

After some database change, the same query takes 400 ms.

The API code hasn't changed, but the API has now become significantly slower.

One possible reason could be an index being removed or no longer being used.

Before:

```text
Index Scan
    |
    v
40 ms
```

After:

```text
Sequential Scan
    |
    v
400 ms
```

In this situation, adding more application servers would not solve the actual problem.

The database query needs to be fixed.

## 8. I would check Redis or other caches

If the application uses caching, I would check the cache hit ratio.

For example, suppose we normally have:

```text
Cache hit ratio = 95%
```

But suddenly it becomes:

```text
Cache hit ratio = 40%
```

That means many more requests are going to the database.

The resulting chain could be:

```text
Cache misses increase
        |
        v
Database traffic increases
        |
        v
Database becomes overloaded
        |
        v
Database latency increases
        |
        v
API latency increases
```

So sometimes the database problem is actually caused by a cache problem.

This is why I would avoid looking at individual components in isolation.

## 9. I would check external services

If the API calls an external service, I would also investigate that dependency.

For example:

```text
Our API
   |
   v
Payment Provider
```

Normally:

```text
Payment Provider response = 50 ms
```

But now:

```text
Payment Provider response = 400 ms
```

Our API will also become slower even though our own application code may be completely healthy.

I would therefore check downstream service latency, timeout rates, network connectivity and rate limiting.

I would also check whether our application is retrying failed requests.

Retries are particularly dangerous during incidents.

For example:

```text
Request
   |
   v
Service B
   |
 Timeout
   |
   v
Retry
   |
   v
Service B
   |
 Timeout
```

If thousands of requests are doing this, the retries themselves can overload Service B and create a **retry storm**.

## 10. I would check traffic and capacity

I would also compare the current traffic with normal traffic.

For example:

```text
Normal:
RPS = 1,000
CPU = 45%
Latency = 100 ms

Current:
RPS = 5,000
CPU = 95%
Latency = 500 ms
```

In this case, the system may simply be reaching its capacity.

Then scaling the application horizontally could be a reasonable mitigation, assuming the application tier is actually the bottleneck and the downstream systems can handle the additional traffic.

This is important:

> I would not scale blindly. I would first identify the constrained resource.

If the database is already overloaded, adding 20 more application instances could make things even worse because now more application instances are sending queries to the same database.

## 11. I would check recent deployments

If the problem started immediately after a deployment, I would compare the new version with the previous version.

For example:

```text
Version 1
    |
    | Latency = 100 ms
    |
    v
Deploy Version 2
    |
    | Latency = 500 ms
    |
    v
Investigation
```

I would check the code changes, configuration changes, feature flags and dependency changes.

If the evidence strongly points to the deployment, rolling back may be the safest immediate mitigation.

After restoring service, I would investigate the exact change that introduced the problem rather than considering the rollback the final solution.

## 12. If the application is running in Kubernetes

If the API is deployed in Kubernetes, I would also check:

* Number of healthy pods
* Pod restarts
* CPU throttling
* Memory limits
* HPA behavior
* Node resource availability
* Readiness and liveness failures
* Network issues

For example, suppose we normally have 10 healthy pods:

```text
10 pods
   |
   v
Normal traffic
```

But because of some infrastructure problem, only 4 pods are healthy:

```text
4 pods
   |
   v
Same traffic
   |
   v
More load per pod
   |
   v
Latency increases
```

That would be a completely different root cause from a slow database query.

## 13. I would use logs, metrics and traces together

I would not depend on only one observability mechanism.

I think about them like this:

**Metrics tell me that something is wrong.**

For example:

```text
P99 latency increased
```

**Logs help me understand what happened.**

For example:

```text
Database connection timeout
```

**Traces help me locate where the time was spent.**

For example:

```text
Database call = 420 ms
```

When all three point in the same direction, I have much stronger evidence for the root cause.

## 14. Finally, I would validate the root cause before fixing it

This is an important part of production debugging.

Suppose I believe the database is responsible.

I wouldn't simply say:

> "The database looks slow, so that's the problem."

I would validate it.

I might compare the slow query with a known-good execution plan, check database CPU, reproduce the query, inspect locks and verify whether fixing the suspected issue actually reduces API latency.

The goal is to move from:

```text
I think this is the problem
```

to:

```text
I have evidence that this is the problem
```

Only then would I apply the permanent fix.

---

# A Real-World Example

Imagine we have an e-commerce API:

```text
GET /products/{productId}
```

Normally:

```text
API latency = 100 ms
```

Suddenly:

```text
API latency = 500 ms
```

I start investigating.

First, I check the traffic.

Traffic hasn't changed significantly.

Then I check application CPU and memory.

They are also normal.

Next, I look at distributed traces and find:

```text
Application processing = 30 ms
Redis = 10 ms
Database = 430 ms
Network = 20 ms
```

So most of the latency is coming from the database.

I then check database metrics and discover that CPU utilization has increased significantly.

I inspect the slow query and its execution plan.

I find that a query which previously used an index is now performing a much more expensive scan.

After investigating recent database changes, I discover that an index was accidentally removed during a schema migration.

The chain is:

```text
Index removed
      ↓
Query becomes expensive
      ↓
Database CPU increases
      ↓
Query latency increases
      ↓
API latency increases
```

I restore the index and monitor the system.

The result is:

```text
Database latency:
430 ms → 40 ms

API latency:
500 ms → 100 ms
```

Now I know not only **what fixed the problem**, but also **why the API became slow in the first place**.

I would then add appropriate monitoring or deployment safeguards so that a similar database change is detected before it reaches production.

---

# Another Example: Cache Failure

Consider a product API that normally gets most product data from Redis.

Normally:

```text
Redis hit ratio = 95%
Database load = Normal
API latency = 100 ms
```

Suddenly Redis starts returning errors.

Now:

```text
Cache hits ↓
Database requests ↑
Database CPU ↑
Database latency ↑
API latency ↑
```

The initial symptom is:

> "API is slow."

The actual root cause is:

> "Cache failure caused a database overload."

This is why debugging distributed systems requires looking at the **entire request path**, not just the API service.

---

# How I Would Answer This in an Interview

If the interviewer expects a concise answer, I would say:

> "If an API suddenly becomes five times slower, I would first confirm the latency increase using metrics like P50, P95 and P99 and determine whether all endpoints are affected or only a particular endpoint. Then I would check when the problem started and correlate it with recent deployments, configuration changes, traffic spikes or infrastructure changes.
>
> After that, I would use distributed tracing to break down the request and identify where the additional latency is being introduced. I would check the application itself for CPU, memory, GC, thread-pool and connection-pool issues, and then investigate dependencies such as Redis, the database and external services.
>
> For the database, I'd look at slow queries, execution plans, locks, connection usage and resource utilization. For caching, I'd check the hit ratio because a cache problem can indirectly overload the database. I'd also check retries and timeouts because they can create cascading failures.
>
> Once I have a hypothesis, I would validate it with metrics and traces before making a change. Depending on the root cause, the mitigation could be rolling back a deployment, scaling the correct component, fixing a query or index, resolving a cache issue, or addressing a downstream dependency. After the fix, I'd continue monitoring to make sure latency has actually returned to normal and then work on preventing the same issue from happening again.
>
> The main thing is that I wouldn't immediately assume the API server itself is the problem. I would first identify where the additional latency is being introduced and then fix that specific bottleneck."

---

# Key Interview Takeaway

The most important mindset is:

> **Don't ask "Why is my API slow?" Ask "Where is the additional time being spent?"**

Then follow the request path:

```text
Client
  ↓
Gateway / Load Balancer
  ↓
Application
  ↓
Cache
  ↓
Database
  ↓
Other Services
  ↓
External Dependencies
```

And use:

```text
Metrics → What is wrong?
Logs    → What happened?
Traces  → Where is it happening?
```

Finally:

```text
Observe
   ↓
Narrow the scope
   ↓
Find the bottleneck
   ↓
Form a hypothesis
   ↓
Validate it
   ↓
Mitigate
   ↓
Fix permanently
   ↓
Prevent recurrence
```
