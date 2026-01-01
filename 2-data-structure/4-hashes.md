# Redis Hashes

## Overview
Redis hashes are collections of field-value pairs, similar to objects, dictionaries, or maps in programming languages. They're ideal for storing and retrieving structured data with multiple attributes. Hashes provide more memory efficiency and better organization than storing multiple related string keys.

## Key Characteristics
- **Structure**: Key → {field1: value1, field2: value2, ...}
- **Data Type**: Hash (mapping/dictionary-like collection)
- **Memory Efficient**: Better than N individual string keys
- **Atomic Operations**: Individual field operations are atomic
- **Partial retrieval**: Can fetch specific fields without loading entire hash
- **Numeric operations**: Direct increment/decrement without serialization

## Common Commands

### Basic Field Operations
| Command | Complexity | Description |
|---------|-----------|-------------|
| `HSET key field value [field value ...]` | O(N) | Set one or more fields |
| `HGET key field` | O(1) | Get single field value |
| `HMGET key field [field ...]` | O(N) | Get multiple field values |
| `HGETALL key` | O(N) | Get all fields and values |
| `HSTRLEN key field` | O(1) | Get value length in bytes |

### Field Management
| Command | Complexity | Description |
|---------|-----------|-------------|
| `HDEL key field [field ...]` | O(N) | Delete one or more fields |
| `HEXISTS key field` | O(1) | Check if field exists |
| `HKEYS key` | O(N) | Get all field names |
| `HVALS key` | O(N) | Get all values |
| `HLEN key` | O(1) | Count total fields |
| `HRANDFIELD key [count]` | O(N) | Get random fields |

### Numeric Operations
| Command | Complexity | Description |
|---------|-----------|-------------|
| `HINCRBY key field increment` | O(1) | Increment integer value |
| `HINCRBYFLOAT key field increment` | O(1) | Increment float value |

### Conditional Operations
| Command | Complexity | Description |
|---------|-----------|-------------|
| `HSETNX key field value` | O(1) | Set only if field doesn't exist |

## Use Cases

### 1. User Profiles
Store complete user information efficiently.

```redis
// Create user profile
HSET user:1 name "John Doe" email "john@example.com" age 30 city "New York" premium 1
HSET user:2 name "Jane Smith" email "jane@example.com" age 28 city "Boston"

// Get specific user info
HGET user:1 email  // Returns: "john@example.com"

// Get multiple fields
HMGET user:1 name email age  // Returns: ["John Doe", "john@example.com", "30"]

// Get entire profile
HGETALL user:1

// Update single field
HSET user:1 city "San Francisco"

// Increment age on birthday
HINCRBY user:1 age 1

// Delete field
HDEL user:1 premium
```

**Real-world scenario**: User account management, CRM systems.

### 2. Product Catalogs
Store product attributes with efficient updates.

```redis
// Create product
HSET product:123 name "Laptop" price 999.99 stock 50 category "Electronics" rating 4.5
HSET product:123 description "High-performance laptop" manufacturer "TechCorp"

// Get product price
HGET product:123 price  // Returns: "999.99"

// Get for display
HMGET product:123 name price rating  // For product list

// Update stock when sold
HINCRBY product:123 stock -1  // Decrement by 1

// Update rating
HINCRBYFLOAT product:123 rating 0.1

// Check if in stock
HGET product:123 stock
```

**Real-world scenario**: E-commerce systems, inventory management, catalog data.

### 3. Session Data
Manage user session information.

```redis
// Create session
HSET session:abc123 user_id "user:1" ip "192.168.1.1" login_time "2024-01-15T10:30:00Z"
HSET session:abc123 last_activity "2024-01-15T10:35:00Z" device "mobile"

// Get session
HGETALL session:abc123

// Update last activity
HSET session:abc123 last_activity "2024-01-15T10:40:00Z"

// Set expiration for cleanup
EXPIRE session:abc123 3600  // 1 hour session timeout
```

**Real-world scenario**: Web session management, API authentication.

### 4. Counters & Metrics
Track metrics with built-in increment operations.

```redis
// Page view counters
HINCRBY stats:2024-01-15 page:home 100
HINCRBY stats:2024-01-15 page:about 50
HINCRBY stats:2024-01-15 page:product 75

// Get all stats
HGETALL stats:2024-01-15

// User statistics
HSET user:1:stats posts 42 followers 1500 following 300 likes 5000
HINCRBY user:1:stats posts 1  // New post
HINCRBY user:1:stats likes 1   // New like
```

**Real-world scenario**: Analytics, page views, performance metrics.

### 5. Configuration Objects
Store application settings hierarchically.

```redis
// Email configuration
HSET config:email smtp_host "mail.example.com" smtp_port 587 from_addr "noreply@example.com"
HSET config:email username "smtp_user" password "encrypted_pwd" tls_enabled 1

// Database configuration
HSET config:db host "localhost" port 5432 database "myapp" pool_size 10

// Get setting
HGET config:email smtp_host

// Get all config
HGETALL config:db
```

**Real-world scenario**: Application configuration, feature flags.

### 6. Shopping Cart
Manage cart items with quantities.

```redis
// Add items to cart
HSET cart:user:123 product:101 2  // 2 units of product 101
HSET cart:user:123 product:202 1  // 1 unit of product 202
HSET cart:user:123 product:303 3  // 3 units of product 303

// Get cart
HGETALL cart:user:123  // Returns all items and quantities

// Update quantity
HINCRBY cart:user:123 product:101 1  // Add 1 more
HSET cart:user:123 product:101 0    // Or set quantity

// Remove item
HDEL cart:user:123 product:101

// Clear cart
DEL cart:user:123
```

**Real-world scenario**: E-commerce carts, shopping baskets.

### 7. Leaderboard with Metadata
Store scores with additional information.

```redis
// Player data
HSET player:1 username "Alice" level 50 experience 95000 last_win "2024-01-15"
HSET player:2 username "Bob" level 48 experience 88000 last_win "2024-01-14"

// Get player info
HGET player:1 username  // Returns: "Alice"
HMGET player:1 level experience

// Update experience
HINCRBY player:1 experience 1000

// Increment level
HINCRBY player:1 level 1
```

## Advanced Patterns

### Conditional Field Update
```redis
// Set only if doesn't exist
HSETNX user:1 created_at "2024-01-15T10:00:00Z"  // Returns 1 (success) first time
HSETNX user:1 created_at "2024-01-15T11:00:00Z"  // Returns 0 (failed) on retry
```

### Batch Operations
```redis
// Multiple updates
HSET product:123 price 99.99 stock 100 available 1 on_sale 0

// Fetch specific fields
HMGET product:123 price stock available
```

### Atomic Increment Across Fields
```redis
// Simulate transaction-like behavior
HINCRBY account:user1 balance -100
HINCRBY account:user2 balance 100
```

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Set field | O(1) | Single field |
| Get field | O(1) | Single field |
| Get all fields | O(N) | N = number of fields |
| Increment | O(1) | Per field |
| Delete field | O(N) | N = number of deleted fields |
| Field check | O(1) | Fast membership test |

## Memory Comparison: Strings vs Hashes

```redis
// Method 1: Multiple string keys
SET user:1:name "John"
SET user:1:email "john@example.com"
SET user:1:age "30"
// Memory: 3 keys + 3 expiration entries + overhead

// Method 2: Single hash (more efficient)
HSET user:1 name "John" email "john@example.com" age "30"
// Memory: 1 key + 3 fields (less overhead)
```

Memory savings: ~30-50% for typical use cases with 5+ fields.

## Limitations

- **No nested hashes**: Cannot store hash within hash (use JSON strings)
- **No sorting by field values**: Use Sorted Sets for ranked data
- **String values only**: All field values must be strings (serialize for objects)
- **No transactions**: Operations not ACID, use Lua for multi-field transactions

## Best Practices

1. **Use HSET for bulk operations**: HSET key f1 v1 f2 v2 is more efficient
2. **Prefer HMGET over HGETALL**: Only fetch needed fields
3. **Use HINCRBY for counters**: Better than GET→increment→SET
4. **Limit hash size**: Keep fields < 1000 per hash
5. **Set TTL on hashes**: Use EXPIRE for temporary data
6. **Index frequently accessed fields**: Consider separate keys for hot data

## Common Pitfalls

1. **HGETALL for large hashes**: O(N) operation loads entire hash
2. **Type confusion**: Redis doesn't distinguish field types, store as strings
3. **Memory for sparse data**: Each field has overhead, don't store many nulls
4. **No atomic multi-key operations**: Cannot atomically update across hashes
5. **Missing TTL**: Hashes without expiration persist indefinitely
