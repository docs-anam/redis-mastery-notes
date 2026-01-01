# Search & Analytics - Data Processing

## Overview

Redis enables fast analytics, aggregation, and search-like functionality for real-time insights.

## Redis Search (RediSearch Module)

### Full-Text Search

```python
from redis import Redis
from redis.commands.search.field import TextField, NumericField
from redis.commands.search.query import Query

r = Redis()

# Create index
schema = (
    TextField("title", weight=5.0),
    TextField("description"),
    NumericField("price"),
)

r.ft('products').create_index(schema)

# Index documents
r.hset('product:1', mapping={'title': 'Laptop', 'description': 'Fast computer', 'price': 999})
r.hset('product:2', mapping={'title': 'Mouse', 'description': 'Wireless mouse', 'price': 25})

# Search
q = Query('@title:Laptop @price:[100 500]')
results = r.ft('products').search(q)
```

---

## Analytics with HyperLogLog

### Unique Visitor Counting

```python
class Analytics:
    def __init__(self, r):
        self.r = r
    
    def track_visitor(self, date, visitor_id):
        """Track unique visitor"""
        # Store in HyperLogLog (constant memory!)
        self.r.pfadd(f'visitors:{date}', visitor_id)
    
    def get_unique_count(self, date):
        """Get unique visitor count"""
        return self.r.pfcount(f'visitors:{date}')
    
    def merge_days(self, dates):
        """Merge stats from multiple days"""
        keys = [f'visitors:{date}' for date in dates]
        return self.r.pfcount(*keys)

# Usage
analytics = Analytics(r)
for i in range(1000):
    analytics.track_visitor('2024-01-01', f'user_{i}')

count = analytics.get_unique_count('2024-01-01')  # ~1000 (accurate)
# Memory used: ~1.5KB regardless of actual count!
```

---

## Aggregation with Sorted Sets

### Top Products by Sales

```python
class ProductAnalytics:
    def __init__(self, r):
        self.r = r
    
    def record_sale(self, product_id, amount):
        """Record product sale"""
        self.r.zincrby('sales:total', amount, product_id)
        self.r.zincrby(f'sales:daily:{date.today()}', amount, product_id)
    
    def get_top_products(self, limit=10):
        """Get top selling products"""
        return self.r.zrevrange('sales:total', 0, limit-1, withscores=True)
    
    def get_daily_top(self, limit=10):
        """Get today's top products"""
        return self.r.zrevrange(
            f'sales:daily:{date.today()}', 0, limit-1, withscores=True
        )

# Usage
analytics = ProductAnalytics(r)
analytics.record_sale('product_1', 100)
analytics.record_sale('product_2', 50)
print(analytics.get_top_products(5))
```

---

## Time-Series Data

### Event Aggregation

```python
from time import time

class TimeSeries:
    def __init__(self, r, interval=60):
        self.r = r
        self.interval = interval  # 60 second buckets
    
    def add_event(self, key, value):
        """Add event to time series"""
        timestamp = int(time() / self.interval)
        self.r.zincrby(key, value, timestamp)
    
    def get_stats(self, key, minutes=60):
        """Get stats for last N minutes"""
        now = int(time() / self.interval)
        past = now - (minutes * 60 // self.interval)
        
        values = self.r.zrange(key, past, now, withscores=True)
        scores = [score for _, score in values]
        
        return {
            'total': sum(scores),
            'avg': sum(scores) / len(scores) if scores else 0,
            'count': len(scores)
        }

# Usage
timeseries = TimeSeries(r)
timeseries.add_event('requests:count', 1)  # Each request
stats = timeseries.get_stats('requests:count', minutes=60)
```

---

## Stream Analytics

### Event Log Analysis

```python
class EventLog:
    def __init__(self, r, stream_name='events'):
        self.r = r
        self.stream = stream_name
    
    def log_event(self, event_type, data):
        """Log event to stream"""
        self.r.xadd(self.stream, {
            'type': event_type,
            'timestamp': time.time(),
            **data
        })
    
    def get_events(self, event_type, hours=1):
        """Get events of type from last N hours"""
        past = time.time() - (hours * 3600)
        
        events = []
        for entry_id, data in self.r.xrange(self.stream):
            if data[b'type'].decode() == event_type:
                events.append(data)
        
        return events
    
    def count_by_type(self):
        """Count events by type"""
        counts = {}
        for entry_id, data in self.r.xrange(self.stream):
            event_type = data[b'type'].decode()
            counts[event_type] = counts.get(event_type, 0) + 1
        
        return counts

# Usage
log = EventLog(r)
log.log_event('user_signup', {'user_id': 123})
log.log_event('login', {'user_id': 123})
print(log.count_by_type())  # {'user_signup': 1, 'login': 1}
```

---

## Bit Operations for Analytics

### Feature Flag Toggle

```python
class FeatureFlags:
    def __init__(self, r):
        self.r = r
    
    def set_feature(self, user_id, feature, enabled):
        """Enable/disable feature for user"""
        bit = 1 if enabled else 0
        self.r.setbit(f'features:{feature}', user_id, bit)
    
    def has_feature(self, user_id, feature):
        """Check if user has feature"""
        return bool(self.r.getbit(f'features:{feature}', user_id))
    
    def count_enabled(self, feature):
        """Count how many users have feature"""
        return self.r.bitcount(f'features:{feature}')

# Usage
flags = FeatureFlags(r)
flags.set_feature(user_id=123, feature='beta_ui', enabled=True)
flags.set_feature(user_id=456, feature='beta_ui', enabled=False)

print(flags.has_feature(123, 'beta_ui'))  # True
print(flags.count_enabled('beta_ui'))  # 1 user
```

---

## Best Practices

### DO: Use HyperLogLog for Cardinality

```python
# ✅ GOOD: Exact count of unique users (12KB)
for user_id in users:
    r.pfadd('unique_users', user_id)

count = r.pfcount('unique_users')  # Exact (with small error margin)
```

### DO: Archive Old Data

```python
# ✅ GOOD: Move old stats to archive
old_stats = r.zrange('stats', 0, -1)
db.insert_batch('stats_archive', old_stats)
r.delete('stats')
```

### DON'T: Store Individual Events in Redis

```python
# ❌ BAD: Too much memory
r.set(f'event:{event_id}', json.dumps(event))

# ✅ GOOD: Aggregate and archive
r.zincrby('event:count', 1, event_type)
db.insert(event)  # Persist to database
```

---

## Common Mistakes

### Mistake 1: No Data Retention Policy
**Problem**: Redis fills with old data
**Solution**: Implement TTL and archival

### Mistake 2: Exact Cardinality with Large Datasets
**Problem**: Memory bloat
**Solution**: Use HyperLogLog (0.81% error, fixed memory)

### Mistake 3: Not Aggregating Data
**Problem**: Individual records kill memory
**Solution**: Use sorted sets, streams, counters

### Mistake 4: No Time Granularity
**Problem**: Can't drill down by time
**Solution**: Create per-minute/hour/day buckets

---

## Next Steps

1. **Learn [Caching Strategies](6-caching.md)** for optimization
2. **Review [Performance](../4-performance/1-intro.md)** for scaling
3. **Explore [Integration Patterns](1-intro.md)** for end-to-end

## Resources

- [RediSearch Documentation](https://redis.io/docs/modules/search/)
- [HyperLogLog Guide](https://redis.io/commands/pfadd)
- [Redis Streams](https://redis.io/commands/xadd)
- [Time-Series Patterns](https://redis.io/docs/data-types/)
