# Redis Server Information

## Overview
The `INFO` command provides comprehensive details about Redis server status, performance metrics, and statistics.

## Command Syntax
```redis
INFO [section]
```

## Main Sections

### Server
- Redis version, OS, architecture
- Process ID and uptime
- TCP port and configuration file path

### Clients
- Connected clients count
- Blocked clients
- Input/output buffers

### Memory
- Used memory (human-readable and bytes)
- Peak memory usage
- Memory fragmentation ratio
- Memory allocator details

### Persistence
- RDB (snapshotting) status
- AOF (Append-Only File) status
- Last save time
- Background operation info

### Stats
- Total connections received
- Total commands processed
- Operations per second
- Keyspace hits and misses
- Evicted keys count

### Replication
- Role (master/slave/sentinel)
- Connected replicas
- Replication offset
- Master replication ID

### CPU
- CPU usage (user and system)
- Child process CPU usage

### Cluster
- Cluster enabled status
- Slots information

### Keyspace
- Database statistics
- Keys count per database
- Expiration info

## Usage Examples
```redis
INFO                    # All sections
INFO memory             # Memory section only
INFO stats              # Statistics section
INFO replication        # Replication info
```

## Practical Applications
- Monitor server health
- Debug performance issues
- Track memory usage
- Verify replication status
- Identify bottlenecks
