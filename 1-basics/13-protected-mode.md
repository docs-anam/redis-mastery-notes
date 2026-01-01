# Redis Protected Mode

## Overview

Protected mode is a security feature introduced in Redis 3.2 that restricts access to unauthenticated clients from external hosts. Prevents accidental exposure of Redis instances.

## What is Protected Mode?

### Default Behavior

```
Protected Mode: ON (default)
├─ Local clients (localhost): ALLOWED
├─ Authenticated clients: ALLOWED
└─ External clients: BLOCKED (if no password set)
```

### When Protected Mode Triggers

```
- Client: Not from localhost
- AND Redis: No password configured
- THEN: Connection blocked with error
```

## Enabling/Disabling

### Configuration File

```conf
# redis.conf
protected-mode yes          # Enabled (default in Redis 6.0+)
# or
protected-mode no           # Disabled (not recommended in production)
```

### Runtime Configuration

```redis
# Check current setting
CONFIG GET protected-mode
# Output: 1) "protected-mode", 2) "yes"

# Change at runtime
CONFIG SET protected-mode yes
```

## Protected Mode Errors

### Error When Blocked

```python
import redis

try:
    # Attempting to connect from remote host
    r = redis.Redis(host='production-redis.com', port=6379)
    r.ping()
except redis.ConnectionError as e:
    print(f"Error: {e}")
    # Error: Error -1 connecting to production-redis.com:6379.
    # DENIED Redis is running in protected mode because no password
    # is set, for security reasons unencrypted connections are
    # disabled.
```

## Solutions to Protected Mode

### Solution 1: Set a Password

```conf
# redis.conf
protected-mode yes
requirepass "strongpassword123"
```

```python
import redis

# Connect with password
r = redis.Redis(
    host='remote-host',
    port=6379,
    password='strongpassword123'
)

r.ping()  # Works!
```

### Solution 2: Bind to Specific Address

```conf
# redis.conf
protected-mode yes

# Bind to specific address instead of all
bind 192.168.1.100
# Now only accessible from that address
```

### Solution 3: Disable Protected Mode

```conf
# redis.conf - NOT RECOMMENDED
protected-mode no

# Now accessible from anywhere without password
# Use only in development or behind firewall!
```

## Best Practices

### 1. Keep Protected Mode Enabled

```conf
# Good: Protected mode on, password required
protected-mode yes
requirepass "secure_password_123"
```

### 2. Use Strong Passwords

```conf
# Good: Strong, random password
requirepass "k9$mP@x2qL$nR&vW3zT"

# Bad: Weak password
requirepass "password"
requirepass "123456"
requirepass "redis"
```

### 3. Network Segmentation

```conf
# Option 1: Bind to localhost only
bind 127.0.0.1

# Option 2: Bind to specific network interface
bind 192.168.1.100

# Option 3: Use VPN/private network
# Deploy Redis only on private network
```

### 4. Use TLS for Remote Connections

```conf
# Enable TLS
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-dh-params-file /path/to/redis.dh

# Optionally keep non-TLS disabled
port 0  # Disable unencrypted
```

```python
import redis

# Connect via TLS
r = redis.Redis(
    host='remote-redis.com',
    port=6380,
    password='password',
    ssl=True,
    ssl_cert_reqs='required',
    ssl_ca_certs='/path/to/ca.pem'
)

r.ping()
```

## Common Scenarios

### Scenario 1: Development Environment

```conf
# Local development (protected mode can be off)
protected-mode no
bind 127.0.0.1
port 6379
```

```python
import redis

# Local connection (no password needed)
r = redis.Redis(host='localhost', port=6379)
r.ping()  # Works!
```

### Scenario 2: Docker/Container

```dockerfile
FROM redis:latest

# Use environment variable for password
ENV REDIS_PASSWORD=your_secure_password

RUN echo "requirepass ${REDIS_PASSWORD}" >> /usr/local/etc/redis/redis.conf
RUN echo "protected-mode yes" >> /usr/local/etc/redis/redis.conf

EXPOSE 6379
CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]
```

```python
import os
import redis

password = os.getenv('REDIS_PASSWORD')
r = redis.Redis(
    host='redis-container',
    port=6379,
    password=password
)

r.ping()
```

### Scenario 3: Production with ACL

```conf
# Redis 6.0+ ACL
aclfile /etc/redis/users.acl

# users.acl
user default on >strongpassword ~* &* +@all
user app-user on >app_password ~* &* +get +set +del +incr
user readonly on >readonly_pass ~* &* +get +ping
```

```python
import redis

# Connect as specific user
r = redis.Redis(
    host='production-redis',
    port=6379,
    username='app-user',
    password='app_password'
)

r.ping()
```

### Scenario 4: High Availability with Sentinel

```conf
# Sentinel configuration
sentinel auth-pass mymaster strongpassword
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

```python
from redis.sentinel import Sentinel

sentinels = [('sentinel1', 26379), ('sentinel2', 26379)]
sentinel = Sentinel(
    sentinels,
    password='sentinel_password',
    socket_timeout=0.1
)

# Get Redis master
master = sentinel.master_for(
    'mymaster',
    socket_timeout=0.1,
    password='strongpassword'
)

master.ping()
```

## Troubleshooting

### Issue: "Protected mode enabled"

```python
# Problem
try:
    r = redis.Redis(host='remote-host', port=6379)
    r.ping()
except redis.ResponseError as e:
    print(f"Error: {e}")
    # DENIED Redis is running in protected mode...

# Solution 1: Add password
redis_cli -a yourpassword

# Solution 2: Set password in config
# requirepass yourpassword

# Solution 3: Disable (not recommended)
# CONFIG SET protected-mode no
```

### Issue: Can't Connect Locally with Protected Mode

```python
# This should work with protected mode on
r = redis.Redis(host='127.0.0.1', port=6379)  # Works!

# But this might not work
r = redis.Redis(host='localhost', port=6379)  # Should also work

# If DNS resolution issue:
r = redis.Redis(host='127.0.0.1', port=6379)  # Use IP instead
```

## Security Considerations

### When to Disable Protected Mode

1. **Never in production** ❌
2. **Behind firewall** - If Redis is behind NAT/firewall
3. **Development only** - Local machine only
4. **Docker private network** - Isolated container network

### Security Checklist

- [ ] Protected mode enabled in production
- [ ] Strong password set (if password auth required)
- [ ] Firewall restricts port 6379
- [ ] Bind to specific address (not 0.0.0.0)
- [ ] Use TLS for remote connections
- [ ] Regular password rotation
- [ ] Monitor connection attempts
- [ ] Disable dangerous commands (FLUSHDB, SHUTDOWN)

## Monitoring Protected Mode

### Check Configuration

```python
import redis

r = redis.Redis()

config = r.config_get('protected-mode')
print(f"Protected mode: {config['protected-mode']}")

requirepass = r.config_get('requirepass')
print(f"Password set: {'Yes' if requirepass['requirepass'] else 'No'}")
```

### Monitor Blocked Connections

```bash
# Using redis-cli with MONITOR
redis-cli -a password MONITOR | grep "DENIED"

# Using INFO
redis-cli -a password INFO stats | grep client_recent_max_input_buffer
```

## Next Steps

- [Security](14-security.md) - Comprehensive security guide
- [Configuration](4-configuration.md) - Full configuration options
- [ACL (Redis 6.0+)](14-security.md) - User-level access control

## Resources

- **Protected Mode**: https://redis.io/docs/management/security/
- **Configuration**: https://redis.io/docs/management/config/
- **TLS Support**: https://redis.io/docs/management/encryption/

## Summary

- Protected mode blocks external unauthenticated connections
- Enabled by default in Redis 3.2+
- Set password to allow external access
- Keep enabled in production
- Use TLS for remote connections
- Bind to specific addresses
- Disable only in isolated development environments
