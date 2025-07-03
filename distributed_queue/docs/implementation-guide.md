# Distributed Message Queue Implementation Guide

## Overview

This document provides detailed implementation guidance for building the distributed message queue system described in the architecture document.

## Directory Structure

```
message-queue/
├── broker/                     # Broker implementation
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/mq/broker/
│   │   │   │   ├── BrokerServer.java
│   │   │   │   ├── PartitionManager.java
│   │   │   │   ├── ReplicationManager.java
│   │   │   │   ├── storage/
│   │   │   │   │   ├── LogSegment.java
│   │   │   │   │   ├── IndexFile.java
│   │   │   │   │   └── OffsetManager.java
│   │   │   │   ├── network/
│   │   │   │   │   ├── RequestHandler.java
│   │   │   │   │   └── NetworkServer.java
│   │   │   │   └── coordination/
│   │   │   │       ├── Controller.java
│   │   │   │       └── ZookeeperClient.java
│   │   │   └── resources/
│   │   │       └── broker.properties
│   │   └── test/
│   ├── pom.xml
│   └── Dockerfile
├── client/                     # Client library
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/mq/client/
│   │   │   │   ├── producer/
│   │   │   │   │   ├── MessageProducer.java
│   │   │   │   │   ├── PartitionRouter.java
│   │   │   │   │   └── BatchManager.java
│   │   │   │   ├── consumer/
│   │   │   │   │   ├── MessageConsumer.java
│   │   │   │   │   ├── ConsumerGroup.java
│   │   │   │   │   └── OffsetCommitter.java
│   │   │   │   ├── common/
│   │   │   │   │   ├── Message.java
│   │   │   │   │   ├── TopicPartition.java
│   │   │   │   │   └── Configuration.java
│   │   │   │   └── network/
│   │   │   │       ├── BrokerConnection.java
│   │   │   │       └── RequestResponse.java
│   │   │   └── resources/
│   │   └── test/
│   ├── pom.xml
│   └── README.md
├── coordination/               # Coordination service
│   ├── zookeeper/
│   └── etcd/
├── monitoring/                 # Monitoring and metrics
│   ├── prometheus/
│   ├── grafana/
│   └── elk/
├── deployment/                 # Deployment configurations
│   ├── kubernetes/
│   ├── docker-compose/
│   └── terraform/
├── scripts/                    # Utility scripts
│   ├── setup.sh
│   ├── cluster-start.sh
│   └── performance-test.sh
├── docs/                       # Documentation
│   ├── api/
│   ├── operations/
│   └── examples/
└── README.md
```

## Core Components Implementation

### 1. Message Format

```java
public class Message {
    private final byte[] key;           // Optional partition key
    private final byte[] value;         // Message payload
    private final Map<String, String> headers; // Metadata
    private final long timestamp;       // Message timestamp
    private final long offset;          // Partition offset
    
    // Serialization methods
    public byte[] serialize() { /* ... */ }
    public static Message deserialize(byte[] data) { /* ... */ }
}
```

### 2. Broker Storage Implementation

#### Log Segment Structure
```java
public class LogSegment {
    private final File logFile;         // Message data
    private final File indexFile;       // Offset index
    private final File timeIndexFile;   // Time-based index
    private final long baseOffset;      // First offset in segment
    private final AtomicLong nextOffset; // Next write offset
    
    public void append(Message message) {
        // 1. Serialize message
        // 2. Write to log file
        // 3. Update index
        // 4. Fsync if required
    }
    
    public List<Message> read(long startOffset, int maxBytes) {
        // 1. Find position using index
        // 2. Read messages from log file
        // 3. Deserialize and return
    }
}
```

#### Partition Manager
```java
public class PartitionManager {
    private final Map<TopicPartition, LogManager> partitions;
    private final ReplicationManager replicationManager;
    
    public void handleProduceRequest(ProduceRequest request) {
        for (TopicPartition tp : request.getPartitions()) {
            LogManager log = partitions.get(tp);
            List<Message> messages = request.getMessages(tp);
            
            // Append to local log
            List<Long> offsets = log.append(messages);
            
            // Replicate to followers
            replicationManager.replicate(tp, messages);
            
            // Send acknowledgment based on ACK setting
            sendAck(request, offsets);
        }
    }
    
    public void handleFetchRequest(FetchRequest request) {
        FetchResponse response = new FetchResponse();
        
        for (TopicPartition tp : request.getPartitions()) {
            LogManager log = partitions.get(tp);
            long offset = request.getOffset(tp);
            int maxBytes = request.getMaxBytes(tp);
            
            List<Message> messages = log.read(offset, maxBytes);
            response.addMessages(tp, messages);
        }
        
        sendResponse(response);
    }
}
```

### 3. Producer Implementation

#### Message Producer
```java
public class MessageProducer {
    private final PartitionRouter router;
    private final BatchManager batchManager;
    private final NetworkClient networkClient;
    
    public Future<RecordMetadata> send(ProducerRecord record) {
        // 1. Determine target partition
        int partition = router.getPartition(record);
        TopicPartition tp = new TopicPartition(record.topic(), partition);
        
        // 2. Add to batch
        FutureRecordMetadata future = new FutureRecordMetadata();
        batchManager.addToBatch(tp, record, future);
        
        return future;
    }
    
    public void flush() {
        batchManager.flushAll();
    }
}
```

#### Partition Router
The **Partition Router** is a critical component in the producer flow that determines which partition a message should be sent to. It's the first step in the producer pipeline and directly impacts message distribution, ordering, and performance.

```java
public interface PartitionRouter {
    /**
     * Determines the target partition for a given producer record.
     * 
     * @param record The producer record to route
     * @param cluster Current cluster metadata
     * @return The partition number (0-based index)
     */
    int getPartition(ProducerRecord record, ClusterMetadata cluster);
}

public class DefaultPartitionRouter implements PartitionRouter {
    private final AtomicInteger counter = new AtomicInteger(0);
    
    @Override
    public int getPartition(ProducerRecord record, ClusterMetadata cluster) {
        String topic = record.topic();
        int numPartitions = cluster.getPartitionCount(topic);
        
        // Strategy 1: If partition is explicitly specified
        if (record.partition() != null) {
            return record.partition();
        }
        
        // Strategy 2: If message has a key, use key-based routing
        if (record.key() != null) {
            return keyBasedPartitioning(record.key(), numPartitions);
        }
        
        // Strategy 3: Round-robin for messages without keys
        return roundRobinPartitioning(numPartitions);
    }
    
    /**
     * Hash-based partitioning using message key.
     * Ensures messages with the same key go to the same partition (ordering guarantee).
     */
    private int keyBasedPartitioning(byte[] key, int numPartitions) {
        // Use consistent hashing to ensure same key always maps to same partition
        int hash = murmurHash(key);
        return Math.abs(hash) % numPartitions;
    }
    
    /**
     * Round-robin partitioning for messages without keys.
     * Distributes load evenly across all partitions.
     */
    private int roundRobinPartitioning(int numPartitions) {
        return counter.getAndIncrement() % numPartitions;
    }
    
    /**
     * MurmurHash implementation for consistent key hashing.
     */
    private int murmurHash(byte[] key) {
        // MurmurHash3 implementation
        final int seed = 0x9747b28c;
        final int m = 0x5bd1e995;
        final int r = 24;
        
        int h = seed ^ key.length;
        int len = key.length;
        int i = 0;
        
        while (len >= 4) {
            int k = (key[i] & 0xff) | ((key[i + 1] & 0xff) << 8) | 
                   ((key[i + 2] & 0xff) << 16) | ((key[i + 3] & 0xff) << 24);
            
            k *= m;
            k ^= k >>> r;
            k *= m;
            
            h *= m;
            h ^= k;
            
            i += 4;
            len -= 4;
        }
        
        // Handle remaining bytes
        switch (len) {
            case 3: h ^= (key[i + 2] & 0xff) << 16;
            case 2: h ^= (key[i + 1] & 0xff) << 8;
            case 1: h ^= (key[i] & 0xff);
                    h *= m;
        }
        
        h ^= h >>> 13;
        h *= m;
        h ^= h >>> 15;
        
        return h;
    }
}
```

#### Custom Partition Strategies

```java
/**
 * Geographic partitioning based on user location.
 */
public class GeographicPartitionRouter implements PartitionRouter {
    private final Map<String, Integer> regionToPartition;
    
    public GeographicPartitionRouter() {
        regionToPartition = Map.of(
            "US-EAST", 0,
            "US-WEST", 1,
            "EU", 2,
            "ASIA", 3
        );
    }
    
    @Override
    public int getPartition(ProducerRecord record, ClusterMetadata cluster) {
        // Extract region from message headers
        String region = record.headers().get("region");
        
        if (region != null && regionToPartition.containsKey(region)) {
            return regionToPartition.get(region);
        }
        
        // Fallback to default strategy
        return new DefaultPartitionRouter().getPartition(record, cluster);
    }
}

/**
 * Load-aware partitioning that considers current partition load.
 */
public class LoadAwarePartitionRouter implements PartitionRouter {
    private final PartitionLoadTracker loadTracker;
    
    @Override
    public int getPartition(ProducerRecord record, ClusterMetadata cluster) {
        String topic = record.topic();
        List<Integer> partitions = cluster.getPartitions(topic);
        
        // Find partition with lowest current load
        return partitions.stream()
                        .min(Comparator.comparing(loadTracker::getPartitionLoad))
                        .orElse(0);
    }
}

/**
 * Time-based partitioning for time-series data.
 */
public class TimeBasedPartitionRouter implements PartitionRouter {
    private final Duration partitionWindow;
    
    public TimeBasedPartitionRouter(Duration partitionWindow) {
        this.partitionWindow = partitionWindow;
    }
    
    @Override
    public int getPartition(ProducerRecord record, ClusterMetadata cluster) {
        long timestamp = record.timestamp() != null ? 
                        record.timestamp() : System.currentTimeMillis();
        
        int numPartitions = cluster.getPartitionCount(record.topic());
        
        // Calculate partition based on time window
        long windowIndex = timestamp / partitionWindow.toMillis();
        return (int) (windowIndex % numPartitions);
    }
}
```

#### Partition Router Configuration

```java
public class PartitionRouterConfig {
    // Partitioning strategy
    public static final String PARTITIONER_CLASS = "partitioner.class";
    public static final String DEFAULT_PARTITIONER = DefaultPartitionRouter.class.getName();
    
    // Key-based partitioning settings
    public static final String HASH_FUNCTION = "partitioner.hash.function"; // "murmur3", "crc32"
    public static final String KEY_ENCODING = "partitioner.key.encoding"; // "utf8", "bytes"
    
    // Round-robin settings
    public static final String ROUND_ROBIN_STICKY_BATCH = "partitioner.round.robin.sticky.batch";
    
    // Custom partitioner settings
    public static final String CUSTOM_PARTITIONER_CONFIG = "partitioner.custom.config";
}
```

#### Sticky Partitioning Optimization

```java
/**
 * Sticky partitioning reduces the number of partitions a producer sends to 
 * during a batch window, improving batching efficiency.
 */
public class StickyPartitionRouter implements PartitionRouter {
    private volatile int stickyPartition = -1;
    private final AtomicInteger counter = new AtomicInteger(0);
    private final int stickyBatchSize;
    
    public StickyPartitionRouter(int stickyBatchSize) {
        this.stickyBatchSize = stickyBatchSize;
    }
    
    @Override
    public int getPartition(ProducerRecord record, ClusterMetadata cluster) {
        // Use explicit partition if specified
        if (record.partition() != null) {
            return record.partition();
        }
        
        // Use key-based routing if key is present
        if (record.key() != null) {
            return keyBasedPartitioning(record.key(), 
                                      cluster.getPartitionCount(record.topic()));
        }
        
        // Sticky partitioning for keyless messages
        return getStickyPartition(cluster.getPartitionCount(record.topic()));
    }
    
    private synchronized int getStickyPartition(int numPartitions) {
        // Switch to new partition after batch size threshold
        if (stickyPartition == -1 || counter.get() >= stickyBatchSize) {
            stickyPartition = ThreadLocalRandom.current().nextInt(numPartitions);
            counter.set(0);
        }
        
        counter.incrementAndGet();
        return stickyPartition;
    }
    
    private int keyBasedPartitioning(byte[] key, int numPartitions) {
        return Math.abs(Arrays.hashCode(key)) % numPartitions;
    }
}
```

#### Impact on Message Ordering and Distribution

**Key-based Partitioning:**
- **Pros**: Guarantees ordering for messages with the same key
- **Cons**: Can create hot partitions if key distribution is uneven
- **Use Case**: User events, financial transactions, state updates

**Round-robin Partitioning:**
- **Pros**: Even load distribution across partitions
- **Cons**: No ordering guarantees across messages
- **Use Case**: Metrics, logs, independent events

**Custom Partitioning:**
- **Pros**: Application-specific optimization
- **Cons**: Requires careful design to avoid hot spots
- **Use Case**: Geographic routing, tenant isolation, time-series data

#### Performance Considerations

```java
public class PartitionRouterMetrics {
    private final Timer partitioningLatency;
    private final Counter[] partitionCounts;
    private final Histogram keyDistribution;
    
    public void recordPartitionSelection(int partition, long latencyNs) {
        partitioningLatency.record(latencyNs, TimeUnit.NANOSECONDS);
        partitionCounts[partition].increment();
    }
    
    public void recordKeyDistribution(byte[] key) {
        if (key != null) {
            keyDistribution.record(Arrays.hashCode(key));
        }
    }
    
    // Monitor partition skew
    public double getPartitionSkew() {
        long[] counts = Arrays.stream(partitionCounts)
                             .mapToLong(Counter::count)
                             .toArray();
        
        double mean = Arrays.stream(counts).average().orElse(0.0);
        double variance = Arrays.stream(counts)
                               .mapToDouble(count -> Math.pow(count - mean, 2))
                               .average()
                               .orElse(0.0);
        
        return Math.sqrt(variance) / mean; // Coefficient of variation
    }
}
```

#### Partition Router in Producer Flow

```java
// ...existing code...
````markdown
### 4. Consumer Implementation

#### Message Consumer
```java
public class MessageConsumer {
    private final ConsumerGroup group;
    private final NetworkClient networkClient;
    private final OffsetCommitter offsetCommitter;
    private final Set<TopicPartition> assignedPartitions;
    
    public ConsumerRecords poll(Duration timeout) {
        // 1. Check for rebalance
        group.checkRebalance();
        
        // 2. Fetch messages from assigned partitions
        Map<TopicPartition, List<Message>> records = fetchMessages();
        
        // 3. Update fetch positions
        updateFetchPositions(records);
        
        return new ConsumerRecords(records);
    }
    
    public void commitSync() {
        Map<TopicPartition, Long> offsets = getCurrentOffsets();
        offsetCommitter.commitSync(offsets);
    }
    
    private Map<TopicPartition, List<Message>> fetchMessages() {
        FetchRequest request = new FetchRequest();
        
        for (TopicPartition tp : assignedPartitions) {
            long offset = getNextFetchOffset(tp);
            request.addPartition(tp, offset, maxFetchBytes);
        }
        
        FetchResponse response = networkClient.send(request);
        return response.getRecords();
    }
}
```

#### Consumer Group Management
```java
public class ConsumerGroup {
    private final String groupId;
    private final GroupCoordinator coordinator;
    private final Set<String> subscribedTopics;
    private volatile boolean needsRebalance = false;
    
    public void subscribe(Collection<String> topics) {
        this.subscribedTopics.clear();
        this.subscribedTopics.addAll(topics);
        this.needsRebalance = true;
    }
    
    public void checkRebalance() {
        if (needsRebalance) {
            performRebalance();
            needsRebalance = false;
        }
    }
    
    private void performRebalance() {
        // 1. Send heartbeat to coordinator
        HeartbeatResponse heartbeat = coordinator.sendHeartbeat();
        
        if (heartbeat.needsRebalance()) {
            // 2. Join group
            JoinGroupResponse joinResponse = coordinator.joinGroup(subscribedTopics);
            
            // 3. Sync group (get partition assignment)
            SyncGroupResponse syncResponse = coordinator.syncGroup(joinResponse);
            
            // 4. Update assigned partitions
            updateAssignment(syncResponse.getAssignment());
        }
    }
}
```

### 4. Consumer Coordinator Implementation

The **Consumer Coordinator** is a critical server-side component that manages consumer groups, coordinates partition assignments, and handles consumer group rebalancing. It acts as the central authority for all consumer group operations.

#### Core Consumer Coordinator
```java
public class ConsumerCoordinator {
    private final Map<String, ConsumerGroupMetadata> groups;
    private final Map<String, Integer> groupCoordinators; // groupId -> brokerId
    private final PartitionAssignor partitionAssignor;
    private final OffsetManager offsetManager;
    private final HeartbeatManager heartbeatManager;
    private final RebalanceManager rebalanceManager;
    
    /**
     * Handle consumer join group request.
     * This is the first phase of the rebalance protocol.
     */
    public JoinGroupResponse handleJoinGroup(JoinGroupRequest request) {
        String groupId = request.getGroupId();
        String memberId = request.getMemberId();
        List<String> supportedProtocols = request.getSupportedProtocols();
        
        ConsumerGroupMetadata group = getOrCreateGroup(groupId);
        
        synchronized (group) {
            // Add member to group
            ConsumerMember member = new ConsumerMember(
                memberId.isEmpty() ? generateMemberId() : memberId,
                request.getClientId(),
                request.getClientHost(),
                supportedProtocols
            );
            
            group.addMember(member);
            
            // Trigger rebalance if this is a new member or protocol change
            if (group.shouldRebalance()) {
                group.prepareRebalance();
                rebalanceManager.scheduleRebalance(groupId);
            }
            
            // Wait for all members to join or timeout
            if (group.allMembersJoined()) {
                return createJoinGroupResponse(group, member);
            } else {
                // Return response indicating waiting for other members
                return JoinGroupResponse.waiting(member.getMemberId());
            }
        }
    }
    
    /**
     * Handle sync group request.
     * This is the second phase where partition assignments are distributed.
     */
    public SyncGroupResponse handleSyncGroup(SyncGroupRequest request) {
        String groupId = request.getGroupId();
        String memberId = request.getMemberId();
        
        ConsumerGroupMetadata group = groups.get(groupId);
        if (group == null) {
            return SyncGroupResponse.error("Group not found");
        }
        
        synchronized (group) {
            ConsumerMember member = group.getMember(memberId);
            if (member == null) {
                return SyncGroupResponse.error("Member not found");
            }
            
            // If this is the group leader, process the assignment
            if (member.isLeader() && request.hasAssignments()) {
                Map<String, List<TopicPartition>> assignments = request.getAssignments();
                group.setAssignments(assignments);
                
                // Persist assignments for failure recovery
                persistGroupAssignments(groupId, assignments);
            }
            
            // Return assignment for this member
            List<TopicPartition> assignment = group.getAssignment(memberId);
            return SyncGroupResponse.success(assignment);
        }
    }
    
    /**
     * Handle heartbeat request to maintain group membership.
     */
    public HeartbeatResponse handleHeartbeat(HeartbeatRequest request) {
        String groupId = request.getGroupId();
        String memberId = request.getMemberId();
        int generationId = request.getGenerationId();
        
        ConsumerGroupMetadata group = groups.get(groupId);
        if (group == null) {
            return HeartbeatResponse.error("UNKNOWN_GROUP");
        }
        
        synchronized (group) {
            // Validate member and generation
            if (!group.isValidMember(memberId, generationId)) {
                return HeartbeatResponse.error("REBALANCE_IN_PROGRESS");
            }
            
            // Update member's last heartbeat time
            group.updateMemberHeartbeat(memberId);
            
            // Check if rebalance is needed
            if (group.needsRebalance()) {
                return HeartbeatResponse.rebalanceRequired();
            }
            
            return HeartbeatResponse.success();
        }
    }
    
    /**
     * Handle leave group request when consumer shuts down gracefully.
     */
    public LeaveGroupResponse handleLeaveGroup(LeaveGroupRequest request) {
        String groupId = request.getGroupId();
        String memberId = request.getMemberId();
        
        ConsumerGroupMetadata group = groups.get(groupId);
        if (group == null) {
            return LeaveGroupResponse.success(); // Group already gone
        }
        
        synchronized (group) {
            group.removeMember(memberId);
            
            // Trigger rebalance if group is not empty
            if (!group.isEmpty()) {
                group.prepareRebalance();
                rebalanceManager.scheduleRebalance(groupId);
            } else {
                // Remove empty group
                groups.remove(groupId);
            }
        }
        
        return LeaveGroupResponse.success();
    }
}
```

#### Partition Assignment Strategies
```java
public interface PartitionAssignor {
    /**
     * Assign partitions to consumer group members.
     */
    Map<String, List<TopicPartition>> assign(ConsumerGroupMetadata group, 
                                            Map<String, Integer> topicPartitionCounts);
}

/**
 * Range partition assignment strategy.
 * Assigns consecutive partitions to each consumer.
 */
public class RangeAssignor implements PartitionAssignor {
    @Override
    public Map<String, List<TopicPartition>> assign(ConsumerGroupMetadata group,
                                                   Map<String, Integer> topicPartitionCounts) {
        Map<String, List<TopicPartition>> assignment = new HashMap<>();
        List<String> members = group.getOrderedMemberIds();
        
        for (String topic : group.getSubscribedTopics()) {
            int partitionCount = topicPartitionCounts.get(topic);
            int memberCount = members.size();
            
            int partitionsPerMember = partitionCount / memberCount;
            int remainingPartitions = partitionCount % memberCount;
            
            int partitionIndex = 0;
            for (int i = 0; i < memberCount; i++) {
                String memberId = members.get(i);
                int assignedPartitions = partitionsPerMember + (i < remainingPartitions ? 1 : 0);
                
                List<TopicPartition> memberPartitions = assignment.computeIfAbsent(
                    memberId, k -> new ArrayList<>()
                );
                
                for (int j = 0; j < assignedPartitions; j++) {
                    memberPartitions.add(new TopicPartition(topic, partitionIndex++));
                }
            }
        }
        
        return assignment;
    }
}

/**
 * Round-robin partition assignment strategy.
 * Distributes partitions evenly across all topics.
 */
public class RoundRobinAssignor implements PartitionAssignor {
    @Override
    public Map<String, List<TopicPartition>> assign(ConsumerGroupMetadata group,
                                                   Map<String, Integer> topicPartitionCounts) {
        Map<String, List<TopicPartition>> assignment = new HashMap<>();
        List<String> members = group.getOrderedMemberIds();
        
        // Collect all partitions
        List<TopicPartition> allPartitions = new ArrayList<>();
        for (String topic : group.getSubscribedTopics()) {
            int partitionCount = topicPartitionCounts.get(topic);
            for (int i = 0; i < partitionCount; i++) {
                allPartitions.add(new TopicPartition(topic, i));
            }
        }
        
        // Sort partitions for consistent assignment
        allPartitions.sort(Comparator.comparing(TopicPartition::topic)
                                   .thenComparing(TopicPartition::partition));
        
        // Round-robin assignment
        for (int i = 0; i < allPartitions.size(); i++) {
            String memberId = members.get(i % members.size());
            assignment.computeIfAbsent(memberId, k -> new ArrayList<>())
                     .add(allPartitions.get(i));
        }
        
        return assignment;
    }
}

/**
 * Sticky partition assignment strategy.
 * Tries to preserve existing assignments during rebalance.
 */
public class StickyAssignor implements PartitionAssignor {
    @Override
    public Map<String, List<TopicPartition>> assign(ConsumerGroupMetadata group,
                                                   Map<String, Integer> topicPartitionCounts) {
        Map<String, List<TopicPartition>> currentAssignment = group.getCurrentAssignment();
        Map<String, List<TopicPartition>> newAssignment = new HashMap<>();
        
        // Start with existing assignments for active members
        for (String memberId : group.getOrderedMemberIds()) {
            List<TopicPartition> existing = currentAssignment.getOrDefault(memberId, 
                                                                          new ArrayList<>());
            newAssignment.put(memberId, new ArrayList<>(existing));
        }
        
        // Find unassigned partitions
        Set<TopicPartition> allPartitions = getAllPartitions(group.getSubscribedTopics(), 
                                                            topicPartitionCounts);
        Set<TopicPartition> assignedPartitions = newAssignment.values().stream()
                                                             .flatMap(List::stream)
                                                             .collect(Collectors.toSet());
        
        List<TopicPartition> unassigned = allPartitions.stream()
                                                      .filter(tp -> !assignedPartitions.contains(tp))
                                                      .collect(Collectors.toList());
        
        // Distribute unassigned partitions to minimize assignment changes
        distributeUnassignedPartitions(newAssignment, unassigned);
        
        return newAssignment;
    }
    
    private void distributeUnassignedPartitions(Map<String, List<TopicPartition>> assignment,
                                              List<TopicPartition> unassigned) {
        List<String> members = new ArrayList<>(assignment.keySet());
        Collections.sort(members); // Ensure deterministic assignment
        
        for (TopicPartition partition : unassigned) {
            // Find member with fewest assigned partitions
            String targetMember = members.stream()
                                        .min(Comparator.comparing(m -> assignment.get(m).size()))
                                        .orElse(members.get(0));
            
            assignment.get(targetMember).add(partition);
        }
    }
}
```

#### Consumer Group State Management
```java
public class ConsumerGroupMetadata {
    private final String groupId;
    private final Map<String, ConsumerMember> members;
    private final Set<String> subscribedTopics;
    private final Map<String, List<TopicPartition>> currentAssignment;
    private volatile GroupState state;
    private volatile int generationId;
    private volatile String protocolType;
    private volatile String protocolName;
    
    public enum GroupState {
        PREPARING_REBALANCE,
        COMPLETING_REBALANCE,
        STABLE,
        DEAD,
        EMPTY
    }
    
    public boolean shouldRebalance() {
        return state == GroupState.PREPARING_REBALANCE ||
               hasNewMembers() ||
               hasLeavingMembers() ||
               hasExpiredMembers();
    }
    
    public void prepareRebalance() {
        this.state = GroupState.PREPARING_REBALANCE;
        this.generationId++;
        
        // Clear current assignments
        currentAssignment.clear();
        
        // Reset member states
        for (ConsumerMember member : members.values()) {
            member.setState(MemberState.AWAITING_SYNC);
        }
    }
    
    public boolean allMembersJoined() {
        return members.values().stream()
                     .allMatch(member -> member.getState() == MemberState.AWAITING_SYNC);
    }
    
    public void completeRebalance(Map<String, List<TopicPartition>> assignment) {
        this.currentAssignment.clear();
        this.currentAssignment.putAll(assignment);
        this.state = GroupState.STABLE;
        
        // Update member states
        for (ConsumerMember member : members.values()) {
            member.setState(MemberState.STABLE);
        }
    }
    
    public boolean hasExpiredMembers() {
        long sessionTimeoutMs = 30000; // Configure based on group settings
        long currentTime = System.currentTimeMillis();
        
        return members.values().stream()
                     .anyMatch(member -> 
                         currentTime - member.getLastHeartbeat() > sessionTimeoutMs);
    }
    
    public void removeExpiredMembers() {
        long sessionTimeoutMs = 30000;
        long currentTime = System.currentTimeMillis();
        
        Iterator<ConsumerMember> iterator = members.values().iterator();
        while (iterator.hasNext()) {
            ConsumerMember member = iterator.next();
            if (currentTime - member.getLastHeartbeat() > sessionTimeoutMs) {
                iterator.remove();
            }
        }
    }
}

public class ConsumerMember {
    private final String memberId;
    private final String clientId;
    private final String clientHost;
    private final List<String> supportedProtocols;
    private volatile MemberState state;
    private volatile long lastHeartbeat;
    private volatile boolean isLeader;
    
    public enum MemberState {
        JOINING,
        AWAITING_SYNC,
        STABLE,
        LEAVING
    }
    
    public void updateHeartbeat() {
        this.lastHeartbeat = System.currentTimeMillis();
    }
    
    public boolean isExpired(long sessionTimeoutMs) {
        return System.currentTimeMillis() - lastHeartbeat > sessionTimeoutMs;
    }
}
```

#### Rebalance Process Implementation
```java
public class RebalanceManager {
    private final ScheduledExecutorService scheduler;
    private final Map<String, ScheduledFuture<?>> pendingRebalances;
    private final ConsumerCoordinator coordinator;
    
    public void scheduleRebalance(String groupId) {
        scheduleRebalance(groupId, Duration.ofMillis(3000)); // Default delay
    }
    
    public void scheduleRebalance(String groupId, Duration delay) {
        // Cancel existing rebalance if pending
        ScheduledFuture<?> existing = pendingRebalances.get(groupId);
        if (existing != null && !existing.isDone()) {
            existing.cancel(false);
        }
        
        // Schedule new rebalance
        ScheduledFuture<?> future = scheduler.schedule(
            () -> executeRebalance(groupId),
            delay.toMillis(),
            TimeUnit.MILLISECONDS
        );
        
        pendingRebalances.put(groupId, future);
    }
    
    private void executeRebalance(String groupId) {
        ConsumerGroupMetadata group = coordinator.getGroup(groupId);
        if (group == null) {
            return;
        }
        
        synchronized (group) {
            try {
                // 1. Remove expired members
                group.removeExpiredMembers();
                
                // 2. Check if group is empty
                if (group.isEmpty()) {
                    group.setState(GroupState.DEAD);
                    coordinator.removeGroup(groupId);
                    return;
                }
                
                // 3. Transition to completing rebalance
                group.setState(GroupState.COMPLETING_REBALANCE);
                
                // 4. Select group leader (first member alphabetically for consistency)
                String leaderId = group.getOrderedMemberIds().get(0);
                group.setLeader(leaderId);
                
                // 5. Compute partition assignment
                Map<String, List<TopicPartition>> assignment = 
                    coordinator.computeAssignment(group);
                
                // 6. Complete rebalance
                group.completeRebalance(assignment);
                
                // 7. Notify all members that rebalance is complete
                notifyRebalanceComplete(group);
                
            } catch (Exception e) {
                // Handle rebalance failure
                group.setState(GroupState.PREPARING_REBALANCE);
                scheduleRebalance(groupId, Duration.ofMillis(5000)); // Retry
            }
        }
    }
    
    private void notifyRebalanceComplete(ConsumerGroupMetadata group) {
        // Send completion notification to all group members
        for (ConsumerMember member : group.getMembers()) {
            // This would typically send a response or notification
            // to the consumer client indicating rebalance completion
        }
    }
}
```

#### Protocol Message Definitions
```java
// Join Group Protocol
public class JoinGroupRequest {
    private final String groupId;
    private final int sessionTimeoutMs;
    private final int rebalanceTimeoutMs;
    private final String memberId;
    private final String groupInstanceId;
    private final String protocolType;
    private final List<String> supportedProtocols;
    private final Map<String, ByteBuffer> protocolMetadata;
}

public class JoinGroupResponse {
    private final int errorCode;
    private final int generationId;
    private final String protocolName;
    private final String leaderId;
    private final String memberId;
    private final Map<String, ByteBuffer> members; // memberId -> metadata
    
    public static JoinGroupResponse waiting(String memberId) {
        return new JoinGroupResponse(0, -1, null, null, memberId, null);
    }
    
    public static JoinGroupResponse success(int generationId, String protocolName,
                                          String leaderId, String memberId,
                                          Map<String, ByteBuffer> members) {
        return new JoinGroupResponse(0, generationId, protocolName, 
                                   leaderId, memberId, members);
    }
}

// Sync Group Protocol
public class SyncGroupRequest {
    private final String groupId;
    private final int generationId;
    private final String memberId;
    private final String groupInstanceId;
    private final Map<String, List<TopicPartition>> assignments;
}

public class SyncGroupResponse {
    private final int errorCode;
    private final List<TopicPartition> assignment;
    
    public static SyncGroupResponse success(List<TopicPartition> assignment) {
        return new SyncGroupResponse(0, assignment);
    }
    
    public static SyncGroupResponse error(String message) {
        return new SyncGroupResponse(-1, null);
    }
}

// Heartbeat Protocol
public class HeartbeatRequest {
    private final String groupId;
    private final int generationId;
    private final String memberId;
    private final String groupInstanceId;
}

public class HeartbeatResponse {
    private final int errorCode;
    private final int throttleTimeMs;
    
    public static HeartbeatResponse success() {
        return new HeartbeatResponse(0, 0);
    }
    
    public static HeartbeatResponse rebalanceRequired() {
        return new HeartbeatResponse(27, 0); // REBALANCE_IN_PROGRESS
    }
    
    public static HeartbeatResponse error(String errorCode) {
        return new HeartbeatResponse(-1, 0);
    }
}
```

#### Consumer Coordinator Integration
```java
public class ConsumerCoordinatorManager {
    private final Map<Integer, ConsumerCoordinator> coordinators; // brokerId -> coordinator
    private final ClusterMetadata clusterMetadata;
    private final ConsistentHashRing hashRing;
    
    /**
     * Get the coordinator for a specific consumer group.
     */
    public ConsumerCoordinator getCoordinator(String groupId) {
        int coordinatorBrokerId = hashRing.getCoordinator(groupId);
        return coordinators.get(coordinatorBrokerId);
    }
    
    /**
     * Handle coordinator migration during broker failures.
     */
    public void handleBrokerFailure(int failedBrokerId) {
        ConsumerCoordinator failedCoordinator = coordinators.get(failedBrokerId);
        if (failedCoordinator == null) {
            return;
        }
        
        // Migrate all groups to new coordinators
        Set<String> affectedGroups = failedCoordinator.getAllGroupIds();
        
        for (String groupId : affectedGroups) {
            int newCoordinatorId = hashRing.getCoordinator(groupId);
            ConsumerCoordinator newCoordinator = coordinators.get(newCoordinatorId);
            
            // Transfer group metadata
            ConsumerGroupMetadata group = failedCoordinator.getGroup(groupId);
            if (group != null) {
                newCoordinator.addGroup(group);
                
                // Trigger rebalance in new coordinator
                newCoordinator.scheduleRebalance(groupId);
            }
        }
        
        // Remove failed coordinator
        coordinators.remove(failedBrokerId);
    }
}
```

### 5. Coordination Service Integration

#### Zookeeper Client
```java
public class ZookeeperClient {
    private final ZooKeeper zookeeper;
    private final String brokerPath = "/brokers/ids";
    private final String topicPath = "/brokers/topics";
    private final String controllerPath = "/controller";
    
    public void registerBroker(int brokerId, String host, int port) {
        String brokerInfo = createBrokerInfo(host, port);
        String path = brokerPath + "/" + brokerId;
        
        zookeeper.create(path, brokerInfo.getBytes(), 
                        ZooDefs.Ids.OPEN_ACL_UNSAFE, 
                        CreateMode.EPHEMERAL);
    }
    
    public void registerAsController(int brokerId) {
        String controllerInfo = createControllerInfo(brokerId);
        
        try {
            zookeeper.create(controllerPath, controllerInfo.getBytes(),
                           ZooDefs.Ids.OPEN_ACL_UNSAFE,
                           CreateMode.EPHEMERAL);
        } catch (KeeperException.NodeExistsException e) {
            // Another broker is already controller
        }
    }
    
    public List<Integer> getAliveBrokers() {
        try {
            List<String> children = zookeeper.getChildren(brokerPath, false);
            return children.stream()
                          .map(Integer::parseInt)
                          .collect(Collectors.toList());
        } catch (Exception e) {
            throw new RuntimeException("Failed to get alive brokers", e);
        }
    }
}
```

### 6. Network Protocol

#### Request/Response Format
```java
// Base request format
public abstract class Request {
    protected final int apiKey;
    protected final int apiVersion;
    protected final int correlationId;
    protected final String clientId;
    
    public abstract byte[] serialize();
}

// Produce request
public class ProduceRequest extends Request {
    private final Map<TopicPartition, List<Message>> records;
    private final int acks;
    private final int timeoutMs;
    
    @Override
    public byte[] serialize() {
        ByteBuffer buffer = ByteBuffer.allocate(calculateSize());
        
        // Write header
        buffer.putInt(apiKey);
        buffer.putInt(apiVersion);
        buffer.putInt(correlationId);
        writeString(buffer, clientId);
        
        // Write body
        buffer.putInt(acks);
        buffer.putInt(timeoutMs);
        buffer.putInt(records.size());
        
        for (Map.Entry<TopicPartition, List<Message>> entry : records.entrySet()) {
            writeTopicPartition(buffer, entry.getKey());
            writeMessages(buffer, entry.getValue());
        }
        
        return buffer.array();
    }
}
```

### 7. Performance Optimizations

#### Zero-Copy Transfer
```java
public class ZeroCopyTransfer {
    public void sendFile(SocketChannel channel, RandomAccessFile file, 
                        long position, long count) throws IOException {
        FileChannel fileChannel = file.getChannel();
        
        // Use transferTo for zero-copy transfer
        long transferred = 0;
        while (transferred < count) {
            long result = fileChannel.transferTo(
                position + transferred,
                count - transferred,
                channel
            );
            transferred += result;
        }
    }
}
```

#### Memory Pool
```java
public class MemoryPool {
    private final Queue<ByteBuffer> availableBuffers;
    private final int bufferSize;
    private final int maxBuffers;
    
    public ByteBuffer allocate() {
        ByteBuffer buffer = availableBuffers.poll();
        if (buffer == null) {
            if (allocatedBuffers.get() < maxBuffers) {
                buffer = ByteBuffer.allocateDirect(bufferSize);
                allocatedBuffers.incrementAndGet();
            } else {
                // Wait for buffer to become available
                buffer = waitForBuffer();
            }
        }
        return buffer;
    }
    
    public void release(ByteBuffer buffer) {
        buffer.clear();
        availableBuffers.offer(buffer);
    }
}
```

## Configuration

### Broker Configuration
```properties
# broker.properties
broker.id=1
listeners=PLAINTEXT://localhost:9092
log.dirs=/var/kafka-logs
num.network.threads=8
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# Log settings
log.segment.bytes=1073741824
log.retention.hours=168
log.retention.bytes=1073741824
log.cleanup.policy=delete

# Replication settings
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Zookeeper settings
zookeeper.connect=localhost:2181
zookeeper.connection.timeout.ms=6000
```

### Client Configuration
```properties
# producer.properties
bootstrap.servers=localhost:9092
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
acks=all
retries=2147483647
batch.size=16384
linger.ms=0
buffer.memory=33554432
compression.type=snappy

# consumer.properties
bootstrap.servers=localhost:9092
group.id=test-group
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer
auto.offset.reset=earliest
enable.auto.commit=true
auto.commit.interval.ms=1000
session.timeout.ms=30000
```

## Testing Strategy

### Unit Tests
- Message serialization/deserialization
- Partition routing logic
- Batch management
- Offset management

### Integration Tests
- Producer-Broker-Consumer flow
- Replication and failover
- Consumer group rebalancing
- Network protocol correctness

### Performance Tests
- Throughput benchmarks
- Latency measurements
- Scalability testing
- Resource utilization

### Chaos Testing
- Broker failures
- Network partitions
- Disk failures
- High load scenarios

## Deployment

### Docker Compose Example
```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  broker1:
    image: message-queue/broker:latest
    depends_on:
      - zookeeper
    environment:
      BROKER_ID: 1
      ZOOKEEPER_CONNECT: zookeeper:2181
      LISTENERS: PLAINTEXT://0.0.0.0:9092
    ports:
      - "9092:9092"

  broker2:
    image: message-queue/broker:latest
    depends_on:
      - zookeeper
    environment:
      BROKER_ID: 2
      ZOOKEEPER_CONNECT: zookeeper:2181
      LISTENERS: PLAINTEXT://0.0.0.0:9093
    ports:
      - "9093:9093"

  broker3:
    image: message-queue/broker:latest
    depends_on:
      - zookeeper
    environment:
      BROKER_ID: 3
      ZOOKEEPER_CONNECT: zookeeper:2181
      LISTENERS: PLAINTEXT://0.0.0.0:9094
    ports:
      - "9094:9094"
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: broker
spec:
  serviceName: broker
  replicas: 3
  selector:
    matchLabels:
      app: broker
  template:
    metadata:
      labels:
        app: broker
    spec:
      containers:
      - name: broker
        image: message-queue/broker:latest
        ports:
        - containerPort: 9092
        env:
        - name: BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ZOOKEEPER_CONNECT
          value: "zookeeper:2181"
        volumeMounts:
        - name: data
          mountPath: /var/kafka-logs
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi
```

## Monitoring

### Key Metrics to Monitor
- **Broker Metrics**: CPU, memory, disk I/O, network I/O
- **Topic Metrics**: Message rate, byte rate, partition count
- **Consumer Metrics**: Lag, consumption rate, rebalance frequency
- **Replication Metrics**: ISR shrink/expand, replication lag

### Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'message-queue-brokers'
    static_configs:
      - targets: ['broker1:8080', 'broker2:8080', 'broker3:8080']
```

### Grafana Dashboard
- Broker health overview
- Topic and partition metrics
- Consumer lag monitoring
- Performance dashboards

This implementation guide provides a solid foundation for building a distributed message queue system with the architecture principles outlined in your notes. The system is designed to be scalable, reliable, and performant while maintaining simplicity in operation.
