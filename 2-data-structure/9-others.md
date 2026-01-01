# Redis Others - Bitmaps & Bitfields

## Overview

Bitmaps use individual bits to store data compactly. Perfect for:
- **Presence Flags**: Online/offline status, membership
- **Compact Storage**: 1 bit per flag (extremely efficient)
- **Analytics**: Cohort analysis, user segments
- **Permissions**: User role flags, feature flags
- **Bloom Filters**: Approximate membership testing

### Why Bitmaps?

- **Extreme Compression**: 1 billion flags = 125 MB
- **O(1) Operations**: Single bit access in constant time
- **Atomic Bit Operations**: SETBIT, GETBIT, BITCOUNT
- **Bulk Operations**: AND, OR, XOR on bit ranges
- **Bit Population**: Count set bits (BITCOUNT)

---

## Core Commands

### Basic Bit Operations

```bash
# Set bit (0 or 1)
SETBIT online:users:2024-01-01 0 1    # User 0 is online

# Get bit value
GETBIT online:users:2024-01-01 0      # Returns: 1

# Set and return old value
GETSET online:users:2024-01-01 0 0    # Get and set to 0
```

### Counting & Analysis

```bash
# Count set bits
BITCOUNT online:users:2024-01-01      # Total online users

# Count in range
BITCOUNT online:users:2024-01-01 0 1  # Bytes 0-1 only

# Position of first set/unset bit
BITPOS online:users:2024-01-01 1      # First online user (bit position)
```

### Bulk Operations

```bash
# AND operation
BITOP AND result key1 key2             # Intersection

# OR operation
BITOP OR result key1 key2              # Union

# XOR operation
BITOP XOR result key1 key2             # Symmetric difference

# NOT operation
BITOP NOT result key                   # Invert bits
```

### Advanced: Bitfields

```bash
# Get bitfield
BITFIELD mykey GET u4 0                # Get 4-bit unsigned at offset 0

# Set bitfield
BITFIELD mykey SET u8 0 255            # Set 8-bit unsigned to 255

# Increment bitfield
BITFIELD mykey INCRBY i8 0 1           # Increment 8-bit signed by 1
```

---

## Practical Examples

### Example 1: Online Status

```python
import redis
r = redis.Redis()

# Mark user online
def mark_online(user_id):
    today = '2024-01-01'
    key = f'online:{today}'
    r.setbit(key, user_id, 1)
    r.expire(key, 7*86400)

# Mark user offline
def mark_offline(user_id):
    today = '2024-01-01'
    key = f'online:{today}'
    r.setbit(key, user_id, 0)

# Check if online
def is_online(user_id):
    today = '2024-01-01'
    key = f'online:{today}'
    return r.getbit(key, user_id) == 1

# Count online
def count_online():
    today = '2024-01-01'
    key = f'online:{today}'
    return r.bitcount(key)

# Usage
mark_online(123)
mark_online(456)
print(f"Online: {count_online()}")  # 2
```

### Example 2: User Segments/Features

```python
# Feature flags
def enable_feature(feature_name, user_id):
    key = f'feature:{feature_name}:users'
    r.setbit(key, user_id, 1)

def has_feature(feature_name, user_id):
    key = f'feature:{feature_name}:users'
    return r.getbit(key, user_id) == 1

# Cohort analysis - users with multiple features
def get_beta_users():
    """Find users with both beta features enabled"""
    r.bitop('AND', 'beta_users', 'feature:beta1:users', 'feature:beta2:users')
    return r.bitcount('beta_users')

# Usage
enable_feature('beta_feature_1', 100)
enable_feature('beta_feature_1', 101)
enable_feature('beta_feature_2', 100)

print(f"Users with both features: {get_beta_users()}")
```

### Example 3: Attendance Tracking

```python
# Track daily attendance
def mark_attendance(user_id, present=True):
    today = '2024-01-01'
    key = f'attendance:{today}'
    r.setbit(key, user_id, 1 if present else 0)

# Get attendance percentage
def attendance_rate():
    today = '2024-01-01'
    key = f'attendance:{today}'
    present = r.bitcount(key)
    total = 1000  # Assume 1000 users
    return (present / total) * 100

# Check for perfect attendance
def perfect_attendance(user_id, days=30):
    """Check if user attended all 30 days"""
    keys = [f'attendance:2024-01-{str(day).zfill(2)}' for day in range(1, days+1)]
    # Find user in all attendance records
    # Complex logic but possible with BITOP
```

### Example 4: Permissions Matrix

```python
# Each user has bits for permissions
# Bit 0: can_read, 1: can_write, 2: can_delete, 3: can_admin

def set_permission(user_id, permission_bit, granted=True):
    key = f'permissions:{user_id}'
    r.setbit(key, permission_bit, 1 if granted else 0)

def has_permission(user_id, permission_bit):
    key = f'permissions:{user_id}'
    return r.getbit(key, permission_bit) == 1

def can_read(user_id):
    return has_permission(user_id, 0)

def can_write(user_id):
    return has_permission(user_id, 1)

def can_delete(user_id):
    return has_permission(user_id, 2)

# Usage
set_permission(100, 0, True)   # can_read
set_permission(100, 1, True)   # can_write
set_permission(100, 2, False)  # can't delete

print(f"User 100 can write: {can_write(100)}")  # True
```

---

## Real-World Pattern: Cohort Analysis

```python
def track_page_view(page_name, user_id):
    """Track which pages users visited"""
    key = f'page:{page_name}:users'
    r.setbit(key, user_id, 1)

def find_cohort(page1, page2):
    """Find users who visited both pages"""
    r.bitop('AND', 'cohort', f'page:{page1}:users', f'page:{page2}:users')
    count = r.bitcount('cohort')
    return count

def find_dropout(landed, converted):
    """Find users who landed but didn't convert"""
    r.bitop('AND', 'temp', f'page:{landed}:users', f'page:{converted}:users')
    # NOT temp (users in landed but not in temp)
    r.bitop('AND', 'dropout', f'page:{landed}:users', 'temp')
    # This is complex - better in application code

# Usage
track_page_view('landing', 100)
track_page_view('landing', 101)
track_page_view('signup', 100)

converted = find_cohort('landing', 'signup')
print(f"Landing → Signup conversion: {converted} users")
```

---

## Performance

```
Operation              Time       Memory (1M bits)
────────────────────────────────────────────────
SETBIT                 O(1)       125 KB (1M items)
GETBIT                 O(1)       Constant
BITCOUNT               O(N)       For N bytes
BITOP (N keys)         O(N)       Returns N bytes
```

**Key Insight**: 1 billion users as flags = 125 MB memory!

---

## Memory Comparison

```
Storage Method          Memory (1B users)    Operation
────────────────────────────────────────────────────
String set              ~1 GB               O(N) check
Hash fields             ~2 GB               O(N) check
Sorted set              ~4 GB               O(1) check
Set                     ~4 GB               O(1) check
Bitmap                  ~125 MB             O(1) check
HyperLogLog             ~12 KB              ~O(1) estimate
```

---

## Best Practices

### 1. Use Bitmaps for Compact Flags

```python
# ✅ DO: Use bitmap for flags
r.setbit('features:user:100', 0, 1)    # 1 bit
r.setbit('features:user:100', 1, 1)

# ❌ DON'T: Use strings
r.set('feature:0:user:100', '1')       # Much more memory
r.set('feature:1:user:100', '1')
```

### 2. Use BITOP for Cohort Analysis

```python
# ✅ DO: Atomic bitwise operations
r.bitop('AND', 'cohort', 'feature:1:users', 'feature:2:users')
count = r.bitcount('cohort')

# ❌ DON'T: Fetch and intersect in app
set1 = r.smembers('feature:1:users')
set2 = r.smembers('feature:2:users')
cohort = set1 & set2
```

### 3. Manage Bit Positions Carefully

```python
# ✅ DO: Use constants for bit meanings
READ = 0
WRITE = 1
DELETE = 2
ADMIN = 3

r.setbit(f'perms:{user_id}', READ, 1)

# ❌ DON'T: Magic numbers scattered
r.setbit(f'perms:{user_id}', 0, 1)     # What does bit 0 mean?
```

### 4. Consider Alternative Structures

```python
# Choose based on use case:
# - 1-100 flags: Use Hash fields
# - 1000s of flags: Use Bitmap
# - Need membership: Use Set
# - Need counting: Use HyperLogLog
```

---

## Common Mistakes

### Mistake 1: Wrong Byte Indexing in BITCOUNT

```python
# ❌ WRONG: Confusing bits and bytes
r.bitcount(key, 0, 10)  # Bytes 0-10 (80-88 bits), not bits!

# ✅ RIGHT: Remember it's byte range
# Bytes 0-1 = bits 0-15
r.bitcount(key, 0, 1)
```

### Mistake 2: Not Setting Expiration

```python
# ❌ WRONG: Bitmap grows forever
r.setbit('online:2020', user_id, 1)
# Years later: still taking space

# ✅ RIGHT: Set TTL
r.setbit('online:2024', user_id, 1)
r.expire('online:2024', 24*3600)  # 24 hours
```

### Mistake 3: Using for Strings

```python
# ❌ WRONG: Bitmap for non-flag data
r.setbit('username', 0, 1)  # Doesn't make sense

# ✅ RIGHT: Use strings for text
r.set('username', 'alice')
```

### Mistake 4: Assuming Bit Order

```python
# ❌ WRONG: Big-endian assumed
user_features = 0b11010100
# Bit positions might not be what you expect

# ✅ RIGHT: Be explicit
FEATURE_A = 0  # First bit
FEATURE_B = 1  # Second bit
r.setbit(key, FEATURE_A, 1)
```

---

## Bitfield Use Cases

For advanced scenarios, BITFIELD allows operations on multi-bit integers:

```python
# Store user levels (4 bits per user)
r.bitfield('user:levels', 'SET', 'u4', 0, 5)    # User 0 = level 5
r.bitfield('user:levels', 'GET', 'u4', 0)      # Read level 5
r.bitfield('user:levels', 'INCRBY', 'i4', 0, 1) # Increment by 1
```

---

## Next Steps

- **[Streams](6-stream.md)** - Use with bitmaps for event tracking
- **[Sorted Sets](5-sorted-set.md)** - Alternative for rankings
- **[HyperLogLog](8-hyperloglog.md)** - For cardinality estimation

