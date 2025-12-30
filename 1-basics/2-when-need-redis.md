
# When We Need Redis

## 1. **Caching Layer**
Reduce database load by storing frequently accessed data in memory.

```
┌─────────────┐
│   Request   │
└──────┬──────┘
    │
    ├─→ Redis Cache (Fast) ✓
    │
    └─→ Database (Slow) ✗
```

**Use case:** User profiles, product catalogs, API responses

---

## 2. **Session Management**
Store user sessions with automatic expiration.

```
┌──────────────────┐
│   User Login     │
└────────┬─────────┘
      │
      ├─→ Redis Session Store
      │   (TTL expiration)
      │
      └─→ Quick access across services
```

---

## 3. **Real-time Analytics**
Track metrics, counters, and statistics instantly.

```
┌──────────────────────┐
│  User Activity       │
└────────┬─────────────┘
      │
    Redis Counters/Sets
      │
    (Pageviews, clicks, unique users)
```

---

## 4. **Message Queue / Pub-Sub**
Decouple services with asynchronous messaging.

```
┌─────────────┐
│  Publisher  │
└──────┬──────┘
    │
    Redis Queue
    │
    ├─→ Consumer 1
    ├─→ Consumer 2
    └─→ Consumer 3
```

---

## 5. **Leaderboards & Sorted Sets**
Rank data efficiently using sorted sets.

```
Redis Sorted Set
├─ User A: 1500 points
├─ User B: 1200 points
└─ User C: 900 points
```

---

## 6. **Rate Limiting**
Control API request frequency per user.

```
User Request → Redis Counter
              │
              ├─ < Limit → Allow
              └─ ≥ Limit → Reject
```

---

## 7. **Distributed Locks**
Ensure single execution across multiple services.

```
┌──────────────────┐
│  Service A       │──┐
└──────────────────┘  │
             ├─→ Redis Lock
┌──────────────────┐  │   (Only one acquires)
│  Service B       │──┤
└──────────────────┘  │
             │
┌──────────────────┐  │
│  Service C       │──┘
└──────────────────┘
```
