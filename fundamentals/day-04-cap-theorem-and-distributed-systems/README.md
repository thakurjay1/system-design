CAP is not “pick any two.” In a real distributed system, network partitions are a failure condition you generally must tolerate.
Therefore, when a partition happens, the practical trade-off is primarily between Consistency and Availability.

__1. Learning Objectives__

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
