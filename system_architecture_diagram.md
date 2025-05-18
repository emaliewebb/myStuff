# System Architecture Diagram

Below is a more detailed architecture diagram for the video streaming platform:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                                  │
├─────────────┬─────────────┬─────────────┬─────────────┬─────────────────┤
│  Web App    │  iOS App    │ Android App │  Smart TV   │  Embedded       │
│  (React)    │  (Swift)    │ (Kotlin)    │  Apps       │  Players        │
└──────┬──────┴──────┬──────┴──────┬──────┴──────┬──────┴────────┬────────┘
       │             │             │             │               │
       └─────────────┼─────────────┼─────────────┼───────────────┘
                     │             │             │
┌────────────────────▼─────────────▼─────────────▼────────────────────────┐
│                           DELIVERY LAYER                                │
├─────────────────────────────┬───────────────────────────────────────────┤
│     Global CDN              │     Edge Caching                          │
│  (CloudFront/Cloudflare)    │     (Regional PoPs)                       │
└─────────────────────────────┴───────────────────┬───────────────────────┘
                                                  │
┌──────────────────────────────────────────────────▼──────────────────────┐
│                           GATEWAY LAYER                                 │
├─────────────────────────────┬───────────────────────────────────────────┤
│     API Gateway             │     Load Balancers                        │
│  (AWS API Gateway/Kong)     │     (ALB/Nginx)                           │
└─────────────────────────────┴───────────────────┬───────────────────────┘
                                                  │
┌──────────────────────────────────────────────────▼──────────────────────┐
│                        SERVICE MESH LAYER                               │
├─────────────────────────────────────────────────────────────────────────┤
│  Service Discovery | Circuit Breaking | Load Balancing | Observability  │
│  (Istio/Linkerd)                                                        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                 ┌─────────────────────────────────────┐
                 │                                     │
┌────────────────▼─────────────┐     ┌─────────────────▼────────────────┐
│    CORE SERVICES LAYER       │     │   SUPPORTING SERVICES LAYER      │
├────────────┬────────────┬────┤     ├────────────┬────────────┬────────┤
│ User       │ Video      │    │     │ Search     │ Analytics  │        │
│ Service    │ Service    │    │     │ Service    │ Service    │        │
├────────────┼────────────┤    │     ├────────────┼────────────┤        │
│ Auth       │ Transcode  │    │     │ Recommend  │ Notification│       │
│ Service    │ Service    │    │     │ Service    │ Service    │        │
├────────────┼────────────┤    │     ├────────────┼────────────┤        │
│ Comment    │ Upload     │    │     │ Trending   │ Moderation │        │
│ Service    │ Service    │    │     │ Service    │ Service    │        │
└────────────┴────────────┴────┘     └────────────┴────────────┴────────┘
         │                  │                │            │
         └──────────────────┼────────────────┼────────────┘
                            │                │
┌───────────────────────────▼────────────────▼───────────────────────────┐
│                        MESSAGING LAYER                                 │
├─────────────────────────────┬───────────────────────────────────────────┤
│     Event Bus               │     Message Queues                        │
│     (Kafka)                 │     (RabbitMQ/SQS)                        │
└─────────────────────────────┴───────────────────┬───────────────────────┘
                                                  │
┌──────────────────────────────────────────────────▼──────────────────────┐
│                        DATA STORAGE LAYER                               │
├────────────┬────────────┬────────────┬────────────┬────────────┬────────┤
│ SQL DB     │ NoSQL DB   │ Redis      │ Time-Series│ Search     │ Data   │
│ (User/Meta)│ (Video/    │ Cache      │ DB         │ Index      │ Lake   │
│ PostgreSQL │ Comments)  │ Cluster    │ (InfluxDB) │ (Elastic)  │ (S3)   │
└────────────┴────────────┴────────────┴────────────┴────────────┴────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────────┐
│                        STORAGE LAYER                                  │
├────────────────────────┬──────────────────────┬─────────────────────────┤
│  Hot Storage           │  Warm Storage        │  Cold Storage           │
│  (S3 Standard)         │  (S3 IA)             │  (S3 Glacier)           │
└────────────────────────┴──────────────────────┴─────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────────┐
│                     OBSERVABILITY LAYER                               │
├────────────────────┬─────────────────────┬───────────────────────────────┤
│  Metrics           │  Logging            │  Tracing                      │
│  (Prometheus)      │  (ELK Stack)        │  (Jaeger)                     │
└────────────────────┴─────────────────────┴───────────────────────────────┘
```

## Key Architecture Components Interaction

### Video Upload Flow

1. User uploads video through web/mobile client
2. API Gateway authenticates and routes request to Upload Service
3. Upload Service:
   - Validates video format and content
   - Stores raw video in S3
   - Creates metadata entry in database
   - Publishes upload event to Kafka
4. Transcoding Service:
   - Consumes upload event
   - Transcodes video to multiple formats/resolutions
   - Stores transcoded versions in S3
   - Updates video metadata with available formats
   - Publishes transcoding complete event
5. Notification Service notifies user of completed processing

### Video Streaming Flow

1. User requests video through client application
2. Request routed through CDN
3. If video segment cached at edge, served directly
4. If not cached:
   - Request forwarded to origin server
   - Video Service retrieves metadata
   - CDN caches video segment for future requests
5. Adaptive bitrate streaming adjusts quality based on user's connection
6. Analytics Service records view and engagement metrics

### Search and Discovery Flow

1. User enters search query
2. API Gateway routes to Search Service
3. Search Service:
   - Queries Elasticsearch index
   - Applies personalization filters
   - Returns ranked results
4. Results displayed to user
5. User interactions feed back to Recommendation Service

### Recommendation Flow

1. User activity (views, likes, etc.) published to Kafka
2. Analytics Service processes events in real-time
3. Recommendation Service:
   - Builds user profile based on activity
   - Generates personalized recommendations
   - Caches recommendations in Redis
4. Recommendations served to user through API Gateway