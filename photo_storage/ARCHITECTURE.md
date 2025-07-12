# Photo Storage System Architecture

## High-Level Architecture

```
                    ┌─────────────────┐
                    │   CDN/Edge      │
                    │   (CloudFlare)  │
                    └─────────┬───────┘
                              │
                    ┌─────────┴───────┐
                    │  Load Balancer  │
                    │   (HAProxy)     │
                    └─────────┬───────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────┴─────┐     ┌─────┴─────┐     ┌─────┴─────┐
    │  Upload   │     │   Read    │     │  Search   │
    │ Service   │     │ Service   │     │ Service   │
    └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
          │                 │                 │
          │         ┌───────┴───────┐         │
          │         │     Cache     │         │
          │         │  (Redis)      │         │
          │         └───────────────┘         │
          │                                   │
    ┌─────┴─────┐                     ┌─────┴─────┐
    │Processing │                     │  Search   │
    │Pipeline   │                     │  Index    │
    │(Kafka)    │                     │(Elastic)  │
    └─────┬─────┘                     └───────────┘
          │
    ┌─────┴─────┐
    │ ML/CV     │
    │Processing │
    │(Spark)    │
    └─────┬─────┘
          │
    ┌─────┴─────┐
    │ Metadata  │
    │    DB     │
    │(Spanner)  │
    └─────┬─────┘
          │
    ┌─────┴─────┐
    │  Object   │
    │ Storage   │
    │  (S3)     │
    └───────────┘
```

## Core Components

### 1. API Gateway & Load Balancing

**Components:**
- **CDN Layer**: CloudFlare for global edge caching
- **Load Balancer**: HAProxy for traffic distribution
- **API Gateway**: Kong or AWS API Gateway for routing

**Responsibilities:**
- SSL termination and security
- Rate limiting and DDoS protection
- Request routing and load distribution
- API versioning and documentation

### 2. Application Services

#### Upload Service
```go
type UploadService struct {
    objectStore   ObjectStore
    metadataDB    MetadataDB
    processor     ProcessingQueue
    deduplicator  Deduplicator
}

func (us *UploadService) UploadPhoto(ctx context.Context, photo *Photo) error {
    // 1. Validate and sanitize
    // 2. Check for duplicates
    // 3. Generate unique ID
    // 4. Upload to object store
    // 5. Store metadata
    // 6. Queue for processing
}
```

#### Read Service
```go
type ReadService struct {
    cache      Cache
    objectStore ObjectStore
    metadataDB  MetadataDB
    cdn        CDN
}

func (rs *ReadService) GetPhoto(ctx context.Context, id string, size ImageSize) (*Photo, error) {
    // 1. Check cache
    // 2. Query metadata
    // 3. Generate signed URL
    // 4. Return CDN link
}
```

#### Search Service
```go
type SearchService struct {
    searchIndex SearchIndex
    mlService   MLService
    cache      Cache
}

func (ss *SearchService) SearchPhotos(ctx context.Context, query *SearchQuery) (*SearchResults, error) {
    // 1. Parse query
    // 2. Search index
    // 3. Apply filters
    // 4. Rank results
}
```

### 3. Storage Layer

#### Object Storage Architecture
```
┌─────────────────────────────────────────┐
│               Object Storage            │
├─────────────────┬───────────────────────┤
│    Primary      │      Backup/Archive   │
│   (Hot Tier)    │     (Cold Tier)       │
├─────────────────┼───────────────────────┤
│   Amazon S3     │    Glacier/Deep       │
│   Multi-AZ      │    Archive            │
│   Replication   │    Cross-region       │
└─────────────────┴───────────────────────┘
```

**Storage Strategy:**
- **Hot Tier**: Frequently accessed photos (last 30 days)
- **Warm Tier**: Moderately accessed photos (30 days - 1 year)
- **Cold Tier**: Rarely accessed photos (>1 year)

#### Metadata Database Schema
```sql
-- Photos table
CREATE TABLE photos (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    filename VARCHAR(255) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(50) NOT NULL,
    width INTEGER NOT NULL,
    height INTEGER NOT NULL,
    taken_at TIMESTAMP,
    uploaded_at TIMESTAMP NOT NULL,
    storage_path VARCHAR(500) NOT NULL,
    checksum VARCHAR(64) NOT NULL,
    processing_status ENUM('pending', 'processing', 'completed', 'failed'),
    tags JSONB,
    metadata JSONB,
    INDEX idx_user_taken_at (user_id, taken_at),
    INDEX idx_checksum (checksum),
    INDEX idx_processing_status (processing_status)
);

-- Albums table
CREATE TABLE albums (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    cover_photo_id UUID,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    is_public BOOLEAN DEFAULT FALSE,
    INDEX idx_user_created_at (user_id, created_at)
);

-- Photo-Album mapping
CREATE TABLE album_photos (
    album_id UUID NOT NULL,
    photo_id UUID NOT NULL,
    added_at TIMESTAMP NOT NULL,
    PRIMARY KEY (album_id, photo_id),
    INDEX idx_photo_id (photo_id)
);
```

### 4. Processing Pipeline

#### Asynchronous Processing
```
Upload → Kafka → Worker Pool → ML/CV Processing → Index Update
    ↓
Metadata DB
    ↓
Object Store
```

**Processing Steps:**
1. **Image Validation**: Format, size, malware scanning
2. **Thumbnail Generation**: Multiple sizes (150px, 300px, 600px, 1200px)
3. **Metadata Extraction**: EXIF data, GPS coordinates, camera info
4. **ML Processing**: Object detection, face recognition, scene analysis
5. **Search Indexing**: Update Elasticsearch with extracted features

#### Worker Service
```go
type ProcessingWorker struct {
    kafka       KafkaConsumer
    imageProc   ImageProcessor
    mlService   MLService
    searchIndex SearchIndex
}

func (pw *ProcessingWorker) ProcessPhoto(ctx context.Context, photoID string) error {
    photo := pw.getPhoto(photoID)
    
    // Generate thumbnails
    thumbnails := pw.imageProc.GenerateThumbnails(photo)
    pw.uploadThumbnails(thumbnails)
    
    // Extract features
    features := pw.mlService.ExtractFeatures(photo)
    pw.updateMetadata(photoID, features)
    
    // Update search index
    pw.searchIndex.IndexPhoto(photoID, features)
    
    return nil
}
```

### 5. Caching Strategy

#### Multi-Level Caching
```
CDN (Edge) → Application Cache (Redis) → Database
     ↓
User Devices
```

**Cache Layers:**
1. **CDN Cache**: Static assets, thumbnails (TTL: 7 days)
2. **Application Cache**: Metadata, search results (TTL: 1 hour)
3. **Database Cache**: Query results (TTL: 15 minutes)

#### Cache Invalidation
```go
type CacheManager struct {
    redis    RedisCluster
    cdn      CDNService
}

func (cm *CacheManager) InvalidatePhoto(photoID string) error {
    // Invalidate metadata cache
    cm.redis.Del(fmt.Sprintf("photo:%s", photoID))
    
    // Invalidate CDN cache
    cm.cdn.PurgeCache([]string{
        fmt.Sprintf("/photos/%s/*", photoID),
    })
    
    return nil
}
```

### 6. Security & Privacy

#### Authentication & Authorization
- **OAuth 2.0** with JWT tokens
- **Role-Based Access Control** (RBAC)
- **API Rate Limiting** per user/IP
- **Request Signing** for critical operations

#### Data Protection
- **Encryption at Rest**: AES-256 for stored data
- **Encryption in Transit**: TLS 1.3 for all communications
- **Access Logging**: Comprehensive audit trails
- **Data Anonymization**: PII protection in logs

### 7. Monitoring & Observability

#### Metrics Collection
```go
type Metrics struct {
    uploadLatency    prometheus.Histogram
    downloadLatency  prometheus.Histogram
    storageUsage    prometheus.Gauge
    errorRate       prometheus.Counter
}

func (m *Metrics) RecordUpload(duration time.Duration, size int64) {
    m.uploadLatency.Observe(duration.Seconds())
    // Additional metrics...
}
```

#### Health Checks
```go
type HealthChecker struct {
    db          Database
    objectStore ObjectStore
    cache      Cache
}

func (hc *HealthChecker) CheckHealth() *HealthStatus {
    return &HealthStatus{
        Database:    hc.checkDatabase(),
        ObjectStore: hc.checkObjectStore(),
        Cache:      hc.checkCache(),
    }
}
```

## Scalability Patterns

### Horizontal Scaling
1. **Stateless Services**: All application services are stateless
2. **Database Sharding**: Shard by user_id for even distribution
3. **Load Balancing**: Round-robin with health checks
4. **Auto-scaling**: Kubernetes HPA based on CPU/memory

### Data Partitioning
```
User Shard = hash(user_id) % num_shards

Shard 0: Users 0, 3, 6, 9...
Shard 1: Users 1, 4, 7, 10...
Shard 2: Users 2, 5, 8, 11...
```

### Geographic Distribution
- **Multi-region deployment** for low latency
- **Regional data residency** for compliance
- **Cross-region replication** for disaster recovery

## Disaster Recovery

### Backup Strategy
- **Continuous backup** of metadata database
- **Cross-region replication** of object storage
- **Point-in-time recovery** capability
- **Automated failover** procedures

### Recovery Procedures
1. **RTO (Recovery Time Objective)**: 4 hours
2. **RPO (Recovery Point Objective)**: 15 minutes
3. **Automated monitoring** and alerting
4. **Disaster recovery testing** quarterly
