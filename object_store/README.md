# Distributed Object Store

A highly available, durable distributed object storage system designed to provide Amazon S3-like functionality with 99.9999% durability and 99.99% availability.

## Overview

This distributed object store provides:
- **REST API**: HTTP-based interface for object operations (GET, PUT, DELETE, LIST)
- **High Durability**: 99.9999% (six 9's) durability through erasure coding and replication
- **High Availability**: 99.99% availability with multi-region deployment
- **Scalable Storage**: Support for small files (KB) to large files (TB+)
- **Multi-tenant**: Bucket-based isolation with access control
- **Metadata Management**: Rich metadata support with efficient indexing

## Quick Start

### Core Operations

```bash
# Create a bucket
PUT /buckets/my-bucket

# Upload an object
PUT /buckets/my-bucket/objects/my-file.txt
Content-Type: text/plain
Content-Length: 1024

# Download an object
GET /buckets/my-bucket/objects/my-file.txt

# List objects
GET /buckets/my-bucket/objects?prefix=logs/&limit=100

# Delete an object
DELETE /buckets/my-bucket/objects/my-file.txt
```

## Architecture Highlights

- **Layered Architecture**: API Gateway → Metadata Service → Storage Nodes
- **Erasure Coding**: 6+3 configuration for optimal durability/storage efficiency
- **Consistent Hashing**: For data distribution and load balancing
- **Multi-Region**: Cross-region replication for disaster recovery
- **Auto-Scaling**: Dynamic scaling based on load and storage requirements

## Documentation

- [Architecture Design](ARCHITECTURE.md) - Core system design and components
- [Project Overview](PROJECT_OVERVIEW.md) - Documentation structure and contents
- [Implementation Guide](docs/implementation-guide.md) - Technical implementation details
- [API Reference](docs/api-reference.md) - Complete REST API documentation
- [Performance Analysis](docs/performance-analysis.md) - Benchmarking and optimization
- [Security Operations](docs/security-operations.md) - Security and operational procedures
- [Deployment Guide](docs/deployment-guide.md) - Production deployment instructions

## Key Features

### Durability (99.9999%)
- Erasure coding with 6+3 configuration
- Cross-region replication
- Automatic data verification and repair
- Multi-level checksums

### Availability (99.99%)
- Multi-zone deployment
- Automatic failover mechanisms
- Load balancing and circuit breakers
- Graceful degradation

### Performance
- Sub-100ms latency for small objects
- Multi-GB/s throughput for large objects
- Intelligent caching and prefetching
- Optimized for various file sizes

### Scalability
- Horizontal scaling of all components
- Consistent hashing for data distribution
- Auto-scaling based on metrics
- Support for petabyte-scale deployments
