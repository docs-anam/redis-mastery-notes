# Cluster - Horizontal Scaling and Sharding

## Overview

Redis Cluster distributes data across multiple nodes using consistent hashing, enabling horizontal scaling and fault tolerance.

### Architecture

```
Cluster with 6 nodes (3 primary shards + 3 replicas)

          Client
            |
    ┌───────┼───────┐
    |       |       |
    v       v       v
┌────────┬────────┬────────┐
│Node 1  │Node 2  │Node 3  │ (Primary shards)
│(0-5460)│(5461-  │(10923- │
│        │ 10922) │16383)  │
└────────┴────────┴────────┘
    |       |       |
    v       v       v
┌────────┬────────┬────────┐
│Node 4  │Node 5  │Node 6  │ (Replica shards)
│(replica│(replica│(replica│
│ of 1)  │ of 2)  │ of 3)  │
└────────┴────────┴────────┘
```

### Key Benefits

✅ **Horizontal Scaling**: Add nodes to increase capacity
✅ **Automatic Sharding**: Data distributed by key hash
✅ **High Availability**: Replicas for fault tolerance
✅ **Linear Performance**: More nodes = more throughput
✅ **No Single Point of Failure**: Any node can fail safely

### Limitations

❌ **Complexity**: More moving parts to manage
❌ **Cross-key Operations**: Limited MGET/MSET across slots
❌ **Transactions**: MULTI/EXEC limited to single slot
❌ **Learning Curve**: Different semantics than single Redis

---

## Core Concepts

### Hash Slots

Redis Cluster uses 16,384 hash slots:

```
Slot = CRC16(key) % 16384

Example:
key = "user:123"
CRC16("user:123") = 12345
12345 % 16384 = 12345 (slot)
↓
Route to node owning slot 12345
```

### Slot Distribution

```
Node 1: Slots 0-5460
Node 2: Slots 5461-10922
Node 3: Slots 10923-16383

Total: 16384 slots (covers all possible values)
```

### Hash Tags

Force keys to same slot:

```
key = "user:123:{account}"
     → Uses CRC16 of "account" only
     
All keys with same tag go to same slot:
"user:123:{tag}" → Slot X
"user:456:{tag}" → Slot X  (same!)
"order:789:{tag}" → Slot X (same!)
```

---

## Setup

### Step 1: Create Configuration

```
# redis-6379.conf
port 6379
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000
databases 16

# Repeat for 6380-6384
```

### Step 2: Start Nodes

```bash
for port in 6379 6380 6381 6382 6383 6384; do
    redis-server redis-${port}.conf &
done
```

### Step 3: Create Cluster

```bash
# Redis 5.0+
redis-cli --cluster create \
    127.0.0.1:6379 \
    127.0.0.1:6380 \
    127.0.0.1:6381 \
    127.0.0.1:6382 \
    127.0.0.1:6383 \
    127.0.0.1:6384 \
    --cluster-replicas 1
```

### Step 4: Verify

```bash
redis-cli -p 6379
> CLUSTER INFO
> CLUSTER NODES
```

---

## Commands Reference

### CLUSTER INFO
Get cluster status.

```redis
CLUSTER INFO
# Shows: cluster_state, cluster_slots_assigned, etc.
```

```python
import redis

r = redis.RedisCluster(startup_nodes=[
    {'host': '127.0.0.1', 'port': 6379}
])

info = r.cluster_info()
print(f"State: {info['cluster_state']}")
```

### CLUSTER NODES
List all nodes.

```redis
CLUSTER NODES
# Shows: node-id, address, role, slots, etc.
```

```python
nodes = r.cluster_nodes()
```

### CLUSTER SLOTS
Get slot-to-node mapping.

```redis
CLUSTER SLOTS
# Returns: [[start, end, [ip, port]], ...]
```

```python
slots = r.execute_command('CLUSTER', 'SLOTS')
```

### CLUSTER MEET
Add node to cluster.

```redis
CLUSTER MEET 127.0.0.1 6385
```

```python
r.execute_command('CLUSTER', 'MEET', '127.0.0.1', 6385)
```

---

## Practical Examples

### Example 1: Basic Cluster Connection

```python
import redis
from rediscluster import RedisCluster

class ClusterConnection:
    def __init__(self, nodes):
        self.cluster = RedisCluster(
            startup_nodes=nodes,
            decode_responses=True
        )
    
    def set_key(self, key, value):
        """SET automatically routed to correct node"""
        self.cluster.set(key, value)
    
    def get_key(self, key):
        """GET automatically routed to correct node"""
        return self.cluster.get(key)
    
    def cluster_status(self):
        """Get cluster health"""
        info = self.cluster.cluster_info()
        return {
            'state': info['cluster_state'],
            'slots_assigned': info.get('cluster_slots_assigned', 0),
            'nodes': self.cluster.cluster_nodes()
        }

# Usage
nodes = [
    {'host': '127.0.0.1', 'port': 6379},
    {'host': '127.0.0.1', 'port': 6380},
    {'host': '127.0.0.1', 'port': 6381}
]

conn = ClusterConnection(nodes)
conn.set_key('user:123', 'Alice')
print(conn.get_key('user:123'))
print(conn.cluster_status())
```

### Example 2: Hash Tag Based Grouping

```python
import redis

class TaggedCluster:
    def __init__(self):
        self.cluster = redis.RedisCluster(
            startup_nodes=[{'host': '127.0.0.1', 'port': 6379}]
        )
    
    def set_user_data(self, user_id, field, value):
        """Store user data in hash with tag"""
        key = f"user:{user_id}:{{{user_id}}}"  # Tag: {user_id}
        self.cluster.hset(key, field, value)
    
    def get_user_data(self, user_id):
        """Get all user data"""
        key = f"user:{user_id}:{{{user_id}}}"
        return self.cluster.hgetall(key)
    
    def atomic_user_update(self, user_id, updates):
        """Atomic update with HASH TAGS"""
        # All keys with same tag go to same slot
        key = f"user:{user_id}:{{{user_id}}}"
        
        pipe = self.cluster.pipeline(transaction=False)
        for field, value in updates.items():
            pipe.hset(key, field, value)
        
        pipe.execute()

# Usage
tagged = TaggedCluster()

tagged.set_user_data(123, 'name', 'Alice')
tagged.set_user_data(123, 'email', 'alice@example.com')

user = tagged.get_user_data(123)
print(user)
```

### Example 3: Cluster Monitoring

```python
import redis
import time
from datetime import datetime

class ClusterMonitor:
    def __init__(self, nodes):
        self.cluster = redis.RedisCluster(
            startup_nodes=nodes,
            skip_full_coverage_check=True
        )
    
    def get_node_info(self):
        """Get info from each node"""
        nodes_info = {}
        
        for node in self.cluster.nodes:
            try:
                info = self.cluster.execute_command(
                    'INFO', node_id=node.name
                )
                nodes_info[node.name] = {
                    'used_memory': info.get('used_memory_human'),
                    'connected_clients': info.get('connected_clients'),
                    'ops_per_sec': info.get('instantaneous_ops_per_sec')
                }
            except:
                nodes_info[node.name] = {'status': 'unavailable'}
        
        return nodes_info
    
    def check_slots(self):
        """Verify all slots assigned"""
        slots_info = self.cluster.execute_command('CLUSTER', 'SLOTS')
        total_slots = 0
        
        for slot_range in slots_info:
            start, end = slot_range[0], slot_range[1]
            total_slots += (end - start + 1)
        
        return {
            'total_slots': total_slots,
            'coverage': (total_slots / 16384) * 100,
            'complete': total_slots == 16384
        }

# Usage
monitor = ClusterMonitor([{'host': '127.0.0.1', 'port': 6379}])

print("Node Info:")
print(monitor.get_node_info())

print("\nSlot Coverage:")
print(monitor.check_slots())
```

---

## Best Practices

### DO: Use Hash Tags for Related Data

```python
# ✅ GOOD: All user data together
r.set(f"user:123:{{{123}}}:name", "Alice")
r.set(f"user:123:{{{123}}}:email", "alice@example.com")
# Both in same slot, can use MULTI/EXEC

# ❌ BAD: Different slots
r.set("user:123:name", "Alice")
r.set("user:123:email", "alice@example.com")
# Different slots, can't use MULTI/EXEC
```

### DO: Plan for Growth

```
Current: 3 nodes, 5460 slots each
Future:  6 nodes, 2730 slots each

Plan:
1. Add 3 new nodes
2. Rebalance slots (takes time)
3. Monitor for performance impact
```

### DO: Monitor Cluster Health

```python
# ✅ GOOD: Regular health checks
def health_check():
    info = cluster.cluster_info()
    if info['cluster_state'] != 'ok':
        alert("Cluster degraded!")
    
    # Check for migrating slots
    nodes = cluster.cluster_nodes()
    for node in nodes:
        if node['migrating_slots']:
            alert(f"Slots migrating on {node['name']}")
```

### DON'T: Cross-Slot Transactions

```python
# ❌ BAD: Keys in different slots
pipe = cluster.pipeline(transaction=True)
pipe.set('key1', 'value1')  # Slot A
pipe.set('key2', 'value2')  # Slot B
pipe.execute()  # ERROR!

# ✅ GOOD: Same slot only
pipe = cluster.pipeline(transaction=True)
pipe.set('key:{tag}:1', 'value1')
pipe.set('key:{tag}:2', 'value2')
pipe.execute()  # OK!
```

### DON'T: Use KEYS or SCAN

```python
# ❌ BAD: KEYS not supported in cluster
keys = cluster.keys('*')  # ERROR!

# ✅ GOOD: Use CLUSTER GETKEYSINSLOT
for slot in range(16384):
    keys = cluster.execute_command('CLUSTER', 'GETKEYSINSLOT', slot, 10)
```

---

## Common Mistakes

### Mistake 1: Ignoring Hash Tags

**Problem**: Data scattered across nodes
```python
# ❌ BAD: User data in different slots
r.set('user:123:name', 'Alice')       # Slot X
r.hset('user:123:profile', ...)       # Slot Y
r.zadd('user:123:scores', ...)        # Slot Z

# Can't use transaction, must check each separately
```

**Solution**: Use hash tags
```python
# ✅ GOOD: All user data in same slot
r.set('user:123:{123}:name', 'Alice')
r.hset('user:123:{123}:profile', ...)
r.zadd('user:123:{123}:scores', ...)
# All routed to same slot, can use MULTI/EXEC
```

### Mistake 2: Not Preparing for Rebalancing

**Problem**: Slow cluster during rebalancing
```
Add node -> Redis rebalances
-> Slots migrating
-> Increased latency
-> No notice to clients
```

**Solution**: Plan and monitor
```python
# Plan rebalancing during low traffic
# Disable pipelining during migration
# Monitor migrating_slots metric
```

### Mistake 3: Single Replica

**Problem**: Node failure loses data
```
3 nodes, 0 replicas
Any node fails -> data loss
```

**Solution**: Deploy with replicas
```
--cluster-replicas 1  # 1 replica per primary
```

### Mistake 4: Not Using Cluster Mode

**Problem**: Cluster client doesn't handle slot migration
```python
# ❌ BAD: Regular redis-py
r = redis.Redis(...)  # Doesn't know about cluster

# ✅ GOOD: Cluster-aware client
r = redis.RedisCluster(...)  # Handles rebalancing
```

### Mistake 5: Wrong Cluster Node Count

**Problem**: Odd number of nodes causes issues
```
5 nodes -> 3 handle slots, 2 failover
Load uneven
```

**Solution**: Even primary count
```
6 nodes: 3 primary + 3 replica (ideal)
9 nodes: 3 primary shards, 2 replicas each
```

---

## Migration Strategies

### Strategy 1: Online Resharding

```bash
# Add new node
redis-cli --cluster add-node 127.0.0.1:6385 127.0.0.1:6379

# Reshard slots
redis-cli --cluster reshard 127.0.0.1:6379 \
    --cluster-from <source-node-id> \
    --cluster-to <dest-node-id> \
    --cluster-slots <number>
```

### Strategy 2: Gradual Migration

```
Day 1: Add nodes (no slot migration)
Day 2: Migrate 10% slots
Day 3: Migrate 20% slots
...
Wait between migrations to monitor
```

---

## Next Steps

1. **Learn [Performance](../4-performance/1-intro.md)** tuning for cluster
2. **Explore [Monitoring](../1-basics/11-server-information.md)** at scale
3. **Study [Persistence](../1-basics/15-persistence.md)** with cluster
4. **Review [Security](../1-basics/14-security.md)** best practices

## Resources

- [Redis Cluster Official](https://redis.io/topics/cluster-tutorial)
- [Cluster Specifications](https://redis.io/topics/cluster-spec)
- [Cluster Commands](https://redis.io/commands/cluster-info)
- [Resharding Guide](https://redis.io/topics/cluster-tutorial#resharding)
