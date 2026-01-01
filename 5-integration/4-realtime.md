# Real-time Systems - WebSockets and Live Updates

## Overview

Redis powers real-time features like live notifications, presence tracking, and instant updates.

## WebSocket Integration

### Socket.IO with Redis Adapter

```python
from flask import Flask
from flask_socketio import SocketIO, emit, join_room
from redis import Redis

app = Flask(__name__)
socketio = SocketIO(app, message_queue='redis://localhost:6379')

# Connected users
r = Redis()

@socketio.on('connect')
def handle_connect():
    user_id = request.sid
    username = request.args.get('username')
    r.hset('online_users', user_id, username)
    emit('user_joined', {'username': username}, broadcast=True)

@socketio.on('disconnect')
def handle_disconnect():
    user_id = request.sid
    username = r.hget('online_users', user_id)
    r.hdel('online_users', user_id)
    emit('user_left', {'username': username}, broadcast=True)

@socketio.on('message')
def handle_message(data):
    # Broadcast to room
    emit('new_message', data, room=data.get('room'), broadcast=True)
    
    # Store in Redis for history
    r.lpush(f"messages:{data.get('room')}", json.dumps(data))
    r.ltrim(f"messages:{data.get('room')}", 0, 99)  # Keep last 100
```

---

## Pub/Sub Real-time

### Live Notifications

```python
class NotificationService:
    def __init__(self, r):
        self.r = r
    
    def notify_user(self, user_id, notification):
        """Send real-time notification"""
        # Publish to user channel
        self.r.publish(f'user:{user_id}:notifications', 
                       json.dumps(notification))
    
    def subscribe_to_notifications(self, user_id, callback):
        """Subscribe to user notifications"""
        pubsub = self.r.pubsub()
        pubsub.subscribe(f'user:{user_id}:notifications')
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                callback(data)

# Usage
notifier = NotificationService(r)

# Send notification
notifier.notify_user(123, {
    'type': 'order_shipped',
    'order_id': 456,
    'tracking_number': 'ABC123'
})

# Listen for notifications (on client/worker)
notifier.subscribe_to_notifications(123, 
    lambda notif: print(f"Got: {notif}"))
```

---

## Presence Tracking

### Online Status

```python
class PresenceManager:
    def __init__(self, r, ttl=300):
        self.r = r
        self.ttl = ttl
    
    def set_online(self, user_id):
        """Mark user as online"""
        self.r.setex(f'presence:{user_id}', self.ttl, '1')
        self.r.zadd('presence:users', {user_id: time.time()})
    
    def is_online(self, user_id):
        """Check if user is online"""
        return self.r.exists(f'presence:{user_id}') > 0
    
    def get_online_users(self):
        """Get all online users"""
        return list(self.r.zrange('presence:users', 0, -1, 
                                   withscores=True))
    
    def get_user_count(self):
        """Get online user count"""
        return self.r.zcard('presence:users')

# Usage
presence = PresenceManager(r)
presence.set_online('user123')

online = presence.is_online('user123')  # True
count = presence.get_user_count()  # 1
```

---

## Activity Streams

### Feed Generation

```python
class ActivityStream:
    def __init__(self, r):
        self.r = r
    
    def post_activity(self, user_id, activity):
        """Post activity"""
        activity['timestamp'] = time.time()
        
        # Add to user's timeline
        self.r.lpush(f'timeline:{user_id}', json.dumps(activity))
        
        # Add to followers' feeds
        followers = self.r.smembers(f'followers:{user_id}')
        for follower_id in followers:
            self.r.lpush(f'feed:{follower_id}', json.dumps(activity))
            self.r.ltrim(f'feed:{follower_id}', 0, 999)  # Keep last 1000
    
    def get_feed(self, user_id, limit=20):
        """Get user's feed"""
        feed = self.r.lrange(f'feed:{user_id}', 0, limit-1)
        return [json.loads(item) for item in feed]
    
    def follow(self, user_id, target_id):
        """Follow user"""
        self.r.sadd(f'followers:{target_id}', user_id)
        self.r.sadd(f'following:{user_id}', target_id)

# Usage
stream = ActivityStream(r)
stream.follow('user1', 'user2')
stream.post_activity('user2', {'type': 'post', 'text': 'Hello'})
feed = stream.get_feed('user1')  # User1 sees User2's activity
```

---

## Live Leaderboard

### Real-time Ranking

```python
class Leaderboard:
    def __init__(self, r, board_name):
        self.r = r
        self.board_name = board_name
    
    def update_score(self, user_id, score):
        """Update user score"""
        self.r.zadd(self.board_name, {user_id: score})
        
        # Notify subscribers
        self.r.publish(f'{self.board_name}:updates', 
                       json.dumps({'user_id': user_id, 'score': score}))
    
    def get_rank(self, user_id):
        """Get user's rank"""
        rank = self.r.zrevrank(self.board_name, user_id)
        return rank + 1 if rank is not None else None
    
    def get_top(self, limit=10):
        """Get top scorers"""
        top = self.r.zrevrange(self.board_name, 0, limit-1, 
                               withscores=True)
        return [(user, score) for user, score in top]

# Usage
leaderboard = Leaderboard(r, 'game:scores')
leaderboard.update_score('player1', 1000)
leaderboard.update_score('player2', 950)
print(leaderboard.get_rank('player1'))  # 1
print(leaderboard.get_top(5))
```

---

## Best Practices

### DO: Use Message Compression

```python
# ✅ GOOD: Compress large messages
import zlib
message = json.dumps(large_data).encode()
compressed = zlib.compress(message)
r.publish(channel, compressed)
```

### DO: Handle Disconnections

```python
# ✅ GOOD: Reconnection with message buffering
pubsub = r.pubsub()
pubsub.subscribe(channel)

while True:
    try:
        for message in pubsub.listen():
            process(message)
    except ConnectionError:
        time.sleep(5)  # Backoff before reconnect
        pubsub = r.pubsub()
        pubsub.subscribe(channel)
```

### DON'T: Store Persistent Data in Pub/Sub

```python
# ❌ BAD: Relying on Pub/Sub for persistence
r.publish('events', data)  # No persistence!

# ✅ GOOD: Use Streams for history
r.xadd('events', {'data': data})  # Persisted, ordered
```

---

## Common Mistakes

### Mistake 1: No Backpressure
**Problem**: Subscribers can't keep up
**Solution**: Implement buffering or rate limiting

### Mistake 2: Message Loss
**Problem**: Subscribers miss messages
**Solution**: Use Streams instead of Pub/Sub

### Mistake 3: Global Broadcast
**Problem**: All users receive all messages
**Solution**: Use channel patterns, room-based routing

### Mistake 4: No Persistence
**Problem**: Activity data lost
**Solution**: Back up events to database periodically

---

## Next Steps

1. **Learn [Search & Analytics](5-search.md)** for data processing
2. **Explore [Caching Strategies](6-caching.md)** for optimization
3. **Review [Advanced Concepts](../3-advanced-concepts/1-intro.md)** for patterns

## Resources

- [Socket.IO Redis Adapter](https://python-socketio.readthedocs.io/)
- [Redis Pub/Sub](https://redis.io/commands/publish)
- [Redis Streams](https://redis.io/commands/xadd)
