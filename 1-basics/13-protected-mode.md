
# Redis Protected Mode

## Overview
Protected mode is a security feature in Redis that prevents unauthorized clients from accessing the database when running in an insecure environment.

## What is Protected Mode?

Protected mode is enabled by default in Redis and acts as a safeguard against accidental exposure of Redis instances to the public internet. When enabled, Redis only accepts connections from:
- `127.0.0.1` (localhost)
- `::1` (IPv6 localhost)
- Unix sockets

## When Protected Mode Activates

Protected mode is automatically enabled when:
- No password is configured (`requirepass` is not set)
- The instance is not running in cluster mode
- The bind directive is set to accept connections from all interfaces (`0.0.0.0`)

## Configuration

### Enable/Disable
```
protected-mode yes|no
```

Set in `redis.conf` or via CLI:
```bash
CONFIG SET protected-mode yes
```

### Related Settings
- `requirepass`: Set a password to allow remote connections
- `bind`: Specify which IP addresses to listen on
- `port`: Configure the listening port

## Security Implications

**With Protected Mode ON:**
- Remote clients are rejected with an error message
- Only local connections are accepted
- Default safe configuration for development

**With Protected Mode OFF:**
- Any client can connect if network allows
- Should only be disabled if properly secured:
    - Strong password set
    - Firewall restrictions in place
    - Running in trusted network environment

## When to Disable

- Production environments with proper security measures
- Cloud deployments with configured authentication
- When using Redis in a private/secure network
