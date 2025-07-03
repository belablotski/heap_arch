# Security and Operations Guide

## Overview

This document covers security configurations, operational procedures, and best practices for running the distributed message queue system in production environments.

## Security Architecture

### Authentication and Authorization

#### 1. SASL Authentication
```java
// SASL configuration for brokers
public class SASLConfig {
    public static Properties getBrokerSASLConfig() {
        Properties props = new Properties();
        props.put("security.inter.broker.protocol", "SASL_SSL");
        props.put("sasl.mechanism.inter.broker.protocol", "PLAIN");
        props.put("sasl.enabled.mechanisms", "PLAIN,SCRAM-SHA-256");
        
        // JAAS configuration
        props.put("listener.name.sasl_ssl.plain.sasl.jaas.config",
            "org.apache.kafka.common.security.plain.PlainLoginModule required " +
            "username=\"admin\" password=\"admin-secret\";");
            
        return props;
    }
}

// Client SASL configuration
public class ClientSASLConfig {
    public static Properties getProducerSASLConfig() {
        Properties props = new Properties();
        props.put("security.protocol", "SASL_SSL");
        props.put("sasl.mechanism", "PLAIN");
        props.put("sasl.jaas.config",
            "org.apache.kafka.common.security.plain.PlainLoginModule required " +
            "username=\"producer\" password=\"producer-secret\";");
            
        return props;
    }
}
```

#### 2. OAuth 2.0 Integration
```java
public class OAuthAuthenticator implements Authenticator {
    private final OAuthTokenValidator tokenValidator;
    
    @Override
    public boolean authenticate(String token, String resource) {
        try {
            TokenValidationResult result = tokenValidator.validate(token);
            
            if (!result.isValid()) {
                return false;
            }
            
            // Check resource permissions
            return hasPermission(result.getPrincipal(), resource);
            
        } catch (Exception e) {
            log.error("OAuth authentication failed", e);
            return false;
        }
    }
    
    private boolean hasPermission(Principal principal, String resource) {
        // Check permissions from OAuth token claims
        Set<String> permissions = principal.getClaims().get("permissions");
        return permissions.contains("read:" + resource) || 
               permissions.contains("write:" + resource);
    }
}
```

#### 3. Access Control Lists (ACLs)
```java
public class ACLManager {
    private final Map<Principal, Set<ResourcePermission>> acls;
    
    public void addACL(Principal principal, ResourceType resourceType, 
                      String resourceName, Operation operation) {
        ResourcePermission permission = new ResourcePermission(
            resourceType, resourceName, operation);
            
        acls.computeIfAbsent(principal, k -> new HashSet<>())
            .add(permission);
    }
    
    public boolean authorize(Principal principal, ResourceType resourceType,
                           String resourceName, Operation operation) {
        Set<ResourcePermission> permissions = acls.get(principal);
        if (permissions == null) {
            return false;
        }
        
        ResourcePermission required = new ResourcePermission(
            resourceType, resourceName, operation);
            
        return permissions.contains(required) ||
               permissions.contains(new ResourcePermission(
                   resourceType, "*", operation));
    }
}

// Usage example
public class SecurityExample {
    public void setupACLs() {
        ACLManager aclManager = new ACLManager();
        
        // Allow producer to write to specific topics
        Principal producerPrincipal = new Principal("User", "producer");
        aclManager.addACL(producerPrincipal, ResourceType.TOPIC, 
                         "orders", Operation.WRITE);
        aclManager.addACL(producerPrincipal, ResourceType.TOPIC, 
                         "payments", Operation.WRITE);
        
        // Allow consumer group to read from topics
        Principal consumerPrincipal = new Principal("User", "consumer-group-1");
        aclManager.addACL(consumerPrincipal, ResourceType.TOPIC, 
                         "orders", Operation.READ);
        aclManager.addACL(consumerPrincipal, ResourceType.GROUP, 
                         "order-processors", Operation.READ);
    }
}
```

### Encryption

#### 1. TLS Configuration
```properties
# Broker TLS configuration
listeners=SSL://0.0.0.0:9093,PLAINTEXT://0.0.0.0:9092
ssl.keystore.location=/var/ssl/private/broker.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/var/ssl/private/broker.truststore.jks
ssl.truststore.password=truststore-password

# Client authentication
ssl.client.auth=required
ssl.enabled.protocols=TLSv1.2,TLSv1.3
ssl.cipher.suites=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384

# Security protocol for inter-broker communication
security.inter.broker.protocol=SSL
```

#### 2. Encryption at Rest
```java
public class EncryptionAtRest {
    private final AESCipher cipher;
    private final KeyManager keyManager;
    
    public void writeEncryptedSegment(LogSegment segment, List<Message> messages) {
        // Encrypt message data
        byte[] plaintext = serializeMessages(messages);
        EncryptionKey key = keyManager.getCurrentKey();
        byte[] ciphertext = cipher.encrypt(plaintext, key);
        
        // Write encrypted data with key reference
        SegmentHeader header = new SegmentHeader(key.getKeyId(), 
                                               ciphertext.length);
        segment.writeHeader(header);
        segment.writeData(ciphertext);
    }
    
    public List<Message> readEncryptedSegment(LogSegment segment) {
        SegmentHeader header = segment.readHeader();
        byte[] ciphertext = segment.readData(header.getDataLength());
        
        // Decrypt using referenced key
        EncryptionKey key = keyManager.getKey(header.getKeyId());
        byte[] plaintext = cipher.decrypt(ciphertext, key);
        
        return deserializeMessages(plaintext);
    }
}
```

#### 3. Key Management
```java
public class KeyManager {
    private final Map<String, EncryptionKey> keys;
    private final KeyRotationScheduler rotationScheduler;
    
    public void rotateKeys() {
        // Generate new key
        EncryptionKey newKey = generateNewKey();
        keys.put(newKey.getKeyId(), newKey);
        
        // Schedule old key for deletion (after retention period)
        scheduleKeyDeletion(getCurrentKey(), Duration.ofDays(30));
        
        // Update current key reference
        setCurrentKey(newKey);
        
        log.info("Key rotation completed. New key ID: {}", newKey.getKeyId());
    }
    
    private EncryptionKey generateNewKey() {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);
        SecretKey secretKey = keyGen.generateKey();
        
        return new EncryptionKey(
            UUID.randomUUID().toString(),
            secretKey.getEncoded(),
            System.currentTimeMillis()
        );
    }
}
```

## Operations and Monitoring

### Cluster Management

#### 1. Broker Lifecycle Management
```java
public class BrokerManager {
    private final List<BrokerNode> brokers;
    private final HealthChecker healthChecker;
    
    public void addBroker(BrokerNode broker) {
        // 1. Start broker process
        broker.start();
        
        // 2. Wait for broker to join cluster
        waitForBrokerRegistration(broker.getId());
        
        // 3. Trigger partition rebalancing
        triggerPartitionRebalance();
        
        // 4. Update load balancer configuration
        updateLoadBalancer();
        
        brokers.add(broker);
        log.info("Broker {} added to cluster", broker.getId());
    }
    
    public void removeBroker(int brokerId) {
        BrokerNode broker = findBroker(brokerId);
        
        // 1. Move partitions to other brokers
        relocatePartitions(brokerId);
        
        // 2. Wait for data migration to complete
        waitForMigrationCompletion(brokerId);
        
        // 3. Gracefully shutdown broker
        broker.shutdown();
        
        // 4. Remove from cluster
        brokers.removeIf(b -> b.getId() == brokerId);
        
        log.info("Broker {} removed from cluster", brokerId);
    }
    
    public void rollingUpdate() {
        for (BrokerNode broker : brokers) {
            // 1. Move leadership away from broker
            transferLeadership(broker.getId());
            
            // 2. Shutdown and update broker
            broker.shutdown();
            broker.update();
            broker.start();
            
            // 3. Wait for broker to catch up
            waitForSync(broker.getId());
            
            // 4. Move to next broker
            Thread.sleep(Duration.ofMinutes(2).toMillis());
        }
    }
}
```

#### 2. Partition Management
```java
public class PartitionManager {
    
    public void rebalancePartitions() {
        List<TopicPartition> partitions = getAllPartitions();
        List<Integer> availableBrokers = getAvailableBrokers();
        
        // Calculate optimal distribution
        Map<TopicPartition, List<Integer>> newAssignment = 
            calculateOptimalAssignment(partitions, availableBrokers);
        
        // Execute reassignment
        for (Map.Entry<TopicPartition, List<Integer>> entry : newAssignment.entrySet()) {
            reassignPartition(entry.getKey(), entry.getValue());
        }
    }
    
    public void increasePartitionCount(String topic, int newPartitionCount) {
        TopicMetadata metadata = getTopicMetadata(topic);
        int currentPartitions = metadata.getPartitionCount();
        
        if (newPartitionCount <= currentPartitions) {
            throw new IllegalArgumentException("New partition count must be greater than current");
        }
        
        // Create new partitions
        for (int i = currentPartitions; i < newPartitionCount; i++) {
            List<Integer> replicas = selectReplicaBrokers(metadata.getReplicationFactor());
            createPartition(topic, i, replicas);
        }
        
        // Update topic metadata
        updateTopicMetadata(topic, newPartitionCount);
    }
    
    private List<Integer> selectReplicaBrokers(int replicationFactor) {
        List<Integer> availableBrokers = getAvailableBrokers();
        Collections.shuffle(availableBrokers);
        return availableBrokers.subList(0, Math.min(replicationFactor, availableBrokers.size()));
    }
}
```

### Monitoring and Alerting

#### 1. Health Checks
```java
public class HealthChecker {
    
    @HealthCheck
    public HealthStatus checkBrokerHealth() {
        try {
            // Check if broker is responsive
            if (!isBrokerResponsive()) {
                return HealthStatus.UNHEALTHY("Broker not responsive");
            }
            
            // Check disk space
            if (getDiskUsagePercent() > 90) {
                return HealthStatus.UNHEALTHY("Disk usage too high");
            }
            
            // Check memory usage
            if (getMemoryUsagePercent() > 85) {
                return HealthStatus.WARNING("Memory usage high");
            }
            
            // Check replication lag
            long maxReplicationLag = getMaxReplicationLag();
            if (maxReplicationLag > Duration.ofMinutes(5).toMillis()) {
                return HealthStatus.WARNING("High replication lag: " + maxReplicationLag + "ms");
            }
            
            return HealthStatus.HEALTHY();
            
        } catch (Exception e) {
            return HealthStatus.UNHEALTHY("Health check failed: " + e.getMessage());
        }
    }
    
    @HealthCheck  
    public HealthStatus checkConsumerLag() {
        Map<String, Long> consumerLags = getConsumerLags();
        
        for (Map.Entry<String, Long> entry : consumerLags.entrySet()) {
            if (entry.getValue() > 100_000) {
                return HealthStatus.WARNING(
                    "High consumer lag for group " + entry.getKey() + ": " + entry.getValue());
            }
        }
        
        return HealthStatus.HEALTHY();
    }
}
```

#### 2. Metrics Collection
```java
public class MetricsCollector {
    private final MeterRegistry meterRegistry;
    
    public void recordMessage(String topic, int partition, long size) {
        Counter.builder("messages.produced")
               .tag("topic", topic)
               .tag("partition", String.valueOf(partition))
               .register(meterRegistry)
               .increment();
               
        Timer.builder("message.size")
             .tag("topic", topic)
             .register(meterRegistry)
             .record(size, TimeUnit.BYTES);
    }
    
    public void recordLatency(String operation, long latencyMs) {
        Timer.builder("operation.latency")
             .tag("operation", operation)
             .register(meterRegistry)
             .record(latencyMs, TimeUnit.MILLISECONDS);
    }
    
    public void recordReplicationLag(String topic, int partition, long lagMs) {
        Gauge.builder("replication.lag")
             .tag("topic", topic)
             .tag("partition", String.valueOf(partition))
             .register(meterRegistry, lagMs);
    }
}
```

### Backup and Recovery

#### 1. Data Backup Strategy
```java
public class BackupManager {
    private final S3Client s3Client;
    private final String backupBucket;
    
    public void backupPartition(String topic, int partition) {
        String backupKey = String.format("backups/%s/%d/%s", 
                                        topic, partition, Instant.now().toString());
        
        // Create snapshot of partition data
        PartitionSnapshot snapshot = createSnapshot(topic, partition);
        
        // Compress and upload to S3
        byte[] compressedData = compress(snapshot.getData());
        
        PutObjectRequest request = PutObjectRequest.builder()
            .bucket(backupBucket)
            .key(backupKey)
            .build();
            
        s3Client.putObject(request, RequestBody.fromBytes(compressedData));
        
        // Store metadata
        BackupMetadata metadata = new BackupMetadata(
            topic, partition, backupKey, snapshot.getOffsetRange());
        storeBackupMetadata(metadata);
        
        log.info("Backup completed for {}/{}: {}", topic, partition, backupKey);
    }
    
    public void restorePartition(String topic, int partition, String backupKey) {
        // Download backup from S3
        GetObjectRequest request = GetObjectRequest.builder()
            .bucket(backupBucket)
            .key(backupKey)
            .build();
            
        ResponseInputStream<GetObjectResponse> response = s3Client.getObject(request);
        byte[] compressedData = response.readAllBytes();
        
        // Decompress and restore
        byte[] data = decompress(compressedData);
        PartitionSnapshot snapshot = PartitionSnapshot.fromBytes(data);
        
        // Restore partition data
        restorePartitionData(topic, partition, snapshot);
        
        log.info("Restore completed for {}/{} from {}", topic, partition, backupKey);
    }
    
    public void scheduleRegularBackups() {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);
        
        // Full backup weekly
        scheduler.scheduleAtFixedRate(this::performFullBackup, 
                                    0, 7, TimeUnit.DAYS);
        
        // Incremental backup daily
        scheduler.scheduleAtFixedRate(this::performIncrementalBackup, 
                                    0, 1, TimeUnit.DAYS);
    }
}
```

#### 2. Disaster Recovery
```java
public class DisasterRecoveryManager {
    
    public void initiateFailover(String primaryDataCenter, String backupDataCenter) {
        log.info("Initiating failover from {} to {}", primaryDataCenter, backupDataCenter);
        
        // 1. Stop writes to primary
        stopWrites(primaryDataCenter);
        
        // 2. Ensure all data is replicated to backup
        waitForReplicationSync(primaryDataCenter, backupDataCenter);
        
        // 3. Promote backup to primary
        promoteToActive(backupDataCenter);
        
        // 4. Update DNS/load balancer
        updateServiceEndpoints(backupDataCenter);
        
        // 5. Resume operations
        resumeOperations(backupDataCenter);
        
        log.info("Failover completed successfully");
    }
    
    public void setupCrossRegionReplication() {
        List<String> regions = Arrays.asList("us-east-1", "us-west-2", "eu-west-1");
        
        for (String region : regions) {
            ClusterConfig config = createClusterConfig(region);
            setupCluster(region, config);
            
            // Configure replication to other regions
            for (String targetRegion : regions) {
                if (!targetRegion.equals(region)) {
                    setupReplication(region, targetRegion);
                }
            }
        }
    }
}
```

### Performance Optimization

#### 1. Query Optimization
```java
public class QueryOptimizer {
    
    public OptimizedFetchPlan optimizeFetchPlan(FetchRequest request) {
        List<TopicPartition> partitions = request.getPartitions();
        
        // Group partitions by broker
        Map<Integer, List<TopicPartition>> partitionsByBroker = 
            partitions.stream()
                     .collect(Collectors.groupingBy(this::getBrokerForPartition));
        
        // Optimize fetch order and batching
        OptimizedFetchPlan plan = new OptimizedFetchPlan();
        
        for (Map.Entry<Integer, List<TopicPartition>> entry : partitionsByBroker.entrySet()) {
            int brokerId = entry.getKey();
            List<TopicPartition> brokerPartitions = entry.getValue();
            
            // Create optimized fetch requests for this broker
            List<FetchRequest> optimizedRequests = 
                createOptimizedFetchRequests(brokerPartitions, request);
            
            plan.addBrokerRequests(brokerId, optimizedRequests);
        }
        
        return plan;
    }
    
    private List<FetchRequest> createOptimizedFetchRequests(
            List<TopicPartition> partitions, FetchRequest originalRequest) {
        
        // Sort partitions by expected data size
        partitions.sort(Comparator.comparing(this::getExpectedDataSize));
        
        List<FetchRequest> requests = new ArrayList<>();
        List<TopicPartition> currentBatch = new ArrayList<>();
        int currentBatchSize = 0;
        
        for (TopicPartition partition : partitions) {
            int expectedSize = getExpectedDataSize(partition);
            
            if (currentBatchSize + expectedSize > MAX_FETCH_BYTES) {
                // Create request for current batch
                if (!currentBatch.isEmpty()) {
                    requests.add(createFetchRequest(currentBatch, originalRequest));
                    currentBatch.clear();
                    currentBatchSize = 0;
                }
            }
            
            currentBatch.add(partition);
            currentBatchSize += expectedSize;
        }
        
        // Handle remaining partitions
        if (!currentBatch.isEmpty()) {
            requests.add(createFetchRequest(currentBatch, originalRequest));
        }
        
        return requests;
    }
}
```

#### 2. Resource Management
```java
public class ResourceManager {
    private final MemoryPool memoryPool;
    private final ThreadPool threadPool;
    
    public void optimizeResourceAllocation() {
        // Monitor current resource usage
        ResourceUsage usage = getCurrentUsage();
        
        // Adjust memory allocation
        if (usage.getMemoryPressure() > 0.8) {
            increaseMemoryPool();
        } else if (usage.getMemoryPressure() < 0.4) {
            decreaseMemoryPool();
        }
        
        // Adjust thread pool size
        if (usage.getCpuUtilization() > 0.8 && usage.getQueueLength() > 100) {
            increaseThreadPoolSize();
        } else if (usage.getCpuUtilization() < 0.3) {
            decreaseThreadPoolSize();
        }
        
        // Adjust I/O parameters
        optimizeIOParameters(usage);
    }
    
    private void optimizeIOParameters(ResourceUsage usage) {
        if (usage.getDiskUtilization() > 0.8) {
            // Increase buffer sizes to reduce I/O frequency
            increaseBatchSizes();
            increaseFlushInterval();
        } else if (usage.getAverageLatency() > TARGET_LATENCY) {
            // Reduce latency at cost of throughput
            decreaseBatchSizes();
            decreaseFlushInterval();
        }
    }
}
```

This security and operations guide provides comprehensive coverage of production deployment considerations, including security hardening, operational procedures, monitoring strategies, and performance optimization techniques for the distributed message queue system.
