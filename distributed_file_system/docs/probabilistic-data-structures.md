# Probabilistic Data Structures in DFS

## Overview

Our distributed file system leverages multiple probabilistic data structures to achieve high performance and storage efficiency. These structures trade perfect accuracy for significant improvements in memory usage and query speed.

## 1. Bloom Filters - Deduplication Engine

### Purpose
Fast negative lookups for chunk existence in our global deduplication index.

### Implementation
```go
type DeduplicationBloomFilter struct {
    bitArray     []uint64      // Packed bit array
    size         uint64        // Total bits
    hashFuncs    []HashFunc    // K hash functions
    itemCount    uint64        // Estimated items added
    capacity     uint64        // Maximum items before rebuild
}

func (bf *DeduplicationBloomFilter) MightContain(chunkHash string) bool {
    for _, hashFunc := range bf.hashFuncs {
        bitIndex := hashFunc(chunkHash) % bf.size
        if !bf.getBit(bitIndex) {
            return false // Definitely not present
        }
    }
    return true // Might be present
}
```

### Performance Impact
- **Memory**: 1.2GB for 1B chunks (vs 100GB full index)
- **Speed**: 70% reduction in database queries
- **False Positive Rate**: 1% (tunable)

## 2. Count-Min Sketch - Access Frequency Tracking

### Purpose
Track chunk access frequency for intelligent data tiering without storing exact counts.

### Implementation
```go
type AccessFrequencyTracker struct {
    sketch    *CountMinSketch
    timeWindow time.Duration
    decay     float64
}

func (aft *AccessFrequencyTracker) RecordAccess(chunkID string, timestamp time.Time) {
    // Apply time decay to older accesses
    decayFactor := math.Exp(-float64(time.Since(timestamp)) / float64(aft.timeWindow))
    aft.sketch.Add(chunkID, uint64(decayFactor * 1000))
}

func (aft *AccessFrequencyTracker) GetTier(chunkID string) StorageTier {
    frequency := aft.sketch.EstimateFrequency(chunkID)
    switch {
    case frequency > 1000: return HotTier
    case frequency > 100:  return WarmTier
    default:               return ColdTier
    }
}
```

### Use Cases
- **Data Tiering**: Automatically move cold data to cheaper storage
- **Cache Eviction**: LFU cache replacement policy
- **Prefetching**: Predict which chunks to prefetch based on patterns

## 3. HyperLogLog - Cardinality Estimation

### Purpose
Estimate unique counts for analytics and capacity planning.

### Implementation
```go
type DFSAnalytics struct {
    uniqueUsers     *HyperLogLog  // Unique users per file
    uniqueFiles     *HyperLogLog  // Unique files per user
    uniqueChunks    *HyperLogLog  // Total unique chunks
    dailyActiveUsers *HyperLogLog  // DAU estimation
}

func (analytics *DFSAnalytics) EstimateUniqueUsers(fileID string) uint64 {
    return analytics.uniqueUsers.EstimateCardinality()
}

func (analytics *DFSAnalytics) EstimateStorageEfficiency() float64 {
    totalChunks := analytics.GetTotalChunkWrites()
    uniqueChunks := analytics.uniqueChunks.EstimateCardinality()
    return float64(uniqueChunks) / float64(totalChunks)
}
```

### Analytics Applications
```yaml
metrics:
  storage_efficiency:
    description: "Deduplication ratio estimation"
    formula: "unique_chunks / total_chunk_writes"
    accuracy: "±3% with 1KB memory"
    
  user_engagement:
    description: "Daily/Monthly active users"
    memory_usage: "1.5KB per metric"
    accuracy: "±2% for millions of users"
    
  file_popularity:
    description: "Unique users per file"
    use_case: "Caching decisions, viral content detection"
```

## 4. Cuckoo Filter - Recently Deleted Chunks

### Purpose
Track recently deleted chunks to prevent premature garbage collection.

### Implementation
```go
type GarbageCollector struct {
    recentlyDeleted *CuckooFilter
    retentionPeriod time.Duration
    cleanupTicker   *time.Ticker
}

func (gc *GarbageCollector) MarkForDeletion(chunkID string) {
    // Add to recently deleted filter
    gc.recentlyDeleted.Insert(chunkID)
    
    // Schedule actual deletion after retention period
    time.AfterFunc(gc.retentionPeriod, func() {
        gc.recentlyDeleted.Delete(chunkID)
        gc.performActualDeletion(chunkID)
    })
}

func (gc *GarbageCollector) CanSafelyDelete(chunkID string) bool {
    // Don't delete if recently marked for deletion
    return !gc.recentlyDeleted.MightContain(chunkID)
}
```

### Benefits
- **Prevents Data Loss**: Grace period for accidental deletions
- **Memory Efficient**: ~1 byte per recently deleted chunk
- **Supports Deletion**: Unlike Bloom filters

## 5. Skip Lists - Metadata Range Queries

### Purpose
Efficient range queries on file metadata (size, timestamps, etc.).

### Implementation
```go
type MetadataIndex struct {
    bySize      *SkipList  // Files sorted by size
    byTimestamp *SkipList  // Files sorted by creation time
    byAccess    *SkipList  // Files sorted by last access
}

func (mi *MetadataIndex) FindFilesBySize(minSize, maxSize uint64) []FileMetadata {
    var results []FileMetadata
    mi.bySize.RangeQuery(minSize, maxSize, func(node *SkipListNode) {
        results = append(results, node.Value.(FileMetadata))
    })
    return results
}

// Use case: Find large files for cleanup
func (mi *MetadataIndex) FindLargeOldFiles(sizeThreshold uint64, ageThreshold time.Duration) []FileMetadata {
    cutoffTime := time.Now().Add(-ageThreshold)
    candidates := mi.FindFilesBySize(sizeThreshold, math.MaxUint64)
    
    var oldLargeFiles []FileMetadata
    for _, file := range candidates {
        if file.CreatedAt.Before(cutoffTime) {
            oldLargeFiles = append(oldLargeFiles, file)
        }
    }
    return oldLargeFiles
}
```

## 6. MinHash - Content Similarity Detection

### Purpose
Detect similar files for advanced deduplication.

### Implementation
```go
type SimilarityDetector struct {
    minHashSize int
    shingleSize int
}

func (sd *SimilarityDetector) ComputeMinHash(content []byte) []uint64 {
    shingles := sd.generateShingles(content)
    minHashes := make([]uint64, sd.minHashSize)
    
    for i := range minHashes {
        minHashes[i] = math.MaxUint64
        hashFunc := sd.getHashFunction(i)
        
        for _, shingle := range shingles {
            hash := hashFunc(shingle)
            if hash < minHashes[i] {
                minHashes[i] = hash
            }
        }
    }
    return minHashes
}

func (sd *SimilarityDetector) EstimateSimilarity(hash1, hash2 []uint64) float64 {
    matches := 0
    for i := range hash1 {
        if hash1[i] == hash2[i] {
            matches++
        }
    }
    return float64(matches) / float64(len(hash1))
}
```

### Applications
- **Delta Compression**: Find similar files for differential storage
- **Duplicate Detection**: Find near-duplicate files
- **Content Recommendations**: Suggest similar content to users

## Performance Comparison

| Data Structure | Memory per Item | Query Time | Use Case |
|---------------|----------------|------------|----------|
| Bloom Filter | 10 bits | O(k) | Membership testing |
| Count-Min Sketch | 32 bits | O(k) | Frequency estimation |
| HyperLogLog | 0.0015 bits | O(1) | Cardinality estimation |
| Cuckoo Filter | 12-16 bits | O(1) | Membership + deletion |
| Skip List | 24-32 bytes | O(log n) | Range queries |
| MinHash | 64 bits × k | O(k) | Similarity estimation |

## Integration Architecture

```go
type ProbabilisticLayer struct {
    // Deduplication
    chunkBloomFilter    *BloomFilter
    
    // Analytics
    accessFrequency     *CountMinSketch
    uniqueUsers         *HyperLogLog
    uniqueChunks        *HyperLogLog
    
    // Garbage Collection
    recentlyDeleted     *CuckooFilter
    
    // Indexing
    metadataSkipLists   *MetadataIndex
    
    // Similarity
    contentSimilarity   *SimilarityDetector
}

func (pl *ProbabilisticLayer) ProcessUpload(file *File) (*ProcessResult, error) {
    chunks := pl.chunkFile(file)
    result := &ProcessResult{}
    
    for _, chunk := range chunks {
        // Check if chunk exists (Bloom filter)
        if !pl.chunkBloomFilter.MightContain(chunk.Hash) {
            // Definitely new chunk
            result.NewChunks = append(result.NewChunks, chunk)
            pl.chunkBloomFilter.Add(chunk.Hash)
            pl.uniqueChunks.Add(chunk.Hash)
        } else {
            // Maybe exists - check database
            exists, err := pl.database.ChunkExists(chunk.Hash)
            if err != nil {
                return nil, err
            }
            if exists {
                result.DeduplicatedChunks = append(result.DeduplicatedChunks, chunk)
            } else {
                // False positive
                result.NewChunks = append(result.NewChunks, chunk)
                pl.uniqueChunks.Add(chunk.Hash)
            }
        }
        
        // Track access frequency
        pl.accessFrequency.Add(chunk.Hash, 1)
    }
    
    // Track unique users
    pl.uniqueUsers.Add(file.UserID)
    
    return result, nil
}
```

## Monitoring and Tuning

### Key Metrics
```yaml
bloom_filter_metrics:
  false_positive_rate: 0.01  # Target 1%
  memory_usage_gb: 1.2
  query_rate_per_second: 1000000
  
count_min_sketch_metrics:
  error_rate: 0.001  # ε parameter
  confidence: 0.99   # δ parameter
  memory_usage_mb: 100
  
hyperloglog_metrics:
  standard_error: 0.02  # ±2% accuracy
  memory_usage_kb: 1.5
  
cuckoo_filter_metrics:
  load_factor: 0.95
  false_positive_rate: 0.001
  deletion_success_rate: 0.99
```

### Auto-Tuning
```go
type ProbabilisticTuner struct {
    monitor *MetricsCollector
}

func (pt *ProbabilisticTuner) TuneBloomFilter() {
    fpr := pt.monitor.GetBloomFilterFPR()
    if fpr > 0.02 {  // Too many false positives
        pt.increaseBloomFilterSize()
    }
    
    queryLoad := pt.monitor.GetDatabaseQueryLoad()
    if queryLoad > 0.8 {  // Database overloaded
        pt.decreaseBloomFilterFPR()
    }
}
```

These probabilistic data structures are essential for achieving our target of serving millions of concurrent users while maintaining storage efficiency. They enable us to make intelligent decisions about data placement, caching, and cleanup with minimal memory overhead.
