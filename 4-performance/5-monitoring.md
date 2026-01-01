# Monitoring - Production Observability

## Overview

Production monitoring ensures early detection of issues, capacity planning, and SLA compliance.

## Key Metrics

### Latency Metrics

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379)

# Measure command latency
latencies = []
for i in range(10000):
    start = time.time()
    r.get(f'key:{i}')
    latencies.append((time.time() - start) * 1000)  # ms

# Calculate percentiles
sorted_latencies = sorted(latencies)
p50 = sorted_latencies[len(latencies) // 2]
p99 = sorted_latencies[int(len(latencies) * 0.99)]
p999 = sorted_latencies[int(len(latencies) * 0.999)]

print(f"P50: {p50}ms, P99: {p99}ms, P999: {p999}ms")
```

**Good Targets**:
- P50 latency: < 1ms
- P99 latency: < 5ms
- P999 latency: < 10ms

### Throughput Metrics

```bash
# Watch operations per second
redis-cli --stat
# Shows ops/sec, connected clients, memory usage

# Measure with redis-benchmark
redis-benchmark -n 100000 -q
# Output: 51975.10 requests per second
```

**Good Targets**:
- Single server: 50k-100k ops/sec
- Cluster: 1M+ ops/sec

### Memory Metrics

```python
info = r.info('memory')
print(f"Used memory: {info['used_memory_human']}")
print(f"Fragmentation: {info['mem_fragmentation_ratio']}")
print(f"Peak memory: {info['used_memory_peak_human']}")

# Monitor growth
# Fragmentation > 1.5 indicates memory inefficiency
```

**Good Targets**:
- Fragmentation: < 1.1
- Memory utilization: 50-80% of maxmemory

---

## Slowlog Analysis

Identify slow operations:

```python
# Configure slowlog
r.config_set('slowlog-log-slower-than', 1000)  # Log > 1ms
r.config_set('slowlog-max-len', 128)

# Run operations
# ...

# Check slowlog
slow = r.slowlog_get(10)
for i, entry in enumerate(slow):
    id_val = entry['id']
    duration = entry['duration']
    command = ' '.join(str(c) for c in entry['command'][:3])
    print(f"{i}. Duration: {duration}µs, Command: {command}")

# Clear slowlog
r.slowlog_reset()
```

---

## Monitoring Tools

### Redis CLI Monitoring

```bash
# Real-time command monitor
redis-cli monitor
# Prints every command in real-time

# Watch stats
redis-cli --stat
# Updates every 1 second

# Check slowlog
redis-cli slowlog get 10
redis-cli slowlog len
redis-cli slowlog reset
```

### Application Monitoring

```python
from prometheus_client import Counter, Histogram
import time

# Prometheus metrics
command_latency = Histogram(
    'redis_command_latency_ms',
    'Command latency in milliseconds',
    buckets=[0.1, 0.5, 1, 5, 10, 50]
)

command_count = Counter(
    'redis_commands_total',
    'Total commands executed',
    ['command_type']
)

def monitored_get(key):
    with command_latency.time():
        result = r.get(key)
        command_count.labels(command_type='GET').inc()
    return result
```

---

## Production Monitoring Checklist

```
Daily Checks:
- [ ] Error rate < 0.1%
- [ ] P99 latency < 5ms
- [ ] Memory fragmentation < 1.5
- [ ] Replication lag < 100ms
- [ ] Slowlog < 5 entries

Weekly Checks:
- [ ] Capacity planning (memory growth trend)
- [ ] Backup success rate 100%
- [ ] Sentinel health (all monitoring)
- [ ] Cluster slot distribution balanced

Monthly Checks:
- [ ] Performance trend analysis
- [ ] Schema optimization opportunities
- [ ] Eviction policy effectiveness
- [ ] Disaster recovery test
```

---

## Best Practices

### DO: Set Alerts

```python
# ✅ GOOD: Alert thresholds
if p99_latency > 10:
    send_alert("High latency detected")

if memory_fragmentation > 1.5:
    send_alert("Memory fragmentation high, consider restart")

if replication_lag > 1000:  # bytes
    send_alert("Replication lag increasing")
```

### DO: Monitor Replication

```python
# ✅ GOOD: Track replication health
info = r.info('replication')
if info['role'] == 'master':
    slave_count = len([k for k in info.keys() if k.startswith('slave')])
    print(f"Connected slaves: {slave_count}")
```

### DON'T: Ignore Slowlog

```python
# ❌ BAD: Never checking slowlog
# Application mysteriously slow

# ✅ GOOD: Regular slowlog review
slow = r.slowlog_get(100)
# Identify patterns: KEYS pattern, SORT, BRPOP blocking, etc.
```

---

## Common Mistakes

### Mistake 1: Not Monitoring Memory Growth
**Problem**: Server runs out of memory without warning
**Solution**: Track memory trend, set maxmemory alerts

### Mistake 2: Ignoring Replication Lag
**Problem**: Stale reads cause data inconsistency
**Solution**: Monitor lag, use master for critical reads

### Mistake 3: Not Baselining Performance
**Problem**: Can't detect degradation
**Solution**: Establish baseline, monitor trends

### Mistake 4: Reactive vs Proactive
**Problem**: Discover issues only after user impact
**Solution**: Set alerts, monitor continuously

### Mistake 5: Too Many Alerts
**Problem**: Alert fatigue, real issues ignored
**Solution**: Tune alerts to signal-to-noise ratio

---

## Next Steps

1. **Review [Advanced Concepts](../3-advanced-concepts/1-intro.md)** for production patterns
2. **Explore [Integration](../5-integration/1-intro.md)** for end-to-end solutions
3. **Plan deployment** using monitoring checklist

## Resources

- [Redis Monitoring](https://redis.io/docs/management/monitoring)
- [Slowlog Documentation](https://redis.io/commands/slowlog-get)
- [Prometheus Integration](https://prometheus.io/docs/instrumenting/exporters/)
- [Memory Analysis](https://redis.io/commands/memory-stats)
