# Redis Databases

## Overview
Redis uses a simple database model based on **logical databases** rather than separate server instances. By default, Redis includes 16 databases (0-15), though this is configurable.

## Key Concepts

### Database Selection
- Each client connects to a specific database using the `SELECT` command
- Default database is `0`
- Syntax: `SELECT <database-number>`

### Database Isolation
- Each database maintains its own **keyspace** (set of keys)
- Keys in one database are completely isolated from others
- Operations on one database don't affect others

### Use Cases
- **Development environments**: Separate databases for different projects
- **Testing**: Isolated test data without affecting production
- **Multi-tenant applications**: Store different customer data separately
- **Caching layers**: Dedicated databases for different cache types

## Important Limitations

- No built-in access control per database
- All databases share the same memory pool
- No transaction isolation between databases
- `FLUSHDB` clears only current database; `FLUSHALL` clears all

## Configuration

Database count is set in `redis.conf`:
```
databases 16
```

## Best Practices

- Use database `0` for production data
- Document which database serves which purpose
- Consider namespace prefixes for large deployments
- Monitor memory usage across all databases