[🏠 Back to Main Notes](../../README.md) • [💻 CS Theory DBMS Module](../../CS%20Theory/02-database-management-systems.md)

# Database Management Systems (DBMS) - Interview Questions & Answers

Refer to the complete, updated 2026 Master Guide in:
👉 **[02. Database Management Systems (DBMS) Master Guide](../../CS%20Theory/02-database-management-systems.md)**
👉 **[07. CS Theory Interview Mastery](../../CS%20Theory/07-cs-theory-interview-mastery.md)**

---

## 🎯 Quick Top Interview Highlights

### 1. What are ACID Properties?
- **Atomicity:** All operations succeed or all roll back (All or Nothing).
- **Consistency:** Database transitions only from one valid state to another valid state.
- **Isolation:** Concurrent transactions execute without interfering (Isolation levels: Read Uncommitted, Read Committed, Repeatable Read, Serializable).
- **Durability:** Committed transactions are permanently written to disk via Write-Ahead Logging (WAL).

### 2. Why use B+ Trees for Database Indexing?
- High fan-out ($100–500$ keys per node) keeps tree depth shallow ($3–4$ levels) for millions of records, minimizing expensive disk seeks.
- Leaf nodes contain all data pointers and form a sorted doubly-linked list for ultra-fast range queries (`BETWEEN x AND y`).

### 3. Normalization (1NF to BCNF)?
- **1NF:** Atomic attribute values; no repeating groups.
- **2NF:** 1NF + No Partial Dependency (all non-prime attributes depend on full candidate key).
- **3NF:** 2NF + No Transitive Dependency ($A \rightarrow B$ and $B \rightarrow C$).
- **BCNF:** 3NF + For every functional dependency $X \rightarrow Y$, $X$ must be a **Super Key**.
