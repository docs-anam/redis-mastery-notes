# Scaling - Distributing Load

## Overview

Scaling enables handling 10-100x more traffic by distributing data and load across multiple nodes.

## Scaling Approaches

### Vertical Scaling (Single Node Optimization)

Maximize single server resources:

```
Strategy              Improvement   Cost      Complexity
─────────────────────────────────────────────────────
Faster CPU           1.5-2x        Medium    Low
More Memory          Variable      Low       Low
SSD Storage          3-5x          Medium    Low
Network upgrades     1.5-2x        Medium    Low
```

**Limits**: Single core bottleneck (~100k ops/sec)

### Horizontal Scaling (Multiple Nodes)

Distribute data and traffic:

```
Approach              Data          Read      Write      Setup Time
──────────────────────────────────────────────────────────────────
Replication          Copy          Scale     Single     5 min
Cluster              Sharded       Scale     Scale      30 min
Sentinel             Copy + HA     Scale     Single     15 min
```

---

## Replication-Based Scaling

Master handles writes, replicas handle reads:

```python
import redis

# Connect to master for writes
master = redis.Redis(host='master.redis.com', port=6379)

# Connect to replicas for reads (round-robin)
read_replicas = [
    redis.Redis(host='replica1.redis.com', port=6379),
    redis.Redis(host='replica2.redis.com', port=6379),
    redis.Redis(host='replica3.redis.com', port=6379)
]

# Write to master
master.set('user:123', 'Alice')

# Read from replicas (may be slightly stale)
from random import choice
user = choice(read_replicas).get('user:123')
```

**Benefits**: Simple, read scaling, high availability
**Trade-off**: Replication lag, single master bottleneck

---

## Cluster-Based Scaling

Data partitioned across nodes via hash slots:

```python
from rediscluster import RedisCluster

# Define cluster nodes
startup_nodes = [
    {"host": "node1", "port": 6379},
    {"host": "node2", "port": 6379},
    {"host": "node3", "port": 6379},
]

# Create cluster connection
rc = RedisCluster(startup_nodes=startup_nodes)

# Data automatically routed to correct slot
rc.set('user:123', 'Alice')  # Slot = CRC16('user:123') % 16384
rc.get('user:123')

# Hash tags for related keys on same node
rc.mset({'user:123:name': 'Alice', 'user:123:email': 'alice@example.com'})
# Both keys hash to same slot due to {user:123}
```

**Benefits**: Linear scaling, true distributed writes/reads
**Trade-off**: Complexity, multi-key operations harder

---

## Scaling Strategy Selection

```
Scenario                   Best Approach       Rationale
─────────────────────────────────────────────────────────
< 100k ops/sec            Single + Vertical   Cost-effective
100k-1M ops/sec           Replication         Simpler than Cluster
> 1M ops/sec              Cluster             Needed for true scale
Write-heavy               Cluster             Master can't scale
Read-heavy                Replication         Cheaper than Cluster
High availability         Sentinel + Rep      Auto-failover
```

---

## Performance Characteristics

| Approach | Read | Write | HA | Setup |
|----------|------|-------|----|----|
| Single | 100k | 100k | No | Trivial |
| Replication | 300k+ | 100k | Yes | Simple |
| Cluster | 1M+ | 1M+ | Yes | Complex |

---

## Best Practices

### DO: Use Cluster for Write Scaling
```python
# ✅ GOOD: Cluster scales both reads and writes
rc = RedisCluster(startup_nodes=nodes)
```

### DO: Monitor Replication Lag
```python
# ✅ GOOD: Check lag before reading
info = r.info('replication')
lag = info['master_repl_offset'] - info['slave_repl_offset']
if lag > 1000:  # 1000 bytes lag
    read_from_master()
```

### DON'T: Cross-Slot Transactions
```python
# ❌ BAD: Keys on different slots
rc.mset({'key1': 'val1', 'key2': 'val2'})  # Will fail!

# ✅ GOOD: Use hash tags
rc.mset({'{tag}:key1': 'val1', '{tag}:key2': 'val2'})
```

---

## Common Mistakes

### Mistake 1: Cluster Too Early
**Problem**: Added complexity before needed
**Solution**: Start with replication, scale to cluster if needed

### Mistake 2: Not Planning Hash Tags
**Problem**: Cross-slot failures in transactions
**Solution**: Design key schema with tags upfront

### Mistake 3: Ignoring Replication Lag
**Problem**: Reading stale data for critical decisions
**Solution**: Monitor lag, use master for critical reads

### Mistake 4: Unbalanced Cluster
**Problem**: Some nodes overloaded, others idle
**Solution**: Regular rebalancing, hash distribution analysis

---

## Next Steps

1. **Learn [Monitoring](5-monitoring.md)** for production observability
2. **Review [Advanced Concepts](../3-advanced-concepts/1-intro.md)** for HA
3. **Explore [Integration Patterns](../5-integration/1-intro.md)**

## Resources

- [Redis Replication](https://redis.io/topics/replication)
- [Redis Cluster](https://redis.io/topics/cluster-tutorial)
- [Scaling Strategies](https://redis.io/topics/scaling)
