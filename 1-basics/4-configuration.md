# Redis Configuration

## Overview
Redis configuration controls server behavior, memory management, persistence, and replication settings. Proper configuration is critical for performance, data durability, and security. Configuration can be applied via configuration files, command-line arguments, or at runtime using Redis commands.

## Configuration Methods

### 1. **Configuration File**
- Default location: `/etc/redis/redis.conf`
- Start Redis with config: `redis-server /path/to/redis.conf`
- Settings applied at startup
- Official Redis config template: https://github.com/redis/redis/blob/unstable/redis.conf

#### Setting Up Configuration File
1. Download the default config from the Redis repository
2. Copy to your preferred location: `sudo cp redis.conf /etc/redis/redis.conf`
3. Edit settings using your preferred editor: `sudo nano /etc/redis/redis.conf`
4. Start Redis with the config file: `redis-server /etc/redis/redis.conf`
5. Verify settings loaded: `redis-cli CONFIG GET "*"`

### 2. **Command Line Arguments**
- Override config file settings: `redis-server --port 6380`
- Useful for testing and temporary changes
- Format: `redis-server --parameter value`

### 3. **Runtime Configuration**
- Use `CONFIG GET` to retrieve settings: `CONFIG GET port`
- Use `CONFIG SET` to modify settings dynamically: `CONFIG SET maxmemory 256mb`
- Changes apply immediately without restart
- Use `CONFIG REWRITE` to persist runtime changes to the config file

## Connecting with redis-cli

Once you've configured Redis and changed the port, use `redis-cli` to connect:

### Basic Connections

**Default port (6379) on localhost:**
```bash
redis-cli
```

**Custom port (e.g., 6380):**
```bash
redis-cli -h localhost -p 6380
```

## Key Configuration Parameters

### Port and Binding
```conf
port 6379                    # Port Redis listens on
bind 127.0.0.1              # Bind to specific IP (127.0.0.1 for localhost only)
protected-mode yes          # Reject unauthenticated connections from non-localhost
tcp-backlog 511             # TCP listen backlog
timeout 0                   # Close connection after client idle (0 = disabled)
```

### Memory Management
```conf
maxmemory 100mb             # Maximum memory Redis can use
maxmemory-policy allkeys-lru # Eviction policy when maxmemory limit reached
```
**Common eviction policies:**
- `noeviction`: Return error when memory limit reached (default)
- `allkeys-lru`: Remove least recently used keys from all keys
- `allkeys-lfu`: Remove least frequently used keys from all keys
- `volatile-lru`: Remove least recently used keys with TTL
- `volatile-lfu`: Remove least frequently used keys with TTL
- `allkeys-random`: Randomly remove any key
- `volatile-random`: Randomly remove keys with TTL
- `volatile-ttl`: Remove keys with shortest TTL first

### Persistence
```conf
save 900 1          # RDB snapshot: save if 1 key changed in 900s
save 300 10         # RDB snapshot: save if 10 keys changed in 300s
save 60 10000       # RDB snapshot: save if 10000 keys changed in 60s
appendonly yes      # Enable AOF (Append-Only File)
appendfsync everysec # AOF fsync: always | everysec | no
appendfilename "appendonly.aof"
dbfilename "dump.rdb"
dir ./              # Directory for RDB and AOF files
```
**Persistence Trade-offs:**
- RDB: Fast, compact snapshots; loses recent changes
- AOF: Durability per command; larger files, slower
- Combination: Better durability with acceptable performance

### Replication
```conf
replicaof no one      # Disable replication (use instead of slaveof)
replica-serve-stale-data yes # Serve stale data when disconnected from master
replica-read-only yes # Replicas accept only read commands
masterauth password   # Password to authenticate with master
```

### Security
```conf
requirepass yourpassword      # Set server password (use ACL in Redis 6+)
masterauth password           # Authenticate with master (deprecated)
rename-command FLUSHDB ""     # Disable dangerous commands (set to empty string)
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG_$(date +%s)"
```

### Logging
```conf
loglevel notice       # Log level: debug | verbose | notice | warning
logfile ""            # Log file path (empty string = stdout)
syslog-enabled no     # Enable syslog logging
syslog-ident redis    # Syslog identity
```

### Clients and Connections
```conf
maxclients 10000      # Maximum number of connected clients
```

## Best Practices

### Development Environment
- Use default configuration with small `maxmemory`
- Disable persistence if data loss is acceptable
- Enable `protected-mode` to prevent accidental access

### Production Environment
- ⚠️ **Always set `maxmemory`** to prevent out-of-memory crashes
- ⚠️ **Enable authentication** with strong passwords
- ⚠️ **Configure persistence** based on durability requirements
- ⚠️ **Bind to specific IP** (not 0.0.0.0) or use firewall rules
- ⚠️ **Disable dangerous commands** that can delete data (FLUSHDB, FLUSHALL, CONFIG)
- Disable `protected-mode` only if using proper firewall/security
- Use `allkeys-lru` or `volatile-lru` for maxmemory-policy to prevent crashes
- Monitor memory usage regularly
- Use Redis 6+ ACL instead of `requirepass` for granular permissions
- Enable logging for debugging and auditing
- Use `CONFIG REWRITE` to save runtime changes
- Keep backups of your configuration file and data

### Performance Optimization
- Adjust `save` frequency based on durability vs. performance needs
- Use `appendfsync no` for better performance (with risk of data loss)
- Tune `tcp-backlog` for high-traffic scenarios
- Set appropriate `timeout` to clean up idle connections
- Use pipelining for batch operations
- Monitor slow log: `CONFIG GET slowlog-*`

### Security Checklist
- [ ] Set `requirepass` or configure ACL
- [ ] Disable/rename dangerous commands
- [ ] Use firewalls to restrict access
- [ ] Keep Redis updated to latest version
- [ ] Use TLS for client connections in production
- [ ] Rotate passwords regularly
- [ ] Audit command logs periodically