# Redis Security

## Overview
Redis is an in-memory data store that requires careful security configuration to protect sensitive data and prevent unauthorized access. Unlike traditional databases with built-in security models, Redis security is primarily configured through application-level controls and network isolation.

⚠️ **Critical**: Redis is designed to be trusted. It assumes all users with access are trustworthy. Never expose Redis directly to the internet.

## Key Security Considerations

### 1. Authentication

#### Setting Password Authentication
```conf
# redis.conf
requirepass "your_strong_password_here"
# Password must be at least 8 characters for production
```

**Connecting with password:**
```bash
# Method 1: Pass password as argument
redis-cli -a "your_strong_password_here"

# Method 2: Authenticate after connecting
redis-cli
> AUTH "your_strong_password_here"
OK
```

#### Redis 6.0+ ACL (Access Control Lists)

**Create users with specific permissions:**
```redis
# Create a read-only user
ACL SETUSER readuser on >password123 ~* &* +get +mget -@all +@read

# Create an admin user
ACL SETUSER admin on >admin_password ~* &* +@all

# List all users
ACL LIST

# Get user details
ACL GETUSER admin

# Delete user
ACL DELUSER readuser
```

**ACL Configuration File (redis.conf or acl.conf):**
```conf
user default on >default_password ~* &* +@all
user readuser on >read_password ~* &* +@read +@fast -flushdb -flushall
user writeuser on >write_password ~* &* +@write +@read
user admin on >admin_password ~* &* +@all ~*
```

#### Best Practices for Authentication
- Use strong passwords (20+ characters, mixed case, numbers, symbols)
- Rotate passwords every 90 days
- Never hardcode passwords (use environment variables, secrets manager)
- Use different credentials for dev, staging, and production
- Prefer ACL over single `requirepass` for production
- Store passwords in environment variables or secret management tools
- Use different permissions for different services

### 2. Network Security

#### Bind Configuration
```conf
# redis.conf - DANGEROUS (exposes to everyone)
bind 0.0.0.0

# RECOMMENDED - Only localhost
bind 127.0.0.1

# RECOMMENDED - Specific internal IP
bind 10.0.1.5

# Multiple addresses
bind 127.0.0.1 10.0.1.5
```

#### Protected Mode
```conf
# redis.conf - Prevent external connections without authentication
protected-mode yes
```
When enabled:
- Rejects connections from non-localhost if no password set
- Prevents accidental exposure
- Still requires authentication even from localhost if requirepass is set

#### Firewall Configuration (iptables example)
```bash
# Allow only from specific IP
sudo iptables -A INPUT -p tcp --dport 6379 -s 10.0.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6379 -j DROP

# Allow specific host
sudo iptables -A INPUT -p tcp --dport 6379 -s 192.168.1.100 -j ACCEPT

# Verify rules
sudo iptables -L -n
```

#### TLS/SSL Configuration
```conf
# redis.conf - Enable TLS
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt
```

**Connecting with TLS:**
```bash
redis-cli --tls \
  --cert /path/to/client.crt \
  --key /path/to/client.key \
  --cacert /path/to/ca.crt \
  -h redis.example.com \
  -p 6380
```

#### VPN/SSH Tunneling
```bash
# SSH tunnel to Redis server
ssh -L 6379:localhost:6379 user@redis-server.com

# Then connect through tunnel
redis-cli -h localhost -p 6379
```

### 3. Configuration Security

#### Disable Dangerous Commands
```conf
# redis.conf - Completely disable commands
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG ""
```

#### Rename Commands to Hidden Names
```conf
# redis.conf - Rename to random/obscure names
rename-command FLUSHDB "FLUSHDB_$(date +%s)"
rename-command CONFIG "CONFIG_random_$(openssl rand -hex 8)"
rename-command SHUTDOWN ""
```

#### Commands to Consider Disabling/Renaming
| Command | Risk | Alternative |
|---------|------|-------------|
| FLUSHDB | Deletes all data in DB | Remove, rename, or ACL restrict |
| FLUSHALL | Deletes all data | Remove, rename, or ACL restrict |
| KEYS | Performance impact, information leak | Use SCAN with ACL |
| CONFIG | Exposes/modifies server config | Remove or use ACL |
| SHUTDOWN | Stops server | Remove or use ACL |
| DEBUG | Low-level debugging | Remove for production |
| MONITOR | Shows all commands | Restrict with ACL |

### 4. Data Protection

#### File Permissions
```bash
# Restrict redis.conf to owner only
chmod 600 /etc/redis/redis.conf

# Restrict RDB dump files
chmod 600 /var/lib/redis/dump.rdb

# Restrict AOF files
chmod 600 /var/lib/redis/appendonly.aof

# Restrict Redis data directory
chmod 700 /var/lib/redis/

# Verify permissions
ls -la /etc/redis/redis.conf
ls -la /var/lib/redis/
```

#### Persistence Security
```conf
# redis.conf - Secure persistence files
dir /var/lib/redis/          # Restricted directory
dbfilename dump.rdb
appendfilename appendonly.aof

# Backup location (different partition/server)
# Ensures data availability and disaster recovery
```

#### Encryption at Rest
Redis doesn't provide built-in encryption. Solutions:
```bash
# 1. Use encrypted filesystems (LUKS, FileVault)
# 2. Encrypt RDB dumps before backup
gpg --encrypt dump.rdb

# 3. Use Redis Enterprise with encryption
# 4. Implement application-level encryption
```

#### Backup Security
```bash
# Create encrypted backup
tar czf redis-backup.tar.gz /var/lib/redis/
gpg --symmetric redis-backup.tar.gz

# Store backup securely
# - Different server/location
# - Access restricted
# - Regularly test restore
```

### 5. Monitoring & Auditing

#### Enable Logging
```conf
# redis.conf
loglevel notice                    # debug, verbose, notice, warning
logfile "/var/log/redis/redis.log"
syslog-enabled yes
syslog-ident redis
```

#### Monitor Commands
```redis
# Monitor all commands in real-time
MONITOR

# Get slow commands
SLOWLOG GET 10

# Check specific command latency
SLOWLOG GET 100 WITHSCORES
```

#### Monitor Security Events
```bash
# Monitor authentication failures
grep "AUTH" /var/log/redis/redis.log

# Monitor dangerous commands
grep -E "FLUSHDB|FLUSHALL|CONFIG" /var/log/redis/redis.log

# Check client connections
redis-cli CLIENT LIST
```

#### Set Up Alerting
```bash
# Script to check for suspicious activity
#!/bin/bash

REDIS_CLI="redis-cli -a password"

# Check client count
CLIENTS=$($REDIS_CLI INFO clients | grep connected_clients)
if [ $CLIENTS -gt 100 ]; then
    echo "Alert: Too many clients" | mail -s "Redis Alert" admin@example.com
fi

# Check memory usage
MEMORY=$($REDIS_CLI INFO memory | grep used_memory_human)
echo "Memory usage: $MEMORY"
```

### 6. Best Practices Checklist

#### Development Environment
- ✅ Enable local access only (bind 127.0.0.1)
- ✅ Use `protected-mode yes`
- ✅ Enable logging
- ✅ Don't require complex passwords
- ✅ Use single database

#### Production Environment
- ✅ **NEVER** expose to internet (use VPN/private network)
- ✅ Enable requirepass with strong password
- ✅ Implement ACL with user-based permissions
- ✅ Bind to specific internal IP only
- ✅ Enable TLS for client connections
- ✅ Disable/rename dangerous commands
- ✅ Restrict file permissions (600 for configs, 700 for directories)
- ✅ Enable and monitor logging
- ✅ Implement firewall rules
- ✅ Regular backups with encryption
- ✅ Keep Redis updated
- ✅ Run Redis as non-root user
- ✅ Use separate Redis instances per environment
- ✅ Implement intrusion detection
- ✅ Regular security audits