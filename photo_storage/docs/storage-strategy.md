# Storage Strategy for Photo Storage System

## Overview

The storage strategy is designed to handle petabytes of photo data with high durability, availability, and cost efficiency. The system uses a tiered storage approach with intelligent data lifecycle management.

## Storage Tiers

### Hot Tier (Immediate Access)
**Characteristics:**
- Sub-second access latency
- High availability (99.99%)
- Recently uploaded photos (last 30 days)
- Frequently accessed content

**Technology:**
- Amazon S3 Standard
- Google Cloud Storage Standard
- 3-way replication across availability zones

**Cost:** ~$0.023/GB/month

### Warm Tier (Frequent Access)
**Characteristics:**
- 1-5 second access latency
- Photos accessed within 30 days - 1 year
- Automatic transition from hot tier

**Technology:**
- Amazon S3 Intelligent-Tiering
- Google Cloud Storage Nearline
- 2-way replication

**Cost:** ~$0.0125/GB/month

### Cold Tier (Infrequent Access)
**Characteristics:**
- Minutes to hours retrieval time
- Photos older than 1 year
- Long-term archival storage

**Technology:**
- Amazon S3 Glacier Flexible Retrieval
- Google Cloud Storage Coldline
- Single region storage with backup

**Cost:** ~$0.004/GB/month

### Archive Tier (Backup & Compliance)
**Characteristics:**
- Hours to days retrieval time
- Compliance and disaster recovery
- Photos older than 7 years

**Technology:**
- Amazon S3 Glacier Deep Archive
- Google Cloud Storage Archive
- Cross-region backup

**Cost:** ~$0.00099/GB/month

## Object Storage Design

### Object Naming Convention
```
/photos/{year}/{month}/{day}/{user_id}/{photo_id}.{ext}
/thumbnails/{size}/{photo_id}.webp
/metadata/{photo_id}.json
```

Example:
```
/photos/2025/07/10/user-123e4567-e89b-12d3-a456-426614174000/photo-987fcdeb-51a2-43d7-8e5f-123456789abc.jpg
/thumbnails/150x150/photo-987fcdeb-51a2-43d7-8e5f-123456789abc.webp
/thumbnails/300x300/photo-987fcdeb-51a2-43d7-8e5f-123456789abc.webp
/thumbnails/600x600/photo-987fcdeb-51a2-43d7-8e5f-123456789abc.webp
/thumbnails/1200x1200/photo-987fcdeb-51a2-43d7-8e5f-123456789abc.webp
```

### Storage Configuration

#### S3 Bucket Configuration
```json
{
  "bucket": "photos-primary",
  "region": "us-west-2",
  "versioning": {
    "status": "Enabled"
  },
  "lifecycleConfiguration": {
    "rules": [
      {
        "id": "PhotoLifecycle",
        "status": "Enabled",
        "transitions": [
          {
            "days": 30,
            "storageClass": "STANDARD_IA"
          },
          {
            "days": 90,
            "storageClass": "GLACIER"
          },
          {
            "days": 365,
            "storageClass": "DEEP_ARCHIVE"
          }
        ]
      }
    ]
  },
  "replication": {
    "role": "arn:aws:iam::account:role/replication-role",
    "rules": [
      {
        "id": "CrossRegionReplication",
        "status": "Enabled",
        "destination": {
          "bucket": "photos-backup-eu-west-1",
          "storageClass": "STANDARD_IA"
        }
      }
    ]
  }
}
```

## Deduplication Strategy

### Content-Based Deduplication

#### Hash-Based Detection
```go
type Deduplicator struct {
    hashIndex map[string]string // checksum -> photo_id
    db        MetadataDB
}

func (d *Deduplicator) CheckDuplicate(photo *Photo) (bool, string, error) {
    // Calculate perceptual hash
    pHash := d.calculatePerceptualHash(photo.Content)
    
    // Check exact match
    if existingID, exists := d.hashIndex[pHash]; exists {
        return true, existingID, nil
    }
    
    // Check similar images (Hamming distance < 5)
    similar := d.findSimilarHashes(pHash, 5)
    if len(similar) > 0 {
        return true, similar[0], nil
    }
    
    return false, "", nil
}

func (d *Deduplicator) calculatePerceptualHash(content []byte) string {
    // Implementation using pHash algorithm
    // Returns 64-bit hash as hex string
}
```

#### Benefits
- **Storage Savings**: 30-40% reduction in storage costs
- **Upload Speed**: Skip duplicate uploads
- **Bandwidth Savings**: Reduced network transfer

### Implementation Details

#### Deduplication Table Schema
```sql
CREATE TABLE photo_deduplication (
    content_hash VARCHAR(64) PRIMARY KEY,
    perceptual_hash VARCHAR(16) NOT NULL,
    photo_id UUID NOT NULL,
    file_size BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL,
    reference_count INTEGER DEFAULT 1,
    INDEX idx_perceptual_hash (perceptual_hash),
    INDEX idx_photo_id (photo_id)
);
```

#### Reference Counting
```go
func (d *Deduplicator) AddReference(contentHash string, userID string) error {
    tx := d.db.Begin()
    defer tx.Rollback()
    
    // Increment reference count
    tx.Exec("UPDATE photo_deduplication SET reference_count = reference_count + 1 WHERE content_hash = ?", contentHash)
    
    // Create user photo record
    tx.Exec("INSERT INTO user_photos (user_id, photo_id, content_hash) VALUES (?, ?, ?)", userID, photoID, contentHash)
    
    return tx.Commit()
}
```

## Compression and Optimization

### Image Compression Pipeline

#### Format Optimization
```go
type ImageOptimizer struct {
    compressor ImageCompressor
    converter  FormatConverter
}

func (io *ImageOptimizer) OptimizeImage(original *Image) (*OptimizedImage, error) {
    result := &OptimizedImage{}
    
    // Convert HEIC to JPEG for compatibility
    if original.Format == "HEIC" {
        converted := io.converter.HEICToJPEG(original, 85)
        result.Formats["jpeg"] = converted
    }
    
    // Generate WebP for modern browsers
    webp := io.converter.ToWebP(original, 80)
    result.Formats["webp"] = webp
    
    // Generate AVIF for maximum compression
    avif := io.converter.ToAVIF(original, 75)
    result.Formats["avif"] = avif
    
    return result, nil
}
```

#### Compression Settings
- **JPEG**: Quality 85 for originals, 70 for thumbnails
- **WebP**: Quality 80 with lossless alpha
- **AVIF**: Quality 75 for maximum compression
- **PNG**: Lossless compression for graphics

### Thumbnail Generation

#### Multiple Resolution Strategy
```go
type ThumbnailGenerator struct {
    sizes []ThumbnailSize
}

type ThumbnailSize struct {
    Width   int
    Height  int
    Quality int
    Format  string
}

var StandardSizes = []ThumbnailSize{
    {150, 150, 70, "webp"},   // Small thumbnail
    {300, 300, 75, "webp"},   // Medium thumbnail  
    {600, 600, 80, "webp"},   // Large thumbnail
    {1200, 1200, 85, "webp"}, // XL thumbnail
}

func (tg *ThumbnailGenerator) GenerateAll(photo *Photo) ([]*Thumbnail, error) {
    var thumbnails []*Thumbnail
    
    for _, size := range tg.sizes {
        thumb, err := tg.generateThumbnail(photo, size)
        if err != nil {
            return nil, err
        }
        thumbnails = append(thumbnails, thumb)
    }
    
    return thumbnails, nil
}
```

## Data Lifecycle Management

### Automated Transitions

#### Lifecycle Rules
```yaml
lifecycle_rules:
  - name: "standard_to_ia"
    transition_days: 30
    from_class: "STANDARD"
    to_class: "STANDARD_IA"
    
  - name: "ia_to_glacier"
    transition_days: 90
    from_class: "STANDARD_IA"  
    to_class: "GLACIER"
    
  - name: "glacier_to_deep_archive"
    transition_days: 365
    from_class: "GLACIER"
    to_class: "DEEP_ARCHIVE"
```

#### Access Pattern Analysis
```go
type AccessAnalyzer struct {
    analytics AnalyticsDB
    lifecycle LifecycleManager
}

func (aa *AccessAnalyzer) AnalyzeAccessPatterns() error {
    // Query access logs
    patterns := aa.analytics.GetAccessPatterns(30) // Last 30 days
    
    for _, pattern := range patterns {
        if pattern.AccessCount == 0 && pattern.DaysSinceLastAccess > 30 {
            // Move to colder tier
            aa.lifecycle.TransitionToColdTier(pattern.PhotoID)
        }
    }
    
    return nil
}
```

### Intelligent Tiering

#### ML-Based Predictions
```python
import pandas as pd
from sklearn.ensemble import RandomForestClassifier

class AccessPredictor:
    def __init__(self):
        self.model = RandomForestClassifier()
        
    def predict_access_probability(self, photo_metadata):
        """Predict probability of photo being accessed in next 30 days"""
        features = self.extract_features(photo_metadata)
        return self.model.predict_proba([features])[0][1]
    
    def extract_features(self, metadata):
        return [
            metadata['days_since_upload'],
            metadata['total_views'],
            metadata['views_last_week'],
            metadata['is_in_album'],
            metadata['has_faces'],
            metadata['file_size'],
            metadata['user_activity_score']
        ]
```

## Cost Optimization

### Storage Cost Analysis

#### Monthly Cost Breakdown (1PB total)
```
Hot Tier (100TB):     100TB × $0.023 = $2,300
Warm Tier (300TB):    300TB × $0.0125 = $3,750  
Cold Tier (500TB):    500TB × $0.004 = $2,000
Archive Tier (100TB): 100TB × $0.00099 = $99

Total Monthly Cost: $8,149
Average Cost per GB: $0.008149
```

#### Cost Optimization Strategies
1. **Aggressive Lifecycle Management**: Move data to colder tiers faster
2. **Compression**: Use modern formats (AVIF, WebP) for 30-50% savings
3. **Deduplication**: Eliminate duplicate storage
4. **Regional Optimization**: Store data closer to users
5. **Reserved Capacity**: Purchase reserved storage for predictable workloads

### Monitoring and Alerting

#### Storage Metrics
```go
type StorageMetrics struct {
    TotalSize        int64
    HotTierSize      int64
    WarmTierSize     int64
    ColdTierSize     int64
    ArchiveTierSize  int64
    DuplicationRatio float64
    CostPerGB        float64
}

func (sm *StorageMetrics) CalculateCostEfficiency() float64 {
    return sm.TotalSize / (sm.HotTierSize*0.023 + sm.WarmTierSize*0.0125 + 
                          sm.ColdTierSize*0.004 + sm.ArchiveTierSize*0.00099)
}
```

#### Alerts
- Storage tier distribution imbalance
- Unusual cost increases
- Failed lifecycle transitions
- Replication failures
- Storage capacity thresholds

## Backup and Disaster Recovery

### Cross-Region Replication
```yaml
replication_config:
  primary_region: "us-west-2"
  backup_regions:
    - "us-east-1"
    - "eu-west-1"
  replication_rule:
    prefix: "photos/"
    status: "Enabled"
    destination_storage_class: "STANDARD_IA"
```

### Recovery Procedures
1. **Point-in-Time Recovery**: Restore from versioned backups
2. **Cross-Region Failover**: Automatic DNS failover to backup region
3. **Data Integrity Checks**: Regular checksum verification
4. **Disaster Recovery Testing**: Monthly DR drills

This storage strategy provides a robust, scalable, and cost-effective solution for storing billions of photos while maintaining high performance and durability.
