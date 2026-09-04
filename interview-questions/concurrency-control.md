# Two Users Update the Same Record at the Same Time. How Do You Handle It?

This is a classic **concurrency problem** in system design.

Suppose we have a product record:

```text
Product ID: 101
Stock: 10
```

Now two users open the same product at almost the same time.

Both see:

```text
Stock = 10
```

User A changes it to:

```text
Stock = 8
```

User B changes it to:

```text
Stock = 5
```

If both updates are processed independently, the final value may depend on which request reaches the database last.

That means one user's update can potentially **overwrite another user's update**.

The first thing I would decide is:

> **What should happen if two users modify the same record concurrently?**

The answer depends on the business requirement.

In general, I would handle this using **concurrency control**. The two most common approaches are **optimistic locking** and **pessimistic locking**.

---

# 1. The Basic Problem — Lost Update

Let's understand the problem first.

Suppose the database contains:

```text
Employee
----------------
id = 101
salary = 50,000
```

Both users read the record at nearly the same time.

```text
             Database
                 |
        salary = 50,000
             /       \
            /         \
       User A        User B
          |             |
      reads 50K      reads 50K
          |             |
      changes to      changes to
        55K             60K
          |             |
          +------┬------+
                 |
              Database
```

If User A updates first:

```text
salary = 55,000
```

Then User B updates:

```text
salary = 60,000
```

The final value is:

```text
60,000
```

User A's update has effectively disappeared.

This is called a **lost update**.

The system accepted both operations, but one user's change was silently overwritten.

---

# 2. First Question: Should Both Updates Be Allowed?

This is where the business requirement becomes important.

There are different scenarios.

For example, if two users are editing a document, we might want conflict detection.

If two users are updating an account balance, simply allowing the last update to win could be dangerous.

If two users are modifying a product description, last-write-wins might be acceptable depending on the application.

So before choosing a technical solution, I would ask:

> **"What is the expected business behavior when concurrent updates happen?"**

Once that is clear, I can choose the appropriate concurrency strategy.

---

# 3. Optimistic Locking

For most normal CRUD-style applications, **optimistic locking** is often a good starting point.

The idea is simple:

> **Assume conflicts are rare, but detect them when they happen.**

We can add a version number to the record.

For example:

```text
Product
-------------------------
id       = 101
name     = Laptop
price    = 50,000
version  = 5
```

Both users read:

```text
version = 5
```

Now User A updates the product.

The update looks conceptually like:

```sql
UPDATE product
SET price = 55_000,
    version = 6
WHERE id = 101
  AND version = 5;
```

The database finds version 5 and updates the record.

Now:

```text
version = 6
```

User B still has the old version:

```text
version = 5
```

When User B tries:

```sql
UPDATE product
SET price = 60_000,
    version = 6
WHERE id = 101
  AND version = 5;
```

the condition no longer matches.

So:

```text
Rows updated = 0
```

That tells the application:

> **"Someone else modified this record after you read it."**

The application can then return a conflict response, typically something like:

```text
HTTP 409 Conflict
```

depending on the API design.

---

# 4. Why Versioning Works

The important part is this condition:

```sql
WHERE id = 101
AND version = 5
```

We're essentially saying:

> "Update this record only if nobody has modified it since I read version 5."

The complete flow is:

```text
User A                         Database
  |                               |
  |---- Read version 5 ---------->|
  |<--- Record + version 5 -------|
  |                               |
  |---- Update WHERE version=5 -->|
  |                               |
  |<--- Success ------------------|
  |                               |
  |                         version = 6


User B
  |
  |---- Read version 5
  |
  |---- Update WHERE version=5
  |
  |        No matching row
  |
  <-------- Conflict
```

This prevents User B from silently overwriting User A's change.

---

# 5. Optimistic Locking with Java / JPA

In a Java application using JPA/Hibernate, this can be implemented very naturally with a version field.

For example:

```java
@Entity
public class Product {

    @Id
    private Long id;

    private String name;

    private BigDecimal price;

    @Version
    private Long version;
}
```

Hibernate uses the version field to detect concurrent modifications.

Conceptually, the generated SQL behaves like:

```sql
UPDATE product
SET price = ?,
    version = ?
WHERE id = ?
  AND version = ?;
```

If another transaction has already changed the record, the version no longer matches.

Hibernate can then report an optimistic locking failure.

The application can translate that into an appropriate business/API response.

---

# 6. What Should the User See?

Suppose User A and User B are editing the same product.

User A saves first.

User B then tries to save an older version.

Instead of silently overwriting User A's changes, the application could respond:

```text
The record was modified by another user.
Please refresh the record and review the latest changes.
```

For an API, something like:

```text
HTTP 409 Conflict
```

could be appropriate.

The frontend can then:

1. Refresh the latest record
2. Show the user what changed
3. Allow them to review
4. Let them submit their changes again

The exact behavior depends on the business requirement.

---

# 7. Pessimistic Locking

The second major approach is **pessimistic locking**.

Here we assume:

> **"Conflicts are likely, so let's lock the record while we're working with it."**

For example:

```sql
SELECT *
FROM product
WHERE id = 101
FOR UPDATE;
```

The database places a lock on the selected row.

Conceptually:

```text
User A
   |
   | SELECT ... FOR UPDATE
   v
Database
   |
   | Row locked
   |
   v
User A modifies record

User B
   |
   | Tries to update same row
   |
   v
WAIT
```

User B may have to wait until User A's transaction completes.

Once User A commits:

```text
Lock released
```

User B can continue depending on the transaction and database behavior.

---

# 8. Optimistic vs Pessimistic Locking

The difference is important for interviews.

| Optimistic Locking               | Pessimistic Locking                          |
| -------------------------------- | -------------------------------------------- |
| Assumes conflicts are rare       | Assumes conflicts are likely                 |
| Doesn't lock during normal reads | Locks resources                              |
| Detects conflict during update   | Prevents/serializes conflicting access       |
| Better concurrency               | Less concurrency                             |
| Usually simpler to scale         | Can create blocking                          |
| Good for CRUD/editing scenarios  | Good for high-contention critical operations |
| Conflict may require retry       | Requests may wait                            |

A simple way to remember it:

```text
Optimistic:
"Let them work and detect the conflict."

Pessimistic:
"Prevent them from conflicting in the first place."
```

---

# 9. When Would I Choose Optimistic Locking?

I would generally prefer optimistic locking when:

* Concurrent updates are relatively uncommon
* We don't want users waiting for locks
* The operation is relatively short
* The system needs good concurrency
* We can safely ask the user/service to retry
* We are building typical CRUD/editing functionality

For example:

```text
Employee profile
Product details
Customer information
Document metadata
User preferences
Configuration records
```

If two administrators rarely edit the same record at exactly the same time, optimistic locking is usually a good fit.

---

# 10. When Would I Choose Pessimistic Locking?

I would consider pessimistic locking when the resource is highly contested and allowing concurrent modifications could cause serious problems.

For example:

```text
Limited inventory
Bank account operations
Critical financial records
Seat booking
Certain reservation systems
```

Suppose only one seat remains:

```text
Flight
Seat A12
Available = 1
```

Two users attempt to book it simultaneously.

We don't want both users to successfully reserve it.

A transaction with appropriate locking can ensure that one transaction obtains the necessary lock and changes the availability before another transaction can make the same decision.

However, I would still carefully consider transaction duration and database contention.

---

# 11. Database Isolation Levels Also Matter

Concurrency isn't only about locks.

Database **transaction isolation levels** determine what one transaction can see while other transactions are modifying data.

Common isolation levels include:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

The exact behavior varies somewhat by database implementation.

Higher isolation generally provides stronger consistency guarantees but can reduce concurrency and increase contention.

For system design, I wouldn't automatically choose the strongest isolation level.

I'd choose the **minimum level that satisfies the business correctness requirement**.

---

# 12. What About Last Write Wins?

Another possible strategy is simply:

> **"The latest update wins."**

For example:

```text
User A → Update → 10:00:01
User B → Update → 10:00:02

Final value = User B's value
```

This is simple, but it can cause lost updates.

It can be acceptable in some systems where overwriting an older value is expected behavior.

But I would not use it blindly for important business data.

For example:

```text
User A changes:
Address = Delhi

User B changes:
Phone = 9999999999
```

If the application sends the entire object during every update, User B's old copy could potentially overwrite User A's address change.

This is one reason **partial updates** and proper concurrency control can be important.

---

# 13. Compare-and-Set

Optimistic locking is closely related to the idea of **compare-and-set**.

The concept is:

```text
Update the record
ONLY IF
the value/version is still what I originally observed.
```

For example:

```sql
UPDATE account
SET balance = 900,
    version = 8
WHERE id = 123
AND version = 7;
```

If one row is updated:

```text
Success
```

If zero rows are updated:

```text
Conflict
```

This pattern is widely useful in distributed systems.

---

# 14. Distributed Systems Make This More Interesting

Now imagine we don't have one application server.

We have:

```text
             Load Balancer
                  |
        +---------+---------+
        |         |         |
        v         v         v
      Pod A     Pod B     Pod C
        |         |         |
        +---------+---------+
                  |
                  v
              Database
```

User A's request might go to Pod A.

User B's request might go to Pod C.

So an in-memory lock inside Pod A is **not enough**.

For example, this is dangerous:

```java
synchronized(productId) {
    updateProduct();
}
```

because the other request could arrive at another application instance.

```text
User A → Pod A → Lock A

User B → Pod B → Lock B
```

They are two different JVMs.

Therefore, for distributed systems, concurrency control usually needs to be coordinated through a shared mechanism such as:

* Database locking
* Optimistic locking using a database version
* Distributed locking where genuinely required
* Atomic operations in a suitable data store

This is a very important interview point.

---

# 15. What About Distributed Locks?

Sometimes people immediately suggest Redis distributed locks.

I would be careful with that answer.

If the resource already lives in a relational database, I would first ask whether the database's own transactional and locking mechanisms are sufficient.

A distributed lock adds complexity:

```text
Application
    |
    v
Distributed Lock
    |
    v
Database
```

Now I have to think about:

* Lock expiration
* Client failures
* Network partitions
* Ownership
* Lock renewal
* Fencing
* Deadlocks
* What happens if the process crashes

So I wouldn't introduce a distributed lock unless the problem actually requires one.

---

# 16. Another Important Case: Atomic Updates

Sometimes we don't even need to read the value first.

Suppose we have inventory:

```text
stock = 10
```

Two users want to buy one item each.

Instead of:

```text
Read stock
Calculate stock - 1
Write stock
```

we can sometimes perform an atomic conditional update:

```sql
UPDATE product
SET stock = stock - 1
WHERE id = 101
AND stock > 0;
```

Then check the number of affected rows.

If:

```text
Rows affected = 1
```

the purchase can proceed.

If:

```text
Rows affected = 0
```

the item was unavailable.

This is often better than doing the read and write as two independent operations because the condition is evaluated atomically by the database.

---

# Real-World Example 1: Editing a Customer Profile

Suppose two support agents open the same customer profile.

Current record:

```text
Name    = Jay
Phone   = 12345
Version = 10
```

Agent A changes the phone number.

Agent B changes the name.

Both originally loaded version 10.

Agent A saves:

```text
Version 10 → Version 11
```

Agent B then tries to save version 10.

The update fails because the database is now at version 11.

Instead of silently overwriting Agent A's change, the system tells Agent B:

```text
"This customer record was modified by another user.
Please refresh before saving."
```

This is a good use case for optimistic locking.

---

# Real-World Example 2: Seat Booking

Imagine a concert has only one seat remaining:

```text
Seat A10
Status = AVAILABLE
```

Two users click **Book** at almost exactly the same time.

We cannot allow:

```text
User A → Success
User B → Success
```

because one seat cannot belong to two users.

We need a concurrency mechanism that guarantees only one successful reservation.

Depending on the architecture, we could use:

* Database transaction + appropriate locking
* Atomic conditional update
* Reservation state with expiration
* Carefully designed distributed coordination

For example:

```sql
UPDATE seats
SET status = 'RESERVED'
WHERE seat_id = 10
AND status = 'AVAILABLE';
```

If the update affects one row:

```text
Reservation successful
```

If it affects zero rows:

```text
Someone else already reserved it
```

This is a very strong pattern because the database performs the state transition atomically.

---

# Real-World Example 3: Bank Account

Suppose an account has:

```text
Balance = ₹10,000
```

Two withdrawal requests arrive simultaneously:

```text
Request A → Withdraw ₹7,000
Request B → Withdraw ₹6,000
```

If both simply read ₹10,000 before either update happens, we could end up with an incorrect result.

For financial operations, I would not rely on simple last-write-wins behavior.

I would use a proper transactional design with concurrency control so that the balance cannot be incorrectly modified.

For example, an atomic conditional operation could ensure that the balance is sufficient before the withdrawal occurs.

The exact implementation depends on the banking requirements, transaction model and database.

---

# How I Would Approach This in Production

If I encountered concurrent updates in a production system, I would first understand the business requirement.

I'd ask:

> "Should both updates be preserved, should one update win, or should the second update be rejected?"

Then I'd identify the type of operation.

For a normal CRUD operation where conflicts are relatively rare, I'd usually start with **optimistic locking**.

For highly contested resources where correctness requires serialization, I'd consider **pessimistic locking or an atomic database operation**.

Then I'd make sure the concurrency control works across all application instances.

I would also consider:

* Transaction boundaries
* Isolation level
* Retry behavior
* Deadlock handling
* Lock duration
* Database capacity
* User experience when conflicts occur

The important thing is that the concurrency strategy should be driven by the **business requirement**, not simply by choosing the most complicated locking mechanism.

---

# Interview-Specific Answer

If the interviewer asks me:

**"Two users update the same record at the same time. How do you handle it?"**

I would answer:

> "I would first clarify what the business expects when two concurrent updates happen. We need to prevent a situation where one user's update is silently lost.
>
> For a typical CRUD application where conflicts are relatively rare, I would generally use optimistic locking. I would add a version field to the record. When the user reads the record, they also get its current version. During the update, I would update the record only if the version is still the same. If another user has already modified it, the version won't match, the update will affect zero rows, and I can return a conflict such as HTTP 409 rather than overwriting the other user's changes.
>
> If the resource is highly contested and we need to serialize access, I would consider pessimistic locking or an atomic database operation. For example, in seat booking, instead of reading the seat and then updating it separately, I could atomically change the seat from AVAILABLE to RESERVED only if it is still available.
>
> I would also make sure the solution works across multiple application instances. An in-memory lock inside one application server isn't sufficient in a distributed system because concurrent requests can reach different servers.
>
> So my choice would depend on the use case: optimistic locking for detecting relatively rare conflicts, and pessimistic locking or atomic operations when we need stronger serialization for highly contested resources."

---

# Key Interview Takeaway

The core problem is:

```text
Two users
    |
    +---- Read same version
    |
    +---- Modify independently
    |
    v
Potential Lost Update
```

The solution is to introduce **concurrency control**.

The three important approaches to remember are:

```text
1. Optimistic Locking
   → Detect conflicts

2. Pessimistic Locking
   → Prevent/serialize conflicts

3. Atomic Operations
   → Make critical state transitions indivisible
```

A simple decision model is:

```text
                 Concurrent Update
                        |
                        v
              What does business need?
                        |
          +-------------+-------------+
          |                           |
     Conflicts rare             High contention
          |                           |
          v                           v
 Optimistic Locking          Pessimistic /
                              Atomic Operation
```

And the most important interview sentence is:

> **"I wouldn't simply use last-write-wins for every system. First I'd understand whether concurrent updates are acceptable, whether conflicts need to be detected, and what level of consistency the business operation requires."**

