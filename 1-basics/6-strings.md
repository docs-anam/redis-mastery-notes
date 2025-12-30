# Redis Strings

## Overview
Strings are the most basic Redis data type. They store sequences of bytes and are binary-safe, meaning they can contain any kind of data (text, numbers, serialized objects).

## Key Characteristics
- **Binary-safe**: Can store any binary data
- **Maximum size**: 512 MB per string
- **Simple key-value pairs**: Straightforward get/set operations
- **Atomic operations**: Increment/decrement operations are atomic

## Common Commands

### Basic Operations
- `SET key value` - Set a string value
- `GET key` - Retrieve a string value
- `GETSET key value` - Set new value and return old value
- `MSET key1 value1 key2 value2` - Set multiple strings
- `MGET key1 key2` - Get multiple strings

### Numeric Operations
- `INCR key` - Increment integer value by 1
- `INCRBY key increment` - Increment by specific amount
- `DECR key` - Decrement integer value by 1
- `DECRBY key decrement` - Decrement by specific amount
- `INCRBYFLOAT key float` - Increment by floating-point value

### String Manipulation
- `APPEND key value` - Append value to existing string
- `STRLEN key` - Get string length
- `GETRANGE key start end` - Get substring
- `SETRANGE key offset value` - Overwrite part of string

### Expiration & Deletion
- `SETEX key seconds value` - Set with expiration time
- `PSETEX key milliseconds value` - Set with millisecond expiration
- `DEL key` - Delete a key

## Common Commands

### Basic Operations
- `SET key value` - Set a string value
- `GET key` - Retrieve a string value
- `GETSET key value` - Set new value and return old value
- `MSET key1 value1 key2 value2` - Set multiple strings
- `MGET key1 key2` - Get multiple strings
- `SETNX key value` - Set only if key does NOT exist (returns 1 if set, 0 if exists)
- `GETDEL key` - Get value and delete the key

### Numeric Operations
- `INCR key` - Increment integer value by 1
- `INCRBY key increment` - Increment by specific amount
- `DECR key` - Decrement integer value by 1
- `DECRBY key decrement` - Decrement by specific amount
- `INCRBYFLOAT key float` - Increment by floating-point value

### String Manipulation
- `APPEND key value` - Append value to existing string
- `STRLEN key` - Get string length
- `GETRANGE key start end` - Get substring
- `SETRANGE key offset value` - Overwrite part of string

### Expiration & Deletion
- `SETEX key seconds value` - Set with expiration time
- `PSETEX key milliseconds value` - Set with millisecond expiration
- `DEL key` - Delete a key
- `EXISTS key` - Check if key exists
- `EXPIRE key seconds` - Set expiration on existing key
- `TTL key` - Get time to live in seconds (-1 if no expiration, -2 if not exists)

## Implementation Examples

### 1. Basic Key-Value Storage
```bash
# Set a simple string value
SET username:1001 "John Doe"
GET username:1001
# Output: "John Doe"

# Set multiple values at once
MSET user:1:name "Alice" user:1:email "alice@example.com" user:2:name "Bob"
MGET user:1:name user:1:email user:2:name
# Output: 
# 1) "Alice"
# 2) "alice@example.com"
# 3) "Bob"
```

### 2. Counters and Increments
```bash
# Initialize a counter
SET page:views 0
INCR page:views
INCR page:views
GET page:views
# Output: "2"

# Increment by specific amount
INCRBY user:1:points 10
INCRBY user:1:points 5
GET user:1:points
# Output: "15"

# Floating-point counter (e.g., temperature, ratings)
SET temp:sensor1 22.5
INCRBYFLOAT temp:sensor1 0.5
GET temp:sensor1
# Output: "23"
```

### 3. Session Management
```bash
# Store user session with expiration (30 seconds)
SETEX session:abc123 30 '{"user_id":101,"role":"admin"}'
GET session:abc123
# Output: '{"user_id":101,"role":"admin"}'

# Session expires after 30 seconds
SLEEP 31
GET session:abc123
# Output: (nil)

# Check expiration time
SETEX session:xyz789 3600 '{"user_id":202}'
TTL session:xyz789
# Output: 3599 (approximately)
```

### 4. Atomic Get and Set
```bash
# Get old value and set new one atomically
SET status:server1 "online"
GETSET status:server1 "offline"
# Output: "online"
GET status:server1
# Output: "offline"

# Set only if doesn't exist
SETNX newuser:1 "John"
# Output: 1 (success)
SETNX newuser:1 "Jane"
# Output: 0 (failed, key exists)
GET newuser:1
# Output: "John"
```

### 5. String Manipulation
```bash
# Append to existing string
SET message "Hello"
APPEND message " World"
GET message
# Output: "Hello World"

# Get substring (0-indexed)
GETRANGE message 0 4
# Output: "Hello"

# Overwrite part of string
SETRANGE message 6 "Redis"
GET message
# Output: "Hello Redis"

# String length
STRLEN message
# Output: 11
```

### 6. Rate Limiting Implementation
```bash
# Simple rate limiting: 10 requests per minute
SET rate:user:1001 10
DECR rate:user:1001
DECR rate:user:1001
GET rate:user:1001
# Output: "8"

# Reset limit with expiration
SETEX rate:user:1001 60 10
# Limit lasts for 60 seconds

# Check if limit exceeded
DECR rate:user:1001
# Output: 9 (allowed)
```

### 7. Caching JSON Objects
```bash
# Store serialized JSON
SET user:1001:profile '{"id":1001,"name":"Alice","email":"alice@example.com","age":28}'
GET user:1001:profile
# Output: '{"id":1001,"name":"Alice","email":"alice@example.com","age":28}'

# Cache with expiration (1 hour)
SETEX product:5001:details 3600 '{"id":5001,"name":"Laptop","price":999.99,"stock":50}'

# Retrieve and use
GET product:5001:details
```

### 8. Atomic Counters for Analytics
```bash
# Page view counter
INCR analytics:page:homepage:views
INCR analytics:page:homepage:views
GET analytics:page:homepage:views
# Output: "2"

# Multi-day analytics with rolling window
SETEX analytics:daily:2025-12-31 86400 100
SETEX analytics:daily:2026-01-01 86400 150
MGET analytics:daily:2025-12-31 analytics:daily:2026-01-01
# Output:
# 1) "100"
# 2) "150"

# Increment and expire in one command
SETEX clicks:campaign:xyz 3600 0
INCR clicks:campaign:xyz
INCR clicks:campaign:xyz
```

### 9. Application Settings Storage
```bash
# Store application configuration
MSET app:config:db_host "localhost" app:config:db_port "5432" app:config:timeout "30"
MGET app:config:db_host app:config:db_port app:config:timeout
# Output:
# 1) "localhost"
# 2) "5432"
# 3) "30"

# Update single setting
SET app:config:debug_mode "true"
```

### 10. Distributed Lock with Expiration
```bash
# Acquire lock
SETNX lock:resource:1 "process-123"
# Output: 1 (success)

# Lock expires after 10 seconds
SETEX lock:resource:1 10 "process-123"

# Check if locked
EXISTS lock:resource:1
# Output: 1 (locked)

# Release lock
DEL lock:resource:1
```

## Use Cases
- Caching user sessions
- Storing counters and analytics
- Rate limiting
- Storing JSON or serialized objects
- Real-time leaderboards
- Distributed locks with TTL
- Feature flags and configuration management
- Activity tracking and timestamps
- Distributed atomic operations

## Best Practices

### Key Naming Conventions
```
user:{id}:profile
user:{id}:session
cache:{type}:{id}
counter:{metric}:{date}
lock:{resource}:{id}
```

### Memory Optimization
- Use smaller key names in production
- Compress large string values if possible
- Set appropriate TTL to prevent memory bloat
- Monitor memory usage with `INFO memory`

### Performance Tips
- Use `MGET`/`MSET` instead of individual `GET`/`SET` for bulk operations
- Use pipelines when executing multiple commands
- Atomic operations are thread-safe (INCR, APPEND, etc.)
- Remember: All operations are O(1) except GETRANGE, SETRANGE, and APPEND which are O(n)

### Data Type Considerations
| Scenario | Best Choice | Reason |
|----------|------------|--------|
| Simple key-value | String | O(1) operations, simplest structure |
| Incrementing counter | String (INCR) | Atomic, fast, thread-safe |
| Session storage | String (JSON) | Fast serialization/deserialization |
| Cache expiration | SETEX | Built-in TTL support |
| Multiple related values | String (JSON) or Hash | String for serialized, Hash for structured |
| Bit operations | String (SETBIT) | Space-efficient for flags |

## Command Comparison

### SET Variants
```bash
SET key value              # Simple set
SETNX key value           # Set if not exists (0 if exists)
SETEX key seconds value   # Set with seconds expiration
PSETEX key ms value       # Set with milliseconds expiration
GETSET key value          # Atomic get-then-set
```

### GET Variants
```bash
GET key                   # Simple get
MGET key1 key2 key3       # Get multiple values
GETRANGE key 0 4          # Get substring
GETDEL key                # Get and delete
GETEX key EX 10           # Get and set expiration
```

## Performance Analysis

| Operation | Time Complexity | Use Case |
|-----------|-----------------|----------|
| SET / GET | O(1) | High-frequency access |
| INCR / DECR | O(1) | Counters, analytics |
| APPEND | O(n) | Build strings gradually |
| STRLEN | O(1) | Quick length check |
| GETRANGE | O(n) | Extract substrings |
| SETRANGE | O(n) | Partial overwrites |
| MGET / MSET | O(n) | Bulk operations |

## Performance
Strings offer O(1) time complexity for most operations, making them extremely fast for simple key-value access patterns.