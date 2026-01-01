# Redis Monitoring with MONITOR

## Overview

The MONITOR command allows real-time observation of all Redis server commands. Useful for debugging, understanding client behavior, and troubleshooting performance issues.

## Basic MONITOR Command

### Viewing All Commands

```redis
# Start monitoring
MONITOR

# Output (all commands from clients):
# 1704067200.123456 [0 127.0.0.1:12345] "SET" "key1" "value1"
# 1704067200.234567 [0 127.0.0.1:12345] "GET" "key1"
# 1704067200.345678 [0 127.0.0.1:54321] "INCR" "counter"
# 1704067200.456789 [0 127.0.0.1:54321] "INCR" "counter"
```

### Format Explanation

```
timestamp [database client_address] "command" "arg1" "arg2" ...
|         |       |                  |         |     |
|         |       |                  |         |     └─ Arguments
|         |       |                  |         └─────── Command name
|         |       |                  └────────────── Quoted strings
|         |       └──────────────────────────────── Client IP:port
|         └──────────────────────────────────────── Current database
└────────────────────────────────────────────────── Unix timestamp
```

## Python Examples

### Basic Monitoring

```python
import redis
from datetime import datetime

def monitor_redis(duration=10):
    """Monitor Redis for specified duration"""
    r = redis.Redis()
    
    # Start monitoring
    monitor = r.client_list()
    
    print(f"Monitoring Redis for {duration} seconds...")
    
    # In practice, you'd use threading or async
    # redis-cli is better for continuous monitoring
    
    # Alternative: Use redis-cli
    # Command: redis-cli MONITOR

monitor_redis()
```

### Filtering Commands

```python
import redis

def get_command_stats():
    """Get stats on command types"""
    r = redis.Redis()
    
    info = r.info('commandstats')
    
    for cmd, stats in sorted(info.items()):
        calls = stats.get('calls', 0)
        usec = stats.get('usec', 0)
        if calls > 0:
            avg = usec / calls
            print(f"{cmd}: {calls} calls, {avg:.2f}μs avg")

get_command_stats()
```

### Client Information

```python
import redis

def monitor_clients():
    """Monitor active clients"""
    r = redis.Redis()
    
    clients = r.client_list()
    
    for client in clients:
        print(f"Client: {client}")
        # Output:
        # id=123 addr=127.0.0.1:12345 ... flags=N ...
        # id=124 addr=127.0.0.1:54321 ... flags=r ...
```

## Real-World Examples

### Example 1: Debug Slow Commands

```bash
# Terminal 1: Monitor Redis
redis-cli MONITOR

# Terminal 2: Run application
python app.py

# Terminal 1 will show all commands executed
# Look for slow operations or unexpected commands
```

### Example 2: Analyze Command Frequency

```python
import redis
import subprocess
from collections import defaultdict

def analyze_commands(duration=10):
    """Analyze command frequency using MONITOR"""
    proc = subprocess.Popen(
        ['redis-cli', 'MONITOR'],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True
    )
    
    commands = defaultdict(int)
    
    import time
    start = time.time()
    
    for line in proc.stdout:
        if time.time() - start > duration:
            break
        
        # Parse command from MONITOR output
        parts = line.split('"')
        if len(parts) >= 2:
            cmd = parts[1]
            commands[cmd] += 1
    
    proc.terminate()
    
    # Print sorted by frequency
    for cmd, count in sorted(commands.items(), key=lambda x: -x[1]):
        print(f"{cmd}: {count}")

analyze_commands()
```

### Example 3: Find Problematic Clients

```bash
#!/bin/bash
# Monitor and find clients making many requests

redis-cli MONITOR | awk '{
    match($3, /\[.*:([0-9]+)\]/, arr)
    port = arr[1]
    cmd = $4
    cmd_count[port]++
    total++
    
    if (total % 100 == 0) {
        print "Top clients:"
        for (p in cmd_count) {
            print "  Port " p ": " cmd_count[p] " commands"
        }
    }
}'
```

## Performance Monitoring

### Command Statistics

```python
import redis

def get_slowest_commands():
    """Get slowest Redis commands"""
    r = redis.Redis()
    
    info = r.info('commandstats')
    
    # Calculate average execution time
    cmds = []
    for cmd_name, stats in info.items():
        if isinstance(stats, dict):
            usec = stats.get('usec', 0)
            calls = stats.get('calls', 0)
            if calls > 0:
                avg_usec = usec / calls
                cmds.append((cmd_name, avg_usec, calls))
    
    # Sort by average time
    for cmd, avg_usec, calls in sorted(cmds, key=lambda x: -x[1])[:10]:
        print(f"{cmd:15} {avg_usec:10.2f}μs avg ({calls} calls)")

get_slowest_commands()
```

### Memory Usage Monitoring

```python
import redis

def monitor_memory():
    """Monitor Redis memory usage"""
    r = redis.Redis()
    
    info = r.info('memory')
    
    print(f"Memory used: {info['used_memory_human']}")
    print(f"Memory peak: {info['used_memory_peak_human']}")
    print(f"Memory fragmentation: {info['mem_fragmentation_ratio']:.2f}")

monitor_memory()
```

### Key Space Statistics

```python
import redis

def monitor_keyspace():
    """Monitor keys across databases"""
    r = redis.Redis()
    
    info = r.info('keyspace')
    
    for db_name, db_info in info.items():
        if isinstance(db_info, dict):
            keys = db_info.get('keys', 0)
            expires = db_info.get('expires', 0)
            print(f"{db_name}: {keys} keys, {expires} expiring")

monitor_keyspace()
```

## Common Monitoring Patterns

### Pattern 1: Real-Time Command Monitoring

```bash
# Watch all commands in real-time
redis-cli MONITOR | grep -E "(SET|GET|INCR|DEL)"

# Watch commands to specific keys
redis-cli MONITOR | grep "user:123"

# Watch specific client
redis-cli MONITOR | grep "127.0.0.1:12345"
```

### Pattern 2: Log Commands to File

```bash
# Log all commands
redis-cli MONITOR > redis_commands.log

# Later, analyze
grep "SET" redis_commands.log | wc -l

grep "SLOW" redis_commands.log
```

### Pattern 3: Monitor Specific Command Types

```bash
# Only see INCR commands
redis-cli MONITOR | grep -E '\[.*\] "INCR"'

# See all write commands
redis-cli MONITOR | grep -E '\[.*\] "(SET|INCR|DEL|LPUSH|SADD)"'

# See all admin commands
redis-cli MONITOR | grep -E '\[.*\] "(FLUSHDB|FLUSHALL|SHUTDOWN|SAVE)"'
```

## Best Practices

### 1. Use MONITOR Carefully in Production

```python
# Good: Monitor briefly, then stop
# MONITOR has performance impact

# Bad: Monitor for extended periods
# Reduces Redis performance significantly
```

### 2. Filter Output

```bash
# Instead of raw MONITOR output:
redis-cli MONITOR

# Better: Filter relevant commands
redis-cli MONITOR | grep "SET\|INCR"

# Or watch specific keys:
redis-cli MONITOR | grep "user:123"
```

### 3. Use Dedicated Monitoring Tools

```python
# Better than MONITOR: Use official tools
# - redis-cli --stat: Shows stats
# - redis-cli --latency: Measures latency
# - redis-cli --latency-history: Latency over time
```

### 4. Combine with Other Metrics

```python
import redis
import time

def comprehensive_monitoring():
    """Monitor multiple metrics"""
    r = redis.Redis()
    
    while True:
        # Memory
        memory = r.info('memory')['used_memory_human']
        
        # Clients
        clients = r.info('clients')['connected_clients']
        
        # Stats
        stats = r.info('stats')
        ops = stats['total_commands_processed']
        
        print(f"Memory: {memory} | Clients: {clients} | Ops: {ops}")
        
        time.sleep(1)

# Run in background
# comprehensive_monitoring()
```

## Troubleshooting with MONITOR

### Debug Slow Requests

```bash
# Terminal 1: Start MONITOR
redis-cli MONITOR

# Terminal 2: Run request
python slow_request.py

# Terminal 1: Look for slow command
# Check timestamp gaps in output
```

### Find Unexpected Commands

```bash
# Look for commands you don't expect
redis-cli MONITOR | grep -v "GET\|SET\|INCR"

# Find FLUSHDB/FLUSHALL
redis-cli MONITOR | grep "FLUSH"

# Find DEL commands
redis-cli MONITOR | grep "DEL"
```

### Analyze Client Behavior

```bash
# See which clients are most active
redis-cli MONITOR | awk '{print $3}' | sort | uniq -c | sort -rn

# See command breakdown by client
redis-cli MONITOR | awk '{print $3, $4}' | sort | uniq -c | sort -rn
```

## Limitations of MONITOR

### Performance Impact
- MONITOR has significant performance overhead
- Reduces throughput by 5-10% per monitoring client
- Don't use in production for extended periods

### Information Limitations
- Shows commands at protocol level (may be pipelined)
- Doesn't show query result times precisely
- Can miss commands if output is slow

### Better Alternatives

```python
import redis

# Use INFO command instead:
r = redis.Redis()

# Command statistics
print(r.info('commandstats'))

# Memory info
print(r.info('memory'))

# Client list
print(r.client_list())

# Latency monitoring
print(r.latency_latest())
```

## Next Steps

- [Configuration](4-configuration.md) - Performance tuning
- [Persistence](15-persistence.md) - Data monitoring
- [Server Information](11-server-information.md) - INFO command

## Resources

- **MONITOR**: https://redis.io/commands/monitor/
- **INFO**: https://redis.io/commands/info/
- **Monitoring**: https://redis.io/docs/management/monitoring/

## Summary

- MONITOR shows all Redis commands in real-time
- Format: timestamp [db client] "command" args...
- Useful for debugging and understanding behavior
- Performance impact: avoid in production
- Better alternatives: INFO, COMMANDSTATS, CLIENT LIST
- Filter output for relevant commands
