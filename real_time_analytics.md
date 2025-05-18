# Real-time Analytics Considerations

This document outlines the real-time analytics architecture for the video streaming platform, focusing on data collection, processing, and insights generation.

## 1. Analytics Pipeline

### 1.1 Data Collection

The platform will collect analytics data from multiple sources:

**Client-side Events:**
- Video playback events (play, pause, seek, quality change)
- User interaction events (likes, comments, shares)
- Error events (buffering, playback failures)
- Performance metrics (startup time, rebuffering)

**Server-side Events:**
- API request logs
- CDN logs
- Authentication events
- Backend service metrics

**Collection Methods:**
- Browser/app SDK for client events
- Server middleware for API events
- Log aggregation for infrastructure events
- Webhook integration for third-party events

**Sample Client SDK Implementation:**
```javascript
class VideoAnalytics {
  constructor(videoId, userId, sessionId) {
    this.videoId = videoId;
    this.userId = userId;
    this.sessionId = sessionId;
    this.events = [];
    this.flushInterval = setInterval(() => this.flush(), 10000);
  }
  
  trackEvent(eventType, eventData) {
    const event = {
      timestamp: Date.now(),
      videoId: this.videoId,
      userId: this.userId,
      sessionId: this.sessionId,
      eventType,
      eventData
    };
    
    this.events.push(event);
    
    // Flush immediately for certain events
    if (['error', 'session_end'].includes(eventType)) {
      this.flush();
    }
  }
  
  flush() {
    if (this.events.length === 0) return;
    
    const eventsToSend = [...this.events];
    this.events = [];
    
    fetch('https://analytics.videoplatform.com/collect', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        events: eventsToSend,
        batchId: uuidv4(),
        clientTimestamp: Date.now()
      }),
      // Use sendBeacon for better reliability when available
      keepalive: true
    }).catch(err => {
      // On failure, add back to queue
      this.events = [...eventsToSend, ...this.events];
    });
  }
}
```

### 1.2 Stream Processing

The platform will process events in real-time using stream processing technologies:

**Technology Selection:**
- Apache Kafka for event ingestion and buffering
- Kafka Streams for stateful processing
- Apache Flink for complex event processing
- Redis Streams for low-latency analytics

**Processing Topology:**
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Event       │     │ Stream      │     │ Analytics   │
│ Collection  ├────►│ Processing  ├────►│ Storage     │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Raw Event   │     │ Processed   │     │ Aggregated  │
│ Storage     │     │ Event Store │     │ Metrics     │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Kafka Streams Implementation:**
```java
StreamsBuilder builder = new StreamsBuilder();

// Raw video view events
KStream<String, VideoViewEvent> viewEvents = builder.stream(
    "video-view-events",
    Consumed.with(Serdes.String(), videoViewEventSerde)
);

// Group by video ID and window
KTable<Windowed<String>, Long> viewCounts = viewEvents
    .groupBy((key, event) -> event.getVideoId(),
             Grouped.with(Serdes.String(), videoViewEventSerde))
    .windowedBy(TimeWindows.of(Duration.ofMinutes(1)))
    .count();

// Publish to view counts topic
viewCounts.toStream()
    .map((windowedId, count) -> KeyValue.pair(
        windowedId.key(),
        new VideoViewCount(
            windowedId.key(),
            count,
            windowedId.window().start(),
            windowedId.window().end()
        )
    ))
    .to("video-view-counts", Produced.with(Serdes.String(), videoViewCountSerde));
```

### 1.3 Real-time Aggregation

The platform will perform real-time aggregations to generate insights:

**Aggregation Types:**
- Time-based windows (1-minute, 5-minute, 1-hour)
- Sliding windows for trend detection
- Session-based aggregation for user journeys
- Dimensional aggregation (by region, device, content type)

**Key Aggregations:**
- Video view counts
- Engagement rates
- Error rates
- Quality of experience metrics
- Audience demographics

**Flink Implementation Example:**
```java
// Define the processing environment
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// Configure event time processing
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

// Create a Kafka consumer
FlinkKafkaConsumer<VideoEvent> kafkaConsumer = new FlinkKafkaConsumer<>(
    "video-events",
    new VideoEventSchema(),
    kafkaProperties
);

// Set watermark strategy
kafkaConsumer.assignTimestampsAndWatermarks(
    WatermarkStrategy
        .<VideoEvent>forBoundedOutOfOrderness(Duration.ofSeconds(5))
        .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
);

// Process the stream
DataStream<VideoEvent> events = env.addSource(kafkaConsumer);

// Calculate view counts per minute
DataStream<VideoViewCount> viewCounts = events
    .filter(event -> event.getType().equals("view"))
    .keyBy(event -> event.getVideoId())
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new CountAggregateFunction(), new ViewCountWindowFunction());

// Calculate engagement rate
DataStream<EngagementRate> engagementRates = events
    .keyBy(event -> event.getVideoId())
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(new EngagementAggregateFunction(), new EngagementWindowFunction());

// Write results to sinks
viewCounts.addSink(new FlinkKafkaProducer<>(
    "video-view-counts",
    new VideoViewCountSchema(),
    kafkaProperties
));

engagementRates.addSink(new FlinkKafkaProducer<>(
    "video-engagement-rates",
    new EngagementRateSchema(),
    kafkaProperties
));
```

## 2. Metrics and KPIs

### 2.1 User Engagement Metrics

**Watch Time Metrics:**
- Total watch time
- Average watch time per video
- Watch time distribution
- Completion rate
- Rewatch rate

**Interaction Metrics:**
- Likes per view
- Comments per view
- Shares per view
- Click-through rate on recommendations
- Subscription conversion rate

**Retention Metrics:**
- Return visitor rate
- Session frequency
- Time between sessions
- Churn prediction score
- Content stickiness

### 2.2 Platform Health Metrics

**Performance Metrics:**
- Video startup time
- Buffering ratio
- Quality switches
- Error rate
- API response time

**Infrastructure Metrics:**
- CDN cache hit ratio
- Origin server load
- Database query performance
- Transcoding queue depth
- Service availability

**Example Dashboard Configuration:**
```json
{
  "dashboard": {
    "title": "Platform Health",
    "refresh": "10s",
    "panels": [
      {
        "title": "Video Startup Time (p95)",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(video_startup_time_bucket[5m])) by (le))",
            "legendFormat": "p95 Startup Time"
          }
        ],
        "thresholds": [
          {
            "value": 2000,
            "colorMode": "warning",
            "op": "gt",
            "line": true,
            "fill": false
          },
          {
            "value": 4000,
            "colorMode": "critical",
            "op": "gt",
            "line": true,
            "fill": false
          }
        ]
      },
      {
        "title": "Buffering Ratio",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(video_buffering_time_sum[5m])) by (region) / sum(rate(video_watch_time_sum[5m])) by (region)",
            "legendFormat": "{{region}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(video_playback_errors_total[5m])) / sum(rate(video_play_count_total[5m]))",
            "legendFormat": "Error Rate"
          }
        ]
      }
    ]
  }
}
```

### 2.3 Content Performance Metrics

**Popularity Metrics:**
- Views (total, unique)
- View velocity
- Trending score
- Peak concurrent viewers
- Geographic distribution

**Quality Metrics:**
- Audience retention curve
- Average percentage viewed
- Skip rate
- Engagement index
- Viewer satisfaction score

**Monetization Metrics:**
- Revenue per view
- Ad completion rate
- Ad click-through rate
- Subscription conversion
- Revenue by content category

### 2.4 Recommendation Performance

**Accuracy Metrics:**
- Click-through rate
- Watch-through rate
- Time spent on recommended content
- Diversity score
- Serendipity score

**System Performance:**
- Recommendation generation time
- Coverage ratio
- Catalog exploration rate
- Cold start performance
- Personalization impact

## 3. Analytics Storage

### 3.1 Time-Series Database

For metrics and monitoring data:

**Technology Selection:**
- InfluxDB for high-write throughput
- Prometheus for metrics collection
- Grafana for visualization
- Custom retention policies by data importance

**Schema Design:**
```
measurement: video_views
tags:
  - video_id
  - region
  - device_type
  - quality
fields:
  - count
  - watch_time
  - unique_viewers
timestamp: event_time

measurement: video_errors
tags:
  - video_id
  - error_type
  - player_version
  - device_type
fields:
  - count
  - impact_score
timestamp: event_time
```

**Retention Policy:**
```sql
-- Create retention policies
CREATE RETENTION POLICY "realtime" ON "analytics" DURATION 1d REPLICATION 3 DEFAULT;
CREATE RETENTION POLICY "hourly" ON "analytics" DURATION 7d REPLICATION 3;
CREATE RETENTION POLICY "daily" ON "analytics" DURATION 90d REPLICATION 3;
CREATE RETENTION POLICY "monthly" ON "analytics" DURATION 365d REPLICATION 3;

-- Create continuous queries for downsampling
CREATE CONTINUOUS QUERY "cq_video_views_hourly" ON "analytics"
BEGIN
  SELECT sum(count) as count, sum(watch_time) as watch_time, count(distinct(unique_viewers)) as unique_viewers
  INTO "hourly"."video_views"
  FROM "realtime"."video_views"
  GROUP BY time(1h), video_id, region
END;

CREATE CONTINUOUS QUERY "cq_video_views_daily" ON "analytics"
BEGIN
  SELECT sum(count) as count, sum(watch_time) as watch_time, sum(unique_viewers) as unique_viewers
  INTO "daily"."video_views"
  FROM "hourly"."video_views"
  GROUP BY time(1d), video_id, region
END;
```

### 3.2 Columnar Storage

For aggregations and analytics:

**Technology Selection:**
- ClickHouse for high-performance analytics
- Parquet file format for data lake storage
- Partition by time for efficient queries
- Materialized views for common queries

**Schema Design:**
```sql
CREATE TABLE video_events (
    event_time DateTime,
    video_id String,
    user_id String,
    session_id String,
    event_type Enum('play', 'pause', 'seek', 'buffer', 'error', 'complete'),
    position UInt32,
    quality String,
    playback_id String,
    device_type String,
    os String,
    browser String,
    country String,
    region String,
    isp String,
    error_code Nullable(String),
    error_message Nullable(String),
    cdn String,
    referrer String
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(event_time)
ORDER BY (video_id, event_time);

-- Materialized view for video views
CREATE MATERIALIZED VIEW video_views_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMMDD(event_date)
ORDER BY (video_id, event_date, device_type, country)
AS SELECT
    video_id,
    toDate(event_time) as event_date,
    device_type,
    country,
    count() as views,
    uniqExact(user_id) as unique_viewers,
    uniqExact(session_id) as sessions
FROM video_events
WHERE event_type = 'play'
GROUP BY video_id, event_date, device_type, country;
```

### 3.3 Data Lake

For raw event data and long-term storage:

**Technology Selection:**
- S3 or equivalent object storage
- Partitioned by date, event type
- Parquet or ORC file format
- Athena/Presto for ad-hoc queries

**Folder Structure:**
```
/data-lake/
  /video-events/
    /year=2023/
      /month=01/
        /day=01/
          part-00000.parquet
          part-00001.parquet
          ...
      /month=02/
        ...
  /user-events/
    ...
  /error-events/
    ...
```

**Athena Query Example:**
```sql
SELECT
  date_trunc('hour', event_time) as hour,
  video_id,
  country,
  count(*) as play_count,
  count(distinct user_id) as unique_viewers,
  avg(case when event_type = 'complete' then 1 else 0 end) as completion_rate
FROM
  video_events
WHERE
  event_type in ('play', 'complete')
  AND date_trunc('day', event_time) >= date_add('day', -7, current_date)
GROUP BY
  date_trunc('hour', event_time),
  video_id,
  country
ORDER BY
  hour DESC,
  play_count DESC
LIMIT 100;
```

### 3.4 OLAP Database

For complex queries and business intelligence:

**Technology Selection:**
- Amazon Redshift or Google BigQuery
- Star schema design
- Materialized views for performance
- Automated ETL pipelines

**Schema Design:**
```sql
-- Fact table
CREATE TABLE fact_video_views (
    view_id BIGINT IDENTITY(1,1),
    video_key INT NOT NULL REFERENCES dim_videos(video_key),
    user_key INT REFERENCES dim_users(user_key),
    time_key INT NOT NULL REFERENCES dim_time(time_key),
    location_key INT REFERENCES dim_locations(location_key),
    device_key INT REFERENCES dim_devices(device_key),
    view_count INT NOT NULL,
    watch_time_seconds INT NOT NULL,
    avg_bitrate INT,
    rebuffer_count INT,
    rebuffer_duration_ms INT,
    exit_before_video_start BOOLEAN,
    video_completed BOOLEAN,
    PRIMARY KEY (view_id)
)
DISTSTYLE KEY
DISTKEY (video_key)
SORTKEY (time_key);

-- Dimension tables
CREATE TABLE dim_videos (
    video_key INT IDENTITY(1,1),
    video_id VARCHAR(50) NOT NULL,
    title VARCHAR(255),
    creator_id VARCHAR(50),
    category VARCHAR(50),
    tags VARCHAR(1000),
    duration_seconds INT,
    upload_date TIMESTAMP,
    is_premium BOOLEAN,
    PRIMARY KEY (video_key)
)
DISTSTYLE ALL;

CREATE TABLE dim_time (
    time_key INT IDENTITY(1,1),
    event_timestamp TIMESTAMP NOT NULL,
    hour INT NOT NULL,
    day INT NOT NULL,
    month INT NOT NULL,
    quarter INT NOT NULL,
    year INT NOT NULL,
    day_of_week INT NOT NULL,
    is_weekend BOOLEAN NOT NULL,
    PRIMARY KEY (time_key)
)
DISTSTYLE ALL
SORTKEY (event_timestamp);
```

## 4. Real-time Dashboards

### 4.1 Executive Dashboard

High-level platform performance:

**Key Metrics:**
- Daily active users
- Total watch time
- Revenue metrics
- Platform health score
- Growth indicators

**Implementation:**
- Real-time metrics from Kafka Streams
- Aggregated in Redis
- Visualized with Grafana or custom dashboard
- Refreshed every minute

### 4.2 Content Creator Dashboard

For video publishers to monitor performance:

**Key Metrics:**
- Real-time views
- Audience demographics
- Engagement metrics
- Revenue analytics
- Comparative performance

**Implementation:**
- Filtered view of platform analytics
- Creator-specific aggregations
- Personalized insights
- Export capabilities

### 4.3 Operational Dashboard

For platform operations team:

**Key Metrics:**
- Service health
- Error rates
- Infrastructure utilization
- CDN performance
- API performance

**Implementation:**
- Integration with monitoring systems
- Alerting thresholds
- Historical comparison
- Drill-down capabilities

## 5. Machine Learning Integration

### 5.1 Real-time Prediction

Using analytics data for ML predictions:

**Use Cases:**
- Content recommendation
- Churn prediction
- Fraud detection
- Content categorization
- Trending content prediction

**Implementation Approach:**
- Feature extraction from event streams
- Online learning models
- Model serving via API
- Feedback loops for improvement

**Recommendation Pipeline:**
```python
def process_user_event(event):
    """
    Process user event for real-time recommendations
    """
    # Extract user and content features
    user_id = event['user_id']
    video_id = event['video_id']
    event_type = event['event_type']
    
    # Update user profile in Redis
    if event_type == 'view':
        redis_client.zincrby(f"user:{user_id}:category_affinity", 
                            1, 
                            get_video_category(video_id))
    
    # Update content popularity
    if event_type == 'view':
        # Increment view count with time decay
        redis_client.zadd("trending_videos", 
                        {video_id: time.time()})
    
    # Generate real-time recommendations
    if event_type in ['view_complete', 'like']:
        # Get user preferences
        categories = redis_client.zrange(f"user:{user_id}:category_affinity", 
                                      0, -1, 
                                      withscores=True,
                                      desc=True)
        
        # Get recently watched videos
        watched_videos = redis_client.lrange(f"user:{user_id}:watched", 0, 100)
        
        # Call recommendation service
        recommendations = recommendation_service.get_recommendations(
            user_id=user_id,
            categories=categories,
            exclude_videos=watched_videos,
            limit=10
        )
        
        # Cache recommendations
        redis_client.setex(f"user:{user_id}:recommendations", 
                        300,  # 5 minutes TTL
                        json.dumps(recommendations))
```

### 5.2 Anomaly Detection

Identifying unusual patterns in real-time:

**Use Cases:**
- Fraud detection
- Content policy violations
- Performance degradation
- Security incidents
- Viral content detection

**Implementation Approach:**
- Statistical anomaly detection
- Time-series analysis
- Clustering for outlier detection
- Supervised classification for known patterns

**Anomaly Detection Pipeline:**
```python
def detect_performance_anomalies(metrics_stream):
    """
    Detect anomalies in performance metrics
    """
    # Create time windows
    windowed_metrics = metrics_stream.window(
        size=timedelta(minutes=5),
        step=timedelta(minutes=1)
    )
    
    # Calculate statistics
    def calculate_stats(window):
        values = [x['value'] for x in window]
        if not values:
            return None
            
        mean = statistics.mean(values)
        stddev = statistics.stdev(values) if len(values) > 1 else 0
        
        return {
            'mean': mean,
            'stddev': stddev,
            'min': min(values),
            'max': max(values),
            'count': len(values)
        }
    
    stats_stream = windowed_metrics.apply(calculate_stats)
    
    # Detect anomalies using Z-score
    def detect_anomaly(current, history):
        if not history or not current:
            return False
            
        z_score = abs(current['value'] - history['mean']) / history['stddev'] if history['stddev'] > 0 else 0
        return z_score > 3  # More than 3 standard deviations
    
    # Apply detection
    anomalies = metrics_stream.join(
        stats_stream.hopping_window(size=timedelta(hours=1)),
        detect_anomaly
    )
    
    return anomalies
```

## 6. Compliance and Privacy

### 6.1 Data Anonymization

Protecting user privacy in analytics:

**Techniques:**
- User ID hashing
- IP address truncation
- Aggregation thresholds
- Data minimization
- Retention limits

**Implementation Example:**
```python
def anonymize_event(event):
    """
    Anonymize sensitive data in analytics events
    """
    # Create a copy to avoid modifying the original
    anonymized = event.copy()
    
    # Hash user ID if present
    if 'user_id' in anonymized and not anonymized.get('user_opted_in_analytics'):
        anonymized['user_id'] = hash_user_id(anonymized['user_id'])
    
    # Truncate IP address
    if 'ip_address' in anonymized:
        anonymized['ip_address'] = truncate_ip(anonymized['ip_address'])
    
    # Remove PII fields
    pii_fields = ['email', 'full_name', 'phone_number', 'device_id']
    for field in pii_fields:
        if field in anonymized:
            del anonymized[field]
    
    return anonymized

def truncate_ip(ip):
    """
    Truncate IP address for anonymization
    IPv4: Keep first 3 octets
    IPv6: Keep first 6 quartets
    """
    try:
        if ':' in ip:  # IPv6
            parts = ip.split(':')
            return ':'.join(parts[:6]) + ':0:0'
        else:  # IPv4
            parts = ip.split('.')
            return '.'.join(parts[:3]) + '.0'
    except:
        return '0.0.0.0'
```

### 6.2 Data Governance

Ensuring compliance with regulations:

**Key Considerations:**
- GDPR compliance
- CCPA compliance
- Data access controls
- Audit logging
- Data subject requests

**Implementation Approach:**
- Data classification framework
- Access control policies
- Automated compliance checks
- Data lineage tracking
- Consent management

## 7. Cost Optimization

### 7.1 Tiered Analytics

Optimizing analytics costs:

**Implementation Approach:**
- Full-resolution data for recent periods
- Downsampled data for historical analysis
- Aggregated data for long-term trends
- Sampling for high-volume events

**Tiering Strategy:**
- Real-time tier: 100% of events, 7 days retention
- Hourly tier: Aggregated hourly, 30 days retention
- Daily tier: Aggregated daily, 1 year retention
- Monthly tier: Aggregated monthly, multi-year retention

### 7.2 Efficient Processing

Reducing processing costs:

**Techniques:**
- Event filtering at source
- Batch processing where possible
- Optimized serialization formats
- Resource-efficient algorithms
- Caching of derived metrics

**Implementation Example:**
```java
// Filter events early in the pipeline
DataStream<VideoEvent> filteredEvents = rawEvents
    .filter(event -> {
        // Filter out heartbeat events except every 30 seconds
        if (event.getType().equals("heartbeat")) {
            return event.getTimestamp() % 30000 == 0;
        }
        // Keep all other event types
        return true;
    });

// Use efficient serialization
env.getConfig().enableForceAvro();
env.getConfig().registerTypeWithKryoSerializer(VideoEvent.class, AvroSerializer.class);

// Batch processing for historical data
if (!isRealTimeMode) {
    env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);
    env.getConfig().setAutoWatermarkInterval(1000);
}
```

## 8. Implementation Roadmap

### Phase 1: Core Analytics
- Implement basic event collection
- Set up Kafka pipeline
- Deploy time-series database
- Create operational dashboards

### Phase 2: Advanced Analytics
- Implement real-time aggregations
- Deploy columnar database
- Create content and user analytics
- Set up data lake architecture

### Phase 3: Intelligence Layer
- Implement ML integrations
- Deploy anomaly detection
- Create predictive analytics
- Build recommendation feedback loops