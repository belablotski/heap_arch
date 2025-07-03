# API Reference Documentation

## Overview

This document provides comprehensive API documentation for the distributed message queue system, including client libraries, REST APIs, and administrative interfaces.

## Client API

### Producer API

#### MessageProducer Class

```java
public class MessageProducer implements AutoCloseable {
    
    /**
     * Creates a new MessageProducer with the specified configuration.
     * 
     * @param config Producer configuration properties
     */
    public MessageProducer(Properties config) { }
    
    /**
     * Sends a message asynchronously.
     * 
     * @param record The producer record to send
     * @return Future that will be completed when the message is acknowledged
     * @throws ProducerException if the message cannot be sent
     */
    public Future<RecordMetadata> send(ProducerRecord record) { }
    
    /**
     * Sends a message asynchronously with a callback.
     * 
     * @param record The producer record to send
     * @param callback Callback to execute when the send completes
     * @return Future that will be completed when the message is acknowledged
     */
    public Future<RecordMetadata> send(ProducerRecord record, Callback callback) { }
    
    /**
     * Sends a message synchronously.
     * 
     * @param record The producer record to send
     * @return RecordMetadata containing offset and timestamp
     * @throws ProducerException if the message cannot be sent
     * @throws InterruptedException if the thread is interrupted
     */
    public RecordMetadata sendSync(ProducerRecord record) 
            throws ProducerException, InterruptedException { }
    
    /**
     * Flushes all pending messages, blocking until complete.
     */
    public void flush() { }
    
    /**
     * Gets the partition metadata for a topic.
     * 
     * @param topic The topic name
     * @return List of partition metadata
     */
    public List<PartitionInfo> partitionsFor(String topic) { }
    
    /**
     * Closes the producer and releases all resources.
     */
    @Override
    public void close() { }
    
    /**
     * Closes the producer with a timeout.
     * 
     * @param timeout Maximum time to wait for close
     */
    public void close(Duration timeout) { }
}
```

#### ProducerRecord Class

```java
public class ProducerRecord {
    private final String topic;
    private final Integer partition;
    private final byte[] key;
    private final byte[] value;
    private final Map<String, String> headers;
    private final Long timestamp;
    
    /**
     * Creates a producer record with topic and value.
     */
    public ProducerRecord(String topic, byte[] value) { }
    
    /**
     * Creates a producer record with topic, key, and value.
     */
    public ProducerRecord(String topic, byte[] key, byte[] value) { }
    
    /**
     * Creates a producer record with topic, partition, key, and value.
     */
    public ProducerRecord(String topic, Integer partition, byte[] key, byte[] value) { }
    
    /**
     * Creates a producer record with all fields.
     */
    public ProducerRecord(String topic, Integer partition, Long timestamp,
                         byte[] key, byte[] value, Map<String, String> headers) { }
    
    // Getters
    public String topic() { return topic; }
    public Integer partition() { return partition; }
    public byte[] key() { return key; }
    public byte[] value() { return value; }
    public Map<String, String> headers() { return headers; }
    public Long timestamp() { return timestamp; }
}
```

#### Producer Configuration

```java
public class ProducerConfig {
    // Bootstrap servers
    public static final String BOOTSTRAP_SERVERS = "bootstrap.servers";
    
    // Serialization
    public static final String KEY_SERIALIZER = "key.serializer";
    public static final String VALUE_SERIALIZER = "value.serializer";
    
    // Acknowledgment
    public static final String ACKS = "acks"; // "0", "1", "all"
    
    // Batching
    public static final String BATCH_SIZE = "batch.size";
    public static final String LINGER_MS = "linger.ms";
    public static final String BUFFER_MEMORY = "buffer.memory";
    
    // Retries
    public static final String RETRIES = "retries";
    public static final String RETRY_BACKOFF_MS = "retry.backoff.ms";
    public static final String REQUEST_TIMEOUT_MS = "request.timeout.ms";
    
    // Compression
    public static final String COMPRESSION_TYPE = "compression.type"; // "none", "gzip", "snappy", "lz4"
    
    // Idempotence
    public static final String ENABLE_IDEMPOTENCE = "enable.idempotence";
    
    // Security
    public static final String SECURITY_PROTOCOL = "security.protocol";
    public static final String SASL_MECHANISM = "sasl.mechanism";
    public static final String SASL_JAAS_CONFIG = "sasl.jaas.config";
}
```

### Consumer API

#### MessageConsumer Class

```java
public class MessageConsumer implements AutoCloseable {
    
    /**
     * Creates a new MessageConsumer with the specified configuration.
     * 
     * @param config Consumer configuration properties
     */
    public MessageConsumer(Properties config) { }
    
    /**
     * Subscribes to the given topics.
     * 
     * @param topics Collection of topic names to subscribe to
     */
    public void subscribe(Collection<String> topics) { }
    
    /**
     * Subscribes to topics matching the given pattern.
     * 
     * @param pattern Regex pattern for topic names
     */
    public void subscribe(Pattern pattern) { }
    
    /**
     * Manually assign specific partitions to this consumer.
     * 
     * @param partitions Collection of TopicPartition to assign
     */
    public void assign(Collection<TopicPartition> partitions) { }
    
    /**
     * Polls for new messages.
     * 
     * @param timeout Maximum time to wait for messages
     * @return ConsumerRecords containing fetched messages
     */
    public ConsumerRecords poll(Duration timeout) { }
    
    /**
     * Commits the current offsets synchronously.
     * 
     * @throws CommitException if the commit fails
     */
    public void commitSync() throws CommitException { }
    
    /**
     * Commits specific offsets synchronously.
     * 
     * @param offsets Map of TopicPartition to OffsetAndMetadata
     */
    public void commitSync(Map<TopicPartition, OffsetAndMetadata> offsets) { }
    
    /**
     * Commits the current offsets asynchronously.
     */
    public void commitAsync() { }
    
    /**
     * Commits specific offsets asynchronously with a callback.
     * 
     * @param offsets Map of TopicPartition to OffsetAndMetadata
     * @param callback Callback to execute when commit completes
     */
    public void commitAsync(Map<TopicPartition, OffsetAndMetadata> offsets, 
                           OffsetCommitCallback callback) { }
    
    /**
     * Seeks to a specific offset for the given partition.
     * 
     * @param partition The partition to seek
     * @param offset The offset to seek to
     */
    public void seek(TopicPartition partition, long offset) { }
    
    /**
     * Seeks to the beginning of all assigned partitions.
     */
    public void seekToBeginning(Collection<TopicPartition> partitions) { }
    
    /**
     * Seeks to the end of all assigned partitions.
     */
    public void seekToEnd(Collection<TopicPartition> partitions) { }
    
    /**
     * Gets the current assignment for this consumer.
     * 
     * @return Set of assigned TopicPartition
     */
    public Set<TopicPartition> assignment() { }
    
    /**
     * Gets the current subscription for this consumer.
     * 
     * @return Set of subscribed topics
     */
    public Set<String> subscription() { }
    
    /**
     * Closes the consumer and releases all resources.
     */
    @Override
    public void close() { }
}
```

#### ConsumerRecord Class

```java
public class ConsumerRecord {
    private final String topic;
    private final int partition;
    private final long offset;
    private final long timestamp;
    private final byte[] key;
    private final byte[] value;
    private final Map<String, String> headers;
    
    // Constructor
    public ConsumerRecord(String topic, int partition, long offset, 
                         long timestamp, byte[] key, byte[] value,
                         Map<String, String> headers) { }
    
    // Getters
    public String topic() { return topic; }
    public int partition() { return partition; }
    public long offset() { return offset; }
    public long timestamp() { return timestamp; }
    public byte[] key() { return key; }
    public byte[] value() { return value; }
    public Map<String, String> headers() { return headers; }
}
```

#### Consumer Configuration

```java
public class ConsumerConfig {
    // Bootstrap servers
    public static final String BOOTSTRAP_SERVERS = "bootstrap.servers";
    
    // Consumer group
    public static final String GROUP_ID = "group.id";
    public static final String GROUP_INSTANCE_ID = "group.instance.id";
    
    // Deserialization
    public static final String KEY_DESERIALIZER = "key.deserializer";
    public static final String VALUE_DESERIALIZER = "value.deserializer";
    
    // Auto offset reset
    public static final String AUTO_OFFSET_RESET = "auto.offset.reset"; // "earliest", "latest", "none"
    
    // Auto commit
    public static final String ENABLE_AUTO_COMMIT = "enable.auto.commit";
    public static final String AUTO_COMMIT_INTERVAL_MS = "auto.commit.interval.ms";
    
    // Fetch settings
    public static final String FETCH_MIN_BYTES = "fetch.min.bytes";
    public static final String FETCH_MAX_WAIT_MS = "fetch.max.wait.ms";
    public static final String MAX_PARTITION_FETCH_BYTES = "max.partition.fetch.bytes";
    
    // Session and heartbeat
    public static final String SESSION_TIMEOUT_MS = "session.timeout.ms";
    public static final String HEARTBEAT_INTERVAL_MS = "heartbeat.interval.ms";
    
    // Max poll
    public static final String MAX_POLL_RECORDS = "max.poll.records";
    public static final String MAX_POLL_INTERVAL_MS = "max.poll.interval.ms";
    
    // Security
    public static final String SECURITY_PROTOCOL = "security.protocol";
    public static final String SASL_MECHANISM = "sasl.mechanism";
    public static final String SASL_JAAS_CONFIG = "sasl.jaas.config";
}
```

## Administrative API

### AdminClient Class

```java
public class AdminClient implements AutoCloseable {
    
    /**
     * Creates a new AdminClient with the specified configuration.
     * 
     * @param config Admin client configuration
     * @return AdminClient instance
     */
    public static AdminClient create(Properties config) { }
    
    /**
     * Creates a new topic.
     * 
     * @param topic NewTopic specification
     * @return CreateTopicsResult containing the operation result
     */
    public CreateTopicsResult createTopics(Collection<NewTopic> topics) { }
    
    /**
     * Deletes topics.
     * 
     * @param topics Collection of topic names to delete
     * @return DeleteTopicsResult containing the operation result
     */
    public DeleteTopicsResult deleteTopics(Collection<String> topics) { }
    
    /**
     * Lists all topics in the cluster.
     * 
     * @return ListTopicsResult containing topic information
     */
    public ListTopicsResult listTopics() { }
    
    /**
     * Describes the specified topics.
     * 
     * @param topics Collection of topic names to describe
     * @return DescribeTopicsResult containing topic descriptions
     */
    public DescribeTopicsResult describeTopics(Collection<String> topics) { }
    
    /**
     * Describes the cluster.
     * 
     * @return DescribeClusterResult containing cluster information
     */
    public DescribeClusterResult describeCluster() { }
    
    /**
     * Lists consumer groups.
     * 
     * @return ListConsumerGroupsResult containing group information
     */
    public ListConsumerGroupsResult listConsumerGroups() { }
    
    /**
     * Describes consumer groups.
     * 
     * @param groupIds Collection of group IDs to describe
     * @return DescribeConsumerGroupsResult containing group descriptions
     */
    public DescribeConsumerGroupsResult describeConsumerGroups(Collection<String> groupIds) { }
    
    /**
     * Gets consumer group offsets.
     * 
     * @param groupId Consumer group ID
     * @return ListConsumerGroupOffsetsResult containing offset information
     */
    public ListConsumerGroupOffsetsResult listConsumerGroupOffsets(String groupId) { }
    
    /**
     * Alters consumer group offsets.
     * 
     * @param groupId Consumer group ID
     * @param offsets Map of TopicPartition to OffsetAndMetadata
     * @return AlterConsumerGroupOffsetsResult containing the operation result
     */
    public AlterConsumerGroupOffsetsResult alterConsumerGroupOffsets(
            String groupId, Map<TopicPartition, OffsetAndMetadata> offsets) { }
    
    /**
     * Creates partitions for existing topics.
     * 
     * @param newPartitions Map of topic name to NewPartitions
     * @return CreatePartitionsResult containing the operation result
     */
    public CreatePartitionsResult createPartitions(Map<String, NewPartitions> newPartitions) { }
    
    /**
     * Alters topic configurations.
     * 
     * @param configs Map of resource to configuration entries
     * @return AlterConfigsResult containing the operation result
     */
    public AlterConfigsResult alterConfigs(Map<ConfigResource, Config> configs) { }
    
    /**
     * Describes configurations.
     * 
     * @param resources Collection of ConfigResource to describe
     * @return DescribeConfigsResult containing configuration descriptions
     */
    public DescribeConfigsResult describeConfigs(Collection<ConfigResource> resources) { }
    
    @Override
    public void close() { }
}
```

### Topic Management

```java
public class NewTopic {
    private final String name;
    private final int numPartitions;
    private final short replicationFactor;
    private final Map<String, String> configs;
    
    /**
     * Creates a new topic with specified partitions and replication factor.
     */
    public NewTopic(String name, int numPartitions, short replicationFactor) { }
    
    /**
     * Creates a new topic with custom replica assignments.
     */
    public NewTopic(String name, Map<Integer, List<Integer>> replicasAssignments) { }
    
    /**
     * Sets topic configuration.
     */
    public NewTopic configs(Map<String, String> configs) { }
}

public class TopicDescription {
    private final String name;
    private final boolean internal;
    private final List<TopicPartitionInfo> partitions;
    
    // Getters
    public String name() { return name; }
    public boolean isInternal() { return internal; }
    public List<TopicPartitionInfo> partitions() { return partitions; }
}

public class TopicPartitionInfo {
    private final int partition;
    private final Node leader;
    private final List<Node> replicas;
    private final List<Node> isr;
    
    // Getters
    public int partition() { return partition; }
    public Node leader() { return leader; }
    public List<Node> replicas() { return replicas; }
    public List<Node> inSyncReplicas() { return isr; }
}
```

## REST API

### Topics Endpoint

#### GET /api/v1/topics
Lists all topics in the cluster.

**Response:**
```json
{
  "topics": [
    {
      "name": "orders",
      "partitions": 10,
      "replication_factor": 3,
      "configs": {
        "retention.ms": "604800000",
        "cleanup.policy": "delete"
      }
    }
  ]
}
```

#### POST /api/v1/topics
Creates a new topic.

**Request:**
```json
{
  "name": "payments",
  "partitions": 20,
  "replication_factor": 3,
  "configs": {
    "retention.ms": "1209600000",
    "cleanup.policy": "delete",
    "compression.type": "snappy"
  }
}
```

**Response:**
```json
{
  "success": true,
  "topic": "payments",
  "message": "Topic created successfully"
}
```

#### GET /api/v1/topics/{topic}
Gets detailed information about a specific topic.

**Response:**
```json
{
  "name": "orders",
  "partitions": [
    {
      "partition": 0,
      "leader": {
        "id": 1,
        "host": "broker1.example.com",
        "port": 9092
      },
      "replicas": [1, 2, 3],
      "isr": [1, 2, 3]
    }
  ],
  "configs": {
    "retention.ms": "604800000",
    "cleanup.policy": "delete"
  }
}
```

#### DELETE /api/v1/topics/{topic}
Deletes a topic.

**Response:**
```json
{
  "success": true,
  "topic": "orders",
  "message": "Topic deleted successfully"
}
```

### Consumer Groups Endpoint

#### GET /api/v1/consumer-groups
Lists all consumer groups.

**Response:**
```json
{
  "groups": [
    {
      "group_id": "order-processors",
      "state": "Stable",
      "protocol": "range",
      "members": 3
    }
  ]
}
```

#### GET /api/v1/consumer-groups/{group}/offsets
Gets consumer group offsets.

**Response:**
```json
{
  "group_id": "order-processors",
  "offsets": [
    {
      "topic": "orders",
      "partition": 0,
      "current_offset": 12345,
      "log_end_offset": 12350,
      "lag": 5
    }
  ]
}
```

#### PUT /api/v1/consumer-groups/{group}/offsets
Resets consumer group offsets.

**Request:**
```json
{
  "offsets": [
    {
      "topic": "orders",
      "partition": 0,
      "offset": 10000
    }
  ]
}
```

### Cluster Endpoint

#### GET /api/v1/cluster
Gets cluster information.

**Response:**
```json
{
  "cluster_id": "Nk1Q2YsAS_CrsNddMH9QjQ",
  "controller": {
    "id": 1,
    "host": "broker1.example.com",
    "port": 9092
  },
  "brokers": [
    {
      "id": 1,
      "host": "broker1.example.com",
      "port": 9092,
      "rack": "us-east-1a"
    }
  ]
}
```

#### GET /api/v1/cluster/health
Gets cluster health status.

**Response:**
```json
{
  "status": "healthy",
  "brokers_online": 3,
  "brokers_total": 3,
  "under_replicated_partitions": 0,
  "offline_partitions": 0
}
```

### Metrics Endpoint

#### GET /api/v1/metrics/brokers
Gets broker metrics.

**Response:**
```json
{
  "brokers": [
    {
      "id": 1,
      "messages_in_per_sec": 5000.0,
      "bytes_in_per_sec": 5242880.0,
      "bytes_out_per_sec": 10485760.0,
      "cpu_usage": 45.2,
      "memory_usage": 78.5,
      "disk_usage": 65.0
    }
  ]
}
```

#### GET /api/v1/metrics/topics/{topic}
Gets topic-specific metrics.

**Response:**
```json
{
  "topic": "orders",
  "messages_in_per_sec": 1000.0,
  "bytes_in_per_sec": 1048576.0,
  "total_messages": 1000000,
  "total_bytes": 1073741824,
  "partitions": [
    {
      "partition": 0,
      "size_bytes": 134217728,
      "messages": 100000,
      "log_end_offset": 100000
    }
  ]
}
```

## Error Handling

### Error Response Format

All API errors follow this format:

```json
{
  "error": {
    "code": "TOPIC_NOT_FOUND",
    "message": "Topic 'nonexistent' does not exist",
    "details": {
      "topic": "nonexistent"
    }
  }
}
```

### Error Codes

| Code | Description |
|------|-------------|
| `TOPIC_NOT_FOUND` | Requested topic does not exist |
| `TOPIC_ALREADY_EXISTS` | Topic with the same name already exists |
| `INVALID_PARTITIONS` | Invalid partition count specified |
| `INVALID_REPLICATION_FACTOR` | Invalid replication factor specified |
| `INSUFFICIENT_REPLICAS` | Not enough brokers for replication factor |
| `GROUP_NOT_FOUND` | Consumer group does not exist |
| `OFFSET_OUT_OF_RANGE` | Requested offset is out of range |
| `INVALID_CONFIG` | Invalid configuration parameter |
| `AUTHORIZATION_FAILED` | Insufficient permissions for operation |
| `AUTHENTICATION_FAILED` | Authentication credentials invalid |
| `CLUSTER_UNHEALTHY` | Cluster is in an unhealthy state |
| `BROKER_NOT_AVAILABLE` | Required broker is not available |

## SDK Examples

### Java Producer Example

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER, StringSerializer.class.getName());
props.put(ProducerConfig.VALUE_SERIALIZER, StringSerializer.class.getName());
props.put(ProducerConfig.ACKS, "all");
props.put(ProducerConfig.RETRIES, 3);
props.put(ProducerConfig.BATCH_SIZE, 16384);

try (MessageProducer producer = new MessageProducer(props)) {
    for (int i = 0; i < 1000; i++) {
        ProducerRecord record = new ProducerRecord(
            "orders", 
            "order-" + i, 
            "Order data " + i
        );
        
        producer.send(record, new Callback() {
            @Override
            public void onCompletion(RecordMetadata metadata, Exception exception) {
                if (exception != null) {
                    exception.printStackTrace();
                } else {
                    System.out.printf("Sent to partition %d, offset %d%n",
                                    metadata.partition(), metadata.offset());
                }
            }
        });
    }
    
    producer.flush();
}
```

### Java Consumer Example

```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID, "order-processors");
props.put(ConsumerConfig.KEY_DESERIALIZER, StringDeserializer.class.getName());
props.put(ConsumerConfig.VALUE_DESERIALIZER, StringDeserializer.class.getName());
props.put(ConsumerConfig.AUTO_OFFSET_RESET, "earliest");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT, false);

try (MessageConsumer consumer = new MessageConsumer(props)) {
    consumer.subscribe(Arrays.asList("orders"));
    
    while (true) {
        ConsumerRecords records = consumer.poll(Duration.ofMillis(100));
        
        for (ConsumerRecord record : records) {
            System.out.printf("Received: key=%s, value=%s, partition=%d, offset=%d%n",
                            record.key(), record.value(), 
                            record.partition(), record.offset());
            
            // Process the record
            processOrder(record.value());
        }
        
        consumer.commitSync();
    }
}
```

This API documentation provides comprehensive coverage of all client and administrative interfaces for the distributed message queue system, including detailed examples and error handling information.
