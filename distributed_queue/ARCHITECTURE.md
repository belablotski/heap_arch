# Distributed Message Queue Architecture Design

## 1. Problem Definition and Design Scope

### Functional Requirements
- **Producers**: Applications that send messages to topics
- **Consumers**: Applications that read messages from topics
- **Message Delivery Semantics**: Configurable (at-least-once, at-most-once, exactly-once)
- **Data Retention**: Configurable retention period (time-based and size-based)
- **Message Ordering**: Guaranteed within partitions
- **Repeatable Consumption**: Multiple consumers can read the same messages
- **Persistence**: Messages are durably stored

### Non-Functional Requirements
- **Throughput**: Support millions of messages per second
- **Latency**: Sub-millisecond to millisecond latency
- **Scalability**: Horizontal scaling of brokers and partitions
- **Availability**: 99.9%+ uptime with automatic failover
- **Reliability**: No message loss under normal operations

### Scale Estimates
- Message size: 1KB - 1MB (typical: 10KB)
- Producers: 1,000 - 10,000 concurrent
- Consumers: 1,000 - 10,000 concurrent
- Topics: 1,000 - 10,000
- Partitions per topic: 10 - 1,000
- Message throughput: 1M - 10M messages/second

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Message Queue Cluster                       │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Producer   │  │  Producer   │  │  Producer   │              │
│  │    App      │  │    App      │  │    App      │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│         │                 │                 │                   │
│         └─────────────────┼─────────────────┘                   │
│                           │                                     │
│  ┌────────────────────────┼─────────────────────────────────┐   │
│  │                    Load Balancer                         │   │
│  └────────────────────────┼─────────────────────────────────┘   │
│                           │                                     │
│  ┌────────────────────────┼─────────────────────────────────┐   │
│  │                   Broker Cluster                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │   │
│  │  │   Broker    │  │   Broker    │  │   Broker    │       │   │
│  │  │      1      │  │      2      │  │      3      │       │   │
│  │  │             │  │             │  │             │       │   │
│  │  │ Partitions  │  │ Partitions  │  │ Partitions  │       │   │
│  │  │ [1,4,7]     │  │ [2,5,8]     │  │ [3,6,9]     │       │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │   │
│  └─────────────────────────┼────────────────────────────────┘   │
│                            │                                    │
│  ┌─────────────────────────┼────────────────────────────────┐   │
│  │                  Coordination Service                    │   │
│  │            (Zookeeper/etcd/Consul)                       │   │
│  └─────────────────────────┼────────────────────────────────┘   │
│                            │                                    │
│  ┌─────────────────────────┼────────────────────────────────┐   │
│  │                   State Storage                          │   │
│  │              (Consumer Offsets)                          │   │
│  └─────────────────────────┼────────────────────────────────┘   │
│                            │                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  Consumer   │  │  Consumer   │  │  Consumer   │              │
│  │   Group A   │  │   Group B   │  │   Group C   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

### Core Components

#### 2.1 Broker
- **Responsibility**: Store and serve messages
- **Partitions**: Each broker hosts multiple partitions
- **Replication**: Maintains replicas for fault tolerance
- **Controller**: One broker acts as cluster controller

#### 2.2 Topic and Partitions
- **Topic**: Logical message category
- **Partition**: Ordered, immutable sequence of messages
- **Offset**: Unique identifier for message position in partition
- **Message Key**: Optional key for partition assignment

#### 2.3 Producer
- **Message Routing**: Determines target partition
- **Batching**: Groups messages for efficiency
- **Acknowledgment**: Configurable durability guarantees

#### 2.4 Consumer Groups
- **Load Balancing**: Partitions distributed among consumers
- **Offset Management**: Tracks consumption progress
- **Rebalancing**: Automatic partition reassignment

#### 2.5 Coordination Service
- **Broker Discovery**: Maintains cluster membership
- **Leader Election**: Controller selection
- **Metadata Management**: Topics, partitions, replicas

## 3. Messaging Patterns

### 3.1 Point-to-Point
- Single consumer per message
- Message deleted after consumption
- Use case: Task queues, work distribution

### 3.2 Publish-Subscribe
- Multiple consumers per message
- Message retention for replay
- Use case: Event streaming, notifications

## 4. Data Flow

### 4.1 Producer Flow
```
Producer → [Partition Router] → [Batch Buffer] → Broker → [Replication] → ACK
```

1. **Message Creation**: Producer creates message with optional key
2. **Partition Selection**: Router determines target partition
3. **Batching**: Messages buffered for batch sending
4. **Network Send**: Batch sent to partition leader
5. **Replication**: Leader replicates to followers
6. **Acknowledgment**: Configurable ACK level (0, 1, all)

### 4.2 Consumer Flow
```
Consumer → [Group Coordinator] → [Partition Assignment] → [Fetch Request] → [Offset Commit]
```

1. **Group Join**: Consumer joins consumer group
2. **Partition Assignment**: Coordinator assigns partitions
3. **Message Fetch**: Consumer fetches messages from partition leaders
4. **Processing**: Consumer processes messages
5. **Offset Commit**: Consumer commits new offset to state storage

## 5. Consistency and Replication

### 5.1 Replication Strategy
- **Leader-Follower**: One leader per partition, multiple followers
- **In-Sync Replicas (ISR)**: Replicas that are caught up with leader
- **Replication Factor**: Configurable (typically 3)

### 5.2 Consistency Levels
- **ACK=0**: No acknowledgment (fire and forget)
- **ACK=1**: Leader acknowledgment only
- **ACK=all**: All ISR acknowledgment (strongest durability)

### 5.3 Failure Handling
- **Broker Failure**: Automatic leader election for affected partitions
- **Network Partitions**: Split-brain prevention through majority consensus
- **Data Corruption**: Checksum validation and repair

## 6. Storage Architecture

### 6.1 Log-Structured Storage
- **Segment Files**: Large immutable files (1GB default)
- **Active Segment**: Current write target
- **Index Files**: Sparse index for fast seeks
- **Time-based Indexing**: For time-range queries

### 6.2 Retention Policies
- **Time-based**: Delete segments older than retention period
- **Size-based**: Delete oldest segments when size limit exceeded
- **Compaction**: Keep only latest value per key (for compacted topics)

## 7. Performance Optimizations

### 7.1 Producer Optimizations
- **Batching**: Reduce network overhead
- **Compression**: Reduce network and storage
- **Async Send**: Non-blocking message publishing

### 7.2 Broker Optimizations
- **Sequential I/O**: Append-only writes, sequential reads
- **Zero-Copy**: Direct memory-to-network transfer
- **Page Cache**: OS-level caching for reads

### 7.3 Consumer Optimizations
- **Fetch Batching**: Request multiple messages per call
- **Prefetching**: Background message fetching
- **Parallel Processing**: Multi-threaded message processing

## 8. Monitoring and Observability

### 8.1 Key Metrics
- **Throughput**: Messages/second per topic/partition
- **Latency**: End-to-end message latency
- **Consumer Lag**: Offset difference between production and consumption
- **Broker Health**: CPU, memory, disk usage
- **Replication Lag**: Delay between leader and followers

### 8.2 Alerting
- **High Consumer Lag**: Consumers falling behind
- **Broker Down**: Broker unavailability
- **Disk Full**: Storage capacity alerts
- **Replication Issues**: ISR shrinkage

## 9. Security

### 9.1 Authentication
- **SASL**: Simple Authentication and Security Layer
- **OAuth**: Token-based authentication
- **Certificate-based**: mTLS authentication

### 9.2 Authorization
- **ACLs**: Topic-level access control
- **Role-based**: User role management
- **Encryption**: TLS for data in transit, encryption at rest

## 10. Deployment Considerations

### 10.1 Hardware Requirements
- **CPU**: High-frequency cores for low latency
- **Memory**: Large page cache for read performance
- **Storage**: Fast SSDs for active segments, HDDs for retention
- **Network**: High bandwidth, low latency

### 10.2 Cluster Sizing
- **Broker Count**: Based on throughput and fault tolerance
- **Partition Count**: Balance parallelism vs overhead
- **Replication Factor**: Typically 3 for production

### 10.3 Operations
- **Rolling Updates**: Zero-downtime deployments
- **Backup and Recovery**: Point-in-time recovery
- **Capacity Planning**: Growth projections and scaling
