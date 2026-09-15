[🏠 Back to CS Theory Index](./README.md) • [⬅️ Prev: Operating Systems](./01-operating-systems.md) • [Next: Computer Networks ➡️](./03-computer-networks.md)

<div align="center">
  <h1>02. Database Management Systems (DBMS)</h1>
  <p><b>Relational Theory, Normalization, ACID, Concurrency Control, MVCC, Indexing, & Sharding</b></p>
</div>

---

## 📑 Module Index
- [1. Data Models & Relational Architecture](#1-data-models--relational-architecture)
  - [Relational Model Concepts (Tuples, Cardinality, Degree, Keys)](#relational-model-concepts)
  - [ER Modeling & Cardinality Constraints (1:1, 1:N, M:N)](#er-modeling)
- [2. Database Normalization](#2-database-normalization)
  - [Update, Insertion & Deletion Anomalies](#database-anomalies)
  - [Functional Dependencies & Forms: 1NF ➔ 2NF ➔ 3NF ➔ BCNF](#normalization-steps)
- [3. Transactions & ACID Properties](#3-transactions--acid-properties)
  - [Atomicity, Consistency, Isolation & Durability](#acid-properties)
  - [Write-Ahead Logging (WAL) & ARIES Recovery](#wal-and-recovery)
- [4. Concurrency Control & Isolation Levels](#4-concurrency-control--isolation-levels)
  - [Concurrency Anomalies: Dirty Reads, Non-Repeatable Reads, Phantom Reads](#concurrency-anomalies)
  - [The 4 ANSI SQL Isolation Levels](#ansi-sql-isolation-levels)
  - [Two-Phase Locking (2PL) vs Multi-Version Concurrency Control (MVCC)](#2pl-vs-mvcc)
- [5. Storage Engines & Indexing Mechanics](#5-storage-engines--indexing-mechanics)
  - [Clustered vs Non-Clustered Indexes](#clustered-vs-non-clustered)
  - [B+ Tree Internals vs LSM Trees (Log-Structured Merge Trees)](#b-plus-trees-vs-lsm)
- [6. Database Scaling & Architecture](#6-database-scaling--architecture)
  - [Read Replicas & Master-Slave Replication (Sync vs Async)](#replication-models)
  - [Sharding Strategies: Hash-based vs Range-based vs Consistent Hashing](#sharding-strategies)

---

## 1. Data Models & Relational Architecture

### Relational Model Concepts

- **Relation (Table):** A set of tuples sharing the same schema.
- **Tuple (Row / Record):** A single data entity instance.
- **Attribute (Column / Field):** A named domain property.
- **Cardinality:** Total number of **rows** in a table.
- **Degree (Arity):** Total number of **columns** in a table.

```
┌─────────────────────────────────────────────────────────────┐
│ Keys Hierarchy in RDBMS                                     │
├─────────────────┬───────────────────────────────────────────┤
│ **Super Key**   │ Any set of attributes uniquely identifying a tuple.│
│ **Candidate Key**│ Minimal Super Key (no redundant attributes).│
│ **Primary Key** │ Candidate Key chosen as unique table identifier (NOT NULL).│
│ **Foreign Key** │ Column referencing Primary Key of another table.│
└─────────────────┴───────────────────────────────────────────┘
```

---

## 2. Database Normalization

Normalization decomposes relations to eliminate data redundancy and prevent operational anomalies.

```
                      [ UNNORMALIZED DATA ]
                                │
                                ▼ (Eliminate repeating groups / multi-valued attributes)
                              [ 1NF ]
                                │
                                ▼ (Eliminate Partial Dependencies: Non-prime depends on subset of Candidate Key)
                              [ 2NF ]
                                │
                                ▼ (Eliminate Transitive Dependencies: Non-prime depends on Non-prime)
                              [ 3NF ]
                                │
                                ▼ (For every functional dependency X -> Y, X MUST be a Super Key)
                             [ BCNF ]
```

### Normal Forms Summary

| Normal Form | Rule Requirement | Example Violation & Fix |
| :--- | :--- | :--- |
| **1NF** | Atomic values in every cell; no multi-valued arrays or nested tables. | `Phones: [123, 456]` $\rightarrow$ Split into separate rows or join table. |
| **2NF** | Must be in 1NF **AND** no Partial Dependency (applies when Composite Primary Key exists). | Composite Key `(StudentID, CourseID)` determines `StudentName`. $\rightarrow$ Move `StudentName` to `Students` table. |
| **3NF** | Must be in 2NF **AND** no Transitive Dependency ($A \rightarrow B$ and $B \rightarrow C$). | `OrderID -> CustomerID -> CustomerAddress`. $\rightarrow$ Move `CustomerAddress` to `Customers` table. |
| **BCNF** | Must be in 3NF **AND** for every $X \rightarrow Y$, $X$ must be a **Super Key**. | Handles subtle overlapping candidate key anomalies. |

---

## 3. Transactions & ACID Properties

A **Transaction** is a logical unit of database processing containing one or more SQL operations.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   ACID GUARANTEES                                      │
├───────────────────┬────────────────────────────────────────────────────────────────────┤
│ **A - Atomicity** │ All operations succeed or entire transaction is rolled back (All or Nothing).│
├───────────────────┼────────────────────────────────────────────────────────────────────┤
│ **C - Consistency**│ Database transitions only from one valid state to another valid state.│
├───────────────────┼────────────────────────────────────────────────────────────────────┤
│ **I - Isolation** │ Concurrent transactions execute without interfering with one another.│
├───────────────────┼────────────────────────────────────────────────────────────────────┤
│ **D - Durability** │ Committed changes are permanently saved even during power failure.│
└───────────────────┴────────────────────────────────────────────────────────────────────┘
```

- **Write-Ahead Logging (WAL):** Before changes are written to the main data pages on disk, log records describing the change are flushed sequentially to append-only disk storage. This guarantees durability and allows crash recovery via REDO/UNDO loops.

---

## 4. Concurrency Control & Isolation Levels

### Concurrency Read Anomalies

1. **Dirty Read:** Transaction $T_1$ reads uncommitted modified data from $T_2$. If $T_2$ rolls back, $T_1$ read fake data.
2. **Non-Repeatable Read (Fuzzy Read):** $T_1$ reads row $R$. $T_2$ updates or deletes row $R$ and commits. $T_1$ rereads $R$ and gets different values.
3. **Phantom Read:** $T_1$ queries a range of rows (`WHERE age > 25`). $T_2$ inserts a new row with `age = 28` and commits. $T_1$ runs the same range query and finds a new "phantom" row.

---

### ANSI SQL Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Mechanism / Overhead |
| :--- | :---: | :---: | :---: | :--- |
| **Read Uncommitted** | ⚠️ Allowed | ⚠️ Allowed | ⚠️ Allowed | No read locks; lowest isolation. |
| **Read Committed** (PostgreSQL default) | 🛡️ Prevented | ⚠️ Allowed | ⚠️ Allowed | Reads use point-in-time snapshot per statement. |
| **Repeatable Read** (MySQL InnoDB default) | 🛡️ Prevented | 🛡️ Prevented | ⚠️ Allowed | Snapshot created at transaction start. |
| **Serializable** | 🛡️ Prevented | 🛡️ Prevented | 🛡️ Prevented | Strict 2PL or Serializable Snapshot Isolation (SSI). |

---

### MVCC (Multi-Version Concurrency Control)

Modern RDBMS (PostgreSQL, MySQL InnoDB) implement MVCC:
- **Readers never block Writers, and Writers never block Readers!**
- When a row is updated, the engine does not overwrite the row immediately; it creates a new version with metadata tracking transaction IDs (`xmin`, `xmax`).
- Each transaction sees a consistent snapshot based on its start timestamp.

---

## 5. Storage Engines & Indexing Mechanics

### Clustered vs Non-Clustered Indexes

- **Clustered Index:** Dictates the physical on-disk storage order of data rows. (Only **1 per table**, usually Primary Key). Leaf nodes contain the **actual row data**.
- **Non-Clustered (Secondary) Index:** Separate data structure. Leaf nodes contain index keys and a pointer (or Primary Key) pointing back to the physical row.

---

### B+ Tree vs LSM Tree

```
               B+ TREE (PostgreSQL, MySQL, Oracle)                     LSM TREE (Cassandra, RocksDB, Scylla)
                     [ Root Node ]                                      [ MemTable (In-Memory RAM Buffer) ]
                     /           \                                                    │ Flush
             [ Internal ]     [ Internal ]                                            ▼
               /      \         /      \                                    [ SSTable Level 0 (Disk) ]
          [ Leaf ]──[ Leaf ]──[ Leaf ]──[ Leaf ]                                      │ Compaction
          (Leaf nodes form sorted linked-list)                                        ▼
          • Best for: READ-heavy workloads & range scans.                    [ SSTable Level 1 (Disk) ]
                                                                        • Best for: WRITE-heavy workloads.
```

---

## 6. Database Scaling & Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  PRIMARY (MASTER) DATABASE                  │
│                     (All Write / Updates)                   │
└──────────────────────────────┬──────────────────────────────┘
                               │ Replication Stream (WAL logs)
                ┌──────────────┴──────────────┐
                ▼                             ▼
   [ READ REPLICA 1 ]            [ READ REPLICA 2 ]
   (Handles Read Queries)        (Handles Read Queries)
```

### Sharding (Horizontal Partitioning)

Distributes rows of a table across multiple independent physical database instances:

1. **Hash-Based Sharding:** $\text{Shard ID} = \text{hash}(\text{user\_id}) \pmod N$. Uniform distribution, but adding new shards requires re-hashing data (solved via **Consistent Hashing**).
2. **Range-Based Sharding:** Splits by ranges (e.g., Shard 1: A–M, Shard 2: N–Z). Prone to hot spots.
3. **Directory-Based Sharding:** Lookup service maps partition keys to database instances.

---

[➡️ Continue to Module 03: Computer Networks](./03-computer-networks.md)
