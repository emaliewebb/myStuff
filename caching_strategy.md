# Caching Strategy Using Redis

This document outlines the comprehensive caching strategy for the video streaming platform using Redis as the primary caching solution.

## 1. Redis Deployment Architecture

### 1.1 Redis Cluster Configuration

The platform will use a Redis Cluster deployment with the following specifications:

- **Cluster Size**: 20 nodes (10 primary + 10 replica)
- **Instance Type**: cache.r5.2xlarge (4 vCPU, 53.5 GB memory)
- **Sharding Strategy**: Hash slot distribution (16384 slots)
- **Replication**: 1 replica per primary node
- **Multi-AZ**: Deployed across 3 availability zones
- **Auto-failover**: Enabled with 5-second detection timeout

### 1.2 Redis Topology

```
┌─────────────────────────────────────────────────────────────┐
│                     Redis Cluster                           │
├─────────────┬─────────────┬─────────────┬─────────────┬─────┤
│ Shard 1     │ Shard 2     │ Shard 3     │ Shard 4     │ ... │
│ Primary+Rep │ Primary+Rep │ Primary+Rep │ Primary+Rep │     │
└─────────────┴─────────────┴─────────────┴─────────────┴─────┘
        │             │             │             │
┌───────▼─────────────▼─────────────▼─────────────▼───────────┐
│                   Cache Client Layer                        │
│     (Connection Pooling, Consistent Hashing, Failover)      │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                   Application Services                      │
└─────────────────────────────────────────────────────────────┘
```

## 2. Cache Types and Implementation

### 2.1 Video Metadata Cache

**Purpose**: Reduce database load for video information retrieval

**Implementation**:
```
Key: video:{video_id}:metadata
Value: JSON serialized video metadata
TTL: 1 hour
```

**Sample Redis Command**:
```redis
SET video:abc123:metadata "{\"title\":\"Example Video\",\"description\":\"...\",\"views\":10000,\"likes\":500}" EX 3600
```

**Invalidation Strategy**:
- On video metadata update
- Atomic update using MULTI/EXEC

### 2.2 User Profile Cache

**Purpose**: Speed up user profile retrieval

**Implementation**:
```
Key: user:{user_id}:profile
Value: JSON serialized user profile data
TTL: 24 hours
```

**Sample Redis Command**:
```redis
SET user:xyz789:profile "{\"username\":\"example_user\",\"display_name\":\"Example User\",\"subscribers\":1500}" EX 86400
```

**Invalidation Strategy**:
- On profile update
- Partial update using HSET for specific fields

### 2.3 Video Recommendations Cache

**Purpose**: Store personalized video recommendations

**Implementation**:
```
Key: user:{user_id}:recommendations
Value: Sorted set of video IDs with scores
TTL: 30 minutes
```

**Sample Redis Command**:
```redis
ZADD user:xyz789:recommendations 100 video1 95 video2 90 video3 85 video4
EXPIRE user:xyz789:recommendations 1800
```

**Invalidation Strategy**:
- Time-based expiration
- On significant user activity (new video watched)

### 2.4 Search Results Cache

**Purpose**: Cache frequent search queries

**Implementation**:
```
Key: search:{normalized_query}:results
Value: JSON array of video IDs
TTL: 10 minutes
```

**Sample Redis Command**:
```redis
SET search:how_to_make_pasta:results "[\"video1\",\"video2\",\"video3\",\"video4\",\"video5\"]" EX 600
```

**Invalidation Strategy**:
- Time-based only
- No explicit invalidation to reduce load

### 2.5 Video View Count Cache

**Purpose**: Handle high-frequency view count updates

**Implementation**:
```
Key: video:{video_id}:views
Value: Counter
Persistence: Periodic flush to database
```

**Sample Redis Commands**:
```redis
INCR video:abc123:views
```

**Database Synchronization**:
- Batch update every 5 minutes
- Lua script for atomic read-and-reset

### 2.6 Popular Videos Cache

**Purpose**: Track trending videos

**Implementation**:
```
Key: trending:videos
Value: Sorted set of video IDs with view velocity as score
TTL: 1 hour
```

**Sample Redis Command**:
```redis
ZADD trending:videos 1500 video1 1200 video2 900 video3
EXPIRE trending:videos 3600
```

**Update Strategy**:
- Periodic recalculation based on view velocity
- Atomic update using MULTI/EXEC

### 2.7 User Session Cache

**Purpose**: Store user session data

**Implementation**:
```
Key: session:{session_id}
Value: Hash of session data
TTL: 2 hours (extended on activity)
```

**Sample Redis Commands**:
```redis
HSET session:sess123 user_id xyz789 last_activity 1623456789 is_premium true
EXPIRE session:sess123 7200
```

**Invalidation Strategy**:
- On logout
- Time-based expiration

### 2.8 Comment Thread Cache

**Purpose**: Cache recent comments for videos

**Implementation**:
```
Key: video:{video_id}:comments:recent
Value: List of serialized comment objects
TTL: 5 minutes
```

**Sample Redis Command**:
```redis
LPUSH video:abc123:comments:recent "{\"user\":\"user1\",\"text\":\"Great video!\",\"timestamp\":1623456789}"
LTRIM video:abc123:comments:recent 0 99
EXPIRE video:abc123:comments:recent 300
```

**Invalidation Strategy**:
- On new comment (push to list)
- Time-based expiration

### 2.9 Video Segment Cache

**Purpose**: Cache frequently accessed video segments at application level

**Implementation**:
```
Key: video:{video_id}:segment:{quality}:{segment_number}
Value: Binary video segment data
TTL: 24 hours
```

**Sample Redis Command**:
```redis
SET video:abc123:segment:720p:42 <binary_data> EX 86400
```

**Storage Strategy**:
- Only cache most frequently accessed segments
- LRU eviction policy

### 2.10 Rate Limiting Cache

**Purpose**: Implement API rate limiting

**Implementation**:
```
Key: ratelimit:{ip_address}:{endpoint}
Value: Counter with sliding window
TTL: Based on rate limit window
```

**Sample Redis Commands**:
```redis
INCR ratelimit:192.168.1.1:video_upload
EXPIRE ratelimit:192.168.1.1:video_upload 3600
```

## 3. Cache Eviction Policies

### 3.1 Memory Management

- **Total Memory Allocation**: 800 GB across cluster
- **maxmemory-policy**: volatile-lru for most instances
- **Memory Distribution**:
  - 50% for video metadata and user data
  - 30% for recommendations and search results
  - 10% for counters and rate limiting
  - 10% for miscellaneous and buffer

### 3.2 Eviction Policies by Cache Type

| Cache Type | Eviction Policy | Justification |
|------------|-----------------|---------------|
| Video Metadata | volatile-lru | Most videos follow power law distribution |
| User Profiles | volatile-ttl | Even access pattern across users |
| Recommendations | volatile-lfu | Frequent access to popular users' recommendations |
| Search Results | volatile-ttl | Query popularity changes rapidly |
| Counters | noeviction | Critical data that must be processed |
| Session Data | volatile-ttl | Natural expiration with session timeout |

## 4. Cache Warming Strategies

### 4.1 Proactive Cache Warming

- **Video Metadata**: Pre-cache trending and popular videos
- **User Profiles**: Pre-cache profiles of active users and content creators
- **Recommendations**: Generate and cache during off-peak hours
- **Search Results**: Pre-cache common search terms based on analytics

### 4.2 Cache Warming Implementation

```python
def warm_video_cache(trending_videos):
    """Warm cache with trending video metadata"""
    pipeline = redis_client.pipeline()
    for video in trending_videos:
        video_data = fetch_video_metadata_from_db(video.id)
        pipeline.set(f"video:{video.id}:metadata", 
                    json.dumps(video_data), 
                    ex=3600)
    pipeline.execute()
```

## 5. Cache Consistency Strategies

### 5.1 Write-Through Cache

For critical data like user profiles and video metadata:

```python
def update_video_metadata(video_id, metadata):
    """Update both database and cache atomically"""
    # Update database
    db_result = update_video_in_database(video_id, metadata)
    
    if db_result.success:
        # Update cache
        redis_client.set(f"video:{video_id}:metadata", 
                        json.dumps(metadata), 
                        ex=3600)
    return db_result
```

### 5.2 Cache-Aside Pattern

For less critical data:

```python
def get_video_comments(video_id, page=1, limit=20):
    """Cache-aside pattern for comments"""
    cache_key = f"video:{video_id}:comments:page:{page}:limit:{limit}"
    
    # Try to get from cache
    cached_data = redis_client.get(cache_key)
    if cached_data:
        return json.loads(cached_data)
    
    # Cache miss, get from database
    comments = fetch_comments_from_db(video_id, page, limit)
    
    # Store in cache
    redis_client.set(cache_key, json.dumps(comments), ex=300)
    
    return comments
```

### 5.3 Event-Based Invalidation

Using message queue for cache invalidation:

```python
def handle_video_update_event(event):
    """Handle video update event from message queue"""
    video_id = event['video_id']
    
    # Invalidate video metadata cache
    redis_client.delete(f"video:{video_id}:metadata")
    
    # Invalidate related caches
    redis_client.delete(f"video:{video_id}:comments:recent")
    # Invalidate search results containing this video
    # ...
```

## 6. Handling Viral Content

### 6.1 Detection Mechanism

```python
def check_for_viral_content():
    """Identify potentially viral videos based on velocity"""
    # Get videos with high view velocity
    high_velocity_videos = redis_client.zrevrange(
        "video:view:velocity", 0, 99, withscores=True)
    
    for video_id, velocity in high_velocity_videos:
        if velocity > VIRAL_THRESHOLD:
            prepare_for_viral_traffic(video_id)
```

### 6.2 Cache Preparation for Viral Content

```python
def prepare_for_viral_traffic(video_id):
    """Prepare caching infrastructure for viral video"""
    # 1. Increase TTL for this video's metadata
    redis_client.expire(f"video:{video_id}:metadata", 86400)  # 24 hours
    
    # 2. Pre-cache video segments at multiple qualities
    cache_video_segments(video_id, qualities=['480p', '720p', '1080p'])
    
    # 3. Allocate dedicated Redis shard if extremely viral
    if is_extremely_viral(video_id):
        allocate_dedicated_cache_resources(video_id)
    
    # 4. Notify CDN to prepare for high traffic
    notify_cdn_for_traffic_spike(video_id)
```

## 7. Monitoring and Optimization

### 7.1 Cache Hit Rate Monitoring

```python
def track_cache_metrics(cache_type, hit):
    """Track cache hit/miss metrics"""
    if hit:
        redis_client.incr(f"stats:cache:{cache_type}:hits")
    else:
        redis_client.incr(f"stats:cache:{cache_type}:misses")
    
    # Calculate hit rate periodically
    if random.random() < 0.01:  # 1% sampling
        hits = int(redis_client.get(f"stats:cache:{cache_type}:hits") or 0)
        misses = int(redis_client.get(f"stats:cache:{cache_type}:misses") or 0)
        total = hits + misses
        
        if total > 0:
            hit_rate = hits / total
            log_metric(f"cache.{cache_type}.hit_rate", hit_rate)
```

### 7.2 Memory Usage Optimization

```python
def optimize_cache_memory():
    """Optimize Redis memory usage based on analytics"""
    # Get memory stats
    memory_stats = redis_client.info("memory")
    
    # If memory usage is high, adjust TTLs
    if memory_stats["used_memory_rss"] > MEMORY_THRESHOLD:
        # Reduce TTL for less critical caches
        redis_client.config_set("maxmemory-policy", "volatile-ttl")
        
        # Reduce TTL for search results
        for key in redis_client.scan_iter("search:*:results"):
            redis_client.expire(key, 300)  # 5 minutes
    else:
        # Normal memory usage, use LRU
        redis_client.config_set("maxmemory-policy", "volatile-lru")
```

## 8. Redis Client Configuration

### 8.1 Connection Pooling

```python
# Redis connection pool configuration
redis_pool = redis.ConnectionPool(
    host=REDIS_HOST,
    port=REDIS_PORT,
    password=REDIS_PASSWORD,
    db=0,
    max_connections=100,
    socket_timeout=2,
    socket_connect_timeout=1,
    retry_on_timeout=True
)

redis_client = redis.Redis(connection_pool=redis_pool)
```

### 8.2 Sentinel Configuration for High Availability

```python
from redis.sentinel import Sentinel

# Configure sentinel
sentinel = Sentinel([
    ('sentinel-1', 26379),
    ('sentinel-2', 26379),
    ('sentinel-3', 26379)
], socket_timeout=0.5)

# Get master for writing
master = sentinel.master_for(
    'mymaster',
    socket_timeout=0.5,
    password=REDIS_PASSWORD
)

# Get slave for reading
slave = sentinel.slave_for(
    'mymaster',
    socket_timeout=0.5,
    password=REDIS_PASSWORD
)
```

## 9. Disaster Recovery

### 9.1 Redis Persistence Configuration

```
# Redis persistence configuration
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
```

### 9.2 Backup Strategy

- **RDB Snapshots**: Every 6 hours
- **AOF Persistence**: Enabled with 1-second fsync
- **Backup Location**: Cross-region S3 bucket
- **Retention Policy**: 7 days of hourly backups

### 9.3 Recovery Procedure

```python
def recover_redis_cluster():
    """Procedure for recovering Redis cluster"""
    # 1. Identify failed nodes
    failed_nodes = identify_failed_nodes()
    
    # 2. Promote replicas if masters failed
    for node in failed_nodes:
        if node.is_master:
            promote_replica(node)
    
    # 3. Restore data from backup if needed
    if cluster_data_loss_detected():
        restore_from_latest_backup()
    
    # 4. Rebalance slots if needed
    rebalance_cluster_slots()
    
    # 5. Verify cluster health
    verify_cluster_health()
```

## 10. Cost Optimization

### 10.1 Redis Instance Sizing

| Cache Type | Instance Type | Count | Monthly Cost |
|------------|--------------|-------|--------------|
| Metadata Cache | cache.r5.2xlarge | 6 | $2,100 |
| User Data Cache | cache.r5.xlarge | 4 | $700 |
| Counters & Lists | cache.r5.large | 6 | $525 |
| Analytics Cache | cache.r5.2xlarge | 4 | $1,400 |
| **Total** | | 20 | **$4,725** |

### 10.2 Cost Reduction Strategies

- **Auto-scaling**: Scale down during off-peak hours
- **Data Tiering**: Move less frequently accessed data to cheaper storage
- **TTL Optimization**: Adjust TTLs based on access patterns
- **Compression**: Enable Redis compression for text-heavy data

## 11. Implementation Roadmap

### Phase 1: Basic Caching
- Implement video metadata and user profile caching
- Set up Redis Cluster with basic configuration
- Implement cache hit/miss monitoring

### Phase 2: Advanced Caching
- Implement recommendation and search result caching
- Set up cache warming strategies
- Implement viral content detection and handling

### Phase 3: Optimization
- Fine-tune eviction policies based on production data
- Implement advanced monitoring and alerting
- Optimize memory usage and connection handling