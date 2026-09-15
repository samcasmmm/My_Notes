[🏠 Back to CS Theory Index](./README.md) • [⬅️ Prev: System Design Blueprints](./06-system-design-blueprints.md)

<div align="center">
  <h1>07. CS Theory Interview Mastery</h1>
  <p><b>50+ High-Yield Core CS Theory & System Design Interview Questions with In-Depth Solutions</b></p>
</div>

---

## 📑 Interview Sections
1. [Part 1: Operating Systems & Concurrency](#part-1-operating-systems--concurrency)
2. [Part 2: Database Management & SQL Internals](#part-2-database-management--sql-internals)
3. [Part 3: Computer Networks & Protocols](#part-3-computer-networks--protocols)
4. [Part 4: Distributed Systems & System Design](#part-4-distributed-systems--system-design)

---

## Part 1: Operating Systems & Concurrency

#### Q1: What exact steps occur during a CPU Context Switch?
1. CPU receives a hardware timer interrupt or system call trap, switching from User Mode to Kernel Mode.
2. Kernel saves current process state (Program Counter, registers, stack pointer) into its **Process Control Block (PCB)**.
3. Scheduler selects the next process according to its scheduling algorithm (e.g., MLFQ, Round Robin).
4. Kernel updates the Memory Management Unit (MMU) with the new process's Page Table Base Register (PTBR), which flushes the **TLB (Translation Lookaside Buffer)**.
5. Kernel loads the new process's registers from its PCB and resumes execution in User Mode.

#### Q2: What is the difference between a Mutex and a Binary Semaphore?
- **Mutex (Mutual Exclusion):** Has the concept of **ownership**. Only the specific thread that locked the mutex can unlock it. Best for protecting critical sections.
- **Binary Semaphore:** Has **no ownership**. Any thread can signal (`V()`) to wake up another thread waiting (`P()`). Best for inter-thread synchronization signaling.

#### Q3: What is Thrashing in Virtual Memory and how does the OS resolve it?
- **Thrashing:** A state where the CPU spends more time swapping pages in and out between RAM and disk than executing actual user instructions (Page Fault rate explodes toward 100%).
- **Resolution:** The OS uses the **Working Set Model** to monitor active pages. If the sum of working sets of all processes exceeds physical RAM, the OS suspends/swaps out entire lower-priority processes to free memory for active ones.

---

## Part 2: Database Management & SQL Internals

#### Q4: Why are B+ Trees preferred over Binary Search Trees or Hash Tables for database indexes?
- **BST / Red-Black Trees:** Have low fan-out ($2$ children per node), resulting in tree depth of $20+$ levels for millions of rows. Each level requires an expensive disk seek.
- **Hash Tables:** Fast $O(1)$ point lookups (`WHERE id = 5`), but completely fail at **Range Queries** (`WHERE age BETWEEN 20 AND 30` requires full table scan).
- **B+ Trees:** High fan-out ($100–500$ keys per node $\rightarrow$ depth only $3–4$ levels). All data resides in leaf nodes that form a **doubly-linked sorted list**, making range queries and sequential scans extremely fast.

#### Q5: How does Multi-Version Concurrency Control (MVCC) prevent locking during reads?
- When a transaction updates a row, PostgreSQL/MySQL creates a new tuple version with metadata tags (`xmin` for creation txn ID, `xmax` for deletion/expiry txn ID).
- When another transaction performs a `SELECT`, it reads the tuple version matching its snapshot timestamp without acquiring read locks. Thus, **Writers do not block Readers, and Readers do not block Writers**.

---

## Part 3: Computer Networks & Protocols

#### Q6: What happens step-by-step when you type `https://www.google.com` into a browser and press Enter?

```
[ Browser Cache / OS DNS ] ──► [ Recursive DNS Query ] ──► (Returns IP: 142.250.190.46)
                                                                      │
[ TLS 1.3 Handshake ] ◄── [ TCP 3-Way Handshake (SYN -> SYN-ACK -> ACK) ]
         │
         ▼
[ HTTP/2 or HTTP/3 GET / Request ] ──► [ Load Balancer (Nginx / ALB) ]
                                                    │
[ Browser DOM Parsing & Critical Rendering Path ] ◄── [ Reverse Proxy / App Server ]
```

1. **DNS Resolution:** Checks Browser Cache $\rightarrow$ OS hosts file $\rightarrow$ Local DNS Resolver $\rightarrow$ Root $\rightarrow$ TLD $\rightarrow$ Authoritative Server.
2. **TCP Handshake:** Client sends `SYN`, Server replies `SYN-ACK`, Client sends `ACK`.
3. **TLS 1.3 Handshake:** Client & Server exchange Diffie-Hellman public key shares and authenticate Server Certificate to establish symmetric encryption.
4. **HTTP Request:** Browser sends `GET / HTTP/2` request.
5. **Server Processing:** Reverse proxy routes to application server; database queried; HTML returned.
6. **Browser Rendering:** Parses HTML $\rightarrow$ Builds DOM Tree $\rightarrow$ Parses CSS $\rightarrow$ CSSOM $\rightarrow$ Render Tree $\rightarrow$ Layout $\rightarrow$ Paint.

#### Q7: How does HTTP/3 eliminate Head-of-Line (HoL) blocking compared to HTTP/2?
- In HTTP/2, all multiplexed streams share a **single TCP connection**. If one TCP packet is dropped on a lossy network, TCP halts delivery of *all* streams until the lost packet is retransmitted.
- HTTP/3 runs on **QUIC (over UDP)**. Streams are completely independent at the transport layer: packet loss in Stream 1 only delays Stream 1, while Streams 2, 3, and 4 continue processing with zero latency interruption.

---

## Part 4: Distributed Systems & System Design

#### Q8: How do you design an Idempotent Payment API?
- **Idempotency Key:** Client generates a unique UUID (e.g., `Idempotency-Key: 9b1deb4d-...`) and attaches it in the HTTP header.
- **Server Mechanics:**
  1. API Gateway checks Redis with `SET key req_hash NX EX 120`.
  2. If key exists with status `"PROCESSING"`, return HTTP `409 Conflict` or wait.
  3. If key exists with status `"COMPLETED"`, immediately return the cached previous response without recharging.
  4. If key does not exist, execute payment within a database transaction and store the final receipt in Redis against the key.

#### Q9: What is the difference between Strong Consistency (Linearizability) and Eventual Consistency?
- **Strong (Linearizable):** Every read across any server in the world is guaranteed to return the most recent write. Paid with higher latency and reduced availability during partitions (e.g., Banking balance).
- **Eventual:** Writes propagate asynchronously. Replicas may return stale data for a few hundred milliseconds, but will eventually synchronize (e.g., YouTube video view counts, Twitter likes).

---

<div align="center">
  <sub><b>My_Notes</b> • Continuous Learning & Interview Mastery</sub><br/>
  <a href="./README.md"><b>Back to CS Theory Index 🏠</b></a>
</div>
