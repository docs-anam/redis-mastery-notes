# Redis Lists

## Overview
Redis Lists are ordered collections of strings, allowing you to store and manipulate sequences of data efficiently. They maintain insertion order and support operations from both ends with O(1) time complexity.

## Key Characteristics
- **Ordered**: Elements maintain insertion order (FIFO or LIFO patterns)
- **Duplicates Allowed**: Same value can appear multiple times
- **Index-based**: Access elements by position (0-indexed)
- **Range Operations**: Retrieve multiple elements in one command
- **Bidirectional**: Efficient operations from both head and tail
- **Blocking Operations**: Support waiting for elements with timeouts

## Common Commands

### Adding Elements
| Command | Complexity | Description |
|---------|-----------|-------------|
| `LPUSH key value [...]` | O(1) | Add to left (head) |
| `RPUSH key value [...]` | O(1) | Add to right (tail) |
| `LINSERT key BEFORE\|AFTER pivot value` | O(N) | Insert relative to pivot |
| `LPUSHX key value` | O(1) | Add only if key exists |
| `RPUSHX key value` | O(1) | Add to tail only if exists |

### Removing Elements
| Command | Complexity | Description |
|---------|-----------|-------------|
| `LPOP key [count]` | O(N) | Remove and return left elements |
| `RPOP key [count]` | O(N) | Remove and return right elements |
| `LREM key count value` | O(N) | Remove occurrences of value |
| `LTRIM key start stop` | O(N) | Keep only elements in range |
| `BLPOP key timeout` | O(1) | Blocking left pop |
| `BRPOP key timeout` | O(1) | Blocking right pop |

### Retrieving Elements
| Command | Complexity | Description |
|---------|-----------|-------------|
| `LLEN key` | O(1) | Get list length |
| `LINDEX key index` | O(N) | Get element at index |
| `LRANGE key start stop` | O(N) | Get range of elements |
| `LPOS key element` | O(N) | Find position of element |
| `LMOVE source dest LEFT\|RIGHT LEFT\|RIGHT` | O(1) | Move element between lists |

### Modifying Elements
| Command | Complexity | Description |
|---------|-----------|-------------|
| `LSET key index value` | O(N) | Set element at index |

## Use Cases

### 1. Message Queues (FIFO Pattern)
Process tasks in the order they arrive. Perfect for job processing systems.

```redis
// Producer
RPUSH task:queue "task:1" "task:2" "task:3"

// Consumer
LPOP task:queue  // Returns "task:1"
```

**Real-world scenario**: Process payment transactions in sequence.

### 2. Activity Feeds & Notifications
Maintain reverse chronological order of user activities.

```redis
// Add activity
LPUSH user:123:feed "user_liked_post_456:2024-01-15T10:30:00Z"
LPUSH user:123:feed "user_commented_789:2024-01-15T09:45:00Z"

// Get latest 10 activities
LRANGE user:123:feed 0 9

// Remove old entries
LTRIM user:123:feed 0 99  // Keep only last 100
```

**Real-world scenario**: Social media feed, news aggregation.

### 3. Undo/Redo Stack
Implement action history for undo operations.

```redis
// Store actions
LPUSH user:123:undo "action:bold_text"
LPUSH user:123:undo "action:italic_text"
LPUSH user:123:undo "action:font_change"

// Undo last action
LPOP user:123:undo  // Returns "action:font_change"

// Redo stack
RPUSH user:123:redo "action:font_change"
```

**Real-world scenario**: Text editors, graphic design software.

### 4. Rate Limiting with Sliding Window
Track requests in a time window to enforce rate limits.

```redis
// Log request timestamp
LPUSH user:123:requests:sliding 1705318200
LPUSH user:123:requests:sliding 1705318205

// Remove requests older than 60 seconds
LREM user:123:requests:sliding 0 1705317000

// Count requests
LLEN user:123:requests:sliding
```

### 5. Redis-based Pub/Sub Alternative
Use blocking operations for simple message distribution.

```redis
// Worker waits for jobs
BRPOP job:queue 0  // Blocks indefinitely

// Producer adds jobs
LPUSH job:queue "{\"type\":\"email\",\"to\":\"user@example.com\"}"
```

### 6. Leaderboard Tracking
Maintain recent top performers.

```redis
// Add scores
RPUSH leaderboard:today "player1:1500"
RPUSH leaderboard:today "player2:1200"
RPUSH leaderboard:today "player3:1800"

// Keep only top 100
LTRIM leaderboard:today 0 99
```

## Performance Characteristics

| Operation | Complexity | Note |
|-----------|-----------|------|
| Head/Tail operations | O(1) | LPUSH, RPUSH, LPOP, RPOP |
| Middle access | O(N) | Requires scanning |
| Range operations | O(S+N) | S = start offset, N = returned elements |
| List length | O(1) | Stored separately, very fast |
| Blocking operations | O(1) | Efficient internal implementation |

## Advanced Patterns

### Work Queue Pattern
```redis
// Producer: Add work
RPUSH queue:pending {job_data}

// Consumer: Get and process
BRPOPLPUSH queue:pending queue:processing 0
// Process...
LREM queue:processing 1 {job_data}
```

### Atomic List Movement
```redis
// Move from source to destination atomically
RPOPLPUSH source_list dest_list
BRPOPLPUSH source_list dest_list 0  // Blocking version
```

### List as Cache
```redis
// Store results
RPUSH cache:recent "result1"
RPUSH cache:recent "result2"

// Keep only N items (LRU-like)
LTRIM cache:recent 0 999  // Keep last 1000
```

## Limitations & Considerations

- **Random Access**: Middle element access is O(N), use Sorted Sets for indexed access
- **Memory Usage**: Large lists consume significant memory, consider compression
- **No Sorting**: Built-in order is insertion order only, use Sorted Sets for scoring
- **Timeout Precision**: Blocking operations use second precision
- **Single-thread**: Blocking operations block the entire connection

## Memory Optimization Tips

1. Use `LPOP key count` to batch pop operations
2. Implement `LTRIM` regularly to limit list size
3. Consider compression for large values
4. Use pipelining to batch commands

## Common Pitfalls

1. **Blocking all consumers**: If all workers block on same key, they'll all wake on one message
2. **Memory leaks**: Forgotten `LTRIM` operations can lead to unbounded list growth
3. **Order assumptions**: Remember lists preserve insertion order, not value order