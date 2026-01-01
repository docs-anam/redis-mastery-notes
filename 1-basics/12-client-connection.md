# Redis Client Connections

## Overview

Client connections are the backbone of Redis communication. Understanding connection management, pooling, and best practices ensures reliable and efficient applications.

## Basic Connection

### Python Connection

```python
import redis

# Basic connection
r = redis.Redis(host='localhost', port=6379, db=0)

# Test connection
r.ping()  # Returns True

# Close connection
r.close()
```

### Connection with Options

```python
import redis

# Detailed connection
r = redis.Redis(
    host='localhost',
    port=6379,
    db=0,
    password=None,           # If password protected
    socket_timeout=5,        # Timeout for operations
    socket_keepalive=True,   # Keep connection alive
    socket_keepalive_options=None,
    retry_on_timeout=False,
    decode_responses=False,  # Auto-decode responses to strings
    ssl=False,               # Use SSL/TLS
    ssl_certfile=None,
    ssl_keyfile=None
)
```

## Connection Pooling

### Connection Pool Basics

```python
import redis

# Automatic pooling
pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    db=0,
    max_connections=10,      # Maximum connections
    socket_keepalive=True
)

r = redis.Redis(connection_pool=pool)

# Multiple clients share same pool
client1 = redis.Redis(connection_pool=pool)
client2 = redis.Redis(connection_pool=pool)

# Safe to share across threads
```

### Thread-Safe Pooling

```python
import redis
from threading import Thread

# Create pool
pool = redis.ConnectionPool(max_connections=5)

def worker(thread_id):
    r = redis.Redis(connection_pool=pool)
    
    for i in range(100):
        r.incr(f'counter:{thread_id}')

# Run multiple threads
threads = []
for i in range(10):
    t = Thread(target=worker, args=(i,))
    threads.append(t)
    t.start()

# Wait for completion
for t in threads:
    t.join()
```

## Connection Management

### CLIENT Commands

```redis
# List all clients
CLIENT LIST

# Output format:
# id=1 addr=127.0.0.1:54321 fd=6 name=myclient age=100 idle=0
# flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0
# obl=0 oll=0 omem=0 tot-mem=1000 events=r cmd=get arg=key

# Get current client info
CLIENT GETNAME

# Set client name
CLIENT SETNAME "myclient"

# Get client ID
CLIENT ID

# Kill client
CLIENT KILL <IP:port>

# Pause clients (milliseconds)
CLIENT PAUSE 1000

# Unblock clients
CLIENT UNBLOCK <client_id>
```

### Python Client Management

```python
import redis

r = redis.Redis()

# Get info about current client
client_info = r.client_getname()  # Get client name
client_id = r.client_id()          # Get client ID

print(f"Client ID: {client_id}")
print(f"Client name: {client_info}")

# Set client name
r.client_setname("my-app-client")

# List all clients
clients = r.client_list()
for client in clients:
    print(client)

# Kill specific client
# r.client_kill_by_address("192.168.1.100:6379")
```

## Connection Best Practices

### 1. Use Connection Pooling

```python
# Good: Share pool across application
pool = redis.ConnectionPool(max_connections=10)
app_redis = redis.Redis(connection_pool=pool)

# Bad: New connection per request
for _ in range(100):
    r = redis.Redis()  # Creates new connection each time
    r.get('key')
```

### 2. Set Appropriate Timeouts

```python
import redis

# Good: Reasonable timeouts
r = redis.Redis(
    socket_timeout=5,        # 5 second timeout
    socket_connect_timeout=2  # 2 second connect timeout
)

# Bad: No timeout (can hang indefinitely)
r = redis.Redis(socket_timeout=None)
```

### 3. Handle Disconnections

```python
import redis
import time

def resilient_operation():
    """Handle reconnections gracefully"""
    r = redis.Redis()
    max_retries = 3
    
    for attempt in range(max_retries):
        try:
            return r.get('key')
        except redis.ConnectionError:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            raise

result = resilient_operation()
```

### 4. Close Connections Properly

```python
import redis
import contextlib

# Using context manager
@contextlib.contextmanager
def get_redis():
    r = redis.Redis()
    try:
        yield r
    finally:
        r.close()

# Usage
with get_redis() as r:
    r.set('key', 'value')
    # Connection auto-closes
```

## Connection Monitoring

### Monitor Active Connections

```python
import redis

def monitor_connections():
    """Monitor active client connections"""
    r = redis.Redis()
    
    # Get client list
    clients = r.client_list()
    
    print(f"Total connections: {len(clients)}")
    
    # Group by state
    idle_clients = 0
    busy_clients = 0
    
    for line in clients.split('\r\n'):
        if not line:
            continue
        
        # Parse client line
        parts = dict(item.split('=') for item in line.split(' '))
        idle_seconds = int(parts.get('idle', 0))
        
        if idle_seconds > 60:
            idle_clients += 1
        else:
            busy_clients += 1
    
    print(f"Busy: {busy_clients}, Idle: {idle_clients}")

monitor_connections()
```

### Connection Pool Statistics

```python
import redis

def check_pool_stats():
    """Check connection pool statistics"""
    r = redis.Redis()
    
    pool = r.connection_pool
    
    print(f"Pool size: {len(pool._available_connections)}")
    print(f"Pool checked out: {len(pool._in_use_connections)}")
    print(f"Max connections: {pool.max_connections}")

check_pool_stats()
```

## Specific Use Cases

### Use Case 1: Web Application

```python
import redis
from flask import Flask, g

app = Flask(__name__)

# Initialize pool at startup
redis_pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=20,
    decode_responses=True
)

@app.before_request
def get_redis():
    """Get Redis connection for request"""
    g.redis = redis.Redis(connection_pool=redis_pool)

@app.teardown_request
def cleanup_redis(error):
    """Clean up connection"""
    if hasattr(g, 'redis'):
        g.redis.connection_pool.disconnect()

@app.route('/set/<key>/<value>')
def set_key(key, value):
    g.redis.set(key, value)
    return f"Set {key}={value}"

@app.route('/get/<key>')
def get_key(key):
    value = g.redis.get(key)
    return f"Value: {value}"
```

### Use Case 2: Background Jobs

```python
import redis
import time
from multiprocessing import Pool

# Shared pool
redis_pool = redis.ConnectionPool(
    max_connections=5,
    decode_responses=True
)

def process_job(job_id):
    """Process job with Redis"""
    r = redis.Redis(connection_pool=redis_pool)
    
    try:
        # Mark job as processing
        r.set(f'job:{job_id}:status', 'processing')
        
        # Do work
        time.sleep(1)
        
        # Mark complete
        r.set(f'job:{job_id}:status', 'complete')
        
        return job_id
    except Exception as e:
        r.set(f'job:{job_id}:status', 'failed')
        raise

# Process multiple jobs
with Pool(processes=4) as pool:
    results = pool.map(process_job, range(10))
```

### Use Case 3: Microservices

```python
import redis

class RedisService:
    """Reusable Redis service"""
    
    _instance = None
    _pool = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._pool = redis.ConnectionPool(
                host='redis-host',
                port=6379,
                max_connections=10,
                decode_responses=True
            )
        return cls._instance
    
    def get_client(self):
        return redis.Redis(connection_pool=self._pool)

# Usage across microservices
redis_svc = RedisService()
r = redis_svc.get_client()
r.set('key', 'value')
```

## Common Issues

### Issue 1: Too Many Connections

```python
# Symptom: "too many connections" error
# Solution: Use connection pooling with appropriate max_connections

pool = redis.ConnectionPool(
    max_connections=10,  # Adjust based on needs
    decode_responses=True
)

r = redis.Redis(connection_pool=pool)
```

### Issue 2: Connection Timeout

```python
# Symptom: Slow/hung requests
# Solution: Set reasonable timeouts

r = redis.Redis(
    socket_timeout=5,        # Operation timeout
    socket_connect_timeout=2  # Connection timeout
)
```

### Issue 3: Memory Leaks

```python
# Bad: Connections not returned to pool
for i in range(100):
    r = redis.Redis()
    r.set('key', 'value')
    # Connection not closed!

# Good: Use context manager
import contextlib

@contextlib.contextmanager
def redis_client():
    r = redis.Redis()
    try:
        yield r
    finally:
        r.close()

for i in range(100):
    with redis_client() as r:
        r.set('key', 'value')
```

## Performance Tips

### 1. Pipeline Requests

```python
# Without pipeline: N round trips
for i in range(1000):
    r.set(f'key{i}', f'value{i}')

# With pipeline: 1 round trip
with r.pipeline() as pipe:
    for i in range(1000):
        pipe.set(f'key{i}', f'value{i}')
    pipe.execute()
```

### 2. Reuse Connections

```python
# Anti-pattern: New connection per op
r1 = redis.Redis()
r1.set('key1', 'value1')
r1.close()

r2 = redis.Redis()
r2.set('key2', 'value2')
r2.close()

# Better: Reuse connection
r = redis.Redis()
r.set('key1', 'value1')
r.set('key2', 'value2')
r.close()
```

### 3. Connection Pooling Size

```python
# Pool size guidance:
# - Web app: max_connections = number of threads (10-20)
# - Worker queue: max_connections = number of workers
# - Batch jobs: max_connections = smaller (5-10)

pool = redis.ConnectionPool(max_connections=15)
```

## Next Steps

- [Security](14-security.md) - Secure connections
- [Configuration](4-configuration.md) - Server-side connection limits
- [Monitoring](10-monitor.md) - Track connection usage

## Resources

- **Client Commands**: https://redis.io/commands/?group=connection
- **Connection Pooling**: https://redis-py.readthedocs.io/
- **Performance**: https://redis.io/docs/management/optimization/

## Summary

- Use connection pooling for efficiency
- Set appropriate timeouts for reliability
- Monitor active connections
- Close connections properly
- Share pools across application
- Handle disconnections gracefully
- Test connection limits under load
