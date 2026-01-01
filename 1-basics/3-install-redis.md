# Redis Installation Guide

## Installation Methods

Redis installation varies by platform. Choose the method that best suits your environment.

## macOS Installation

### Using Homebrew (Recommended)

```bash
# Install Redis
brew install redis

# Start Redis service
brew services start redis

# Verify installation
redis-cli ping
# Output: PONG

# View Redis information
redis-cli info
```

### Uninstall Redis
```bash
brew services stop redis
brew uninstall redis
```

### Manual Installation on macOS

```bash
# Download latest Redis
wget https://download.redis.io/redis-stable.tar.gz
tar -xzf redis-stable.tar.gz
cd redis-stable

# Compile
make
make test  # Run tests

# Install
sudo make install

# Create directory for data files
mkdir -p /usr/local/var/db/redis

# Start Redis server
redis-server /usr/local/etc/redis.conf
```

## Linux Installation

### Ubuntu/Debian

```bash
# Update package list
sudo apt-get update

# Install Redis
sudo apt-get install redis-server

# Verify installation
redis-cli --version

# Check status
sudo systemctl status redis-server

# Start/Stop Redis
sudo systemctl start redis-server
sudo systemctl stop redis-server
sudo systemctl restart redis-server

# Enable auto-start on boot
sudo systemctl enable redis-server
```

### CentOS/RHEL/Fedora

```bash
# Install Redis
sudo dnf install redis
# or
sudo yum install redis

# Start Redis
sudo systemctl start redis
sudo systemctl enable redis

# Check status
sudo systemctl status redis
```

### From Source (All Linux)

```bash
# Install build dependencies
sudo apt-get install build-essential tcl

# Download Redis
cd /tmp
wget https://download.redis.io/redis-stable.tar.gz
tar -xzf redis-stable.tar.gz
cd redis-stable

# Compile
make
make test  # Optional: run test suite

# Install
sudo make install PREFIX=/usr/local

# Create directory for data files
sudo mkdir -p /var/lib/redis
sudo chown redis:redis /var/lib/redis

# Copy configuration file
sudo cp redis.conf /etc/redis/redis.conf

# Create systemd service
sudo nano /etc/systemd/system/redis.service
```

### Systemd Service File Example
```ini
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/bin/kill -s TERM $MAINPID
Restart=on-failure
Type=notify

[Install]
WantedBy=multi-user.target
```

## Docker Installation

### Quick Start
```bash
# Run Redis container
docker run -d -p 6379:6379 --name redis redis:latest

# Connect to Redis
docker exec -it redis redis-cli

# Stop Redis
docker stop redis
docker rm redis
```

### Docker Compose (Recommended)

Create `docker-compose.yml`:
```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped

volumes:
  redis_data:
```

Run with:
```bash
docker-compose up -d
docker-compose down
docker-compose logs -f redis
```

### Docker with Persistence
```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:latest \
  redis-server --appendonly yes
```

## Windows Installation

### Using WSL (Windows Subsystem for Linux)

```bash
# Enable WSL
wsl --install

# Open WSL and install like Ubuntu
sudo apt-get update
sudo apt-get install redis-server

# Start Redis
redis-server
```

### Using Memurai (Windows Native)

1. Download from: https://github.com/microsoftarchive/memurai-releases
2. Run installer
3. Redis will run as Windows Service
4. Access via `redis-cli`

### Using Docker (Recommended)
See Docker section above

## Verification

After installation, verify Redis is working:

```bash
# Test connection
redis-cli ping
# Expected output: PONG

# Get server info
redis-cli info server

# Test basic commands
redis-cli SET mykey "Hello"
redis-cli GET mykey
# Expected output: "Hello"
```

## Configuration

### Default Configuration Location
```
macOS:   /usr/local/etc/redis.conf
Linux:   /etc/redis/redis.conf
Docker:  /usr/local/etc/redis/redis.conf
```

### Essential Configuration Parameters

```conf
# Server
port 6379              # Default port
bind 127.0.0.1         # Bind to localhost only
timeout 0              # Client timeout (0 = no timeout)

# Memory
maxmemory 256mb        # Maximum memory (set to your needs)
maxmemory-policy allkeys-lru  # Eviction policy

# Persistence
save 900 1             # RDB: save if 1 key changed in 900s
save 300 10            # RDB: save if 10 keys changed in 300s
save 60 10000          # RDB: save if 10000 keys changed in 60s
appendonly yes         # Enable AOF persistence
appendfsync everysec   # Sync AOF every second

# Logging
loglevel notice        # Log level: debug, verbose, notice, warning
logfile ""             # Log to stdout
```

## Starting Redis

### As Foreground Process
```bash
redis-server
```

### With Custom Configuration
```bash
redis-server /path/to/redis.conf
```

### As Background Service
```bash
# macOS
brew services start redis

# Linux (systemd)
sudo systemctl start redis-server

# Linux (manual background)
redis-server &
```

## Client Connection

### Using redis-cli

```bash
# Interactive mode
redis-cli

# Then type commands:
> PING
PONG
> SET name John
OK
> GET name
"John"
> EXIT

# Or single command
redis-cli PING
redis-cli SET name John
redis-cli GET name
```

### Using Python
```python
import redis

# Connect
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Test
if r.ping():
    print("Connected to Redis!")

# Basic operations
r.set('name', 'John')
print(r.get('name'))
```

### Using Node.js
```javascript
const redis = require('redis');

const client = redis.createClient({
  host: 'localhost',
  port: 6379
});

client.on('connect', () => {
  console.log('Connected to Redis');
});

client.set('name', 'John', (err, reply) => {
  client.get('name', (err, reply) => {
    console.log(reply);
  });
});
```

## Installation Troubleshooting

### "redis-cli: command not found"
```bash
# Add Redis to PATH
export PATH="/usr/local/bin:$PATH"

# Or reinstall with correct prefix
sudo make install PREFIX=/usr/local
```

### Port Already in Use
```bash
# Find process using port 6379
lsof -i :6379

# Kill the process
kill -9 <PID>

# Or use different port
redis-server --port 6380
```

### Permission Denied
```bash
# Fix file permissions
sudo chown -R redis:redis /var/lib/redis
sudo chmod 755 /var/lib/redis
```

### Connection Refused
```bash
# Check if Redis is running
ps aux | grep redis

# Check if listening on port
netstat -tuln | grep 6379

# Verify configuration
cat /etc/redis/redis.conf | grep bind
```

## Verifying Correct Installation

### Test Commands
```redis
PING
# Output: PONG

SELECT 0
# Output: OK

SET testkey "test value"
# Output: OK

GET testkey
# Output: "test value"

DEL testkey
# Output: (integer) 1

INFO
# Output: Server information
```

### Python Test Script
```python
import redis
import json

try:
    r = redis.Redis(host='localhost', port=6379, decode_responses=True)
    
    # Test connection
    r.ping()
    print("✓ Connected to Redis")
    
    # Test string
    r.set('test:string', 'hello')
    assert r.get('test:string') == 'hello'
    print("✓ String operations work")
    
    # Test list
    r.lpush('test:list', 'item1', 'item2')
    assert len(r.lrange('test:list', 0, -1)) == 2
    print("✓ List operations work")
    
    # Test hash
    r.hset('test:hash', 'field1', 'value1')
    assert r.hget('test:hash', 'field1') == 'value1'
    print("✓ Hash operations work")
    
    # Test set
    r.sadd('test:set', 'member1', 'member2')
    assert r.scard('test:set') == 2
    print("✓ Set operations work")
    
    # Cleanup
    r.delete('test:string', 'test:list', 'test:hash', 'test:set')
    
    print("\n✓ All tests passed! Redis is working correctly.")
    
except Exception as e:
    print(f"✗ Error: {e}")
```

## Next Steps

- [Configuration & Tuning](4-configuration.md)
- [Basic String Operations](6-strings.md)
- [Persistence](15-persistence.md)
- [Security](14-security.md)

## Resources

- **Official Downloads**: https://redis.io/download
- **Installation Guide**: https://redis.io/docs/getting-started/installation/
- **Docker Hub**: https://hub.docker.com/_/redis
- **GitHub**: https://github.com/redis/redis

## Summary

Redis installation is straightforward:
- **macOS**: `brew install redis`
- **Linux**: `apt-get install redis-server` or from source
- **Docker**: `docker run -p 6379:6379 redis:latest`
- **Windows**: Use WSL, Memurai, or Docker

Verify with `redis-cli ping` which should return `PONG`.
