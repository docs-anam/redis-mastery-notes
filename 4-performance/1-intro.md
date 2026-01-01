# Performance - Optimization and Scaling

## Table of Contents
1. [Overview](#overview)
2. [Performance Metrics](#performance-metrics)
3. [Optimization Layers](#optimization-layers)
4. [Learning Paths](#learning-paths)
5. [Benchmarking Strategy](#benchmarking-strategy)

---

## Overview

Redis performance optimization spans multiple layers: application design, network, commands, data structures, configuration, and infrastructure.

### Performance Levels

```
Layer 1: Application Design
  ↓ (use right patterns)
Layer 2: Data Structure Selection
  ↓ (choose optimal structures)
Layer 3: Command Optimization
  ↓ (use efficient commands)
Layer 4: Network Tuning
  ↓ (reduce latency)
Layer 5: Configuration
  ↓ (tune parameters)
Layer 6: Infrastructure
  ↓ (scale horizontally)
Maximum Performance
```

### Key Metrics

```
Metric              Good            Excellent        Target
────────────────────────────────────────────────────────────
Latency (p99)       <5ms            <1ms            <500µs
Throughput          50k ops/sec     100k ops/sec    1M ops/sec
CPU Usage           <70%            <50%            <30%
Memory Hit Rate     >99%            >99.9%          >99.99%
Network Util        <60%            <40%            <20%
```

### Common Bottlenecks

```
Bottleneck              Symptom              Solution
─────────────────────────────────────────────────────
Network latency         High RTT             Pipelining
Memory fragmentation    High memory usage    Allocation tuning
CPU bound              100% CPU             Split keys
Network bound          Constant traffic     Compression
Slow commands          High latency         SCAN instead of KEYS
Large values           Memory spike         Serialization
```

---

## Performance Metrics

### Latency

```
Latency Type        Typical         Acceptable      Target
───────────────────────────────────────────────────────
Local network       0.1-0.5ms       <1ms            <100µs
Same datacenter     1-5ms           <10ms           <1ms
Cross-region        50-200ms        <500ms          <100ms
```

### Throughput

```
Scenario                    Typical      Optimized
──────────────────────────────────────────────
Single client              5-10k ops/sec   50k ops/sec
100 pipelined commands     50k ops/sec     500k ops/sec
Cluster (3 shards)         100k ops/sec    1M+ ops/sec
```

### Memory Efficiency

```
Metric                  Measurement      Optimization
──────────────────────────────────────────────────────
Memory per key          100-500 bytes    20-50 bytes
Fragmentation ratio     1.1-1.5x         1.01-1.05x
Eviction rate           <1%              ~0%
```

---

## Optimization Layers

### Layer 1: Application Design
- ✅ Use pipelining for batch operations
- ✅ Cache frequently accessed data
- ✅ Batch operations into transactions
- ✅ Use connection pooling

### Layer 2: Data Structure Selection
- ✅ Strings for simple values
- ✅ Hashes for objects
- ✅ Sets for uniqueness
- ✅ Sorted Sets for rankings
- ✅ Lists for queues
- ✅ Streams for events

### Layer 3: Command Optimization
- ✅ Use GET vs MGET appropriately
- ✅ Replace KEYS with SCAN
- ✅ Batch DEL operations
- ✅ Use bit operations for flags

### Layer 4: Network Tuning
- ✅ Enable TCP_NODELAY
- ✅ Tune SO_KEEPALIVE
- ✅ Increase receive buffer
- ✅ Use local connections

### Layer 5: Configuration Tuning
- ✅ Set maxmemory correctly
- ✅ Choose eviction policy
- ✅ Tune slowlog threshold
- ✅ Set timeout appropriately

### Layer 6: Infrastructure
- ✅ Use SSD storage
- ✅ Adequate memory (full dataset)
- ✅ Multi-core servers
- ✅ Fast network

---

## Learning Paths

### Path 1: Application Optimization (2-3 days)
Focus: Client-side improvements without changing Redis

```
1. Pipelining → Batch operations
2. Connection pooling → Reuse connections
3. Lua scripting → Reduce round-trips
4. Batching → Group operations
Goal: 5-10x performance improvement
```

### Path 2: Command Optimization (2-3 days)
Focus: Choosing faster commands and patterns

```
1. Data structure selection → Right tool for job
2. Command efficiency → MGET vs GET
3. Scanning → SCAN instead of KEYS
4. Transactions → MULTI/EXEC
Goal: 2-5x performance improvement
```

### Path 3: System Tuning (3-4 days)
Focus: Configuration and infrastructure changes

```
1. Configuration → Tune parameters
2. Memory optimization → Eviction policies
3. Persistence → RDB vs AOF
4. Replication → Master-replica setup
Goal: 20-50% performance improvement
```

### Path 4: Scaling (4-5 days)
Focus: Distributing load across multiple Redis instances

```
1. Replication → Read scaling
2. Cluster → Horizontal scaling
3. Sentinel → High availability
4. Monitoring → Production readiness
Goal: Linear scaling with more nodes
```

---

## Benchmarking Strategy

### Step 1: Baseline Measurement
```bash
# Measure current performance
redis-benchmark -h localhost -p 6379 -n 100000

# With pipelining
redis-benchmark -h localhost -p 6379 -n 100000 -P 16
```

### Step 2: Identify Bottleneck
```
Network?   → Pipelining, compression
CPU?       → Optimize commands, Lua scripts
Memory?    → Eviction policy, data structure
Disk?      → Async writes, persistence config
```

### Step 3: Apply Optimization
```
Change one variable at a time
Measure improvement
Document results
```

### Step 4: Monitor Production
```
Track latency (p50, p99, p999)
Monitor memory usage
Watch CPU utilization
Check network throughput
```

---

## Performance Profile

Before diving into optimization, understand your workload:

```
Workload Type       Write %    Value Size    Key Count
──────────────────────────────────────────────────────
Cache (hot)         80%        Small         <1M
Analytics           5%         Large         >100M
Time-series         90%        Medium        <10M
Sessions            60%        Small         10-100M
User data           40%        Medium        1-10M
```

---

## Quick Wins

Immediate improvements (no code changes):

1. **Enable Pipelining**
   - 5-10x improvement for batch operations
   - Works with existing code

2. **Use Connection Pooling**
   - Reduces connection overhead
   - Simple configuration change

3. **Tune Slowlog**
   - Identify problematic commands
   - No performance cost

4. **Increase Maxmemory**
   - Reduce evictions
   - Requires adequate RAM

5. **Enable Persistence Async**
   - Reduces write latency
   - Minimal data loss risk

---

## Topics in This Section

1. **[1-intro.md](1-intro.md)** - Overview and strategies (this file)
2. **[2-optimization.md](2-optimization.md)** - Application-level improvements
3. **[3-benchmarking.md](3-benchmarking.md)** - Measuring and profiling
4. **[4-scaling.md](4-scaling.md)** - Distributed systems scaling
5. **[5-monitoring.md](5-monitoring.md)** - Production observability

---

## Next Steps

Choose your learning path:

1. **Application focus** → Start with [2-optimization.md](2-optimization.md)
2. **Benchmarking focus** → Start with [3-benchmarking.md](3-benchmarking.md)
3. **Infrastructure focus** → Start with [4-scaling.md](4-scaling.md)
4. **Production focus** → Start with [5-monitoring.md](5-monitoring.md)

## Resources

- [Redis Performance](https://redis.io/topics/performance)
- [Redis Benchmarking](https://redis.io/topics/benchmarking)
- [Latency Monitoring](https://redis.io/topics/latency-monitor)
