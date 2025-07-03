# Performance Analysis and Benchmarks

## Overview

This document provides detailed performance analysis, benchmarking strategies, and optimization techniques for the distributed message queue system.

## Performance Characteristics

### Throughput Analysis

#### Producer Throughput
- **Single Producer**: 50,000 - 100,000 messages/second
- **Multiple Producers**: 500,000 - 1,000,000+ messages/second
- **Factors affecting throughput**:
  - Batch size (larger batches = higher throughput)
  - Compression (reduces network overhead)
  - Acknowledgment level (acks=0 > acks=1 > acks=all)
  - Number of partitions
  - Message size

#### Consumer Throughput
- **Single Consumer**: 100,000 - 200,000 messages/second
- **Consumer Group**: 1,000,000+ messages/second (with sufficient partitions)
- **Factors affecting throughput**:
  - Fetch batch size
  - Processing logic complexity
  - Number of partitions
  - Consumer parallelism

### Latency Analysis

#### End-to-End Latency Components
```
Producer → [Network] → Broker → [Replication] → [Network] → Consumer
    ↓           ↓         ↓          ↓            ↓         ↓
  1-2ms      1-5ms    1-10ms     5-50ms       1-5ms     1-2ms
```

#### Latency Optimization Strategies
1. **Producer-side**:
   - Use async sends
   - Optimize batch size vs linger time
   - Use local buffering
   - Reduce serialization overhead

2. **Broker-side**:
   - Use SSDs for active segments
   - Optimize JVM garbage collection
   - Use page cache effectively
   - Minimize network hops

3. **Consumer-side**:
   - Use prefetching
   - Optimize deserialization
   - Use parallel processing
   - Reduce offset commit frequency

## Benchmarking Framework

### Test Scenarios

#### 1. Throughput Benchmark
```java
public class ThroughputBenchmark {
    private static final int NUM_MESSAGES = 10_000_000;
    private static final int MESSAGE_SIZE = 1024; // 1KB
    private static final int NUM_PRODUCERS = 10;
    private static final int NUM_PARTITIONS = 50;
    
    @Test
    public void testProducerThroughput() {
        // Setup producers with different configurations
        List<Producer> producers = createProducers(NUM_PRODUCERS);
        
        long startTime = System.currentTimeMillis();
        
        // Send messages concurrently
        CountDownLatch latch = new CountDownLatch(NUM_PRODUCERS);
        for (Producer producer : producers) {
            executor.submit(() -> {
                try {
                    sendMessages(producer, NUM_MESSAGES / NUM_PRODUCERS);
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        long endTime = System.currentTimeMillis();
        
        double throughput = NUM_MESSAGES / ((endTime - startTime) / 1000.0);
        System.out.println("Throughput: " + throughput + " messages/sec");
    }
    
    @Test
    public void testConsumerThroughput() {
        // Create consumer group
        ConsumerGroup group = new ConsumerGroup("benchmark-group");
        List<Consumer> consumers = createConsumers(group, NUM_PARTITIONS);
        
        long startTime = System.currentTimeMillis();
        AtomicLong processedMessages = new AtomicLong(0);
        
        // Start consuming
        for (Consumer consumer : consumers) {
            executor.submit(() -> {
                while (processedMessages.get() < NUM_MESSAGES) {
                    ConsumerRecords records = consumer.poll(Duration.ofMillis(100));
                    processedMessages.addAndGet(records.count());
                }
            });
        }
        
        // Wait for completion
        while (processedMessages.get() < NUM_MESSAGES) {
            Thread.sleep(100);
        }
        
        long endTime = System.currentTimeMillis();
        double throughput = NUM_MESSAGES / ((endTime - startTime) / 1000.0);
        System.out.println("Consumer Throughput: " + throughput + " messages/sec");
    }
}
```

#### 2. Latency Benchmark
```java
public class LatencyBenchmark {
    private static final int NUM_SAMPLES = 100_000;
    
    @Test
    public void testEndToEndLatency() {
        Producer producer = createProducer();
        Consumer consumer = createConsumer();
        
        List<Long> latencies = new ArrayList<>();
        CountDownLatch latch = new CountDownLatch(NUM_SAMPLES);
        
        // Consumer thread
        executor.submit(() -> {
            while (latch.getCount() > 0) {
                ConsumerRecords records = consumer.poll(Duration.ofMillis(10));
                for (ConsumerRecord record : records) {
                    long sendTime = Long.parseLong(record.value());
                    long latency = System.currentTimeMillis() - sendTime;
                    latencies.add(latency);
                    latch.countDown();
                }
            }
        });
        
        // Producer thread
        executor.submit(() -> {
            for (int i = 0; i < NUM_SAMPLES; i++) {
                String timestamp = String.valueOf(System.currentTimeMillis());
                producer.send(new ProducerRecord("test-topic", timestamp));
                Thread.sleep(10); // Send at 100 messages/sec
            }
        });
        
        latch.await();
        
        // Calculate percentiles
        Collections.sort(latencies);
        System.out.println("P50: " + latencies.get(NUM_SAMPLES / 2) + "ms");
        System.out.println("P95: " + latencies.get((int)(NUM_SAMPLES * 0.95)) + "ms");
        System.out.println("P99: " + latencies.get((int)(NUM_SAMPLES * 0.99)) + "ms");
    }
}
```

#### 3. Scalability Benchmark
```java
public class ScalabilityBenchmark {
    
    @Test
    public void testHorizontalScaling() {
        int[] brokerCounts = {1, 3, 5, 7, 9};
        int[] partitionCounts = {10, 50, 100, 200, 500};
        
        for (int brokers : brokerCounts) {
            for (int partitions : partitionCounts) {
                ClusterSetup cluster = setupCluster(brokers, partitions);
                
                double throughput = measureThroughput(cluster);
                double latency = measureLatency(cluster);
                
                System.out.printf("Brokers: %d, Partitions: %d, " +
                                "Throughput: %.2f msgs/sec, Latency: %.2f ms%n",
                                brokers, partitions, throughput, latency);
                
                cluster.shutdown();
            }
        }
    }
}
```

### Stress Testing

#### 1. High Load Test
```java
public class StressTest {
    
    @Test
    public void testHighLoadStability() {
        // Create high load scenario
        int numProducers = 100;
        int numConsumers = 50;
        int messagesPerProducer = 1_000_000;
        
        // Monitor system resources
        SystemMonitor monitor = new SystemMonitor();
        monitor.start();
        
        try {
            // Run high load test for extended period
            runHighLoadTest(numProducers, numConsumers, messagesPerProducer);
            
            // Verify system stability
            assertSystemStability(monitor);
            
        } finally {
            monitor.stop();
        }
    }
    
    @Test
    public void testMemoryLeaks() {
        // Run continuous load and monitor memory usage
        MemoryMonitor memMonitor = new MemoryMonitor();
        
        for (int i = 0; i < 1000; i++) {
            runBatch(1000); // Send 1000 messages
            
            if (i % 100 == 0) {
                System.gc();
                long memoryUsage = memMonitor.getUsedMemory();
                System.out.println("Memory usage after " + (i * 1000) + 
                                 " messages: " + memoryUsage + " MB");
                
                // Assert memory usage is not continuously growing
                assertMemoryStable(memoryUsage);
            }
        }
    }
}
```

#### 2. Failure Scenarios
```java
public class FailureTest {
    
    @Test
    public void testBrokerFailure() {
        Cluster cluster = setupCluster(3); // 3 brokers
        
        // Start producing messages
        Producer producer = createProducer();
        startContinuousProducing(producer);
        
        // Kill one broker
        cluster.killBroker(1);
        
        // Verify system continues to operate
        assertSystemOperational(cluster);
        assertNoMessageLoss();
        
        // Restart broker
        cluster.startBroker(1);
        
        // Verify recovery
        assertFullRecovery(cluster);
    }
    
    @Test
    public void testNetworkPartition() {
        Cluster cluster = setupCluster(5);
        
        // Create network partition (2 brokers isolated)
        cluster.partitionNetwork(Arrays.asList(1, 2), Arrays.asList(3, 4, 5));
        
        // Verify split-brain prevention
        assertSingleActiveController(cluster);
        assertConsistency(cluster);
        
        // Heal partition
        cluster.healNetworkPartition();
        
        // Verify recovery
        assertConsistency(cluster);
    }
}
```

## Performance Tuning Guide

### JVM Tuning

#### Garbage Collection Optimization
```bash
# G1GC settings for low-latency workloads
-XX:+UseG1GC
-XX:MaxGCPauseMillis=20
-XX:InitiatingHeapOccupancyPercent=35
-XX:+ExplicitGCInvokesConcurrent
-XX:MaxInlineLevel=15

# Heap sizing
-Xms8g
-Xmx8g
-XX:NewRatio=1

# GC logging
-Xloggc:gc.log
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+UseGCLogFileRotation
```

#### Memory Management
```bash
# Direct memory for zero-copy operations
-XX:MaxDirectMemorySize=2g

# Reduce object allocation overhead
-XX:+UseTLAB
-XX:TLABSize=1m

# Compressed OOPs for heap < 32GB
-XX:+UseCompressedOops
```

### Operating System Tuning

#### File System
```bash
# Mount options for performance
mount -o noatime,data=writeback /dev/sda1 /var/kafka-logs

# Increase file descriptor limits
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf

# Kernel parameters
echo 'vm.swappiness=1' >> /etc/sysctl.conf
echo 'vm.dirty_background_ratio=5' >> /etc/sysctl.conf
echo 'vm.dirty_ratio=60' >> /etc/sysctl.conf
echo 'vm.dirty_expire_centisecs=12000' >> /etc/sysctl.conf
```

#### Network Tuning
```bash
# TCP buffer sizes
echo 'net.core.rmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' >> /etc/sysctl.conf

# TCP window scaling
echo 'net.ipv4.tcp_window_scaling = 1' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 65536 16777216' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 16777216' >> /etc/sysctl.conf
```

### Broker Configuration Tuning

#### High Throughput Configuration
```properties
# Network threads
num.network.threads=16
num.io.threads=16

# Socket buffers
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600

# Log settings
log.segment.bytes=1073741824
log.flush.interval.messages=10000
log.flush.interval.ms=1000

# Replication
num.replica.fetchers=4
replica.fetch.max.bytes=1048576
```

#### Low Latency Configuration
```properties
# Reduce batching
log.flush.interval.messages=1
log.flush.interval.ms=1

# Faster leader election
controller.socket.timeout.ms=30000
controller.message.queue.size=10

# Network optimizations
socket.send.buffer.bytes=131072
socket.receive.buffer.bytes=131072
```

### Client Configuration Tuning

#### Producer Tuning
```properties
# High throughput
batch.size=65536
linger.ms=100
compression.type=lz4
buffer.memory=67108864

# Low latency
batch.size=1
linger.ms=0
compression.type=none
```

#### Consumer Tuning
```properties
# High throughput
fetch.min.bytes=65536
fetch.max.wait.ms=500
max.partition.fetch.bytes=1048576

# Low latency
fetch.min.bytes=1
fetch.max.wait.ms=1
```

## Monitoring and Alerting

### Key Performance Metrics

#### Broker Metrics
```java
public class BrokerMetrics {
    // Throughput metrics
    @Metric("messages_in_per_sec")
    public double getMessagesInPerSecond();
    
    @Metric("bytes_in_per_sec")
    public double getBytesInPerSecond();
    
    @Metric("bytes_out_per_sec")
    public double getBytesOutPerSecond();
    
    // Latency metrics
    @Metric("request_local_time_ms")
    public Histogram getRequestLocalTime();
    
    @Metric("request_remote_time_ms")
    public Histogram getRequestRemoteTime();
    
    // Resource metrics
    @Metric("active_controller_count")
    public int getActiveControllerCount();
    
    @Metric("offline_partitions_count")
    public int getOfflinePartitionsCount();
    
    @Metric("under_replicated_partitions")
    public int getUnderReplicatedPartitions();
}
```

#### Performance Alerts
```yaml
# Prometheus alerting rules
groups:
- name: message-queue-performance
  rules:
  - alert: HighLatency
    expr: histogram_quantile(0.95, request_local_time_ms) > 100
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High request latency detected"
      
  - alert: LowThroughput
    expr: rate(messages_in_per_sec[5m]) < 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Low message throughput detected"
      
  - alert: ConsumerLag
    expr: consumer_lag > 100000
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "High consumer lag detected"
```

### Performance Monitoring Dashboard

#### Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "title": "Message Queue Performance",
    "panels": [
      {
        "title": "Throughput",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(messages_in_per_sec[1m])",
            "legendFormat": "Messages In/sec"
          },
          {
            "expr": "rate(messages_out_per_sec[1m])",
            "legendFormat": "Messages Out/sec"
          }
        ]
      },
      {
        "title": "Latency Distribution",
        "type": "heatmap",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, request_local_time_ms)",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, request_local_time_ms)",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, request_local_time_ms)",
            "legendFormat": "P99"
          }
        ]
      }
    ]
  }
}
```

This performance analysis framework provides comprehensive tooling for measuring, optimizing, and monitoring the distributed message queue system's performance characteristics.
