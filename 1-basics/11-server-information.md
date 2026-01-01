# Redis Server Information (INFO Command)

## Overview

The INFO command provides comprehensive server statistics covering memory usage, performance, replication, persistence, and more. Essential for monitoring and diagnostics.

## Basic INFO Command

### Server Section

```redis
INFO server

# Output:
# redis_version:7.0.0
# redis_mode:standalone
# os:Linux 5.10.0
# arch_bits:64
# gcc_version:11.2.0
# process_id:1234
# uptime_in_seconds:86400
# uptime_in_days:1
```

### Getting All Information

```redis
# Get all sections
INFO

# Get specific section
INFO memory
INFO stats
INFO clients
INFO replication

# Get multiple sections
INFO memory stats clients
```

## Key Sections

### Memory Information

```python
import redis

r = redis.Redis()

mem = r.info('memory')

print(f"Used memory: {mem['used_memory_human']}")
print(f"Peak memory: {mem['used_memory_peak_human']}")
print(f"Total memory: {mem['total_system_memory_human']}")
print(f"Fragmentation: {mem['mem_fragmentation_ratio']:.2f}")
print(f"Fragmentation bytes: {mem['mem_fragmentation_bytes']}")
```

### Client Information

```python
import redis

r = redis.Redis()

clients = r.info('clients')

print(f"Connected clients: {clients['connected_clients']}")
print(f"Blocked clients: {clients['blocked_clients']}")
print(f"Max connections: {clients['maxclients']}")
```

### Statistics

```python
import redis

r = redis.Redis()

stats = r.info('stats')

print(f"Total commands: {stats['total_commands_processed']}")
print(f"Connections received: {stats['total_connections_received']}")
print(f"Operations/sec: {stats['instantaneous_ops_per_sec']}")
print(f"Evicted keys: {stats['evicted_keys']}")
print(f"Expired keys: {stats['expired_keys']}")
print(f"Keyspace hits: {stats['keyspace_hits']}")
print(f"Keyspace misses: {stats['keyspace_misses']}")
```

### Keyspace Information

```python
import redis

r = redis.Redis()

keyspace = r.info('keyspace')

for db_name, db_info in keyspace.items():
    if isinstance(db_info, dict):
        print(f"{db_name}: {db_info['keys']} keys, {db_info['expires']} expiring")
        
        # Calculate hit rate
        hits = db_info.get('hits', 0)
        misses = db_info.get('misses', 0)
        if hits + misses > 0:
            hit_rate = hits / (hits + misses) * 100
            print(f"  Hit rate: {hit_rate:.1f}%")
```

### Replication Information

```python
import redis

r = redis.Redis()

rep = r.info('replication')

print(f"Role: {rep['role']}")  # 'master' or 'slave'
print(f"Connected slaves: {rep['connected_slaves']}")
print(f"Master repl offset: {rep['master_repl_offset']}")
```

## Real-World Examples

### Example 1: Health Check Script

```python
import redis
import json

def redis_health_check():
    """Check Redis health"""
    r = redis.Redis()
    
    health = {
        'status': 'healthy',
        'issues': []
    }
    
    # Check connection
    try:
        r.ping()
    except:
        health['status'] = 'critical'
        health['issues'].append('Cannot connect to Redis')
        return health
    
    # Check memory usage
    mem = r.info('memory')
    used_pct = mem['used_memory'] / mem['total_system_memory'] * 100
    if used_pct > 80:
        health['status'] = 'warning'
        health['issues'].append(f'High memory usage: {used_pct:.1f}%')
    
    # Check eviction
    stats = r.info('stats')
    if stats['evicted_keys'] > 0:
        health['status'] = 'warning'
        health['issues'].append(f"Keys evicted: {stats['evicted_keys']}")
    
    # Check cache hit rate
    hits = stats['keyspace_hits']
    misses = stats['keyspace_misses']
    if hits + misses > 0:
        hit_rate = hits / (hits + misses)
        if hit_rate < 0.8:
            health['issues'].append(f'Low hit rate: {hit_rate*100:.1f}%')
    
    return health

status = redis_health_check()
print(json.dumps(status, indent=2))
```

### Example 2: Performance Monitoring

```python
import redis
import time

def monitor_performance():
    """Monitor Redis performance metrics"""
    r = redis.Redis()
    
    # Get baseline
    stats1 = r.info('stats')
    time1 = time.time()
    
    # Wait a bit
    time.sleep(5)
    
    # Get new stats
    stats2 = r.info('stats')
    time2 = time.time()
    
    # Calculate rates
    elapsed = time2 - time1
    cmd_diff = stats2['total_commands_processed'] - stats1['total_commands_processed']
    cmds_per_sec = cmd_diff / elapsed
    
    # Get current operations/sec
    ops_per_sec = stats2['instantaneous_ops_per_sec']
    
    # Memory info
    mem = r.info('memory')
    
    print(f"Commands/sec: {cmds_per_sec:.0f}")
    print(f"Instantaneous ops/sec: {ops_per_sec}")
    print(f"Memory used: {mem['used_memory_human']}")
    print(f"Memory fragmentation: {mem['mem_fragmentation_ratio']:.2f}")

monitor_performance()
```

### Example 3: Capacity Planning

```python
import redis

def capacity_analysis():
    """Analyze capacity and plan scaling"""
    r = redis.Redis()
    
    mem = r.info('memory')
    stats = r.info('stats')
    keyspace = r.info('keyspace')
    
    # Count total keys
    total_keys = sum(
        db['keys'] for db in keyspace.values() 
        if isinstance(db, dict)
    )
    
    # Average bytes per key
    avg_bytes = mem['used_memory'] / max(total_keys, 1)
    
    # Growth rate
    uptime = mem['uptime_in_seconds']
    cmds = stats['total_commands_processed']
    cmds_per_sec = cmds / uptime if uptime > 0 else 0
    
    # Projections
    memory_limit = 1024 * 1024 * 1024  # 1GB
    available = memory_limit - mem['used_memory']
    years_to_full = (available / (avg_bytes * cmds_per_sec * 86400)) / 365
    
    print(f"Total keys: {total_keys}")
    print(f"Average bytes/key: {avg_bytes:.0f}")
    print(f"Commands/sec: {cmds_per_sec:.0f}")
    print(f"Years until memory full: {years_to_full:.1f}")

capacity_analysis()
```

### Example 4: Export Metrics

```python
import redis
import csv
from datetime import datetime

def export_metrics():
    """Export Redis metrics to CSV"""
    r = redis.Redis()
    
    info = r.info()
    timestamp = datetime.now().isoformat()
    
    with open('redis_metrics.csv', 'a', newline='') as f:
        writer = csv.writer(f)
        
        # Header
        writer.writerow(['timestamp', 'metric', 'value'])
        
        # Memory metrics
        mem = info['memory']
        writer.writerow([timestamp, 'used_memory', mem['used_memory']])
        writer.writerow([timestamp, 'used_memory_peak', mem['used_memory_peak']])
        
        # Stats
        stats = info['stats']
        writer.writerow([timestamp, 'total_commands', stats['total_commands_processed']])
        writer.writerow([timestamp, 'evicted_keys', stats['evicted_keys']])

export_metrics()
```

## Performance Analysis

### Cache Hit Ratio

```python
import redis

def calculate_hit_ratio():
    """Calculate cache hit ratio"""
    r = redis.Redis()
    
    stats = r.info('stats')
    
    hits = stats['keyspace_hits']
    misses = stats['keyspace_misses']
    total = hits + misses
    
    if total == 0:
        ratio = 0
    else:
        ratio = hits / total
    
    print(f"Hits: {hits}")
    print(f"Misses: {misses}")
    print(f"Total: {total}")
    print(f"Hit ratio: {ratio*100:.1f}%")
    
    # Interpretation
    if ratio > 0.9:
        print("Status: Excellent")
    elif ratio > 0.75:
        print("Status: Good")
    elif ratio > 0.5:
        print("Status: Fair - consider optimization")
    else:
        print("Status: Poor - significant room for improvement")

calculate_hit_ratio()
```

### Memory Efficiency

```python
import redis

def analyze_memory_efficiency():
    """Analyze memory usage efficiency"""
    r = redis.Redis()
    
    mem = r.info('memory')
    keyspace = r.info('keyspace')
    
    # Calculate metrics
    used = mem['used_memory']
    peak = mem['used_memory_peak']
    fragmentation = mem['mem_fragmentation_ratio']
    
    # Count keys
    total_keys = sum(
        db['keys'] for db in keyspace.values() 
        if isinstance(db, dict)
    )
    
    bytes_per_key = used / max(total_keys, 1)
    
    print(f"Memory usage: {mem['used_memory_human']}")
    print(f"Peak memory: {mem['used_memory_peak_human']}")
    print(f"Fragmentation ratio: {fragmentation:.2f}")
    
    if fragmentation > 1.5:
        print("⚠️  High fragmentation - consider restarting")
    
    print(f"Bytes per key: {bytes_per_key:.0f}")
    print(f"Total keys: {total_keys}")

analyze_memory_efficiency()
```

## Interpretation Guide

| Metric | Meaning | Target |
|--------|---------|--------|
| used_memory | Total memory consumed | < 80% of maxmemory |
| mem_fragmentation_ratio | Fragmentation factor | < 1.5 |
| connected_clients | Active connections | < maxclients |
| instantaneous_ops_per_sec | Operations/sec | Depends on load |
| keyspace_hits | Cache lookups succeeded | High |
| keyspace_misses | Cache lookups failed | Low |
| evicted_keys | Keys removed by eviction | 0 (if possible) |
| expired_keys | Keys that expired | Varies |

## Best Practices

### 1. Monitor Regularly

```python
import redis
import time

def continuous_monitoring(interval=60):
    """Monitor Redis continuously"""
    r = redis.Redis()
    
    while True:
        info = r.info()
        
        # Check critical metrics
        mem = info['memory']
        used_pct = mem['used_memory'] / mem['total_system_memory'] * 100
        
        stats = info['stats']
        evicted = stats['evicted_keys']
        
        if used_pct > 90:
            print(f"⚠️  Critical: Memory at {used_pct:.1f}%")
        
        if evicted > 0:
            print(f"⚠️  Keys being evicted: {evicted}")
        
        time.sleep(interval)

# Run in background
# continuous_monitoring()
```

### 2. Set Up Alerts

```python
import redis
import smtplib

def check_and_alert():
    """Check metrics and send alerts"""
    r = redis.Redis()
    
    info = r.info()
    mem = info['memory']
    
    used_pct = mem['used_memory'] / mem['total_system_memory'] * 100
    
    if used_pct > 85:
        # Send alert
        send_alert(f"Redis memory at {used_pct:.1f}%")

def send_alert(message):
    """Send alert email"""
    # Implementation depends on your setup
    pass
```

### 3. Track Trends

```python
import redis
from datetime import datetime

def track_trends():
    """Track metrics over time"""
    r = redis.Redis()
    
    metrics = []
    
    for _ in range(10):
        info = r.info()
        metrics.append({
            'timestamp': datetime.now(),
            'memory': info['memory']['used_memory'],
            'ops': info['stats']['instantaneous_ops_per_sec']
        })
        
        time.sleep(60)
    
    # Analyze trend
    memory_increase = metrics[-1]['memory'] - metrics[0]['memory']
    print(f"Memory increase over 10 mins: {memory_increase / (1024*1024):.2f} MB")
```

## Next Steps

- [Configuration](4-configuration.md) - Tuning parameters
- [Performance](10-monitor.md) - Real-time monitoring
- [Persistence](15-persistence.md) - Data durability

## Resources

- **INFO Command**: https://redis.io/commands/info/
- **Metrics**: https://redis.io/docs/management/monitoring/

## Summary

- INFO provides comprehensive Redis statistics
- Use sections to get specific information
- Monitor memory, clients, and hit rate regularly
- Watch for memory fragmentation and evicted keys
- Track trends to identify issues early
- Set up alerts for critical metrics
