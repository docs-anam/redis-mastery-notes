# Redis Persistence

## Overview
Redis persistence enables data durability by saving in-memory data to disk, allowing recovery after server restarts or failures.

## RDB (Redis Database)

### What is RDB?
Point-in-time snapshot of the entire dataset compressed into a binary file.

### How it Works
- `SAVE`: Blocks all client requests until snapshot completes
- `BGSAVE`: Fork background process, non-blocking
- `LASTSAVE`: Returns timestamp of last save

### Configuration
```
save 900 1          # Save if 1 key changed in 900 seconds
save 300 10         # Save if 10 keys changed in 300 seconds
save 60 10000       # Save if 10000 keys changed in 60 seconds
```

### Pros
- Compact file format
- Fast loading on startup
- Good for backups

### Cons
- Data loss between snapshots
- Expensive fork operation on large datasets

## AOF (Append-Only File)

### What is AOF?
Log every write command executed, replaying them on restart.

### Modes
- `appendfsync always`: Fsync after every command (safest, slowest)
- `appendfsync everysec`: Fsync every second (default, balanced)
- `appendfsync no`: OS decides fsync (fastest, less safe)

### Configuration
```
appendonly yes
appendfilename "appendonly.aof"
```

### Pros
- More durable, minimal data loss
- Readable format
- Automatic rewriting

### Cons
- Larger file size
- Slower than RDB

## Hybrid Approach
Use both RDB and AOF together:
- RDB for fast recovery
- AOF for durability protection

## Best Practices
- Enable persistence based on data criticality
- Monitor disk space
- Test recovery procedures
- Use `BGSAVE` over `SAVE`
- Configure rewrite thresholds for AOF