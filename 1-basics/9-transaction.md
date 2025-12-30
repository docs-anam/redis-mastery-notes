# Redis Transactions

## Overview
Redis transactions allow you to execute a series of commands atomically. All commands in a transaction are executed sequentially without interruption from other clients.

## Key Commands

### MULTI
Starts a transaction block. All subsequent commands are queued until `EXEC` is called. Returns `OK` when successful.

```
MULTI
# Response: OK
```

### EXEC
Executes all queued commands atomically in order and returns array of results. Returns `null` if transaction was aborted by WATCH.

```
EXEC
# Response: array of command results or null
```

### DISCARD
Cancels the transaction and discards all queued commands. Returns `OK`.

```
DISCARD
# Response: OK
```

### WATCH
Monitors one or more keys. If a watched key is modified before `EXEC`, the transaction is aborted. Useful for optimistic locking.

```
WATCH key1 key2 key3
# Response: OK
```

### UNWATCH
Cancels monitoring for all keys. Also called automatically by EXEC or DISCARD.

```
UNWATCH
# Response: OK
```

## Transaction Flow

1. **WATCH** (optional) - Monitor keys for changes
2. **MULTI** - Begin transaction (returns OK)
3. Queue commands - Each command returns QUEUED
4. **EXEC** - Execute all commands atomically, or **DISCARD** to cancel

## Basic Examples

### Example 1: Simple Transaction
```redis
MULTI
OK
SET key1 "value1"
QUEUED
SET key2 "value2"
QUEUED
INCR counter
QUEUED
GET key1
QUEUED
EXEC
1) OK
2) OK
3) (integer) 1
4) "value1"
```

### Example 2: DISCARD Transaction
```redis
MULTI
OK
SET mykey "oldvalue"
QUEUED
INCR mykey
QUEUED
DISCARD
OK
GET mykey
(nil)  # Original value unchanged
```

### Example 3: Bank Transfer (Atomic Operation)
```redis
# Transfer $10 from account1 to account2
WATCH account1 account2
OK
MULTI
OK
DECRBY account1 10
QUEUED
INCRBY account2 10
QUEUED
EXEC
1) (integer) 90
2) (integer) 110
```

### Example 4: WATCH - Detected Modification
```redis
# Terminal 1
WATCH mykey
OK
MULTI
OK
SET mykey "newvalue"
QUEUED

# Terminal 2 (modifies watched key)
SET mykey "changed"
OK

# Terminal 1 continues
EXEC
(nil)  # Transaction aborted because watched key was modified
```

## Transaction Error Handling

### Type 1: Queuing Errors (Before EXEC)
```redis
MULTI
OK
SET mykey value
QUEUED
INCR mykey          # Error: mykey is not an integer
ERR value is not an integer or out of range

EXEC
# Returns null or error
```

### Type 2: Execution Errors (During EXEC)
```redis
MULTI
OK
SET mykey "string"
QUEUED
INCR mykey          # Will fail during EXEC
QUEUED
EXEC
# Partial execution - first command succeeds, second fails
```

## Important Notes

- Commands are queued, not executed immediately after MULTI (return QUEUED)
- Errors during queueing cause the transaction to fail
- Redis does not support rollbacks (DISCARD cancels but doesn't roll back previous operations)
- WATCH provides optimistic locking mechanism
- Transactions are isolated per client connection
- EXEC automatically calls UNWATCH
- All commands in transaction execute in same order on the server

## Transaction vs Pipeline

| Feature | Transaction | Pipeline |
|---------|-------------|----------|
| Atomicity | ✅ Yes (MULTI/EXEC) | ❌ No (unless transaction=True) |
| Order guaranteed | ✅ Yes | ✅ Yes |
| Rollback support | ❌ No | ❌ No |
| WATCH support | ✅ Yes | ✅ Yes (with transaction=True) |
| Network efficiency | ✅ Yes | ✅ Yes |
| Speed | Fast | Faster (no atomicity) |
| Best for | Data consistency | Bulk operations |

## When to Use Transactions

### Use Transactions When:
- ✅ Data consistency is critical (bank transfers, inventory)
- ✅ Multiple related keys must be updated together
- ✅ You need atomicity guarantees
- ✅ Optimistic locking with WATCH is beneficial
- ✅ Preventing race conditions is important

### Use Pipelines (Non-transactional) When:
- ✅ Bulk operations without atomicity needs
- ✅ Performance is more critical than consistency
- ✅ Simple fire-and-forget operations
- ✅ No dependencies between commands

### Use Lua Scripting When:
- ✅ Complex logic needed with atomicity
- ✅ Decision logic depends on Redis data
- ✅ Need true server-side logic execution
- ✅ Reduce network round-trips for complex operations