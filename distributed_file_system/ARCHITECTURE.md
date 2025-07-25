# Distributed File System Architecture

## System Overview

The distributed file system is built on a microservices architecture optimized for storage efficiency, high availability, and scalability. The system separates concerns into distinct layers: storage, metadata, coordination, and client interface.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                            │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Web Clients   │  Mobile Apps    │     API Clients             │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                      Gateway Layer                              │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Load Balancer  │  API Gateway    │   Session Manager           │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                    Service Layer                                │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ Metadata Service│ Dedup Service   │   Compression Service       │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                     Storage Layer                               │
├─────────────────┬─────────────────┬─────────────────────────────┤
│  Chunk Servers  │   Index Store   │      Cold Storage           │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

| Component            | Purpose / Function                                                                                   |
|----------------------|-----------------------------------------------------------------------------------------------------|
| **Client Layer**     | Interfaces for users and applications to interact with the file system (web, mobile, API clients).  |
| **Gateway Layer**    | Entry point for requests; handles load balancing, API routing, and session management.              |
| Load Balancer        | Distributes incoming traffic, provides failover, SSL, and DDoS protection.                          |
| API Gateway          | Routes requests, handles authentication, protocol translation, and resilience features.              |
| Session Manager      | Tracks client sessions, enforces consistency, manages connection state.                             |
| **Service Layer**    | Core logic for metadata, deduplication, and compression.                                            |
| Metadata Service     | Manages file system structure, metadata, chunk mapping, and consistency logs.                       |
| Deduplication Service| Identifies and eliminates duplicate data chunks to save storage.                                    |
| Compression Service  | Compresses data using adaptive algorithms based on data type and access patterns.                   |
| **Storage Layer**    | Stores actual file data and indexes, manages data durability and tiering.                           |
| Chunk Servers        | Store file chunks, handle replication, caching, and compaction.                                     |
| Index Store          | Maintains indexes for fast lookup of files, chunks, and metadata.                                   |
| Cold Storage         | Archives infrequently accessed data to cost-effective storage (e.g., S3/Glacier).                   |

## Core Architecture Components

### 1. Gateway Layer

#### Load Balancer
- **Technology**: HAProxy with keepalived for HA
- **Features**: 
  - Health checking and automatic failover
  - Geographic routing for optimal performance
  - SSL termination and connection pooling
  - Rate limiting and DDoS protection

#### API Gateway
- **Technology**: Envoy Proxy with custom plugins
- **Responsibilities**:
  - Request routing and protocol translation
  - Authentication and authorization
  - Request/response transformation
  - Circuit breaking and retry logic

#### Session Manager
- **Technology**: Redis Cluster with Sentinel
- **Functions**:
  - Client session tracking and affinity
  - Read-after-write consistency enforcement
  - Client preference caching
  - Connection state management

### 2. Service Layer

#### Metadata Service
- **Architecture**: Sharded master-slave clusters
- **Data Store**: etcd for coordination + PostgreSQL for metadata
- **Scaling Strategy**: Consistent hashing with virtual nodes

**Key Features**:
- Namespace management and file system tree structure
- File metadata (size, timestamps, permissions, checksums)
- Chunk location mapping and placement policies
- Transaction log for consistency guarantees

**Sharding Strategy**:
```
Hash(file_path) % num_shards → metadata_shard_id
```

```mermaid
flowchart TD
    A[API Request: File Operation] --> B[Hash file_path to find shard]
    B --> C[Route to Metadata Shard]
    C --> D[Read/Write Metadata in PostgreSQL]
    D --> E[Update etcd Coordination]
    E --> F[Update Transaction Log]
    F --> G[Respond to Client / Next Service]

    style A fill:#e3f2fd,stroke:#2196f3,stroke-width:2px
    style D fill:#f8bbd0,stroke:#c2185b,stroke-width:2px
    style F fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
```

#### Metadata Database Overview

The metadata database is the backbone of the file system's namespace and coordination. It is deployed as a sharded PostgreSQL cluster, where each shard consists of a primary server and two read replicas for high availability and read scaling. Failover is managed automatically using Patroni or similar orchestration tools.

- **Primary Server**: Handles all writes and strong consistency operations.
- **Read Replicas**: Serve read-only queries, offloading traffic from the primary.
- **Sharding**: Each file path is mapped to a shard using consistent hashing.
- **Coordination**: etcd is used for distributed coordination, leader election, and configuration management.

**Deployment Example:**
```
Shard 1: pg-primary-1 (RW) + pg-replica-1a (RO) + pg-replica-1b (RO)
Shard 2: pg-primary-2 (RW) + pg-replica-2a (RO) + pg-replica-2b (RO)
...
```

#### Metadata Entity-Relation Diagram

```mermaid
erDiagram
    FILES {
        uuid id PK
        text name
        uuid parent_id FK
        bigint size
        timestamp created_at
        timestamp updated_at
        text permissions
        text checksum
    }
    CHUNKS {
        uuid id PK
        uuid file_id FK
        int chunk_index
        text chunk_hash
        bigint size
        text storage_location
    }
    USERS {
        uuid id PK
        text username
        text email
        text role
    }
    FILES ||--o{ CHUNKS : contains
    USERS ||--o{ FILES : owns
```

#### Deduplication Service
- **Engine**: Content-Defined Chunking (CDC) with rolling hash
- **Hash Algorithm**: SHA-256 for chunk fingerprints
- **Index**: Distributed hash table for chunk → location mapping

**Deduplication Pipeline**:
1. **Chunking**: Variable-size chunks (avg 64KB, range 32KB-128KB)
2. **Fingerprinting**: SHA-256 hash of chunk content
3. **Lookup**: Check global dedup index for existing chunks
4. **Storage**: Store only unique chunks, reference existing ones

#### Compression Service
- **Multi-tier Strategy**: 
  - Level 1: LZ4 for hot data (fast compression/decompression)
  - Level 2: ZSTD for warm data (balanced)
  - Level 3: LZMA for cold data (maximum compression)

**Adaptive Compression**:
- Machine learning models predict optimal compression algorithm
- Based on file type, access patterns, and compression ratios
- Automatic recompression during data tiering

### 3. Storage Layer

#### Chunk Servers
- **Technology**: Custom storage engine built in Rust
- **Storage Format**: Log-structured merge trees (LSM trees)
- **Replication**: Erasure coding (Reed-Solomon) with configurable redundancy

**Storage Architecture**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    Chunk Server Node                            │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Write Cache   │   Read Cache    │     Compaction Engine       │
├─────────────────┼─────────────────┼─────────────────────────────┤
│   LSM Tree L0   │   LSM Tree L1   │     LSM Tree L2+            │
├─────────────────┼─────────────────┼─────────────────────────────┤
│     SSD Tier    │    HDD Tier     │      Cold Storage           │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

```mermaid
flowchart TD
    A[Client Upload] --> B[API Gateway]
    B --> C[Metadata Service]
    C --> D[Deduplication Service]
    D --> E[Compression Service]
    E --> F[Chunk Servers]
    F --> G[Storage - SSD/HDD/Cold]
    G --> H[Metadata Update]
    H --> I[Indexes]

    style A fill:#e3f2fd,stroke:#2196f3,stroke-width:2px
    style F fill:#fff3e0,stroke:#fb8c00,stroke-width:2px
    style G fill:#f1f8e9,stroke:#43a047,stroke-width:2px
```

#### Index Store
- **Primary Index**: B+ trees for file path → metadata mapping
- **Secondary Indexes**:
  - Content hash → chunk locations
  - Access time → data tiering decisions
  - File size → storage optimization strategies

#### Cold Storage
- **Technology**: S3-compatible object storage with Glacier integration
- **Criteria**: Files not accessed for 90+ days
- **Process**: Transparent migration with metadata updates

## Data Flow Architecture

### Write Path
```
Client → Gateway → Metadata Service → Dedup Service → Compression Service → Chunk Servers
```

1. **Client Upload**: File uploaded through API gateway
2. **Chunking**: File split into variable-size chunks
3. **Deduplication**: Check if chunks already exist
4. **Compression**: Apply appropriate compression algorithm
5. **Placement**: Store chunks across multiple nodes with erasure coding
6. **Metadata Update**: Update file system metadata and indexes

### Read Path
```
Client → Gateway → Session Cache → Metadata Service → Chunk Servers → Client
```

1. **Request**: Client requests file through API
2. **Session Check**: Verify read-after-write consistency requirements
3. **Metadata Lookup**: Get chunk locations from metadata service
4. **Chunk Retrieval**: Fetch chunks from storage nodes
5. **Assembly**: Decompress and reassemble file
6. **Delivery**: Stream result to client

## Consistency Model

### Read-After-Write Consistency
- **Session Affinity**: Client requests routed to same gateway node
- **Write Tracking**: Track client writes in session store
- **Read Verification**: Ensure reads see all previous writes from same client

### Eventual Consistency Implementation
- **Vector Clocks**: Track causality between operations
- **Conflict Resolution**: Last-writer-wins with timestamp ordering
- **Repair Process**: Background anti-entropy process for consistency

### Consistency Levels
```go
type ConsistencyLevel int

const (
    Eventual    ConsistencyLevel = iota  // Best performance
    ReadOwn                              // Read-after-write for same client
    Strong                               // Full consistency (optional, slow)
)
```

## High Availability Design

### Failure Domains
- **Rack-level**: Protect against rack failures
- **Datacenter-level**: Multi-AZ deployment
- **Region-level**: Cross-region replication for disaster recovery

### Replication Strategy
- **Metadata**: 3-way synchronous replication
- **Data**: Erasure coding (6+3) for 99.9999% durability
- **Hot Data**: Additional replication for performance

### Failover Mechanisms
- **Automatic Detection**: Health checks every 30 seconds
- **Fast Failover**: Sub-minute recovery for gateway failures
- **Data Recovery**: Automatic rebuilding of failed storage nodes

## Performance Optimizations

### Caching Strategy
- **L1 Cache**: In-memory cache at gateway layer (1GB per node)
- **L2 Cache**: Distributed Redis cache (100GB cluster-wide)
- **L3 Cache**: SSD-based cache at storage nodes (1TB per node)

### Data Placement Optimization
- **Geographic Proximity**: Place data near requesting clients
- **Load Balancing**: Distribute load across storage nodes
- **Performance Tiers**: Hot/warm/cold data classification

### Network Optimization
- **Connection Pooling**: Reuse connections between services
- **Compression**: Compress inter-service communication
- **Batching**: Batch small operations for better throughput

## Monitoring and Observability

### Metrics Collection
- **System Metrics**: CPU, memory, disk, network utilization
- **Application Metrics**: Request rates, latency, error rates
- **Business Metrics**: Storage efficiency, deduplication ratios

### Distributed Tracing
- **Technology**: Jaeger for request tracing
- **Sampling**: Adaptive sampling based on request importance
- **Correlation**: Trace requests across all microservices

### Alerting
- **SLA Monitoring**: Track against performance targets
- **Anomaly Detection**: ML-based detection of unusual patterns
- **Escalation**: Automated escalation for critical issues
