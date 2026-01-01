# Redis Configuration

## Overview

Redis configuration controls server behavior, performance, persistence, security, and memory management. Configuration can be done via config file or runtime commands.

## Configuration Methods

### 1. Configuration File (redis.conf)

Location:
- macOS: `/usr/local/etc/redis.conf`
- Linux: `/etc/redis/redis.conf`
- Docker: `/usr/local/etc/redis/redis.conf`

Start Redis with config:
```bash
redis-server /path/to/redis.conf
```

### 2. Command Line Overrides

```bash
redis-server --port 6380 --maxmemory 256mb
```

### 3. Runtime Configuration (CONFIG commands)

```redis
# Get current value
CONFIG GET maxmemory
# Output: 1) "maxmemory", 2) "0"

# Set new value (some parameters only)
CONFIG SET maxmemory 256mb

# Get all parameters
CONFIG GET *

# Rewrite config file
CONFIG REWRITE
```

## Essential Configuration Parameters

### Server Configuration

```conf
# Network
port 6379                    # TCP port to listen on
bind 127.0.0.1              # IP to bind (127.0.0.1 = localhost only)
bind 0.0.0.0                # Bind to all interfaces (less secure)
timeout 0                   # Close idle clients after X seconds (0 = disabled)

# Server
databases 16                # Number of databases (0-15)
daemonize no                # Run as background daemon (systemd: no)
pidfile /var/run/redis.pid  # PID file location
loglevel notice             # Log level: debug, verbose, notice, warning
logfile ""                  # Log file (empty = stdout)

# Name
server_name "Redis Server"  # Informational only
```

### Memory Configuration (CRITICAL)

```conf
# Maximum memory
maxmemory 256mb             # Maximum memory Redis can use (IMPORTANT!)
                            # Set to 70-80% of available RAM
                            # Examples: 256mb, 1gb

# Eviction policy (what to delete when maxmemory reached)
maxmemory-policy allkeys-lru

# Available policies:
# - volatile-lru: Evict least recently used keys WITH expiration
# - volatile-lfu: Evict least frequently used keys WITH expiration
# - volatile-ttl: Evict keys WITH shortest TTL
# - volatile-random: Evict random keys WITH expiration
# - allkeys-lru: Evict least recently used keys (ANY key)
# - allkeys-lfu: Evict least frequently used keys (ANY key)
# - allkeys-random: Evict random keys (ANY key)
# - noeviction: Error when memory exceeded (don't delete anything)

# Memory optimization
maxmemory-samples 5         # Number of samples for eviction policy
```

### Persistence Configuration

#### RDB (Snapshots)

```conf
# Save to disk snapshot
# Format: save <seconds> <changes>
save 900 1                  # Save every 15 minutes if 1 key changed
save 300 10                 # Save every 5 minutes if 10 keys changed
save 60 10000               # Save every 60 seconds if 10000 keys changed

# Or disable RDB completely
# save ""

# RDB Options
stop-writes-on-bgsave-error yes  # Stop writes if save fails
rdbcompression yes               # Compress RDB file
rdbchecksum yes                  # Include checksum
dbfilename dump.rdb              # RDB filename
dir /var/lib/redis              # Directory for persistence files
```

#### AOF (Append-Only File)

```conf
# Enable AOF persistence
appendonly no               # Enable AOF (yes/no)

# AOF filename
appendfilename "appendonly.aof"

# fsync policy
appendfsync everysec       # Fsync options:
                           # - always: fsync on every write (safe, slow)
                           # - everysec: fsync every 1 second (good balance)
                           # - no: let OS decide (fastest, least safe)

# AOF rewrite
auto-aof-rewrite-percentage 100  # Rewrite when AOF 100% larger
auto-aof-rewrite-min-size 64mb    # Min size before rewrite
```

### Client Configuration

```conf
# Connection handling
tcp-backlog 511             # TCP backlog size
tcp-keepalive 300          # Send keepalive every 300 seconds

# Client timeout
timeout 0                  # Close idle connections (0 = never)

# Max clients
maxclients 10000           # Maximum simultaneous connections
```

### Replication Configuration

```conf
# Slave/Replica configuration
replicaof NO ONE            # Set up as replica of another instance
                            # Example: replicaof localhost 6379

replica-read-only yes       # Replicas are read-only
replica-serve-stale-data yes # Serve stale data if sync is in progress
```

### Security Configuration

```conf
# Requirepass
requirepass your_password   # Require password for all commands

# ACL configuration (Redis 6.0+)
aclfile /etc/redis/users.acl  # ACL file location

# Command renaming (security through obscurity)
rename-command FLUSHDB ""   # Disable command (empty = disable)
rename-command FLUSHALL ""
rename-command KEYS ""
```

## Performance Tuning

### For High Throughput

```conf
# Disable persistence for maximum speed (if data loss acceptable)
save ""
appendonly no

# Increase buffer sizes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# Increase TCP backlog
tcp-backlog 1024
```

### For Data Safety

```conf
# Use both persistence methods
save 60 1000
appendonly yes
appendfsync always

# Increase client buffer
client-output-buffer-limit normal 256mb 256mb 60
```

### For Memory Efficiency

```conf
# Evict aggressively
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# Disable unnecessary features
save ""

# Optimize key eviction
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
```

## Configuration Examples

### Development Setup

```conf
# Simple configuration for development
port 6379
bind 127.0.0.1
databases 16
maxmemory 256mb
maxmemory-policy allkeys-lru

# Persistence not critical
save 900 1
appendonly no

# Logging
loglevel notice
```

### Production Setup

```conf
# Production configuration
port 6379
bind 127.0.0.1              # Restrict access
requirepass complex_password # Require authentication

# Memory settings
maxmemory 8gb              # Adjust to your server
maxmemory-policy allkeys-lru

# Persistence (both methods for safety)
save 60 1000
appendonly yes
appendfsync everysec
dbfilename dump.rdb
dir /var/lib/redis

# Replication
replicaof master_host 6379

# Logging
loglevel warning           # Less verbose in production
logfile /var/log/redis.log

# Client limits
maxclients 10000
timeout 300

# Performance
tcp-backlog 1024
```

### Cache-Only Setup

```conf
# Optimized for caching (data loss acceptable)
port 6379
bind 127.0.0.1

# No persistence - maximum speed
save ""
appendonly no

# Memory management
maxmemory 4gb
maxmemory-policy allkeys-lru

# Network
tcp-backlog 2048
timeout 0

# Logging
loglevel warning
```

## Runtime Configuration Changes

Some parameters can be changed while Redis is running:

```redis
# Check current setting
CONFIG GET maxmemory
# Output: 1) "maxmemory", 2) "256000000"

# Change parameter
CONFIG SET maxmemory 512mb
# Output: OK

# Change log level
CONFIG SET loglevel warning

# Get all parameters
CONFIG GET *

# Rewrite config file (persistent)
CONFIG REWRITE
```

## Configuration Best Practices

### 1. Set Appropriate maxmemory
```conf
# DO: Set to ~70-80% of system RAM
maxmemory 8gb

# DON'T: Leave as 0 (unlimited)
# This will crash when RAM is full
```

### 2. Choose Appropriate Eviction Policy
```conf
# For cache: allkeys-lru (evict least recently used)
maxmemory-policy allkeys-lru

# For session: volatile-lru (evict keys WITH TTL)
maxmemory-policy volatile-lru

# For critical data: noeviction (error on OOM)
maxmemory-policy noeviction
```

### 3. Configure Persistence Based on Needs
```conf
# For important data: Both methods
save 60 1000
appendonly yes
appendfsync everysec

# For cache: None
save ""
appendonly no

# For balance: RDB only
save 300 10
appendonly no
```

### 4. Use Correct Bind Address
```conf
# Development (local only)
bind 127.0.0.1

# In containers/services (all interfaces)
bind 0.0.0.0

# Multiple interfaces
bind 127.0.0.1 192.168.1.10
```

### 5. Set Appropriate Timeout
```conf
# Development/testing
timeout 0

# Production
timeout 300  # Close idle clients after 5 minutes
```

## Monitoring Configuration

```redis
# View memory usage
INFO memory

# View current configuration
CONFIG GET *

# View server info
INFO server

# View stats
INFO stats
```

## Configuration Validation

```bash
# Check syntax
redis-server /path/to/redis.conf --test-memory 256mb

# Get all config
redis-cli CONFIG GET '*' | head -20
```

## Common Configuration Mistakes

### ❌ Not Setting maxmemory
```conf
# Bad: No limit
# Redis will use all RAM, crash when full
```

### ❌ Wrong Eviction Policy
```conf
# Bad: noeviction with no persistence
maxmemory-policy noeviction
# Will error when full - data loss
```

### ❌ Weak Authentication
```conf
# Bad: Simple password
requirepass "password"

# Good: Complex password
requirepass "k9$mP@x2qL$nR&vW3zT"
```

### ❌ No Persistence in Production
```conf
# Bad for important data
save ""
appendonly no
# Will lose data on restart
```

## Next Steps

- [Strings Data Type](6-strings.md)
- [Persistence](15-persistence.md)
- [Security](14-security.md)
- [Monitoring](10-monitor.md)

## Resources

- **Configuration Documentation**: https://redis.io/docs/management/config/
- **Config Options**: https://redis.io/docs/management/config-file/

## Summary

Key configuration decisions:
- **maxmemory**: Set to 70-80% of RAM
- **maxmemory-policy**: Choose based on use case (usually allkeys-lru)
- **Persistence**: Choose RDB, AOF, both, or none based on needs
- **requirepass**: Always set in production
- **bind**: Restrict to needed addresses only

Start with provided examples, then customize for your needs.
