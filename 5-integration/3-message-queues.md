# Message Queues - Task Processing

## Overview

Redis-based job queues enable asynchronous task processing at scale without external dependencies.

## Job Queue Pattern

### Basic Queue Implementation

```python
import json
from datetime import datetime

class JobQueue:
    def __init__(self, r, queue_name='jobs'):
        self.r = r
        self.queue_name = queue_name
    
    def enqueue(self, task_type: str, task_data: dict, priority: int = 0):
        """Add job to queue"""
        job = {
            'type': task_type,
            'data': task_data,
            'created_at': datetime.now().isoformat(),
            'status': 'pending'
        }
        
        if priority > 0:
            # High priority: use sorted set
            self.r.zadd(f'{self.queue_name}:priority', {json.dumps(job): -priority})
        else:
            # Normal priority: use list
            self.r.rpush(self.queue_name, json.dumps(job))
    
    def dequeue(self, timeout: int = 0):
        """Get next job from queue"""
        if timeout:
            # Blocking pop with timeout
            data = self.r.blpop(self.queue_name, timeout)
            if data:
                return json.loads(data[1])
        else:
            # Non-blocking
            data = self.r.lpop(self.queue_name)
            if data:
                return json.loads(data)
        return None
    
    def queue_size(self):
        """Get number of pending jobs"""
        return self.r.llen(self.queue_name)

# Usage
queue = JobQueue(r)
queue.enqueue('send_email', {'to': 'user@example.com', 'subject': 'Hello'})

# Worker
while True:
    job = queue.dequeue(timeout=5)
    if job:
        process_job(job)
```

---

## Celery with Redis

### Setup

```python
# celery_config.py
from celery import Celery

app = Celery('myapp')
app.conf.update(
    broker_url='redis://localhost:6379/0',
    result_backend='redis://localhost:6379/1',
    task_serializer='json',
    accept_content=['json'],
    result_expires=3600,
)

# tasks.py
@app.task
def send_email(to, subject, body):
    """Send email asynchronously"""
    # Implementation
    return f"Email sent to {to}"

@app.task
def process_image(image_id):
    """Process image with retry logic"""
    try:
        # Heavy processing
        image = Image.objects.get(id=image_id)
        image.process()
    except Exception as exc:
        # Retry up to 3 times
        raise send_email.retry(exc=exc, countdown=60, max_retries=3)
```

### Usage

```python
# Enqueue task
send_email.delay(to='user@example.com', subject='Hello')

# Get result
result = send_email.apply_async(
    (to, subject),
    countdown=60  # Run in 60 seconds
)

# Wait for result (blocking)
email_result = result.get(timeout=300)

# Check status
if result.ready():
    print(f"Result: {result.result}")
else:
    print("Task still running")
```

---

## Scheduled Tasks

### APScheduler with Redis

```python
from apscheduler.schedulers.background import BackgroundScheduler
from redis import Redis

def scheduled_job():
    """Background job"""
    print("Running scheduled job")

scheduler = BackgroundScheduler()
scheduler.add_job(
    scheduled_job,
    'cron',
    hour=2,  # Run at 2 AM
    minute=0
)
scheduler.start()

# Store last execution in Redis
r = Redis()
r.set('last_scheduled_run', datetime.now().isoformat())
```

---

## Dead Letter Queue

### Error Handling

```python
class ReliableQueue:
    def __init__(self, r, queue_name='jobs'):
        self.r = r
        self.queue = queue_name
        self.processing = f'{queue_name}:processing'
        self.failed = f'{queue_name}:failed'
    
    def enqueue(self, task):
        self.r.rpush(self.queue, json.dumps(task))
    
    def claim_job(self, worker_id):
        """Move job from queue to processing"""
        job = self.r.rpoplpush(self.queue, self.processing)
        if job:
            self.r.hset(self.processing, json.loads(job)['id'], worker_id)
            return json.loads(job)
        return None
    
    def complete_job(self, job_id):
        """Mark job as complete"""
        self.r.hdel(self.processing, job_id)
    
    def fail_job(self, job_id, error):
        """Move failed job to DLQ"""
        self.r.hset(self.failed, job_id, error)
        self.r.hdel(self.processing, job_id)
    
    def requeue_failed(self, max_retries=3):
        """Requeue failed jobs"""
        failed = self.r.hgetall(self.failed)
        for job_id, error in failed.items():
            retry_count = self.r.hincrby(f'retries:{job_id}', 'count', 1)
            if retry_count <= max_retries:
                # Requeue
                self.r.rpush(self.queue, job_id)
                self.r.hdel(self.failed, job_id)
```

---

## Best Practices

### DO: Use Message Format

```python
# ✅ GOOD: Structured message format
message = {
    'id': str(uuid4()),
    'type': 'email_task',
    'version': 1,
    'data': {'to': 'user@example.com'},
    'priority': 1,
    'max_retries': 3,
    'created_at': time.time()
}
queue.enqueue(message)
```

### DO: Handle Failures

```python
# ✅ GOOD: Retry with backoff
job = queue.dequeue()
try:
    process_job(job)
except TemporaryError as e:
    job['retries'] = job.get('retries', 0) + 1
    if job['retries'] <= 3:
        queue.enqueue(job)
except PermanentError as e:
    queue.failed.add(job, str(e))
```

### DON'T: Process Without Dequeuing

```python
# ❌ BAD: Job lost if crash before completion
job = queue.lpop(queue_name)
process_job(job)

# ✅ GOOD: Acknowledge after processing
job = queue.rpoplpush(queue_name, processing)
try:
    process_job(job)
    queue.lrem(processing, 0, job)  # Remove from processing
except:
    pass  # Stays in processing, can be retried
```

---

## Common Mistakes

### Mistake 1: No Error Handling
**Problem**: Failed jobs lost forever
**Solution**: Implement DLQ and retry logic

### Mistake 2: No Job Acknowledgment
**Problem**: Same job processed multiple times
**Solution**: Use processing queue, acknowledge on completion

### Mistake 3: No Timeout
**Problem**: Stuck workers block queue
**Solution**: Implement timeout, re-enqueue if stalled

### Mistake 4: Single Worker
**Problem**: Bottleneck, single point of failure
**Solution**: Scale workers horizontally with same queue

---

## Monitoring

```python
def queue_stats():
    pending = r.llen('jobs')
    processing = r.llen('jobs:processing')
    failed = r.hlen('jobs:failed')
    
    return {
        'pending': pending,
        'processing': processing,
        'failed': failed,
        'total': pending + processing + failed
    }

# Check health
stats = queue_stats()
if stats['failed'] > 100:
    alert('Too many failed jobs')
```

---

## Next Steps

1. **Learn [Real-time Systems](4-realtime.md)** for live updates
2. **Explore [Search & Analytics](5-search.md)** for data processing
3. **Review [Caching Strategies](6-caching.md)** for optimization

## Resources

- [Celery Documentation](https://docs.celeryproject.org/)
- [APScheduler Guide](https://apscheduler.readthedocs.io/)
- [Redis Queue (RQ)](https://python-rq.org/)
