# Distributed Message Queue Architecture

A comprehensive design for a scalable, reliable, and high-performance distributed message queue system.

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Components](#architecture-components)
3. [Design Deep Dive](#design-deep-dive)
4. [Implementation Details](#implementation-details)
5. [Deployment & Operations](#deployment--operations)

## System Overview

This distributed message queue system is designed with the following core principles:
- **Availability**: High availability through replication and fault tolerance
- **Reliability**: Guaranteed message delivery with configurable semantics
- **Scalability**: Horizontal scaling through partitioning and clustering
- **Performance**: Optimized for high throughput and low latency

### Key Features

- Publish-Subscribe and Point-to-Point messaging patterns
- Configurable message delivery semantics (at-least-once, at-most-once, exactly-once)
- Message ordering guarantees within partitions
- Configurable data retention policies
- Consumer groups with automatic load balancing
- High availability through replication
- Monitoring and observability

## Documentation

### Core Documentation
- **[Architecture Design](./ARCHITECTURE.md)** - Comprehensive system architecture and design decisions
- **[Implementation Guide](./docs/implementation-guide.md)** - Detailed implementation guidance with code examples
- **[API Reference](./docs/api-reference.md)** - Complete API documentation for clients and admin interfaces
- **[Performance Analysis](./docs/performance-analysis.md)** - Performance benchmarking and optimization guide
- **[Security & Operations](./docs/security-operations.md)** - Security configurations and operational procedures
- **[Deployment Guide](./docs/deployment-guide.md)** - Production deployment instructions and configurations

### Key Features Implemented

✅ **High Availability & Reliability**
- Leader-follower replication with configurable replication factor
- In-Sync Replica (ISR) management for data consistency
- Automatic failover and leader election
- Configurable message delivery semantics (at-least-once, at-most-once, exactly-once)

✅ **Scalability & Performance**
- Horizontal scaling through partitioning
- Producer batching and compression
- Zero-copy transfers for optimal performance
- Consumer groups with automatic load balancing
- Memory-mapped files for efficient I/O

✅ **Message Ordering & Persistence**
- FIFO ordering within partitions
- Durable message storage with configurable retention
- Offset-based message positioning
- Write-ahead log (WAL) architecture

✅ **Security**
- SASL authentication (PLAIN, SCRAM-SHA-256)
- OAuth 2.0 integration
- TLS encryption for data in transit
- Encryption at rest with key rotation
- Fine-grained Access Control Lists (ACLs)

✅ **Monitoring & Observability**
- Comprehensive metrics collection
- Health checks and alerting
- Grafana dashboards
- Performance monitoring and bottleneck detection

## Quick Start

1. **Clone and Setup**
   ```bash
   git clone <repository>
   cd heap_arch
   ```

2. **Docker Deployment**
   ```bash
   # Start the cluster
   docker-compose up -d
   
   # Verify cluster health
   curl http://localhost:8080/health
   ```

3. **Create a Topic**
   ```bash
   curl -X POST http://localhost:8080/api/v1/topics \
     -H "Content-Type: application/json" \
     -d '{
       "name": "test-topic",
       "partitions": 10,
       "replication_factor": 3
     }'
   ```

4. **Producer Example**
   ```java
   Properties props = new Properties();
   props.put("bootstrap.servers", "localhost:9092");
   props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
   props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
   
   try (MessageProducer producer = new MessageProducer(props)) {
       ProducerRecord record = new ProducerRecord("test-topic", "key1", "Hello World!");
       producer.send(record);
   }
   ```

5. **Consumer Example**
   ```java
   Properties props = new Properties();
   props.put("bootstrap.servers", "localhost:9092");
   props.put("group.id", "test-group");
   props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
   props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
   
   try (MessageConsumer consumer = new MessageConsumer(props)) {
       consumer.subscribe(Arrays.asList("test-topic"));
       while (true) {
           ConsumerRecords records = consumer.poll(Duration.ofMillis(100));
           for (ConsumerRecord record : records) {
               System.out.printf("Received: %s%n", record.value());
           }
       }
   }
   ```

## Performance Characteristics

- **Throughput**: 1M+ messages/second with proper tuning
- **Latency**: Sub-millisecond to low milliseconds end-to-end
- **Scalability**: Horizontal scaling to hundreds of brokers
- **Durability**: Configurable replication with strong consistency guarantees