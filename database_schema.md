# Database Schema Design

This document provides a detailed database schema design for the video streaming platform, including entity relationships and optimization strategies.

## 1. Database Technology Selection

The platform uses a polyglot persistence approach with different database technologies for different data types:

| Data Type | Database Technology | Justification |
|-----------|---------------------|---------------|
| User data | PostgreSQL | Strong consistency, ACID transactions, complex relationships |
| Video metadata | MongoDB | Schema flexibility, document structure matches video metadata |
| Comments | MongoDB | Schema flexibility, high write throughput |
| Analytics | ClickHouse | Column-oriented for analytical queries, high compression |
| Search index | Elasticsearch | Full-text search capabilities, faceting, relevance scoring |
| Caching | Redis | In-memory performance, data structures, pub/sub |
| Time-series metrics | InfluxDB | Optimized for time-series data, retention policies |

## 2. Detailed Schema Design

### 2.1 User Database (PostgreSQL)

#### Users Table
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_login TIMESTAMP,
    is_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    account_type VARCHAR(20) DEFAULT 'standard',
    two_factor_enabled BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

#### User Profiles Table
```sql
CREATE TABLE user_profiles (
    profile_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    display_name VARCHAR(100),
    bio TEXT,
    profile_picture_url VARCHAR(255),
    banner_url VARCHAR(255),
    country VARCHAR(100),
    language VARCHAR(50),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    channel_url VARCHAR(100) UNIQUE,
    total_subscribers INT DEFAULT 0,
    total_videos INT DEFAULT 0,
    total_views BIGINT DEFAULT 0
);

CREATE INDEX idx_user_profiles_user_id ON user_profiles(user_id);
CREATE INDEX idx_user_profiles_channel_url ON user_profiles(channel_url);
```

#### Subscriptions Table
```sql
CREATE TABLE subscriptions (
    subscription_id UUID PRIMARY KEY,
    subscriber_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    channel_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    notification_enabled BOOLEAN DEFAULT TRUE,
    UNIQUE(subscriber_id, channel_id)
);

CREATE INDEX idx_subscriptions_subscriber ON subscriptions(subscriber_id);
CREATE INDEX idx_subscriptions_channel ON subscriptions(channel_id);
```

#### User Settings Table
```sql
CREATE TABLE user_settings (
    setting_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    autoplay_enabled BOOLEAN DEFAULT TRUE,
    theme VARCHAR(20) DEFAULT 'light',
    content_language VARCHAR(50) DEFAULT 'en',
    content_region VARCHAR(50) DEFAULT 'US',
    restricted_mode BOOLEAN DEFAULT FALSE,
    privacy_settings JSONB,
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_settings_user_id ON user_settings(user_id);
```

#### Watch History Table
```sql
CREATE TABLE watch_history (
    history_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    video_id VARCHAR(50) NOT NULL,
    watched_at TIMESTAMP NOT NULL DEFAULT NOW(),
    watch_duration INT NOT NULL, -- in seconds
    watch_percentage FLOAT NOT NULL,
    completed BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_watch_history_user_id ON watch_history(user_id);
CREATE INDEX idx_watch_history_video_id ON watch_history(video_id);
CREATE INDEX idx_watch_history_watched_at ON watch_history(watched_at);
```

#### Playlists Table
```sql
CREATE TABLE playlists (
    playlist_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    title VARCHAR(100) NOT NULL,
    description TEXT,
    visibility VARCHAR(20) DEFAULT 'private',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    thumbnail_url VARCHAR(255),
    video_count INT DEFAULT 0
);

CREATE INDEX idx_playlists_user_id ON playlists(user_id);
```

#### Playlist Items Table
```sql
CREATE TABLE playlist_items (
    item_id UUID PRIMARY KEY,
    playlist_id UUID REFERENCES playlists(playlist_id) ON DELETE CASCADE,
    video_id VARCHAR(50) NOT NULL,
    added_at TIMESTAMP NOT NULL DEFAULT NOW(),
    position INT NOT NULL
);

CREATE INDEX idx_playlist_items_playlist_id ON playlist_items(playlist_id);
CREATE INDEX idx_playlist_items_video_id ON playlist_items(video_id);
```

### 2.2 Video Metadata (MongoDB)

#### Videos Collection
```json
{
  "_id": "ObjectId",
  "video_id": "string",
  "user_id": "uuid",
  "title": "string",
  "description": "string",
  "category": "string",
  "tags": ["string"],
  "visibility": "enum(public, private, unlisted)",
  "upload_status": "enum(processing, ready, failed)",
  "processing_status": {
    "upload_complete": "boolean",
    "transcoding_complete": "boolean",
    "thumbnail_generation_complete": "boolean",
    "error": "string"
  },
  "duration": "number",
  "views": "number",
  "likes": "number",
  "dislikes": "number",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "published_at": "timestamp",
  "thumbnails": [
    {
      "url": "string",
      "width": "number",
      "height": "number",
      "type": "enum(auto, custom)"
    }
  ],
  "video_files": [
    {
      "quality": "enum(240p, 360p, 480p, 720p, 1080p)",
      "format": "string",
      "codec": "string",
      "url": "string",
      "size": "number",
      "bitrate": "number"
    }
  ],
  "captions": [
    {
      "language": "string",
      "auto_generated": "boolean",
      "url": "string"
    }
  ],
  "engagement_score": "number",
  "monetization_enabled": "boolean",
  "age_restriction": "boolean",
  "content_rating": "string",
  "location": {
    "latitude": "number",
    "longitude": "number",
    "name": "string"
  },
  "statistics": {
    "comments_count": "number",
    "shares": "number",
    "watch_time": "number",
    "avg_view_duration": "number",
    "completion_rate": "number"
  },
  "engagement_metrics": {
    "first_24h_views": "number",
    "first_week_views": "number",
    "click_through_rate": "number",
    "unique_viewers": "number"
  }
}
```

#### Indexes for Videos Collection
```javascript
db.videos.createIndex({ "video_id": 1 }, { unique: true });
db.videos.createIndex({ "user_id": 1 });
db.videos.createIndex({ "created_at": -1 });
db.videos.createIndex({ "views": -1 });
db.videos.createIndex({ "engagement_score": -1 });
db.videos.createIndex({ "tags": 1 });
db.videos.createIndex({ "category": 1 });
db.videos.createIndex({ 
  "title": "text", 
  "description": "text", 
  "tags": "text" 
}, { 
  weights: { 
    "title": 10, 
    "tags": 5, 
    "description": 1 
  } 
});
```

### 2.3 Comments (MongoDB)

#### Comments Collection
```json
{
  "_id": "ObjectId",
  "comment_id": "uuid",
  "video_id": "string",
  "user_id": "uuid",
  "parent_comment_id": "uuid or null",
  "content": "string",
  "likes": "number",
  "dislikes": "number",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "is_pinned": "boolean",
  "is_edited": "boolean",
  "is_hidden": "boolean",
  "replies_count": "number",
  "user_info": {
    "username": "string",
    "display_name": "string",
    "profile_picture_url": "string"
  },
  "sentiment_score": "number",
  "has_replies": "boolean"
}
```

#### Indexes for Comments Collection
```javascript
db.comments.createIndex({ "video_id": 1, "created_at": -1 });
db.comments.createIndex({ "user_id": 1 });
db.comments.createIndex({ "parent_comment_id": 1 });
db.comments.createIndex({ "likes": -1 });
db.comments.createIndex({ "video_id": 1, "parent_comment_id": 1, "created_at": -1 });
```

### 2.4 Analytics Data (ClickHouse)

#### Video Views Table
```sql
CREATE TABLE video_views (
    event_time DateTime,
    video_id String,
    user_id UUID,
    session_id String,
    ip_address String,
    country String,
    region String,
    city String,
    device_type String,
    browser String,
    os String,
    referrer String,
    watch_time UInt32,
    quality String,
    buffering_events UInt8,
    buffering_duration UInt16,
    exit_point UInt32,
    is_autoplay UInt8,
    is_muted UInt8
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (video_id, event_time);
```

#### User Engagement Table
```sql
CREATE TABLE user_engagement (
    event_time DateTime,
    event_type Enum('view', 'like', 'dislike', 'comment', 'share', 'subscribe'),
    user_id UUID,
    video_id String,
    channel_id UUID,
    session_id String,
    engagement_time UInt32,
    device_type String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);
```

## 3. Database Sharding Strategy

### 3.1 PostgreSQL Sharding

**Sharding Key**: `user_id`

**Sharding Strategy**:
- Hash-based sharding across multiple PostgreSQL instances
- Each shard contains complete user data (profile, settings, subscriptions)
- 10 shards initially, with capacity to expand to 100 shards

**Implementation**:
- Use Citus extension for PostgreSQL
- Distribute tables by user_id
- Keep reference tables replicated across all shards

### 3.2 MongoDB Sharding

**Videos Collection**:
- Shard Key: `{ user_id: 1, created_at: -1 }`
- Sharding Strategy: Range-based on user_id with hash on created_at
- Benefits: Co-locates videos by same creator, distributes load evenly

**Comments Collection**:
- Shard Key: `{ video_id: 1 }`
- Sharding Strategy: Hash-based on video_id
- Benefits: Keeps comments for same video together, enables efficient pagination

### 3.3 ClickHouse Sharding

**Sharding Strategy**:
- Shard by event_time (month)
- Further partition by video_id hash
- Replicate each shard for high availability

## 4. Data Access Patterns

### 4.1 Common Queries and Optimizations

#### User Profile Retrieval
```sql
-- Optimized with user_id index
SELECT u.username, u.email, p.*
FROM users u
JOIN user_profiles p ON u.user_id = p.user_id
WHERE u.user_id = ?;
```

#### Video Metadata Retrieval (MongoDB)
```javascript
// Optimized with video_id index
db.videos.findOne({ video_id: "video123" });
```

#### Video Comments Retrieval (MongoDB)
```javascript
// Optimized with compound index on video_id, parent_comment_id, created_at
db.comments.find({ 
  video_id: "video123", 
  parent_comment_id: null 
})
.sort({ created_at: -1 })
.limit(20);
```

#### User Recommendations
```javascript
// Optimized with user activity data and content-based filtering
db.videos.find({
  category: { $in: userPreferredCategories },
  video_id: { $nin: userWatchedVideos },
  visibility: "public",
  upload_status: "ready"
})
.sort({ engagement_score: -1 })
.limit(20);
```

### 4.2 Denormalization Strategy

To optimize read performance, the following denormalization strategies are implemented:

1. **User Info in Comments**: Store basic user info (username, display name, profile picture) in comments to avoid joins
2. **Video Statistics**: Maintain counter fields (views, likes, comments) in the video document
3. **Channel Info**: Store channel metrics (subscriber count, video count) in user_profiles
4. **Engagement Metrics**: Pre-calculate and store engagement scores

## 5. Data Migration and Evolution Strategy

### 5.1 Schema Evolution

- Use schema versioning for MongoDB collections
- Implement backward-compatible changes
- For PostgreSQL, use transactional schema migrations

### 5.2 Data Migration

- Use background jobs for large-scale data migrations
- Implement dual-write pattern during transitions
- Validate data consistency after migrations

## 6. Database Backup Strategy

### 6.1 PostgreSQL Backup

- Daily full backups
- Continuous WAL archiving
- Point-in-time recovery capability
- Cross-region backup replication

### 6.2 MongoDB Backup

- Daily snapshots
- Continuous oplog backup
- Automated restore testing
- Geo-redundant storage

### 6.3 ClickHouse Backup

- Table-level backups
- Incremental backups for large tables
- Backup verification process

## 7. Database Security

### 7.1 Access Control

- Role-based access control for all databases
- Service-specific database users
- Least privilege principle
- Regular permission audits

### 7.2 Data Protection

- Encryption at rest for all databases
- TLS for all database connections
- Sensitive data encryption at column level
- Regular security scanning