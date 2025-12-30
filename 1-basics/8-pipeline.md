# Redis Pipeline

## Overview
Redis Pipeline is a technique that allows you to send multiple commands to the Redis server without waiting for replies after each command. This significantly improves performance by reducing network round-trips.

## How It Works
1. Client prepares multiple commands
2. Client sends all commands to server in a batch (without waiting for responses)
3. Server processes all commands sequentially (maintains order)
4. Server sends all replies back in a single response
5. Client receives and processes all replies at once

### Pipeline vs Regular Requests
**Without Pipeline (3 commands, 3 round-trips):**
```
Client → Server: SET key1 value1     → wait for OK
Client ← Server: OK
Client → Server: SET key2 value2     → wait for OK
Client ← Server: OK
Client → Server: GET key1            → wait for value1
Client ← Server: value1
Total: 3 network round-trips (~30-50ms latency)
```

**With Pipeline (3 commands, 1 round-trip):**
```
Client → Server: [SET key1 value1, SET key2 value2, GET key1]
Client ← Server: [OK, OK, value1]
Total: 1 network round-trip (~10-15ms latency)
```

## Benefits
- **Reduced Latency**: Minimizes network round-trip time
- **Improved Throughput**: Process multiple commands faster
- **Better Performance**: Especially useful for bulk operations
- **Network Efficiency**: Fewer packets transmitted

## Use Cases
- Bulk data insertion
- Batch updates
- Data migration
- Bulk data retrieval
- High-frequency operations

## Implementation Example

### Python (redis-py)
```python
import redis

# Connect to Redis
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Non-transactional pipeline
pipe = r.pipeline(transaction=False)
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
pipe.set('key3', 'value3')
pipe.get('key1')
pipe.get('key2')
results = pipe.execute()
print(results)
# Output: [True, True, True, 'value1', 'value2']
```

### Node.js (redis)
```javascript
const redis = require('redis');
const client = redis.createClient();

const pipeline = client.multi();
pipeline.set('key1', 'value1');
pipeline.set('key2', 'value2');
pipeline.get('key1');
pipeline.get('key2');
pipeline.exec((err, replies) => {
    console.log(replies);
    // Output: ['OK', 'OK', 'value1', 'value2']
});
```

### Ruby (redis gem)
```ruby
require 'redis'

redis = Redis.new(host: 'localhost', port: 6379)

results = redis.pipelined do
  redis.set('key1', 'value1')
  redis.set('key2', 'value2')
  redis.get('key1')
  redis.get('key2')
end
puts results.inspect
# Output: ["OK", "OK", "value1", "value2"]
```

### Using redis-cli with Pipeline
```bash
# Create a file: commands.txt
SET key1 value1
SET key2 value2
SET key3 value3
GET key1
GET key2
GET key3

# Execute with pipeline
cat commands.txt | redis-cli --pipe
```