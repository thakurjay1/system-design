# The Same Payment Request Arrives Twice. How Do You Prevent Duplicate Payments?

This is a very important **distributed systems and payment-system design problem**.

If the same payment request reaches my payment service twice, I need to make sure that the customer is **charged only once**, even if the request is duplicated because of retries, network issues, client bugs, message redelivery, or multiple API calls.

The key concept I would use here is:

> **Idempotency.**

The basic idea is:

> **The same payment operation should produce the same final result, even if the request is processed multiple times.**

---

# 1. First, Understand Why Duplicate Requests Happen

Duplicate requests are actually quite common in distributed systems.

For example:

```text id="8y9m2k"
Customer
   |
   | POST /payments
   v
Payment Service
   |
   |---- Payment Provider
   |
   X Network timeout
```

The customer doesn't receive the response.

From the customer's perspective:

> "I don't know whether my payment succeeded."

So the client retries:

```text id="9v3r4x"
POST /payments
```

Now the server receives the same payment request again.

The dangerous situation is:

```text id="w5p6tc"
Request 1 → Charge ₹1,000 → SUCCESS
Request 2 → Charge ₹1,000 → SUCCESS
```

The customer has now been charged:

```text
₹2,000
```

instead of:

```text
₹1,000
```

This is exactly what we need to prevent.

---

# 2. Use an Idempotency Key

The most common solution is an **idempotency key**.

The client generates a unique key for a particular payment operation.

For example:

```text id="q7z5p1"
Idempotency-Key: 7f2a8c91-45e2-4a7e-9f23-123456789abc
```

The client sends the same key when retrying the same operation.

So:

```text id="4w8kq9"
Request 1:
Idempotency-Key = ABC123

Request 2:
Idempotency-Key = ABC123
```

The server understands:

> "These are attempts to perform the same operation."

---

# 3. Store the Idempotency Key

I would maintain an idempotency record in a persistent store.

For example:

```text id="2s8r5v"
payment_id        = PAY123
idempotency_key   = ABC123
user_id           = USER45
amount            = 1000
currency          = INR
status            = SUCCESS
provider_ref      = TXN789
```

The important part is that the idempotency key has a **unique constraint**.

For example:

```sql id="4h9c2a"
CREATE UNIQUE INDEX
ON payment_request(idempotency_key);
```

Now the database itself helps guarantee that the same operation isn't inserted multiple times.

---

# 4. Basic Request Flow

The payment flow could look like this:

```text id="j3k7p2"
Client
   |
   | POST /payments
   | Idempotency-Key = ABC123
   v
Payment Service
   |
   | Check idempotency key
   |
   +---- Doesn't exist
   |          |
   |          v
   |     Create payment record
   |          |
   |          v
   |     Process payment
   |          |
   |          v
   |        SUCCESS
   |          |
   |          v
   |     Store result
   |
   v
Return response
```

Now suppose the request arrives again:

```text id="9y2j4q"
Client
   |
   | Same Idempotency-Key = ABC123
   v
Payment Service
   |
   | Check idempotency key
   |
   v
Already exists
   |
   v
Return previous result
```

The second request does **not** create another payment.

---

# 5. Why Checking Redis Alone Is Not Enough

A common answer is:

> "I'll store the idempotency key in Redis."

Redis can definitely be useful, but I would be careful about making Redis the **only source of truth** for a financial transaction.

For example:

```text id="f2c7v8"
Request
   |
   v
Redis
   |
   v
Payment Provider
```

What happens if Redis loses the key?

The next retry could look like a completely new request.

```text id="6j8s1q"
Payment succeeded
      ↓
Redis key lost
      ↓
Retry arrives
      ↓
System thinks it's new
      ↓
Potential duplicate payment
```

For critical payment state, I would generally persist the idempotency information and payment state in a durable database.

Redis can still be used as a **fast lookup/cache**, but I wouldn't rely on an ephemeral cache as the only protection against duplicate financial operations.

---

# 6. The Race Condition Is Important

There is another subtle problem.

Suppose two identical requests arrive at exactly the same time.

```text id="k6x9s2"
Request A ----------------\
                           > Payment Service
Request B ----------------/
```

If both execute:

```text id="r5d1q7"
Check idempotency key
```

before either inserts it, both could see:

```text
Key doesn't exist
```

Then both might process the payment.

So this is dangerous:

```text id="b7w4k3"
Request A → Check → Doesn't exist
Request B → Check → Doesn't exist
Request A → Charge
Request B → Charge
```

Therefore:

> **The check and creation of the idempotency record need concurrency-safe handling.**

A unique database constraint is extremely useful here.

For example:

```sql id="5c8n2r"
INSERT INTO payment_request (
    idempotency_key,
    status
)
VALUES ('ABC123', 'PROCESSING');
```

with:

```sql id="0s3k5m"
UNIQUE(idempotency_key)
```

Only one request can successfully create the record.

The other request gets a duplicate-key/unique-constraint result and knows that another request is already processing or has processed that operation.

---

# 7. Don't Just Store the Key — Store the Result

A good idempotency implementation should store enough information to reproduce the result.

For example:

```text id="x3q9p7"
Idempotency Key: ABC123

Payment ID: PAY123
Amount: ₹1,000
Currency: INR
Status: SUCCESS
Provider Reference: TXN789
Response: Payment successful
```

Now if the client retries:

```text id="h4v8s2"
Request
   |
   v
ABC123
   |
   v
Existing payment found
   |
   v
Return stored result
```

We don't need to charge the customer again.

---

# 8. What If the First Request Is Still Processing?

This is an important edge case.

Suppose:

```text id="m9k4r2"
Request A
   |
   v
Payment processing...
```

At exactly the same time:

```text id="p2x8c6"
Request B
   |
   v
Same idempotency key
```

The payment isn't completed yet.

What should Request B do?

One approach is to return something like:

```text id="w6f3z1"
Payment is currently being processed.
```

or:

```text
HTTP 202 Accepted
```

depending on the API contract.

The system can also allow the second request to wait briefly, then return the current status.

The important point is:

> **Request B must not initiate another payment.**

---

# 9. Payment State Machine

For a robust payment system, I would maintain explicit payment states.

For example:

```text id="j8q2p5"
CREATED
   |
   v
PROCESSING
   |
   +------> SUCCESS
   |
   +------> FAILED
   |
   +------> UNKNOWN
```

`UNKNOWN` is particularly important.

Why?

Because distributed systems can fail at awkward points.

Consider:

```text id="r8x5m1"
Our Service
     |
     | Charge request
     v
Payment Provider
     |
     | Payment succeeds
     |
     X Response lost
     |
Our Service
```

Our service doesn't know whether the payment succeeded.

If we simply mark it as `FAILED` and retry the payment, we could accidentally charge the customer twice.

So I would distinguish:

```text
FAILED
```

from:

```text
UNKNOWN
```

An `UNKNOWN` state means:

> "We don't currently know the final result."

We should reconcile the payment with the provider before initiating another charge.

---

# 10. The Most Dangerous Failure Scenario

This is a very common interview discussion.

Imagine:

```text id="5g2v8q"
Client
   |
   v
Payment Service
   |
   | Charge ₹1,000
   v
Payment Provider
   |
   | SUCCESS
   |
   X Response lost
   |
Payment Service
```

Our service never receives the success response.

From our perspective:

```text
Payment = UNKNOWN
```

The client retries.

If we blindly send another charge:

```text id="p7s3d9"
Retry
  |
  v
Payment Provider
  |
  v
Another ₹1,000 charge
```

Now we have a duplicate payment.

The correct approach is to use the same idempotency key with the payment provider if the provider supports it.

For example:

```text id="x4n8q2"
Our Idempotency Key
        |
        v
Payment Provider
        |
        v
Same operation
```

Then the provider can also recognize the duplicate request and return the original result.

This gives us protection across both sides:

```text
Client
   ↓
Our Payment Service
   ↓
Payment Provider
```

---

# 11. Idempotency Should Exist at the Payment Provider Too

This is a very important distinction.

Suppose our service is idempotent:

```text id="u5c2r7"
Client
   ↓
Our Service
   ↓
Database
```

But when we call the payment provider, the provider doesn't support idempotency.

We can still have a difficult failure case.

For example:

```text id="k9x4s3"
Our Service
   |
   | Charge request
   v
Payment Provider
   |
   | SUCCESS
   |
   X Network failure
   |
Our Service
```

We don't know whether the provider charged the customer.

Therefore, if possible, I would pass a stable idempotency key to the payment provider as well.

For example:

```text id="v3j6q8"
Client
   |
   | Idempotency-Key = ABC123
   v
Our Payment Service
   |
   | Idempotency-Key = ABC123
   v
Payment Provider
```

Now the same operation can be recognized at both layers.

---

# 12. Database Transaction Boundaries Matter

Another common mistake is thinking:

> "I'll put the database insert and payment-provider call inside one transaction."

That doesn't solve the distributed transaction problem.

For example:

```text id="d5r8p2"
BEGIN DB TRANSACTION

Insert payment record

Call Payment Provider
       |
       X
   Network failure

ROLLBACK
```

The database transaction can roll back our database changes.

But the external payment provider may already have processed the payment.

So:

```text id="q6m2x9"
Database → Rolled back
Payment Provider → Payment succeeded
```

We now have inconsistent state.

This is why payment systems require careful handling of distributed operations rather than assuming a local database transaction can cover everything.

---

# 13. Use a Durable Payment Record

I would create a payment record before attempting the actual payment.

For example:

```text id="s4j7n3"
Payment
------------------------------
payment_id       = PAY123
idempotency_key  = ABC123
order_id         = ORD456
amount           = 1000
status           = CREATED
```

Then:

```text id="2k8m5q"
CREATED
   ↓
PROCESSING
   ↓
SUCCESS / FAILED / UNKNOWN
```

This gives us something durable to reconcile against.

---

# 14. What If the Client Sends the Same Key With Different Data?

This is another important edge case.

Suppose:

```text id="q2w8k5"
First request:

Idempotency-Key = ABC123
Amount = ₹1,000
```

Then someone sends:

```text id="g4n7p1"
Second request:

Idempotency-Key = ABC123
Amount = ₹5,000
```

We should **not** treat these as the same request.

The idempotency key should represent one specific operation.

So I would store a request fingerprint or relevant request parameters.

For example:

```text id="t6r3m9"
Idempotency-Key = ABC123
Amount = 1000
Currency = INR
Order = ORD456
```

On a retry, the request should match the original operation.

If the same key is used with different important parameters, I would reject it as an invalid reuse.

This prevents accidental or malicious misuse of idempotency keys.

---

# 15. How Long Should We Keep the Idempotency Key?

We also need to decide the retention period.

We shouldn't necessarily keep every idempotency record forever.

For example:

```text id="n7k4s2"
Idempotency Record
      |
      v
Retention period
      |
      v
Archive/Delete
```

The correct period depends on:

* Payment lifecycle
* Client retry behavior
* Provider behavior
* Business requirements
* Compliance requirements
* Reconciliation requirements

For payment systems, I would generally be conservative because financial operations are much more sensitive than ordinary API requests.

---

# 16. Idempotency vs Duplicate Detection

These are related but slightly different.

**Duplicate detection** asks:

> "Have I seen this request before?"

**Idempotency** asks:

> "If I see this request again, can I safely return the same result without performing the operation again?"

For payments, we need both.

---

# 17. Idempotency Key vs Payment ID

I would also distinguish between:

```text
Order ID
Payment ID
Idempotency Key
Provider Transaction ID
```

They may represent different concepts.

For example:

```text id="c8m2r7"
Order ID
ORD-1001

Payment ID
PAY-5001

Idempotency Key
ABC123

Provider Transaction ID
TXN-9001
```

A retry of the same payment attempt should generally reuse the same idempotency key.

A completely new payment attempt may have a different idempotency key.

This distinction becomes important when dealing with retries, refunds, partial payments and multiple payment attempts for the same order.

---

# Real-World Example 1: Customer Clicks Pay Twice

Suppose the customer clicks the **Pay** button twice.

The frontend sends:

```text id="8k3v5q"
Request 1 → Idempotency-Key = ABC123
Request 2 → Idempotency-Key = ABC123
```

Our service receives both.

First request:

```text id="m2n7x4"
ABC123 → Doesn't exist
       ↓
Create payment
       ↓
Process
       ↓
SUCCESS
```

Second request:

```text id="p6q9s1"
ABC123 → Already exists
       ↓
Return existing result
```

Customer gets:

```text
Payment successful
```

but is charged only once.

---

# Real-World Example 2: Network Timeout

This is even more important.

```text id="z5c8n2"
Client
  |
  | Payment request
  v
Payment Service
  |
  | Charge
  v
Payment Provider
  |
  | SUCCESS
  |
  X Response lost
```

Client sees:

```text
Timeout
```

So it retries with the same key:

```text id="r4k7p3"
Idempotency-Key = ABC123
```

Our service checks the payment record.

If the status is already:

```text id="1h8s5m"
SUCCESS
```

we return the previous result.

If it is:

```text
UNKNOWN
```

we reconcile with the provider before trying another charge.

We **do not blindly charge again**.

---

# Real-World Example 3: Two Requests Arrive Simultaneously

```text id="u7m3k9"
Request A ----------------\
                           \
                            → Payment Service
                           /
Request B ----------------/
```

Both have:

```text
Idempotency-Key = ABC123
```

Both try to create the payment.

The database has:

```sql id="v8x2m6"
UNIQUE(idempotency_key)
```

Only one succeeds.

```text id="g4r9p2"
Request A → INSERT → SUCCESS
Request B → INSERT → DUPLICATE
```

Request B then checks the existing payment record rather than initiating another payment.

This prevents a race condition.

---

# Recommended High-Level Architecture

A simplified payment architecture could look like:

```text id="s9k2m4"
                Client
                   |
                   | Idempotency-Key
                   v
             API Gateway
                   |
                   v
            Payment Service
                   |
          +--------+--------+
          |                 |
          v                 v
    Payment Database       Redis
          |
          | Durable state
          v
    Payment Processing
          |
          v
   Payment Provider
          |
          v
   Provider Transaction
```

Redis can help with fast lookups and temporary coordination, but the durable payment record should be maintained in a reliable persistent store.

For asynchronous architectures, we could also introduce Kafka/RabbitMQ or another messaging system, but the same idempotency principle still applies because messages can be delivered more than once.

---

# What About Message Queues?

Suppose we process payments asynchronously:

```text id="f8m2c7"
Payment API
    |
    v
Message Queue
    |
    v
Payment Worker
```

The queue may deliver the same message more than once.

For example:

```text id="d3p7x1"
Message 123
   ↓
Worker processes payment
   ↓
Worker crashes before acknowledging
   ↓
Queue redelivers Message 123
```

Now the worker sees the same payment operation again.

Therefore, asynchronous processing also needs idempotency.

The worker can use:

```text id="b5n8q4"
Payment ID / Idempotency Key
```

to determine whether the operation has already been processed.

This is one reason I would not say:

> "Kafka guarantees exactly-once, so duplicate payments aren't possible."

Even when a messaging system provides strong processing guarantees in some parts of the pipeline, the complete end-to-end payment operation involves external systems and failure boundaries.

For financial operations, I still design explicit idempotency and reconciliation.

---

# What About Exactly-Once Processing?

This is a common interview trap.

People often say:

> "I'll use exactly-once processing."

In distributed systems, I would be careful with that statement.

There are different meanings of exactly-once:

```text id="m8q3v6"
Exactly-once message processing
            ≠
Exactly-once business effect
```

Even if my internal system processes a message exactly once, the external payment provider may receive a request, process it, and then our response may be lost.

Therefore, for payments, I focus on:

> **Exactly-once business effect**, achieved through idempotency, durable state, provider-side idempotency where available, and reconciliation.

---

# Interview-Specific Answer

If the interviewer asks:

**"The same payment request arrives twice. How do you prevent duplicate payments?"**

I would answer:

> "I would make the payment API idempotent. The client should send a unique idempotency key for each payment operation, and the same key should be reused when the client retries that operation.
>
> On the server side, I would persist the idempotency key along with the payment record and put a unique constraint on it. When the first request arrives, we create the payment record and process the payment. If the same key arrives again, we don't create another payment; we return the existing payment result or the current processing status.
>
> I would also handle the race condition where two identical requests arrive at exactly the same time. I wouldn't rely only on a check-then-insert operation because both requests could see that the key doesn't exist. A database unique constraint or another atomic mechanism should guarantee that only one request creates the payment operation.
>
> Another important case is when we send the payment to an external provider, the provider successfully charges the customer, but our response is lost. In that situation, we must not blindly retry the charge. I'd use the same idempotency key with the payment provider if it supports idempotency, or reconcile the payment status before retrying.
>
> I'd also maintain explicit payment states such as CREATED, PROCESSING, SUCCESS, FAILED and potentially UNKNOWN, because a network failure doesn't necessarily mean that the payment failed.
>
> So the main idea is: use a stable idempotency key, persist it durably, enforce uniqueness, make concurrent requests safe, and handle uncertain provider responses through idempotency and reconciliation."

---

# Key Interview Takeaway

Remember this flow:

```text id="p3x7k9"
Payment Request
      |
      v
Generate Idempotency Key
      |
      v
Check/Create Payment Record
      |
      +---- Already exists?
      |          |
      |          +---- YES → Return existing result/status
      |
      +---- NO
           |
           v
       PROCESSING
           |
           v
    Payment Provider
           |
       +---+---+
       |       |
    SUCCESS   UNKNOWN
       |       |
       v       v
    SUCCESS  Reconcile
```

The most important rule is:

> **Never assume that a timeout means the payment failed.**

And the strongest interview statement is:

> **"For payments, I don't want just duplicate-request protection; I want duplicate-business-effect protection."**

That means even if the same request is delivered twice, retried five times, or the network fails after the provider processes it, the customer should still experience **one payment, not multiple charges**.

### Quick Revision

```text id="c6m8r1"
Duplicate Payment Problem
        |
        v
Idempotency
        |
        +---- Unique Idempotency Key
        |
        +---- Durable Payment Record
        |
        +---- Unique DB Constraint
        |
        +---- Safe Concurrent Insert
        |
        +---- Store Previous Result
        |
        +---- Provider Idempotency
        |
        +---- UNKNOWN State
        |
        +---- Reconciliation
        |
        v
One Business Operation
→ One Payment Effect
```
