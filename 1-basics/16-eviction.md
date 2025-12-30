# Redis Eviction Policies

## Overview
Eviction in Redis is the process of removing keys from the database when memory limits are reached. It's controlled by the `maxmemory` configuration and `maxmemory-policy` settings.

## Memory Limits
- `maxmemory`: Sets the maximum memory Redis can use
- When limit is exceeded, eviction policies determine which keys to remove
- Prevents out-of-memory errors and server crashes

## Eviction Policies

### No Eviction
- **noeviction**: Returns errors when memory is full; no keys are removed

### LRU (Least Recently Used)
- **allkeys-lru**: Evicts least recently used keys from all keys
- **volatile-lru**: Evicts least recently used keys with TTL only

### LFU (Least Frequently Used)
- **allkeys-lfu**: Evicts least frequently used keys from all keys
- **volatile-lfu**: Evicts least frequently used keys with TTL only

### Random
- **allkeys-random**: Randomly evicts any key
- **volatile-random**: Randomly evicts keys with TTL only

### TTL-Based
- **volatile-ttl**: Evicts keys with shortest remaining TTL first

## Configuration
```bash
maxmemory 100mb
maxmemory-policy allkeys-lru
```

## Best Practices
- Choose policy based on access patterns
- Monitor memory usage regularly
- Use volatile policies for cache-like data
- Use allkeys policies for critical data with fallback logic
- Test eviction behavior under load

## Performance Considerations
- Eviction has minimal performance impact
- LRU/LFU provide better hit rates than random policies
- Sampling-based approximation reduces CPU overhead