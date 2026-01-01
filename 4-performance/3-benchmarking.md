# Benchmarking - Measuring and Profiling

## Overview

Systematic benchmarking identifies bottlenecks and validates optimizations. Cannot optimize what you don't measure.

## Tools

### redis-benchmark

Built-in benchmarking tool:

```bash
# Basic benchmark
redis-benchmark -n 100000

# With pipelining
redis-benchmark -n 100000 -P 16

# GET operations only
redis-benchmark -t get -n 100000

# Against specific server
redis-benchmark -h redis.example.com -p 6379 -n 100000
```

### Custom Benchmarking

```python
import redis
import time

class Benchmarker:
    def __init__(self):
        self.r = redis.Redis(host='localhost', port=6379)
    
    def benchmark_operation(self, operation, iterations=10000):
        """Benchmark single operation"""
        start = time.time()
        for i in range(iterations):
            operation(i)
        elapsed = time.time() - start
        
        ops_per_sec = iterations / elapsed
        latency_ms = (elapsed * 1000) / iterations
        
        return {
            'ops_per_sec': ops_per_sec,
            'latency_ms': latency_ms,
            'total_time': elapsed
        }

# Measure GET performance
benchmarker = Benchmarker()
results = benchmarker.benchmark_operation(
    lambda i: benchmarker.r.get(f'key:{i}')
)
print(f"GET: {results['ops_per_sec']:.0f} ops/sec")
print(f"Latency: {results['latency_ms']:.3f}ms")
```

---

## Profiling

### Identify Slow Operations

```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Enable slowlog
r.config_set('slowlog-log-slower-than', 1000)  # 1ms
r.config_set('slowlog-max-len', 128)

# Run operations
# ...

# Check slowlog
slow = r.slowlog_get(10)
for entry in slow:
    print(f"Duration: {entry['duration']}µs")
    print(f"Command: {entry['command']}")
```

### Memory Profiling

```python
# Get memory usage
info = r.info('memory')
print(f"Used memory: {info['used_memory_human']}")
print(f"Fragmentation: {info['mem_fragmentation_ratio']}")

# Identify large keys
doctor = r.execute_command('MEMORY', 'DOCTOR')
print(doctor)
```

---

## Performance Benchmarks

### Typical Numbers

| Operation | Throughput | Latency |
|-----------|-----------|---------|
| GET | 100k ops/sec | 0.5ms |
| SET | 100k ops/sec | 0.5ms |
| INCR | 100k ops/sec | 0.5ms |
| LPUSH | 50k ops/sec | 1ms |
| SADD | 50k ops/sec | 1ms |
| HSET | 50k ops/sec | 1ms |

### Optimized Numbers (Pipelined)

| Operation | Throughput | Latency |
|-----------|-----------|---------|
| GET | 500k+ ops/sec | 0.1ms |
| SET | 500k+ ops/sec | 0.1ms |
| Mixed | 300k+ ops/sec | 0.2ms |

---

## Best Practices

### DO: Test Before Production

```python
# ✅ GOOD: Benchmark before deploying
baseline = benchmarker.benchmark_operation(operation)
assert baseline['latency_ms'] < 1.0, "Latency too high"
```

### DO: Monitor in Production

```python
# ✅ GOOD: Continuous latency tracking
latencies = []
for i in range(1000):
    start = time.time()
    r.get(f'key:{i}')
    latencies.append((time.time() - start) * 1000)

p99 = sorted(latencies)[int(len(latencies) * 0.99)]
print(f"P99 latency: {p99}ms")
```

### DON'T: Benchmark Unrealistic Workload

```bash
# ❌ BAD: All GET operations
redis-benchmark -t get

# ✅ GOOD: Mixed realistic workload
redis-benchmark -t get,set -q
```

---

## Common Mistakes

### Mistake 1: Benchmarking Only Reads
**Problem**: Write performance unknown until production
**Solution**: Test mixed workloads (80/20, 50/50)

### Mistake 2: Ignoring Network Latency
**Problem**: Local benchmarks unrealistic
**Solution**: Test from actual application server location

### Mistake 3: Not Considering Value Size
**Problem**: Small values benchmark unrealistically
**Solution**: Use realistic value sizes

### Mistake 4: No Baseline
**Problem**: Can't detect degradation
**Solution**: Establish baseline, monitor trends

---

## Next Steps

1. **Continue [Scaling](4-scaling.md)** for larger deployments
2. **Learn [Monitoring](5-monitoring.md)** for production
3. **Review [Advanced Concepts](../3-advanced-concepts/1-intro.md)**

## Resources

- [Redis Benchmarking Guide](https://redis.io/topics/benchmarking)
- [Slowlog Documentation](https://redis.io/commands/slowlog-get)
- [Memory Analysis](https://redis.io/commands/memory-stats)
