[🏠 Back to CS Theory Index](./README.md) • [⬅️ Prev: Software Engineering](./05-software-engineering-design.md) • [Next: CS Theory Interview Mastery ➡️](./07-cs-theory-interview-mastery.md)

<div align="center">
  <h1>06. System Design Blueprints</h1>
  <p><b>Production Architectural Blueprints: Rate Limiters, URL Shorteners, Real-Time Chat, Notifications, & Inventory</b></p>
</div>

---

## 📑 Module Index
1. [Blueprint 1: Distributed Rate Limiter](#blueprint-1-distributed-rate-limiter)
2. [Blueprint 2: Scalable URL Shortener (Bitly)](#blueprint-2-scalable-url-shortener)
3. [Blueprint 3: Real-Time Chat System (WhatsApp / Slack)](#blueprint-3-real-time-chat-system)
4. [Blueprint 4: Multi-Channel Notification Service](#blueprint-4-multi-channel-notification-service)
5. [Blueprint 5: Distributed Inventory Reservation & Idempotent Checkout](#blueprint-5-distributed-inventory-reservation)

---

## Blueprint 1: Distributed Rate Limiter

### Requirements
- Limit requests per user/IP (e.g., 100 requests per minute).
- Minimal latency overhead (<2ms), distributed across API Gateway instances.

```
                    ┌─────────────────────────┐
                    │     CLIENT REQUEST      │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       API GATEWAY       │
                    └────────────┬────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────┐
│ ATOMIC REDIS SLIDING WINDOW (Lua Script)                    │
│ 1. Remove timestamps older than (now - 60s) (ZREMRANGEBYSCORE)│
│ 2. Count current elements in sorted set (ZCARD)             │
│ 3. If count < 100:                                          │
│      ZADD key now now ──► Set EXPIRE ──► HTTP 200 OK        │
│    Else:                                                    │
│      Return HTTP 429 Too Many Requests (Retry-After header) │
└─────────────────────────────────────────────────────────────┘
```

---

## Blueprint 2: Scalable URL Shortener

### Key Mechanics: Base62 vs Hashing
- MD5/SHA256 of URL produces 128/256-bit hashes (too long).
- **Solution: Distributed ID Generator (Twitter Snowflake) + Base62 Encoding:**
  - Base62 characters: `[0-9, a-z, A-Z]` (62 chars).
  - A 7-character string in Base62 can represent $62^7 \approx 3.5 \text{ Trillion}$ unique short URLs!

```
[ Long URL: https://example.com/very/long/path ] ──► [ API Gateway ]
                                                             │
                                                             ▼
                                              [ Snowflake ID Generator ]
                                              (Generates 64-bit int: 18928392190)
                                                             │
                                                             ▼
                                              [ Base62 Encoder: "7bK9xQ2" ]
                                                             │
                                                             ▼
                                  ┌──────────────────────────┴──────────────────────────┐
                                  ▼                                                     ▼
                     [ Redis Cache (LRU 20%) ]                                [ PostgreSQL / MongoDB ]
                     key: "7bK9xQ2" -> long_url                                (Primary Storage)
```

---

## Blueprint 3: Real-Time Chat System

### Architecture (WebSockets + Redis Pub/Sub + ScyllaDB)

```
[ User A (Mobile) ]                      [ User B (Mobile) ]
       │ (WebSocket Connection)                 │ (WebSocket Connection)
       ▼                                        ▼
[ WS Gateway Server 1 ]                  [ WS Gateway Server 2 ]
       │                                        ▲
       ├──► Publishes message to Redis Pub/Sub ──┘ (Delivers to User B's open WS connection)
       │
       ▼ (Async Queue: Kafka)
[ Message Ingestion Worker ] ──► [ ScyllaDB / Cassandra (Wide-Column) ]
                                  PK: (chat_id), Clustering: (created_at DESC)
```

- **Session Management:** Redis Hash stores `user_id -> ws_server_id` to route messages directly to the server holding the receiver's persistent TCP connection.
- **Message Storage:** Cassandra/ScyllaDB delivers fast sequential writes and efficient range queries for fetching past 50 chat messages.

---

## Blueprint 4: Multi-Channel Notification Service

```
[ Microservices (Order, Auth, Payment) ]
                   │
                   ▼ (Produces Notification Request Event)
┌─────────────────────────────────────────────────────────────┐
│ KAFKA TOPIC: "notification-events"                          │
│ Partitioned by User ID (Ensures FIFO per user)              │
└──────────────┬──────────────────────────────┬───────────────┘
               │                              │
               ▼ (Priority Queue: High)       ▼ (Bulk Marketing: Low)
       [ Worker Service ]             [ Worker Service ]
               │                              │
       ┌───────┼──────────────────────┐       │
       ▼       ▼                      ▼       ▼
  [ FCM/APNS ] [ Twilio SMS ]   [ SendGrid Email ]
 (Mobile Push) (OTP Codes)      (Order Receipt)
```

- **Features:**
  - **Rate Limiting & De-duplication:** Avoids spamming users with duplicate alerts within 5 minutes.
  - **User Preference Engine:** Checks if user has disabled marketing push notifications before dispatching.

---

## Blueprint 5: Distributed Inventory Reservation

### The Problem: Overselling Hot Items (Flash Sales)

```
               [ 10,000 Concurrent Users Checkout 10 PS5 Consoles ]
```

### The Solution: Two-Phase Redis Reservation with TTL

```
┌─────────────────────────────────────────────────────────────┐
│ 1. USER CLICKS CHECKOUT                                     │
│    • Atomic Redis Lua Script: DECRBY stock 1                │
│    • If stock >= 0:                                         │
│        Create Reservation Key in Redis with 10-Minute TTL   │
│        SET reservation:ord_123 1 EX 600                     │
│      Else:                                                  │
│        Return "Sold Out" immediately (Zero DB load!)        │
├─────────────────────────────────────────────────────────────┤
│ 2. USER PAYS WITHIN 10 MINUTES                              │
│    • Payment webhook succeeds ──► Commit Order to SQL DB    │
│    • Delete reservation key in Redis                        │
├─────────────────────────────────────────────────────────────┤
│ 3. USER ABANDONS CHECKOUT                                   │
│    • Redis Key expires (TTL 600s) ──► Keyspace notification │
│    • INCRBY stock 1 (Automatically restocks inventory!)     │
└─────────────────────────────────────────────────────────────┘
```

---

[➡️ Continue to Module 07: CS Theory Interview Mastery](./07-cs-theory-interview-mastery.md)
