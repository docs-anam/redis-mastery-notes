# Redis Client Connection

## Overview
Redis client connection is the process by which clients establish communication with a Redis server to execute commands and manage data. Understanding connection management is critical for application performance, reliability, and security.

## Connection Basics

### Default Connection
- **Host**: localhost (127.0.0.1)
- **Port**: 6379 (default, configurable)
- **Protocol**: RESP (REdis Serialization Protocol) version 2 or 3
- **Timeout**: 0 (no timeout by default)
- **Database**: 0 (of 16 available)

### Connection Methods
1. **TCP Sockets** - Standard network connection over TCP/IP
   - Works over network
   - Default protocol
   - Requires host and port configuration

2. **Unix Domain Sockets** - Local machine connections only
   - Lower latency than TCP
   - No network overhead
   - Better for local applications
   - Specified as: `redis-cli -s /tmp/redis.sock`

3. **TLS/SSL** - Encrypted connections for security
   - Port 6380 (default for encrypted)
   - Requires certificate configuration
   - Adds encryption overhead (~5-10% slower)
   - Required for remote connections in production

## Connection Process

```
Client → TCP Handshake → Optional AUTH → Optional SELECT → Ready for Commands
                ↓              ↓                ↓
            ~1-3ms       Verify Credentials  Choose DB (0-15)
```

### Connection Lifecycle
1. **Handshake** - TCP connection established (3-way handshake)
2. **Authentication** - Optional AUTH command if requirepass is set
3. **Database Selection** - SELECT command to choose database (0-15)
4. **Command Execution** - Send and receive commands
5. **Connection Termination** - QUIT command or timeout

### Connection State
- **IDLE**: Waiting for commands
- **BLOCKED**: Waiting on blocking operation (BLPOP, BRPOP, etc.)
- **PUBSUB**: In publish/subscribe mode (limited commands)
- **MONITOR**: Monitoring all commands (special mode)
- **TRANSACTION**: In MULTI/EXEC block

## Connection Configuration

### Client Configuration Parameters
- **timeout**: Connection timeout in seconds (0 = no timeout)
- **keepalive**: TCP keepalive probe interval (improves broken connection detection)
- **buffer_size**: Read/write buffer size (larger = better throughput, more memory)
- **retry_on_timeout**: Auto-retry on connection timeout
- **connection_retries**: Number of retry attempts before failing
- **socket_connect_timeout**: Timeout for initial socket connection
- **socket_keepalive_interval**: Interval for TCP keepalive probes

### Server Configuration (redis.conf)

#### Timeout Settings
```conf
timeout 0                  # Client timeout: 0 = never timeout (recommended for production)
tcp-backlog 511           # Pending connection queue size
tcp-keepalive 300         # TCP keepalive probe interval in seconds
```

#### Connection Limits
```conf
maxclients 10000          # Maximum allowed client connections
# Note: Actual limit depends on OS file descriptor limits
```

#### Security Settings
```conf
bind 127.0.0.1            # Bind to specific IPs
protected-mode yes        # Require AUTH for non-localhost connections
requirepass yourpassword   # Enable password authentication
```

### Recommended Configuration for Production
```conf
timeout 0                 # Never timeout clients
tcp-keepalive 60          # Detect dead connections quickly
maxclients 10000          # Adjust based on application needs
requirepass "strong_password_here"
bind 10.0.0.5             # Bind to internal network IP
protected-mode yes        # Extra security layer
```

## Monitoring Connections

### CLIENT Commands for Connection Management

#### List All Connections
```redis
CLIENT LIST
# Output: space-separated key=value pairs for each client
# Includes: id, addr, fd, name, age, idle, flags, db, sub, psub, etc.
```

#### Get Client Information
```redis
CLIENT INFO
# Returns detailed information about current connection
```

#### Set Connection Name
```redis
CLIENT SETNAME "my-app-connection"
# Useful for identifying connections in CLIENT LIST
```

#### Get Connection Name
```redis
CLIENT GETNAME
# Returns the name set by CLIENT SETNAME or nil
```

#### Kill Client Connection
```redis
CLIENT KILL addr IP:PORT
# Force disconnect a specific client
# Useful for cleaning up zombie connections
```

#### Pause All Clients
```redis
CLIENT PAUSE timeout_ms
# Pause all client replies for specified duration
# Useful for maintenance operations
```

### Connection Information Fields Explained
| Field | Meaning |
|-------|---------|
| id | Unique client ID |
| addr | Client IP address and port |
| fd | File descriptor number |
| name | Client name (set by SETNAME) |
| age | Connection age in seconds |
| idle | Idle time in seconds |
| flags | Connection flags (N=normal, S=slave, M=master, x=close-after-write) |
| db | Selected database number |
| sub | Subscribed channels count |
| psub | Pattern subscriptions count |
| qbuf | Query buffer size (bytes) |
| cmd | Last command executed |

### Monitoring Connection Health
```redis
# Check number of connected clients
INFO clients
# Returns: connected_clients, blocked_clients, maxclients

# Monitor specific connection
CLIENT LIST | grep "your-app-name"

# Check server capacity
CONFIG GET maxclients
```