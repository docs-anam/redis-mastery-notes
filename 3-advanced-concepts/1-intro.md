# Redis Advanced Concepts - Overview

## Table of Contents
1. [Overview](#overview)
2. [Topics Covered](#topics-covered)
3. [Prerequisites](#prerequisites)
4. [Learning Path](#learning-path)
5. [Key Concepts](#key-concepts)

---

## Overview

Advanced Redis concepts move beyond basic data types to powerful features for production systems. These include:

- **Message Passing**: Real-time communication between processes
- **Atomic Scripts**: Complex operations as single transactions
- **Data Replication**: High availability and redundancy
- **Automatic Failover**: Zero-downtime deployments
- **Horizontal Scaling**: Distribute data across multiple nodes

---

## Topics Covered

### 1. **Pub/Sub (Publish/Subscribe)**
Real-time messaging for decoupled systems.
- **Use Cases**: Chat, notifications, live feeds
- **Channels**: Named communication channels
- **Patterns**: Wildcard subscriptions
- **Limitations**: No persistence, no message queue

### 2. **Lua Scripting**
Atomic server-side scripts for complex operations.
- **Atomicity**: Entire script runs without interruption
- **Efficiency**: Reduces round-trips to Redis
- **Safety**: Deterministic execution
- **Use Cases**: Complex transactions, conditional updates

### 3. **Replication**
Master-slave data synchronization for redundancy.
- **Master-Slave**: One primary, multiple replicas
- **Sync Methods**: Full and partial resynchronization
- **Lag**: Potential for stale reads
- **Use Cases**: Backup, read scaling, high availability

### 4. **Sentinel**
Automatic failover and monitoring.
- **Monitoring**: Watches master health
- **Failover**: Automatic promotion of replica
- **Configuration**: Client discovery
- **Use Cases**: Production high availability

### 5. **Cluster**
Horizontal scaling and data distribution.
- **Sharding**: Data split across nodes by key hash
- **Resilience**: Multiple replicas per shard
- **Scalability**: Add nodes to increase capacity
- **Use Cases**: Massive datasets, linear scaling

---

## Prerequisites

Before diving into advanced topics, ensure you understand:

✅ **From 1-basics**:
- Installation and configuration
- All basic commands
- Persistence (RDB/AOF)
- Security and authentication

✅ **From 2-data-structures**:
- All data structures
- Performance characteristics
- When to use each structure

✅ **Technical Skills**:
- TCP/networking concepts
- Distributed systems basics
- Command-line tools (redis-cli, redis-benchmark)

---

## Learning Path

### Path 1: Real-Time Applications (2-3 days)
```
1. Pub/Sub → Real-time messaging
2. Streams → Event sourcing
3. Scripting → Complex event handling
Goal: Build chat or notification system
```

### Path 2: High Availability (3-4 days)
```
1. Replication → Data redundancy
2. Sentinel → Automatic failover
3. Monitoring → Health checks
Goal: Production-ready HA setup
```

### Path 3: Scaling (4-5 days)
```
1. Cluster basics → Sharding concepts
2. Cluster operations → Node management
3. Migration → Moving data safely
Goal: Multi-node Redis cluster
```

### Path 4: Production Excellence (5-7 days)
```
1. Scripting → Atomic operations
2. Performance tuning → Benchmarking
3. Replication + Sentinel → Combined HA
4. Monitoring → Observability
Goal: Enterprise-grade Redis system
```

---

## Key Concepts

### Consistency vs Availability

```
System Type          Consistency    Availability    Use Case
─────────────────────────────────────────────────────────────
Single Redis        ✅ Strong      ⚠️ Single point  Dev/cache
Master-Slave        ⚠️ Eventually   ✅ High         Read scaling
Sentinel            ✅ Strong      ✅ High         HA production
Cluster             ⚠️ Eventually   ✅ High         Massive scale
```

### Replication Lag

When using replicas, understand:
- **Synchronous replication**: Redis doesn't wait by default
- **Lag window**: Replica may be seconds behind
- **Read-from-replica**: Accept potential stale data
- **Critical writes**: Always write to master

### Sharding Strategies

```
Approach              Pros              Cons              Best For
──────────────────────────────────────────────────────────────
Client-side           Simple            Complex logic     Small scale
Proxy-based           Transparent       Extra component   Medium scale
Redis Cluster         Built-in          Complex           Large scale
```

---

## Production Checklist

Before deploying advanced Redis:

**Replication**
- [ ] Test failover scenarios
- [ ] Monitor replication lag
- [ ] Plan for split-brain scenarios
- [ ] Document recovery procedures

**Sentinel**
- [ ] Configure quorum correctly
- [ ] Monitor sentinel processes
- [ ] Test automatic failover
- [ ] Plan sentinel deployment

**Cluster**
- [ ] Understand key hash slots
- [ ] Plan shard distribution
- [ ] Test node failures
- [ ] Monitor cluster health
- [ ] Plan maintenance windows

**Scripting**
- [ ] Understand atomicity guarantees
- [ ] Test script error handling
- [ ] Monitor script execution time
- [ ] Document all custom scripts

---

## Common Pitfalls

### Replication
❌ Assuming replicas are strongly consistent
❌ Writing to replicas (data loss on failover)
❌ No monitoring of replication lag
❌ Insufficient replicas for failover

### Sentinel
❌ Not testing failover scenarios
❌ Running all sentinels on same network
❌ Incorrect quorum configuration
❌ No client-side failover handling

### Cluster
❌ Using commands that span multiple keys
❌ No handling of slot migration
❌ Insufficient replicas per shard
❌ Not planning for growth

### Scripting
❌ Long-running scripts blocking Redis
❌ Non-deterministic scripts
❌ Assuming script atomicity = no bugs
❌ Not monitoring script performance

---

## Next Steps

Choose your learning path:

1. **[Pub/Sub](2-pubsub.md)** - Real-time messaging
2. **[Scripting](3-scripting.md)** - Atomic server-side operations
3. **[Replication](4-replication.md)** - Data redundancy
4. **[Sentinel](5-sentinel.md)** - Automatic failover
5. **[Cluster](6-cluster.md)** - Horizontal scaling

## Resources

- [Redis Replication Documentation](https://redis.io/topics/replication)
- [Redis Sentinel Documentation](https://redis.io/topics/sentinel)
- [Redis Cluster Documentation](https://redis.io/topics/cluster-tutorial)
- [Lua Scripting Guide](https://redis.io/commands/eval)
