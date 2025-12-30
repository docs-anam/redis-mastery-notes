# Redis FLUSH Commands

## Overview
Redis provides FLUSH commands to delete all keys from the database(s). These are destructive operations that should be used with caution.

## FLUSHDB
Deletes all keys from the currently selected database.

```bash
FLUSHDB [ASYNC | SYNC]
```

**Options:**
- `ASYNC`: Flushes the database asynchronously in the background
- `SYNC`: Flushes the database synchronously (blocking)

**Use Cases:**
- Clearing a specific database during development
- Resetting test environments

## FLUSHALL
Deletes all keys from all databases.

```bash
FLUSHALL [ASYNC | SYNC]
```

**Options:**
- `ASYNC`: Flushes all databases asynchronously
- `SYNC`: Flushes all databases synchronously

**Use Cases:**
- Complete server reset
- Clearing entire Redis instance

## Key Differences

| Command | Scope | Impact |
|---------|-------|--------|
| FLUSHDB | Current database only | Limited |
| FLUSHALL | All databases | Complete wipe |

## Important Considerations
- **Irreversible**: Deleted keys cannot be recovered
- **Performance**: SYNC mode blocks the server; use ASYNC for production
- **AOF**: Operations are logged if AOF persistence is enabled
- **RDB**: Not automatically saved; use BGSAVE if needed

## Best Practices
- Always backup before flush operations
- Use ASYNC in production to avoid blocking
- Implement access controls to prevent accidental flushes