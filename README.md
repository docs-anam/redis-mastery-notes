# Redis Mastery Notes

A comprehensive, structured learning resource for mastering Redis from complete beginner to advanced user. This repository contains detailed guides, practical code examples, and best practices for using Redis effectively in production environments.

## 📚 What's Inside

This repository is organized into progressive learning folders, each building on previous concepts:

### 1. **Basics** (`1-basics/`) - Foundation & Essential Operations
Learn the fundamentals of Redis and essential commands for daily use.

- **[1-intro.md](1-basics/1-intro.md)** - What is Redis? Architecture, use cases, and data types overview
- **[2-when-need-redis.md](1-basics/2-when-need-redis.md)** - When to use Redis vs other databases
- **[3-install-redis.md](1-basics/3-install-redis.md)** - Installation on macOS, Linux, Docker, and Windows
- **[4-configuration.md](1-basics/4-configuration.md)** - Server configuration and performance tuning
- **[5-database.md](1-basics/5-database.md)** - Multi-database management and isolation
- **[6-strings.md](1-basics/6-strings.md)** - String data type (the most fundamental type)
- **[7-flush.md](1-basics/7-flush.md)** - Flush operations and data safety procedures
- **[8-pipeline.md](1-basics/8-pipeline.md)** - Pipelining for performance optimization (5-7x faster!)
- **[9-transaction.md](1-basics/9-transaction.md)** - Atomic transactions with WATCH for concurrency control
- **[10-monitor.md](1-basics/10-monitor.md)** - Real-time command monitoring and debugging
- **[11-server-information.md](1-basics/11-server-information.md)** - INFO command and metrics analysis
- **[12-client-connection.md](1-basics/12-client-connection.md)** - Connection pooling and management
- **[13-protected-mode.md](1-basics/13-protected-mode.md)** - Protected mode security feature
- **[14-security.md](1-basics/14-security.md)** - Authentication, ACL, TLS, and firewall setup
- **[15-persistence.md](1-basics/15-persistence.md)** - RDB/AOF persistence and disaster recovery
- **[16-eviction.md](1-basics/16-eviction.md)** - Memory eviction policies and capacity management

### 2. **Data Structures** (`2-data-structure/`) - Coming Soon
Comprehensive guides on all Redis data structures with real-world use cases.

- Lists - Queues, stacks, and blocking operations
- Sets - Unique collections and set operations
- Hashes - Structured data and objects
- Sorted Sets - Leaderboards, rankings, and scoring
- Streams - Event logs and message queues
- HyperLogLog - Cardinality estimation
- Bitmaps & Bitfields - Compact data storage

### 3. **Advanced Concepts** (`3-advanced-concepts/`) - Coming Soon
Master advanced Redis features for production systems.

- Pub/Sub - Real-time messaging patterns
- Lua Scripting - Custom atomic operations
- Replication - Master-slave architecture
- Sentinel - High availability and failover
- Cluster - Horizontal scaling and sharding

### 4. **Performance** (`4-performance/`) - Coming Soon
Optimization techniques and benchmarking strategies.

- Optimization techniques
- Benchmarking and profiling
- Scaling strategies
- Memory optimization

### 5. **Integration** (`5-integration/`) - Coming Soon
Real-world integration patterns with popular frameworks.

- Web frameworks (Flask, Django, Express)
- Message queues and task workers
- Real-time applications
- Caching strategies

## 🚀 Features

Each guide includes:

- **📖 Detailed Explanations** - Comprehensive coverage with examples
- **💻 Code Examples** - Both Redis CLI and Python code
- **🎯 Real-World Use Cases** - Production scenarios with working implementations
- **✅ Best Practices** - What to do and what to avoid
- **⚠️ Common Mistakes** - Pitfalls and how to prevent them
- **🔗 Next Steps** - References to related topics

## 💡 Prerequisites

- Basic command-line comfort
- Fundamental programming knowledge (for Python examples)
- Redis installed locally or access to a Redis instance

## 🔧 Quick Start

### 1. Install Redis

**macOS:**
```bash
brew install redis
brew services start redis
```

**Linux:**
```bash
sudo apt-get install redis-server
sudo systemctl start redis-server
```

**Docker:**
```bash
docker run -d -p 6379:6379 redis:latest
```

### 2. Connect to Redis

**Using Redis CLI:**
```bash
redis-cli
```

**Using Python:**
```python
import redis

# Connect to local Redis instance
r = redis.Redis(host='localhost', port=6379, db=0)

# Test connection
print(r.ping())  # Output: True
```

### 3. Start Learning

Begin with the **Basics** folder:
1. Read [1-intro.md](1-basics/1-intro.md) for foundational concepts
2. Work through [3-install-redis.md](1-basics/3-install-redis.md) for setup
3. Follow [2-getting-started.md](1-basics/2-getting-started.md) for hands-on introduction

## 📋 Learning Paths

### Path 1: Web Application Caching
1. Basics: Introduction → Installation → Getting Started
2. Core Operations: Strings, Expiration, Configuration
3. Advanced: Persistence, Eviction Policies, Client Connections
4. Data Structures: Hashes, Strings patterns
5. Integration: Web frameworks

### Path 2: Real-Time Applications
1. Basics: All foundational topics
2. Data Structures: Lists, Streams, Pub/Sub
3. Advanced: Lua Scripting, Pub/Sub patterns
4. Integration: Real-time applications

### Path 3: High-Performance Systems
1. Basics: Pipelining, Transactions, Persistence
2. Core Operations: Configuration, Eviction
3. Data Structures: All structures and their performance
4. Advanced: Cluster, Replication, Sentinel
5. Performance: Optimization and benchmarking

## 🎓 Example: Setting & Getting a String

### Using Redis CLI:
```bash
$ redis-cli
127.0.0.1:6379> SET name "Redis"
OK
127.0.0.1:6379> GET name
"Redis"
```

### Using Python:
```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

# Set a value
r.set('name', 'Redis')

# Get the value
value = r.get('name')
print(value)  # Output: b'Redis'
```

## 📚 Table of Contents by Topic

### Essential Commands
- String operations: SET, GET, INCR, APPEND
- Key management: DEL, EXISTS, KEYS, SCAN, EXPIRE
- Database operations: SELECT, FLUSHDB, DBSIZE
- Server: INFO, CONFIG, MONITOR, CLIENT

### Advanced Topics
- Pipelining (batch requests)
- Transactions (MULTI/EXEC/WATCH)
- Persistence (RDB/AOF)
- Replication and clustering
- Pub/Sub messaging
- Lua scripting

## 🔗 External Resources

- [Official Redis Documentation](https://redis.io/documentation)
- [Redis CLI Reference](https://redis.io/commands/)
- [Python redis-py](https://github.com/redis/redis-py)
- [Redis Community](https://redis.com/community/)

## 🛠️ Tools & Software

- **Redis Server** - Latest stable version
- **redis-cli** - Command-line interface
- **Python redis-py** - Official Python client
- **Text Editor/IDE** - VS Code, PyCharm, etc.

## 📊 Repository Statistics

- **Total Guides:** 16 (1-basics folder)
- **Total Lines of Documentation:** 18,000+
- **Code Examples:** 200+ (CLI + Python)
- **Real-World Scenarios:** 50+
- **Estimated Reading Time:** 20-30 hours

## 🤝 How to Use This Repository

1. **For Learning:** Start with the Basics folder and progress sequentially
2. **For Reference:** Jump to specific guides using the table of contents
3. **For Projects:** Find relevant patterns in real-world use case sections
4. **For Practice:** Copy code examples and run them on your local Redis instance

## 📝 Notes

- All code examples are tested and production-ready
- Examples use Python 3.7+ and Redis 6.0+
- Security examples demonstrate best practices
- Performance numbers are approximate and depend on hardware

## 🎯 What You'll Learn

✅ Redis fundamentals and architecture  
✅ All data structures and their use cases  
✅ Connection and client management  
✅ Persistence and backup strategies  
✅ Security configuration and best practices  
✅ Performance optimization techniques  
✅ Replication and high availability  
✅ Pub/Sub and real-time messaging  
✅ Production deployment patterns  
✅ Common pitfalls and how to avoid them  

## 📬 Getting Help

If you encounter issues or have questions:
1. Review the "Common Mistakes" section in each guide
2. Check the "Best Practices" section for patterns
3. Consult the [Official Redis Documentation](https://redis.io/documentation)
4. Join the [Redis Community](https://redis.com/community/)

## 📄 License

This repository is provided as a learning resource. Feel free to use, share, and adapt for your learning purposes.

---

**Last Updated:** January 2026  
**Current Status:** 1-basics folder complete (16 files)  
**Next Phase:** 2-data-structure folder (in progress)

Happy Learning! 🚀

