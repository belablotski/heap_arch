# OpenTelemetry Observability

## Overview

Our distributed file system leverages OpenTelemetry (OTel) for comprehensive observability across all components. This provides unified telemetry collection, correlation, and analysis for traces, metrics, and logs.

## Why OpenTelemetry for DFS?

### 1. **Distributed Request Tracing**
Track file operations across multiple services:
```
Upload Request Flow:
Client → Gateway → Metadata → Dedup → Compression → Storage
  ↓        ↓          ↓        ↓         ↓           ↓
Span1   Span2      Span3    Span4     Span5       Span6
```

### 2. **Performance Bottleneck Detection**
Identify slow components in the storage optimization pipeline:
- Deduplication lookup times
- Compression algorithm performance
- Network latency between services
- Storage I/O performance

### 3. **Storage Efficiency Analytics**
Track optimization effectiveness with detailed metrics:
- Deduplication ratios per file type
- Compression performance by algorithm
- Cache hit rates across tiers

## OpenTelemetry Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    DFS Services with OTel SDKs                  │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Gateway       │   Metadata      │     Storage                 │
│   Service       │   Service       │     Service                 │
│   + OTel SDK    │   + OTel SDK    │     + OTel SDK              │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                  OTel Collector (Agent Mode)                    │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Receivers     │   Processors    │     Exporters               │
│   (OTLP, Jaeger)│   (Batch, Mem)  │     (Jaeger, Prometheus)    │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────────┐
│                    Observability Backends                       │
├─────────────────┬─────────────────┬─────────────────────────────┤
│   Jaeger        │   Prometheus    │     Elasticsearch           │
│   (Traces)      │   (Metrics)     │     (Logs)                  │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

## Implementation

### 1. Service Instrumentation

#### Go Services (Gateway, Metadata, Dedup)
```go
package main

import (
    "context"
    "log"
    
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.17.0"
)

type DFSService struct {
    tracer  trace.Tracer
    meter   metric.Meter
    logger  *slog.Logger
}

func initOpenTelemetry(serviceName string) (*trace.TracerProvider, error) {
    // Create resource
    res := resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName(serviceName),
        semconv.ServiceVersion("v1.0.0"),
        semconv.DeploymentEnvironment("production"),
        attribute.String("dfs.component", serviceName),
    )
    
    // Create trace exporter
    traceExporter, err := otlptracehttp.New(context.Background(),
        otlptracehttp.WithEndpoint("http://otel-collector:4318"),
        otlptracehttp.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    // Create trace provider
    tp := trace.NewTracerProvider(
        trace.WithBatcher(traceExporter),
        trace.WithResource(res),
        trace.WithSampler(trace.TraceIDRatioBased(0.1)), // 10% sampling
    )
    
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.TraceContext{})
    
    return tp, nil
}

// File upload with comprehensive tracing
func (s *DFSService) UploadFile(ctx context.Context, req *UploadRequest) (*UploadResponse, error) {
    ctx, span := s.tracer.Start(ctx, "dfs.upload_file",
        trace.WithAttributes(
            attribute.String("file.path", req.Path),
            attribute.Int64("file.size", req.Size),
            attribute.String("user.id", req.UserID),
            attribute.String("content.type", req.ContentType),
        ),
    )
    defer span.End()
    
    // Add file size to metrics
    s.fileSizeHistogram.Record(ctx, req.Size,
        metric.WithAttributes(
            attribute.String("operation", "upload"),
            attribute.String("content.type", req.ContentType),
        ),
    )
    
    // Chunk the file
    chunks, err := s.chunkFile(ctx, req.Data)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "chunking failed")
        return nil, err
    }
    
    span.SetAttributes(attribute.Int("chunks.count", len(chunks)))
    
    // Process each chunk with child spans
    var processedChunks []*ProcessedChunk
    for i, chunk := range chunks {
        chunkCtx, chunkSpan := s.tracer.Start(ctx, "dfs.process_chunk",
            trace.WithAttributes(
                attribute.Int("chunk.index", i),
                attribute.String("chunk.hash", chunk.Hash),
                attribute.Int64("chunk.size", chunk.Size),
            ),
        )
        
        processed, err := s.processChunk(chunkCtx, chunk)
        if err != nil {
            chunkSpan.RecordError(err)
            chunkSpan.SetStatus(codes.Error, "chunk processing failed")
            chunkSpan.End()
            continue
        }
        
        chunkSpan.SetAttributes(
            attribute.Bool("chunk.deduplicated", processed.WasDeduplicated),
            attribute.String("compression.algorithm", processed.CompressionAlgorithm),
            attribute.Float64("compression.ratio", processed.CompressionRatio),
        )
        chunkSpan.End()
        
        processedChunks = append(processedChunks, processed)
    }
    
    // Calculate final metrics
    deduplicationRatio := s.calculateDeduplicationRatio(processedChunks)
    totalCompressionRatio := s.calculateCompressionRatio(processedChunks)
    
    span.SetAttributes(
        attribute.Float64("optimization.deduplication_ratio", deduplicationRatio),
        attribute.Float64("optimization.compression_ratio", totalCompressionRatio),
        attribute.Int("chunks.deduplicated", s.countDeduplicated(processedChunks)),
        attribute.Int("chunks.new", s.countNew(processedChunks)),
    )
    
    // Record storage efficiency metrics
    s.deduplicationRatio.Record(ctx, deduplicationRatio)
    s.compressionRatio.Record(ctx, totalCompressionRatio)
    
    span.SetStatus(codes.Ok, "upload completed successfully")
    return &UploadResponse{
        FileID: generateFileID(),
        Size:   req.Size,
        Chunks: len(processedChunks),
    }, nil
}
```

#### Rust Storage Engine Instrumentation
```rust
use opentelemetry::{
    global,
    trace::{Tracer, TraceContextExt},
    Context, KeyValue,
};
use opentelemetry_otlp::WithExportConfig;
use tracing::{info, instrument, span, Level};
use tracing_opentelemetry::OpenTelemetrySpanExt;

pub struct StorageEngine {
    tracer: Box<dyn Tracer + Send + Sync>,
}

impl StorageEngine {
    #[instrument(
        name = "storage.write_chunk",
        fields(
            chunk.hash = %chunk_hash,
            chunk.size = chunk_data.len(),
            storage.tier = %tier
        )
    )]
    pub async fn write_chunk(
        &self,
        chunk_hash: &str,
        chunk_data: &[u8],
        tier: StorageTier,
    ) -> Result<WriteResult, StorageError> {
        let span = span!(Level::INFO, "chunk_write");
        let _enter = span.enter();
        
        // Start compression with child span
        let compression_result = self.compress_chunk(chunk_data, tier).await?;
        
        span.record("compression.algorithm", &compression_result.algorithm.as_str());
        span.record("compression.ratio", &compression_result.ratio);
        span.record("compressed.size", &compression_result.compressed_size);
        
        // Write to storage with another child span
        let write_start = std::time::Instant::now();
        let storage_result = self.write_to_storage(
            chunk_hash,
            &compression_result.data,
            tier,
        ).await?;
        let write_duration = write_start.elapsed();
        
        span.record("storage.write_duration_ms", &write_duration.as_millis());
        span.record("storage.location", &storage_result.location.as_str());
        
        // Record metrics
        self.record_write_metrics(chunk_data.len(), compression_result.ratio, write_duration);
        
        info!(
            chunk_hash = %chunk_hash,
            compressed_size = compression_result.compressed_size,
            write_duration_ms = write_duration.as_millis(),
            "Chunk written successfully"
        );
        
        Ok(WriteResult {
            location: storage_result.location,
            compressed_size: compression_result.compressed_size,
            compression_ratio: compression_result.ratio,
        })
    }
}
```

### 2. Custom Metrics for Storage Optimization

```go
type DFSMetrics struct {
    // File operation metrics
    fileUploads         metric.Int64Counter
    fileDownloads       metric.Int64Counter
    fileSizeHistogram   metric.Int64Histogram
    
    // Storage efficiency metrics
    deduplicationRatio  metric.Float64Histogram
    compressionRatio    metric.Float64Histogram
    storageEfficiency   metric.Float64Gauge
    
    // Performance metrics
    chunkProcessingTime metric.Int64Histogram
    uploadLatency       metric.Int64Histogram
    downloadLatency     metric.Int64Histogram
    
    // Infrastructure metrics
    diskUtilization     metric.Float64Gauge
    networkThroughput   metric.Int64Counter
    cacheHitRate        metric.Float64Gauge
}

func initMetrics() (*DFSMetrics, error) {
    meter := otel.Meter("dfs")
    
    fileUploads, err := meter.Int64Counter("dfs.file.uploads.total",
        metric.WithDescription("Total number of file uploads"),
        metric.WithUnit("1"),
    )
    if err != nil {
        return nil, err
    }
    
    deduplicationRatio, err := meter.Float64Histogram("dfs.deduplication.ratio",
        metric.WithDescription("Deduplication ratio achieved"),
        metric.WithUnit("1"),
        metric.WithExplicitBucketBoundaries(0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0),
    )
    if err != nil {
        return nil, err
    }
    
    return &DFSMetrics{
        fileUploads:        fileUploads,
        deduplicationRatio: deduplicationRatio,
        // ... initialize other metrics
    }, nil
}

// Record comprehensive metrics for each operation
func (m *DFSMetrics) RecordUpload(ctx context.Context, req *UploadRequest, result *UploadResult) {
    labels := []attribute.KeyValue{
        attribute.String("content.type", req.ContentType),
        attribute.String("storage.tier", string(result.StorageTier)),
        attribute.Bool("deduplicated", result.WasDeduplicated),
    }
    
    m.fileUploads.Add(ctx, 1, metric.WithAttributes(labels...))
    m.fileSizeHistogram.Record(ctx, req.Size, metric.WithAttributes(labels...))
    m.deduplicationRatio.Record(ctx, result.DeduplicationRatio, metric.WithAttributes(labels...))
    m.compressionRatio.Record(ctx, result.CompressionRatio, metric.WithAttributes(labels...))
    m.uploadLatency.Record(ctx, result.Duration.Milliseconds(), metric.WithAttributes(labels...))
}
```

### 3. Structured Logging with Correlation

```go
import (
    "log/slog"
    "go.opentelemetry.io/otel/trace"
)

type CorrelatedLogger struct {
    logger *slog.Logger
}

func (cl *CorrelatedLogger) InfoCtx(ctx context.Context, msg string, args ...any) {
    spanCtx := trace.SpanContextFromContext(ctx)
    if spanCtx.IsValid() {
        args = append(args,
            slog.String("trace_id", spanCtx.TraceID().String()),
            slog.String("span_id", spanCtx.SpanID().String()),
        )
    }
    cl.logger.InfoContext(ctx, msg, args...)
}

// Usage in services
func (s *DeduplicationService) ProcessChunk(ctx context.Context, chunk *Chunk) error {
    s.logger.InfoCtx(ctx, "Processing chunk for deduplication",
        slog.String("chunk.hash", chunk.Hash),
        slog.Int64("chunk.size", chunk.Size),
        slog.String("file.path", chunk.FilePath),
    )
    
    exists, err := s.bloomFilter.MightContain(chunk.Hash)
    if err != nil {
        s.logger.ErrorCtx(ctx, "Bloom filter check failed",
            slog.String("chunk.hash", chunk.Hash),
            slog.String("error", err.Error()),
        )
        return err
    }
    
    s.logger.InfoCtx(ctx, "Bloom filter result",
        slog.String("chunk.hash", chunk.Hash),
        slog.Bool("might_exist", exists),
    )
    
    return nil
}
```

### 4. OTel Collector Configuration

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
        
  prometheus:
    config:
      scrape_configs:
        - job_name: 'dfs-services'
          static_configs:
            - targets: ['gateway:8080', 'metadata:8081', 'storage:8082']

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
    send_batch_max_size: 2048
    
  memory_limiter:
    limit_mib: 512
    
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert
      - key: cluster
        value: dfs-cluster
        action: upsert

exporters:
  # Traces to Jaeger
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
      
  # Metrics to Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: dfs
    
  # Logs to Elasticsearch
  elasticsearch:
    endpoints: ["http://elasticsearch:9200"]
    index: "dfs-logs"
    
  # Debug exporter for troubleshooting
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [jaeger, logging]
      
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch, resource]
      exporters: [prometheus]
      
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [elasticsearch, logging]
```

### 5. Kubernetes Deployment with OTel

```yaml
# k8s/otel-collector.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: dfs-system
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector-contrib:latest
        args:
          - "--config=/etc/otelcol-contrib/otel-collector-config.yaml"
        volumeMounts:
        - name: otel-collector-config-vol
          mountPath: /etc/otelcol-contrib/
        env:
        - name: K8S_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        resources:
          limits:
            memory: 512Mi
            cpu: 500m
          requests:
            memory: 256Mi
            cpu: 250m
      volumes:
      - name: otel-collector-config-vol
        configMap:
          name: otel-collector-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: dfs-system
data:
  otel-collector-config.yaml: |
    # Include the collector config from above
```

### 6. Service Deployment with OTel SDK

```yaml
# k8s/gateway-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dfs-gateway
  namespace: dfs-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dfs-gateway
  template:
    metadata:
      labels:
        app: dfs-gateway
    spec:
      containers:
      - name: gateway
        image: dfs/gateway:latest
        env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://otel-collector:4318"
        - name: OTEL_SERVICE_NAME
          value: "dfs-gateway"
        - name: OTEL_SERVICE_VERSION
          value: "v1.0.0"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "service.name=dfs-gateway,service.version=v1.0.0,deployment.environment=production"
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8081
          name: metrics
```

## Observability Dashboards

### 1. Grafana Dashboard for Storage Efficiency

```json
{
  "dashboard": {
    "title": "DFS Storage Optimization",
    "panels": [
      {
        "title": "Deduplication Ratio",
        "type": "stat",
        "targets": [
          {
            "expr": "avg(dfs_deduplication_ratio)",
            "legendFormat": "Current Ratio"
          }
        ]
      },
      {
        "title": "Compression Efficiency by Algorithm",
        "type": "graph",
        "targets": [
          {
            "expr": "avg by (algorithm) (dfs_compression_ratio)",
            "legendFormat": "{{algorithm}}"
          }
        ]
      },
      {
        "title": "Storage Tier Distribution",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (tier) (dfs_chunks_total)",
            "legendFormat": "{{tier}}"
          }
        ]
      }
    ]
  }
}
```

### 2. Jaeger Trace Analysis

Custom trace queries for common scenarios:
```
# Find slow upload operations
service="dfs-gateway" operation="dfs.upload_file" duration>5s

# Analyze deduplication performance
service="dfs-dedup" tag="chunks.deduplicated:>50"

# Track compression algorithm effectiveness
service="dfs-compression" tag="compression.ratio:>3.0"
```

## Benefits of OTel Integration

### 1. **End-to-End Visibility**
- Complete request flow from client to storage
- Cross-service dependency mapping
- Performance bottleneck identification

### 2. **Storage Optimization Insights**
- Real-time deduplication effectiveness
- Compression algorithm performance comparison
- Data tiering decision analysis

### 3. **Operational Excellence**
- Automated SLA monitoring
- Proactive issue detection
- Capacity planning insights

### 4. **Debugging Capabilities**
- Distributed trace correlation
- Error root cause analysis
- Performance regression detection

This comprehensive OpenTelemetry integration transforms our distributed file system into a fully observable system, enabling data-driven optimization decisions and proactive operational management.
