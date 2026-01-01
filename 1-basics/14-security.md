# Redis Security

## Overview

Redis security involves authentication, network isolation, command restrictions, and TLS encryption. Multi-layered approach prevents unauthorized access and data breaches.

## Authentication

### Basic Password Authentication

```conf
# redis.conf
requirepass "your_secure_password"
```

```python
import redis

# Connect with password
r = redis.Redis(
    host='localhost',
    port=6379,
    password='your_secure_password'
)

r.ping()  # Returns True
```

### Password Requirements

```
Good password:
- At least 16 characters
- Mix of uppercase, lowercase, numbers, symbols
- Random, not dictionary words
- Example: "k9$mP@x2qL$nR&vW3zT"

Weak passwords to avoid:
- "redis" (too common)
- "password123" (predictable)
- "admin" (default-like)
```

## ACL (Access Control List) - Redis 6.0+

### User Management

```redis
# List all users
ACL LIST

# Create user
ACL SETUSER alice on >password ~* &* +@all
ACL SETUSER bob on >password ~user:* &* +get +set +del

# Get user info
ACL GETUSER alice

# Delete user
ACL DELUSER bob

# Save ACL config
ACL SAVE
```

### ACL Syntax Breakdown

```
ACL SETUSER <username> <flags> <passwords> <patterns> <permissions>

Flags:
  on/off              - Enable/disable user
  +nopass             - No password required
  
Passwords:
  >password           - Add password
  <password           - Remove password
  #hash               - SHA256 hash
  
Key Patterns:
  ~*                  - All keys
  ~user:*             - Keys matching pattern
  ~*:session:*        - Nested patterns
  
Permissions:
  +@all               - All permissions
  +@admin             - Admin commands
  +@read              - Read commands
  +get +set +del      - Specific commands
  -flushdb            - Deny specific command
```

### Python ACL Examples

```python
import redis

r = redis.Redis(password='admin_password')

# Create application user (read-write)
r.acl_setuser(
    'app-user',
    enabled=True,
    passwords=['app_password'],
    keys=['~app:*'],
    categories=['+@all']
)

# Create read-only user
r.acl_setuser(
    'readonly',
    enabled=True,
    passwords=['readonly_pass'],
    keys=['~*'],
    categories=['+@read']
)

# Connect as app user
app_redis = redis.Redis(
    username='app-user',
    password='app_password'
)

app_redis.get('app:key')  # Works
app_redis.flushdb()       # Denied - not in permissions
```

## Network Security

### Bind to Specific Address

```conf
# redis.conf

# Option 1: Localhost only
bind 127.0.0.1

# Option 2: Specific interface
bind 192.168.1.100

# Option 3: Multiple interfaces
bind 127.0.0.1 192.168.1.100

# Option 4: Disable network (socket only)
bind ""
```

### Disable Dangerous Commands

```conf
# redis.conf

# Disable commands entirely
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG ""
rename-command SHUTDOWN ""

# Rename to obscure commands
rename-command FLUSHDB "xyzzy_flushdb_xyzzy"
rename-command EVAL "xyzzy_eval_xyzzy"
```

### TLS/SSL Encryption

```conf
# redis.conf
port 0                          # Disable unencrypted port
tls-port 6380                   # TLS port
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-dh-params-file /path/to/redis.dh
tls-ca-cert-file /path/to/ca.pem

# Require TLS for replicas
tls-replication yes

# Require TLS for cluster
tls-cluster yes
```

```python
import redis

# Connect via TLS
r = redis.Redis(
    host='secure-redis.com',
    port=6380,
    password='password',
    ssl=True,
    ssl_cert_reqs='required',
    ssl_ca_certs='/path/to/ca.pem',
    ssl_certfile='/path/to/client.crt',
    ssl_keyfile='/path/to/client.key'
)

r.ping()
```

## Firewall Configuration

### iptables (Linux)

```bash
# Allow only specific IP
sudo iptables -A INPUT -p tcp -s 192.168.1.100 --dport 6379 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP

# Allow from specific range
sudo iptables -A INPUT -p tcp -s 192.168.1.0/24 --dport 6379 -j ACCEPT
```

### UFW (Ubuntu)

```bash
# Allow from specific IP
sudo ufw allow from 192.168.1.100 to any port 6379

# Allow from subnet
sudo ufw allow from 192.168.1.0/24 to any port 6379

# Deny all others
sudo ufw default deny incoming
```

## Monitoring and Auditing

### ACL Logging

```redis
# Enable ACL logging
ACL LOG RESET
ACL LOG GET 10

# Output shows denied commands and failed auth attempts
```

### CLIENT INFO

```python
import redis

r = redis.Redis(password='password')

# List all clients
clients = r.client_list()
for client in clients:
    print(client)

# Get current client info
info = r.client_info()
print(f"Client: {info}")

# Monitor client connections
# Terminal: redis-cli
# redis> CLIENT LIST
# redis> MONITOR
```

## Security Patterns

### Pattern 1: Development vs Production

```python
import os
import redis

def get_redis():
    """Get Redis with environment-specific config"""
    env = os.getenv('ENVIRONMENT', 'development')
    
    if env == 'production':
        return redis.Redis(
            host=os.getenv('REDIS_HOST'),
            port=int(os.getenv('REDIS_PORT', 6379)),
            password=os.getenv('REDIS_PASSWORD'),
            ssl=True,
            username=os.getenv('REDIS_USERNAME'),
            # ACL enabled
        )
    else:  # Development
        return redis.Redis(
            host='localhost',
            port=6379
        )

r = get_redis()
```

### Pattern 2: Credential Management

```python
import redis
import os
from cryptography.fernet import Fernet

def get_secure_redis():
    """Get Redis with encrypted credentials"""
    # Load from environment or secure vault
    host = os.getenv('REDIS_HOST')
    port = int(os.getenv('REDIS_PORT', 6379))
    
    # Encrypted password from vault
    encrypted_password = os.getenv('REDIS_PASSWORD_ENCRYPTED')
    key = os.getenv('ENCRYPTION_KEY')
    
    cipher = Fernet(key)
    password = cipher.decrypt(encrypted_password).decode()
    
    return redis.Redis(
        host=host,
        port=port,
        password=password,
        ssl=True
    )

r = get_secure_redis()
```

### Pattern 3: Key Expiration and Cleanup

```python
import redis

class SecureRedis:
    def __init__(self, password):
        self.r = redis.Redis(
            password=password,
            decode_responses=True
        )
    
    def set_temporary(self, key, value, ttl=3600):
        """Set with automatic expiration"""
        self.r.setex(key, ttl, value)
    
    def cleanup_expired(self):
        """Remove expired sensitive keys"""
        # Redis does this automatically
        # But can manually cleanup if needed
        pass

secure_redis = SecureRedis('password')
secure_redis.set_temporary('session:abc123', '{}', ttl=1800)
```

## Security Checklist

### Development Environment
- [ ] Protected mode enabled
- [ ] Localhost binding only
- [ ] No sensitive data stored
- [ ] Password set (even if weak)
- [ ] MONITOR disabled

### Production Environment
- [ ] Protected mode enabled ✓✓✓
- [ ] Strong password (16+ chars, random)
- [ ] ACL configured with roles
- [ ] TLS/SSL encryption enabled
- [ ] Specific IP binding (firewall)
- [ ] Dangerous commands disabled (FLUSHDB, etc)
- [ ] Regular password rotation
- [ ] ACL logging enabled
- [ ] Firewall rules configured
- [ ] Regular security audits

### Compliance
- [ ] Data encryption at rest (RDB/AOF encrypted)
- [ ] Encryption in transit (TLS)
- [ ] Access logging
- [ ] Audit trails
- [ ] Backup security
- [ ] Disaster recovery plan

## Common Attacks and Prevention

### Attack 1: Brute Force Attack

**Prevention:**
```python
# Use strong passwords
password = "k9$mP@x2qL$nR&vW3zT"  # 32 character, mixed

# Limit connection attempts (firewall level)
# fail2ban example
# bantime = 3600  # Ban for 1 hour
# maxretry = 5    # After 5 failed attempts
```

### Attack 2: Man-in-the-Middle (MITM)

**Prevention:**
```conf
# Use TLS to encrypt traffic
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
```

### Attack 3: Unauthorized Command Execution

**Prevention:**
```conf
# Disable dangerous commands
rename-command FLUSHDB ""
rename-command EVAL ""
rename-command SCRIPT ""
rename-command SHUTDOWN ""
```

### Attack 4: Data Exfiltration

**Prevention:**
```python
# Use READ-ONLY users
r.acl_setuser('readonly', categories=['+@read'])

# Encrypt sensitive data before storing
from cryptography.fernet import Fernet
cipher = Fernet(key)
encrypted = cipher.encrypt(sensitive_data.encode())
r.set('encrypted_key', encrypted)
```

## Next Steps

- [Persistence](15-persistence.md) - Secure backup strategies
- [Protected Mode](13-protected-mode.md) - Network isolation
- [Configuration](4-configuration.md) - All security settings

## Resources

- **Redis Security**: https://redis.io/docs/management/security/
- **ACL**: https://redis.io/docs/management/acl/
- **TLS**: https://redis.io/docs/management/encryption/

## Summary

- Use strong passwords and ACL for authentication
- Bind to specific addresses and use firewalls
- Enable TLS for remote connections
- Disable dangerous commands
- Monitor and audit access
- Encrypt data at rest and in transit
- Regular security reviews and updates
