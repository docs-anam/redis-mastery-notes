# Installing Redis

## Overview
Redis is an open-source, in-memory data structure store. This guide covers installation methods across different operating systems.

## Prerequisites
- Administrative access to your system
- Terminal/Command prompt
- Basic command-line knowledge

## Installation Methods

### macOS (Homebrew)
```bash
brew install redis
brew services start redis
```

### Linux (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```

### Linux (CentOS/RHEL)
```bash
sudo yum install redis
sudo systemctl start redis
sudo systemctl enable redis
```

### Windows
- Download Redis from [Microsoft's Redis GitHub](https://github.com/microsoftarchive/redis)
- Or use Windows Subsystem for Linux (WSL) with Ubuntu installation

### Docker
```bash
docker pull redis
docker run -d -p 6379:6379 redis
```

## Verify Installation
```bash
redis-cli --version
redis-cli ping
# Expected output: PONG
## What is Redis Server and Redis CLI

### Redis Server
**Redis Server** is the main Redis daemon process that manages the in-memory data store and handles client connections on port 6379 (default). The server stores all your data in RAM for ultra-fast access, which is why Redis is so performant. Once you start the Redis server, it runs continuously in the background, listening for incoming client connections.

**Key characteristics:**
- Runs as a background daemon process
- Listens on port 6379 by default
- Manages all in-memory data structures
- Handles multiple client connections simultaneously
- Persists data (optional) based on configuration

### Redis CLI (Client)
**Redis CLI** is the command-line interface client tool that allows you to connect to a running Redis Server and execute commands. It's the primary way to interact with Redis from the terminal for testing, debugging, and managing data.

**Key characteristics:**
- Lightweight client interface
- Connects to Redis server on localhost:6379 by default
- Execute commands and view results interactively
- Supports both interactive mode and script execution
- Essential for development and troubleshooting

## Running Redis Server and Client

### Starting Redis Server

After installation, start the Redis server:

**macOS:**
```bash
redis-server
# Or if using Homebrew services:
brew services start redis
```

**Linux:**
```bash
sudo systemctl start redis-server
# Or run directly:
redis-server
```

**Docker:**
```bash
docker run -d -p 6379:6379 redis
```

The server will start listening on port 6379. You should see output like:
```
* The server is now ready to accept connections on port 6379
```

### Connecting with Redis CLI

Once the server is running, open a new terminal and connect using the CLI:

```bash
redis-cli
# Expected output:
# 127.0.0.1:6379>
```

You're now connected! Try a basic command:
```bash
redis-cli ping
# Expected output: PONG
```

### Interactive Commands Example

Once connected with `redis-cli`, you can interact with Redis:

```bash
# Set a key-value pair
SET mykey "Hello Redis"

# Get the value
GET mykey
# Output: "Hello Redis"

# Check server status
INFO
```

### Exiting Redis CLI
```bash
EXIT
# or press Ctrl+D
```

## Installation URLs
- **Official Redis**: https://redis.io/download
- **Windows (Microsoft Archive)**: https://github.com/microsoftarchive/redis
- **Docker Hub**: https://hub.docker.com/_/redis
- **Homebrew**: https://formulae.brew.sh/formula/redis


## Configuration
- Config file location: `/etc/redis/redis.conf` (Linux) or `/usr/local/etc/redis.conf` (macOS)
- Default port: 6379
- Bind address: 127.0.0.1 (localhost)

## Next Steps
- Learn basic Redis commands
- Configure persistence options
- Set up replication or clustering