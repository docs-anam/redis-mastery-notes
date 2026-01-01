# Other Redis Data Structures

This page covers specialized Redis data structures that don't fit into the core categories but are powerful for specific use cases.

## Bitmaps (Bit Operations on Strings)

### Overview
Bitmaps enable storing and manipulating bit-level information within string keys. Despite being called a "bitmap," they're actually bit operations on regular strings, allowing for efficient bit manipulation with O(1) get/set operations.

### Key Characteristics
- **Bit-level operations**: Set, get, count, bitwise operations on bits
- **Memory efficient**: 8 bits per byte, extremely compact
- **Fast operations**: O(1) for get/set, O(N) for count
- **Bitwise logic**: AND, OR, XOR, NOT operations
- **String-based**: Stored as strings internally

### Common Commands

| Command | Complexity | Description |
|---------|-----------|-------------|
| `SETBIT key offset 0/1` | O(1) | Set bit at offset |
| `GETBIT key offset` | O(1) | Get bit at offset |
| `BITCOUNT key [range]` | O(N) | Count set bits |
| `BITPOS key 0/1 [range]` | O(N) | Find first bit value |
| `BITOP AND/OR/XOR/NOT dest key [key ...]` | O(N) | Bitwise operations |
| `BITFIELD key GET/SET/INCRBY` | O(1) | Complex bit operations |

### Use Cases

#### 1. User Activity Tracking
Track user activity with daily granularity using minimal memory.

```redis
// User activity per day (365 days = 46 bytes)
// Bit 0 = Jan 1, Bit 1 = Jan 2, etc.

// User was active on day 10
SETBIT user:123:activity 10 1

// Check if user was active on day 10
GETBIT user:123:activity 10  // Returns: 1

// Count active days (user engagement metric)
BITCOUNT user:123:activity  // Returns: total active days

// Find first active day
BITPOS user:123:activity 1  // Returns: offset of first 1
```

**Real-world scenario**: User engagement tracking, login streaks, activity heatmaps.

#### 2. Online/Offline Status
Track online status for millions of users with minimal memory.

```redis
// 1 million users = ~125KB memory
SETBIT online:users 123456 1  // User 123456 comes online
SETBIT online:users 789012 1  // User 789012 comes online

// Check if user online
GETBIT online:users 123456  // Returns: 1 (online)

// Count online users
BITCOUNT online:users  // Returns: number of online users

// User goes offline
SETBIT online:users 123456 0
```

**Real-world scenario**: Online status indicators, presence detection, concurrent users.

#### 3. Permissions/Flags
Store multiple boolean flags per user.

```redis
// User permissions (bit positions)
// Bit 0 = can_read, Bit 1 = can_write, Bit 2 = can_delete, etc.

SETBIT user:123:permissions 0 1  // Grant read
SETBIT user:123:permissions 1 1  // Grant write
SETBIT user:123:permissions 2 0  // No delete

// Check permission
GETBIT user:123:permissions 1  // Returns: 1 (can write)

// All permissions as bitmask
BITCOUNT user:123:permissions  // Returns: 2 (2 permissions granted)
```

**Real-world scenario**: Role-based access control, feature flags, permission systems.

#### 4. Bitwise Operations & Analytics
Find common patterns between user sets.

```redis
// Daily active users
SETBIT dau:2024-01-15 123 1  // Users who were active
SETBIT dau:2024-01-15 456 1
SETBIT dau:2024-01-15 789 1

// Previous day
SETBIT dau:2024-01-14 123 1
SETBIT dau:2024-01-14 456 1
SETBIT dau:2024-01-14 321 1

// Find users active BOTH days (AND operation)
BITOP AND common dau:2024-01-15 dau:2024-01-14
BITCOUNT common  // Returns: 2 (users 123, 456)

// Find users active on EITHER day (OR operation)
BITOP OR total dau:2024-01-15 dau:2024-01-14
BITCOUNT total  // Returns: 4

// Find users active on 15th but NOT 14th (AND NOT)
BITOP AND exclusive dau:2024-01-15 dau:2024-01-14
```

**Real-world scenario**: Cohort analysis, user segmentation, retention analysis.

## Bitfields

### Overview
Bitfields (BITFIELD command) enable storing and manipulating multiple integers of arbitrary width within a single string. More powerful than basic bitmaps, allowing structured data in compressed form.

### Key Characteristics
- **Variable width**: Support integers of 1-64 bits
- **Atomic operations**: Get, set, and increment operations
- **Overflow handling**: WRAP, SAT (saturate), FAIL modes
- **Efficient storage**: Pack multiple integers into single string
- **Complex operations**: Batch operations in single command

### Common Commands

| Command | Complexity | Description |
|---------|-----------|-------------|
| `BITFIELD key GET u8 0` | O(1) | Get unsigned 8-bit at offset 0 |
| `BITFIELD key SET u8 0 100` | O(1) | Set unsigned 8-bit |
| `BITFIELD key INCRBY u8 0 1` | O(1) | Increment value |

### Use Cases

#### 1. Sensor Data Compression
Store multiple sensor readings compactly.

```redis
// Store temperature (signed 8-bit) and humidity (unsigned 8-bit)
// One measurement = 16 bits = 2 bytes
// 1 year of hourly readings = 8760 hours * 2 bytes = 17.5KB

// Add temperature reading
BITFIELD sensor:1:data SET i8 0 23  // temp: 23°C

// Add humidity reading  
BITFIELD sensor:1:data SET u8 8 65  // humidity: 65%

// Read back
BITFIELD sensor:1:data GET i8 0   // Returns: 23
BITFIELD sensor:1:data GET u8 8   // Returns: 65

// Store multiple readings compactly
// Hour 1: temp=20, humidity=60
BITFIELD sensor:log SET i8 0 20 SET u8 8 60

// Hour 2: temp=22, humidity=62 (offset 16 bits = 2 bytes)
BITFIELD sensor:log SET i8 16 22 SET u8 24 62

// Increment temperature
BITFIELD sensor:log INCRBY i8 0 2  // Add 2 degrees
```

**Real-world scenario**: IoT data storage, time-series compression, sensor logging.

#### 2. Game Leaderboard Scores
Store player stats compactly.

```redis
// Player stats using variable-width integers
// Health (u8: 0-255), Mana (u8: 0-255), Level (u16: 0-65535)

// Player 1 stats
BITFIELD players SET u8 0 250 SET u8 8 180 SET u16 16 45
// Health: 250, Mana: 180, Level: 45

// Read health
BITFIELD players GET u8 0  // Returns: 250

// Increase level
BITFIELD players INCRBY u16 16 1  // Level += 1

// Decrease health (clamped to 0)
BITFIELD players INCRBY u8 0 -10 OVERFLOW SAT
```

**Real-world scenario**: Game state storage, player stats, character data.

#### 3. Efficient Counters
Store multiple counters in single key.

```redis
// Multiple page view counters (16-bit each)
// Page 1: bits 0-15, Page 2: bits 16-31, etc.

// Increment page 1 views
BITFIELD page:views INCRBY u16 0 1

// Increment page 2 views
BITFIELD page:views INCRBY u16 16 1

// Get all views
BITFIELD page:views GET u16 0 GET u16 16  // Returns both counters
```

**Real-world scenario**: Multi-dimensional counters, analytics, metrics storage.

## Comparison of Data Structures

### Memory Efficiency
```
Storing 1 million booleans:
- Set (1 element): ~50MB
- Bitmap: ~125KB
- Efficiency: 400x smaller

Storing user activity (365 days):
- Set of dates: ~365 * 8 bytes = 2.9KB
- Bitmap: ~46 bytes
- Efficiency: 63x smaller
```

### When to Use Each

| Structure | Best For | Memory | Speed |
|-----------|----------|--------|-------|
| Bitmap | Flags, boolean data, activity | Minimal | Very fast |
| Bitfield | Structured numeric data | Very minimal | Very fast |
| Set | Membership, uniqueness | Moderate | Fast |
| Sorted Set | Rankings, ranges | Moderate | Fast |
| Hash | Objects, attributes | Moderate | Fast |
| List | Queues, sequences | Moderate | Fast |
| String | Simple values | Varies | Very fast |

## Best Practices

1. **Use Bitmap for flags**: Most memory-efficient for booleans
2. **Use Bitfield for structured data**: When you have multiple numeric fields
3. **Combine with expiration**: Use EXPIRE for time-limited data
4. **Batch operations**: Use BITFIELD with multiple operations
5. **Document bit positions**: Clear comments on what each bit means
6. **Consider endianness**: Be aware of byte order in bitwise operations
7. **Monitor memory**: Even compressed data can accumulate

## Limitations & Considerations

### Bitmaps
- Limited to 512MB strings (~4 billion bits)
- Operations return integers, no native "bit array" return
- No range operations like Sets
- Requires manual bit offset management

### Bitfields
- Complex syntax can be error-prone
- Difficult to debug bit-level operations
- No native iteration over fields
- Overflow modes need careful consideration

## Common Pitfalls

1. **Bit offset confusion**: Off-by-one errors in bit positions
2. **Memory limits**: Strings limited to 512MB
3. **Overflow handling**: Unexpected WRAP vs SAT vs FAIL behavior
4. **Type width assumptions**: Forgetting actual bit width constraints
5. **Endianness issues**: Different systems may interpret bits differently
6. **Readability sacrifice**: Optimizing memory at cost of code clarity
