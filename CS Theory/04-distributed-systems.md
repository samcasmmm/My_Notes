[🏠 Back to CS Theory Index](./README.md) • [⬅️ Prev: Computer Networks](./03-computer-networks.md) • [Next: Software Engineering & Design ➡️](./05-software-engineering-design.md)

<div align="center">
  <h1>04. Distributed Systems & Cloud Architecture</h1>
  <p><b>CAP Theorem, PACELC, Consensus (Raft), Distributed Transactions (Saga/2PC), & Caching Patterns</b></p>
</div>

---

## 📑 Module Index
- [1. Fundamental Distributed Theorems](#1-fundamental-distributed-theorems)
  - [CAP Theorem (Brewer's Conjecture)](#cap-theorem)
  - [PACELC Theorem (Latency vs Consistency in Normal Operations)](#pacelc-theorem)
- [2. Consistency Models](#2-consistency-models)
  - [Linearizability (Strong) vs Sequential vs Causal vs Eventual Consistency](#consistency-models)
- [3. Distributed Consensus & Coordination](#3-distributed-consensus--coordination)
  - [Raft Consensus Algorithm (Leader Election, Log Replication, Term)](#raft-consensus)
  - [Distributed Locks (Redis Redlock vs ZooKeeper / etcd)](#distributed-locking)
- [4. Distributed Transactions & Patterns](#4-distributed-transactions--patterns)
  - [Two-Phase Commit (2PC) & Its Pitfalls](#two-phase-commit)
  - [The Saga Pattern: Choreography vs Orchestration](#the-saga-pattern)
  - [Transactional Outbox Pattern & Event-Driven Architecture](#transactional-outbox-pattern)
- [5. Distributed Caching Strategies](#5-distributed-caching-strategies)
  - [Cache-Aside vs Read-Through vs Write-Through vs Write-Back](#caching-strategies)
  - [Cache Failures: Cache Avalanche, Stampede (Thundering Herd) & Penetration](#cache-failures-and-fixes)
- [6. Resilience & Fault Tolerance](#6-resilience--fault-tolerance)
  - [Circuit Breaker, Retry with Exponential Backoff & Jitter](#resilience-patterns)
  - [Idempotency Key Design for Payments & APIs](#idempotency-keys)

---

## 1. Fundamental Distributed Theorems

### CAP Theorem

In any asynchronous distributed network subject to network partitions ($P$), a system can guarantee at most **two** out of three properties:

```
                            CONSISTENCY (C)
                         (Every read receives the
                           latest write or error)
                                 / \
                                /   \
                               /     \
                              /  RDBMS\
                       (CP)  / (Single)\  (CA)
                   HBase,   /   Node    \  Traditional
                   etcd,   /             \ RDBMS
                   CockroachDB            \
                          /               \
                         /                 \
  PARTITION TOLERANCE (P)─────────────────── AVAILABILITY (A)
 (System continues despite         (AP)    (Every request receives
  dropped network packets)      Cassandra,  a non-error response,
                                DynamoDB,   without guarantee it's
                                 CouchDB    the latest write)
```

> [!IMPORTANT]
> Because physical network cables and routers **will inevitably fail** ($P$ is mandatory in real-world distributed networks), the real choice is between **CP** (Consistency over Availability) or **AP** (Availability over Consistency).

---

### PACELC Theorem

Extends CAP by considering behavior during **normal operations** (when there is NO partition):

$$\text{If Partition } (P): \text{Choose between } \mathbf{Availability} (A) \text{ vs } \mathbf{Consistency} (C)$$
$$\mathbf{Else} (E): \text{Choose between } \mathbf{Latency} (L) \text{ vs } \mathbf{Consistency} (C)$$

- **MongoDB / HBase (PC/EC):** Prioritizes Consistency during partitions; prioritizes Consistency (paying latency cost) during normal times.
- **DynamoDB / Cassandra (PA/EL):** Prioritizes Availability during partitions; prioritizes low Latency during normal operations.

---

## 2. Consistency Models

```
Strongest ────────────────────────────────────────────────────────────────────────► Weakest
Linearizable (Strong) ──► Sequential ──► Causal ──► Read-Your-Writes ──► Eventual Consistency
```

- **Linearizable (Strong Consistency):** A global real-time clock ordering exists. As soon as a write completes, all subsequent reads across all nodes immediately return that value.
- **Eventual Consistency:** If no new updates are made, all replicas will eventually converge to the same value. (e.g., DNS, Social Media likes).
- **Read-Your-Writes Consistency:** A user will always see their own recent updates, even if other users see older versions for a few milliseconds.

---

## 3. Distributed Consensus & Coordination

### Raft Consensus Algorithm

Used in **etcd (Kubernetes)** and **CockroachDB** to maintain a replicated state machine across $2F + 1$ nodes (tolerating $F$ node failures):

```
[ Follower ] ──(Heartbeat Timeout)──► [ Candidate ] ──(Wins Majority Votes)──► [ Leader ]
     ▲                                                                           │
     └────────────────────────(Discovers Higher Term)────────────────────────────┘
```

1. **Leader Election:** If a Follower receives no heartbeats from Leader within a randomized timeout (150–300ms), it transitions to Candidate, increments `term`, votes for itself, and requests votes from peers.
2. **Log Replication:** Leader accepts client writes, appends to its log, and broadcasts `AppendEntries` RPCs. Once committed on a majority of nodes, Leader applies entry to state machine and responds to client.

---

## 4. Distributed Transactions & Patterns

### The Saga Pattern (Long-Running Distributed Transactions)

In microservices, ACID transactions across multiple databases fail because distributed locks cause high latency and single points of failure.

A **Saga** is a sequence of local transactions where each local transaction updates the database and publishes an event to trigger the next step. If a step fails, the Saga executes **Compensating Transactions** to undo past changes:

```
[ Create Order ] ──► [ Deduct Balance ] ──► [ Reserve Inventory ] ──► [ Ship Order ]
                                                    │ (FAILED: Out of Stock)
                                                    ▼
[ Cancel Order ] ◄── [ Refund Balance ] ◄── [ Compensation Triggered ]
```

- **Choreography:** Services listen to domain events directly without a central coordinator (good for simple workflows).
- **Orchestration:** A central Saga Orchestrator service commands each participant which transaction to execute next (standard for complex enterprise flows).

---

### Transactional Outbox Pattern

Guarantees **Atomic State Update + Message Publishing** without 2-phase commit:

```
                  LOCAL DATABASE TRANSACTION
┌─────────────────────────────────────────────────────────────┐
│ 1. INSERT INTO orders (order_id, total, status)             │
│ 2. INSERT INTO outbox_events (event_id, payload, sent=false)│
└──────────────────────────────┬──────────────────────────────┘
                               │ Commit Transaction (100% Atomic)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ MESSAGE RELAY / CDC (Debezium / Poller Engine)              │
│ Reads outbox table ──► Publishes to Kafka ──► Mark Sent=True│
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Distributed Caching Strategies

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. Cache-Aside (Lazy Loading - Most Common)                                            │
│    • App reads Cache. If Miss, reads DB, writes to Cache, returns data.                │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 2. Read-Through                                                                        │
│    • App queries Cache library directly; Cache fetches from DB automatically on miss. │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 3. Write-Through                                                                       │
│    • App writes to Cache, and Cache writes synchronously to DB before returning.       │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ 4. Write-Back (Write-Behind)                                                           │
│    • App writes to Cache immediately; Cache queues async batch writes to DB later.     │
│    • Extreme write throughput, but risks data loss if cache crashes before DB flush.   │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Critical Caching Failures & Mitigations

1. **Cache Stampede (Thundering Herd):** A popular hot key expires, causing 10,000 concurrent requests to hit the database simultaneously.
   - *Fix:* Use **Mutex Locks (singleflight)** on cache miss or **Probabilistic Early Expiration (XFetch)**.
2. **Cache Penetration:** Malicious requests query non-existent keys (e.g., `id = -999`), bypassing cache and hitting DB every time.
   - *Fix:* Store null values with short TTL or use a **Bloom Filter**.
3. **Cache Avalanche:** Many cached keys expire at the exact same second, flooding the DB.
   - *Fix:* Add **Randomized Jitter** to TTL values (`TTL = 3600s + rand(0, 300s)`).

---

## 6. Resilience & Fault Tolerance

### Circuit Breaker Pattern

Prevents cascading service failures in microservice networks:

```
                      [ CLOSED ] (Normal operations)
                          │ (Failure rate exceeds threshold e.g. 50%)
                          ▼
                       [ OPEN ]  (Immediately fails fast with fallback; no requests sent)
                          │ (Timeout expires e.g. 30s)
                          ▼
                     [ HALF-OPEN ] (Sends canary test requests)
                       /        \
(Success: Reset counter)       (Failure: Trip back to OPEN)
          ▼                              ▼
      [ CLOSED ]                      [ OPEN ]
```

---

[➡️ Continue to Module 05: Software Engineering & Design](./05-software-engineering-design.md)
