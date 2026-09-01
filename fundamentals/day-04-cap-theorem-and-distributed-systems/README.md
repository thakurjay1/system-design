CAP is not “pick any two.” In a real distributed system, network partitions are a failure condition you generally must tolerate.
Therefore, when a partition happens, the practical trade-off is primarily between Consistency and Availability.

__Learning Objectives__

By the end of Day 4, you should be able to:
```
Explain what a distributed system is.
Explain why distributed systems are fundamentally difficult.
Understand distributed state.
Understand network partitions.
Understand different failure models.
Define:
Consistency
Availability
Partition Tolerance
Explain CAP theorem without using the misleading "pick any two" interpretation.
Understand why Partition Tolerance is generally not optional in real distributed systems.
Understand the difference between:
Strong consistency
Eventual consistency
Availability
Fault tolerance
Analyze a distributed system during a network partition.
Understand CP, AP and the limitations of the CA terminology.
Understand CAP through real-world scenarios.
Understand quorum-based systems.
Understand how replication creates CAP trade-offs.
Understand PACELC.
Understand why latency vs consistency matters even when there is no failure.
Reason about CAP/PACELC during system-design interviews.
Apply CAP/PACELC to databases, caches, messaging systems and multi-region architectures.
```


**1. Why Are We Studying CAP?**

Before CAP, imagine a very simple application.
```
                 ┌─────────────┐
User ───────────►│ Application │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │  Database   │
                 └─────────────┘
```
There is one database.

If the application writes:

balance = $100

and immediately reads it:

balance = $100

there is no distributed synchronization problem.

Now suppose the application grows.

We deploy databases in multiple locations:
```
                 ┌─────────────┐
                 │ Application │
                 └──────┬──────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
       ┌────────────┐       ┌────────────┐
       │ Database A │◄─────►│ Database B │
       │   India    │       │  Europe    │
       └────────────┘       └────────────┘
```
Now we have a new problem:

What happens if Database A and Database B cannot communicate?

That single question leads us directly into CAP.


__2. What Is a Distributed System?__

A distributed system is a collection of independent computers that cooperate over a network to provide a unified service.

Examples:

- Distributed databases
- Kafka clusters
- Kubernetes clusters
- Microservices
- Cloud applications
- Multi-region applications
- Distributed caches
- Search clusters
- Object storage
- Payment systems


A simplified architecture:
```
                    Internet
                       │
              ┌────────┴────────┐
              │ Load Balancer   │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Server A     Server B     Server C
          │            │            │
          └────────────┼────────────┘
                       │
                Distributed DB
             ┌─────────┼─────────┐
             ▼         ▼         ▼
           Node A    Node B    Node C
```
The important word is:

Network

The nodes need the network to coordinate.

And networks can:
- Delay messages
- Drop messages
- Duplicate messages
- Reorder messages
- Disconnect nodes
- Become congested
- Become partially unavailable
- Fail between regions

This is where distributed systems become difficult.


**3. The Fundamental Distributed Systems Problem**

Consider two replicas.
```
        Replica A                 Replica B

       balance=100               balance=100
            │                         │
            │                         │
            └──────── Network ─────────┘
```
Client sends:

SET balance = 50

to Replica A.

Replica A updates:

balance = 50

But before Replica A can replicate the change:
```
             X
            / \
           /   \
          A     B

      balance   balance
        50        100
```
Now another user reads from B.

They receive:

balance = 100

But another user reading A gets:

balance = 50

We now have:

Distributed state divergence

This is the heart of many distributed-systems problems.


**4. What Is Distributed State?**

State means information that the system remembers.

Examples:
- User balance
- Order status
- Inventory quantity
- Shopping cart
- Session information
- Payment status
- Account permissions
- Message offsets
- Configuration

In a distributed system, the same logical state may exist on multiple nodes.
```
                 Logical State
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Replica A    Replica B    Replica C

       balance=100  balance=100  balance=100
```
The challenge is:

How do we ensure that all replicas have an acceptable view of the same logical state?

There are several possible answers.

Option 1 — Synchronous replication

Don't acknowledge the write until replicas agree.
```
Client
  │
  ▼
Node A
  │
  ├────────► Node B
  │             │
  ├────────► Node C
  │             │
  ◄─────────────┘
  │
  ▼
ACK
```
Advantages:

- Stronger consistency

Disadvantages:

- Higher latency
- Network failure can prevent progress
- Availability can decrease

Option 2 — Asynchronous replication

Acknowledge immediately.
```
Client
  │
  ▼
Node A
  │
  ▼
ACK
  │
  ├────────► Node B
  └────────► Node C
```
Advantages:

- Lower latency
- Higher availability

Disadvantages:

- Replicas can temporarily disagree
- Reads can return stale data
- Replication lag can occur
- Failures can cause lost/uncommitted updates depending on design

This is where CAP/PACELC becomes important.


**5. Network Partition**

A network partition occurs when communicating nodes become unable to reliably communicate with each other.

Example:

              Network
                 X
                / \
               /   \
              /     \
        ┌─────┐     ┌─────┐
        │ A   │     │ B   │
        │     │     │     │
        └─────┘     └─────┘

Node A is alive.

Node B is alive.

But:
```
A ─────X───── B
```
Messages cannot successfully travel between them.

This is extremely important:

A network partition does not necessarily mean the machines are dead.

Both machines may be perfectly healthy.

The communication path is broken.


**6. Real-World Partition Examples**

A partition can happen because of:

- Router failure
- Network switch failure
- Firewall rule
- DNS failure
- Cloud networking problem
- Availability Zone failure
- Region connectivity failure
- Cable failure
- Packet loss
- Network congestion
- Kubernetes networking problem
- Service discovery failure
- Security policy
- Misconfiguration

Example:
```
                Internet
                   │
          ┌────────┴────────┐
          │                 │
       Region A          Region B
          │                 │
       DB Node A         DB Node B
          │                 │
          └───────X─────────┘
```
Both regions are operational.

But they cannot communicate.

This is a partition.


**7. Failure Model**

Before designing a distributed system, we need to ask:

What failures are we assuming can happen?

This is called the failure model.

Typical failures include:

7.1 Crash failure
A node stops responding.

Node A ──X

7.2 Network failure
Messages cannot reach another node.

A ─────X───── B

7.3 Message delay
Message eventually arrives but very late.

A ────────────────► B
     10 seconds

The system may have already assumed:

B is dead

when it was actually just slow.

7.4 Message loss

A ───────X──────► B

The message never arrives.

7.5 Message duplication
A request may arrive more than once.

A ───────► B
A ───────► B

This is why distributed systems often require:

Idempotency
Request IDs
Deduplication
Exactly-once-like processing semantics at the application level

7.6 Message reordering

Suppose:

Message 1 = balance = 100
Message 2 = balance = 50

Network delivers:

Message 2
Message 1

Replica might incorrectly end up with:

balance = 100

instead of:

balance = 50

Distributed systems therefore need mechanisms such as:

 - Sequence numbers
 - Logical clocks
 - Version numbers
 - Timestamps
 - Consensus protocols
 - Conflict resolution


**8. CAP Theorem**

CAP stands for:

C = Consistency
A = Availability
P = Partition Tolerance

The formal result was developed from Eric Brewer's conjecture and formally proved by Seth Gilbert and Nancy Lynch in 2002. The formal result concerns an asynchronous distributed system and establishes that strong consistency and availability cannot both be guaranteed when network partitions are possible.

The simplified statement is:

A distributed system cannot simultaneously guarantee strong consistency, availability, and partition tolerance under the CAP model.

But don't memorize that sentence alone.

You need to understand why.


**9. CAP — Consistency**
CAP's "Consistency" should not be confused with general database consistency from ACID.

In CAP discussions, consistency is generally referring to strong/atomic consistency, commonly understood as linearizability.

Simplified:
Once a successful write completes, every subsequent read should see that write or a newer value.

Example:

Initial:
balance = 100

Client A:
SET balance = 50
Write succeeds.

Client B immediately reads:
GET balance

Strong consistency requires:
50

not:
100


**10. CAP Consistency Example**

Imagine bank account:

Account A
Balance = ₹10,000

Two ATM systems:
```
ATM 1 ─────► Database A
ATM 2 ─────► Database B
```
If Database A says:
₹10,000

and Database B says:
₹10,000

then:

ATM 1 withdraw ₹8,000
Database A becomes:
₹2,000

But if Database B hasn't received the update yet:
Database B = ₹10,000

ATM 2 might allow another:
₹8,000 withdrawal

Now the system has potentially allowed:
₹16,000 withdrawal
from:
₹10,000

This is why consistency matters for certain workloads.


**11. CAP — Availability**

CAP availability means:
Every request to a non-failing node receives a response within the required model's bounds, rather than being indefinitely rejected or waiting forever.

Simplified:
```
Request
   │
   ▼
System
   │
   ▼
Response
```
Even if some nodes are unreachable, the system continues responding according to its availability guarantee.

Example:
```
Node A   Node B   Node C
  ✓        X        ✓
```
An available system might still process a request through A or C.


**12. Important: Availability ≠ Uptime**

Don't confuse:

Availability

with:

"Server is online"

In distributed systems, availability is about whether requests can receive acceptable responses.

A service could have:

Server = technically running

but:

Request = timeout

From the user's perspective:

System = unavailable

Availability is therefore an end-to-end property, not merely a machine being powered on.

AWS also distinguishes availability from reliability and provides quantitative measures such as MTBF and MTTR for system availability.


**13. CAP — Partition Tolerance**

Partition tolerance means:
The system continues to operate despite communication failures that divide the distributed nodes.

Example:
```
           NETWORK PARTITION

     ┌─────────────┐
     │             │
     │   Region A   │
     │   DB A       │
     │             │
     └──────┬──────┘
            X
            X
            X
     ┌──────┴──────┐
     │             │
     │   Region B   │
     │   DB B       │
     │             │
     └─────────────┘
```
Partition tolerance asks:
Can the architecture survive this communication failure?


**14. The Most Important CAP Insight**
This is where many developers misunderstand CAP.

People often say:

Choose 2:

C
A
P

This is misleading.

The better mental model is:
```
                    NORMAL OPERATION
                          │
                          ▼
                  C + A can coexist
                          │
                          ▼
                 NETWORK PARTITION
                          │
                  ┌───────┴───────┐
                  ▼               ▼
              Consistency    Availability
                  │               │
                CP              AP
```
Why?
Because Partition Tolerance isn't usually something you voluntarily "choose to turn on."

If your system spans:

- Multiple machines
- Availability zones
- Regions
- Data centers
- Internet-connected services

then communication failures are unavoidable possibilities.

AWS explicitly notes that most distributed systems need to tolerate network failures, making the practical choice during a partition a trade-off between consistency and availability.


**15. Why Can't We Have C + A + P?**

Let's reason instead of memorizing.

Consider:
```
            Client 1
               │
               ▼
             Node A
               │
               X
               X   PARTITION
               X
             Node B
               ▲
               │
            Client 2
```
Initially:
A = 100
B = 100

Client 1 writes:
A = 50

But partition prevents:

A ─────X───── B

So now:
A = 50
B = 100

Client 2 asks B:
GET balance

What should B return?

Choice 1 — Return 100
Then B is available.

But:
A = 50
B = 100

The system cannot guarantee strong consistency.

So:

Availability ✓
Partition Tolerance ✓
Strong Consistency ✗

This is AP behavior.

Choice 2 — Refuse the request

B says:
"I cannot confirm the latest state."

Then consistency can be preserved.

But the request does not receive the desired successful response.

So:

Consistency ✓
Partition Tolerance ✓
Availability ✗

This is CP behavior.
