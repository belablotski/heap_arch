# Implementation Guide

## Overview

This guide provides step-by-step instructions for implementing the metrics monitoring and alerting system, including infrastructure setup, component deployment, and configuration.

## Prerequisites

### Infrastructure Requirements

```yaml
# infrastructure-requirements.yml
minimum_infrastructure:
  kafka_cluster:
    nodes: 3
    memory_per_node: "16GB"
    disk_per_node: "500GB SSD"
    cpu_per_node: "4 cores"
    
  timescaledb_cluster:
    primary_nodes: 2
    replica_nodes: 2
    memory_per_node: "32GB"
    disk_per_node: "1TB SSD"
    cpu_per_node: "8 cores"
    
  clickhouse_cluster:
    shards: 3
    replicas: 2
    memory_per_node: "64GB"
    disk_per_node: "2TB SSD"
    cpu_per_node: "16 cores"
    
  redis_cluster:
    nodes: 6
    memory_per_node: "32GB"
    disk_per_node: "200GB SSD"
    cpu_per_node: "4 cores"
    
  application_servers:
    collectors: 4
    alert_engines: 2
    gateways: 3
    memory_per_server: "16GB"
    cpu_per_server: "8 cores"
```

### Software Dependencies

```bash
# Required software versions
Docker >= 20.10
Kubernetes >= 1.21
PostgreSQL >= 14 (for TimescaleDB)
ClickHouse >= 22.0
Redis >= 7.0
Apache Kafka >= 3.0
Python >= 3.9
Node.js >= 16 (for dashboards)
```

## Phase 1: Core Infrastructure Setup

### Step 1: Kafka Cluster Deployment

```yaml
# kafka-cluster.yml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: metrics-kafka
  namespace: monitoring
spec:
  kafka:
    version: 3.0.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.0"
    storage:
      type: persistent-claim
      size: 500Gi
      class: fast-ssd
    resources:
      requests:
        memory: 16Gi
        cpu: "4"
      limits:
        memory: 16Gi
        cpu: "4"
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Gi
      class: fast-ssd
    resources:
      requests:
        memory: 4Gi
        cpu: "2"
      limits:
        memory: 4Gi
        cpu: "2"
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

```bash
# Deploy Kafka cluster
kubectl apply -f https://strimzi.io/install/latest?namespace=monitoring
kubectl apply -f kafka-cluster.yml

# Wait for cluster to be ready
kubectl wait kafka/metrics-kafka --for=condition=Ready --timeout=300s -n monitoring
```

### Step 2: Create Kafka Topics

```bash
# create-kafka-topics.sh
#!/bin/bash

KAFKA_CLUSTER="metrics-kafka"
NAMESPACE="monitoring"

# Create metrics topic with partitions based on metric names
kubectl exec -it ${KAFKA_CLUSTER}-kafka-0 -n ${NAMESPACE} -- bin/kafka-topics.sh \
  --create \
  --topic metrics-raw \
  --partitions 20 \
  --replication-factor 3 \
  --config retention.ms=86400000 \
  --config compression.type=snappy \
  --bootstrap-server localhost:9092

# Create alerts topic
kubectl exec -it ${KAFKA_CLUSTER}-kafka-0 -n ${NAMESPACE} -- bin/kafka-topics.sh \
  --create \
  --topic alerts \
  --partitions 10 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --bootstrap-server localhost:9092

# Create aggregated metrics topic
kubectl exec -it ${KAFKA_CLUSTER}-kafka-0 -n ${NAMESPACE} -- bin/kafka-topics.sh \
  --create \
  --topic metrics-aggregated \
  --partitions 10 \
  --replication-factor 3 \
  --config retention.ms=2592000000 \
  --bootstrap-server localhost:9092

echo "Kafka topics created successfully"
```

### Step 3: TimescaleDB Setup (Hot Storage)

```yaml
# timescaledb-deployment.yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: timescaledb
  namespace: monitoring
spec:
  serviceName: timescaledb
  replicas: 2
  selector:
    matchLabels:
      app: timescaledb
  template:
    metadata:
      labels:
        app: timescaledb
    spec:
      containers:
      - name: timescaledb
        image: timescale/timescaledb-ha:pg14-latest
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "metrics"
        - name: POSTGRES_USER
          value: "metrics_user"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: timescaledb-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            memory: "32Gi"
            cpu: "8"
          limits:
            memory: "32Gi"
            cpu: "8"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 1Ti
```

```bash
# Deploy TimescaleDB
kubectl apply -f timescaledb-deployment.yml

# Create database schema
kubectl exec -it timescaledb-0 -n monitoring -- psql -U metrics_user -d metrics -c "
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE metrics_hot (
    time TIMESTAMPTZ NOT NULL,
    metric_name TEXT NOT NULL,
    value DOUBLE PRECISION NOT NULL,
    labels JSONB NOT NULL,
    host TEXT,
    region TEXT,
    environment TEXT
);

SELECT create_hypertable('metrics_hot', 'time', chunk_time_interval => INTERVAL '1 hour');

CREATE INDEX idx_metrics_hot_name_time ON metrics_hot (metric_name, time DESC);
CREATE INDEX idx_metrics_hot_host_time ON metrics_hot (host, time DESC);
CREATE INDEX idx_metrics_hot_labels ON metrics_hot USING GIN (labels);

ALTER TABLE metrics_hot SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'metric_name, host',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('metrics_hot', INTERVAL '2 hours');
SELECT add_retention_policy('metrics_hot', INTERVAL '7 days');
"
```

### Step 4: ClickHouse Cluster Setup (Warm Storage)

```yaml
# clickhouse-cluster.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clickhouse-config
  namespace: monitoring
data:
  config.xml: |
    <yandex>
        <logger>
            <level>information</level>
            <console>true</console>
        </logger>
        <http_port>8123</http_port>
        <tcp_port>9000</tcp_port>
        <listen_host>::</listen_host>
        <max_connections>1000</max_connections>
        <keep_alive_timeout>3</keep_alive_timeout>
        <max_concurrent_queries>100</max_concurrent_queries>
        <uncompressed_cache_size>8589934592</uncompressed_cache_size>
        <mark_cache_size>5368709120</mark_cache_size>
        <path>/var/lib/clickhouse/</path>
        <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
        <users_config>users.xml</users_config>
        <default_profile>default</default_profile>
        <default_database>default</default_database>
        <remote_servers>
            <metrics_cluster>
                <shard>
                    <replica>
                        <host>clickhouse-0.clickhouse.monitoring.svc.cluster.local</host>
                        <port>9000</port>
                    </replica>
                    <replica>
                        <host>clickhouse-1.clickhouse.monitoring.svc.cluster.local</host>
                        <port>9000</port>
                    </replica>
                </shard>
                <shard>
                    <replica>
                        <host>clickhouse-2.clickhouse.monitoring.svc.cluster.local</host>
                        <port>9000</port>
                    </replica>
                    <replica>
                        <host>clickhouse-3.clickhouse.monitoring.svc.cluster.local</host>
                        <port>9000</port>
                    </replica>
                </shard>
            </metrics_cluster>
        </remote_servers>
    </yandex>
  users.xml: |
    <yandex>
        <profiles>
            <default>
                <max_memory_usage>10000000000</max_memory_usage>
                <use_uncompressed_cache>0</use_uncompressed_cache>
                <load_balancing>random</load_balancing>
            </default>
        </profiles>
        <users>
            <default>
                <password></password>
                <networks incl="networks" replace="replace">
                    <ip>::/0</ip>
                </networks>
                <profile>default</profile>
                <quota>default</quota>
            </default>
        </users>
        <quotas>
            <default>
                <interval>
                    <duration>3600</duration>
                    <queries>0</queries>
                    <errors>0</errors>
                    <result_rows>0</result_rows>
                    <read_rows>0</read_rows>
                    <execution_time>0</execution_time>
                </interval>
            </default>
        </quotas>
    </yandex>
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: clickhouse
  namespace: monitoring
spec:
  serviceName: clickhouse
  replicas: 4
  selector:
    matchLabels:
      app: clickhouse
  template:
    metadata:
      labels:
        app: clickhouse
    spec:
      containers:
      - name: clickhouse
        image: clickhouse/clickhouse-server:22.8
        ports:
        - containerPort: 9000
        - containerPort: 8123
        resources:
          requests:
            memory: "64Gi"
            cpu: "16"
          limits:
            memory: "64Gi"
            cpu: "16"
        volumeMounts:
        - name: config
          mountPath: /etc/clickhouse-server/config.xml
          subPath: config.xml
        - name: config
          mountPath: /etc/clickhouse-server/users.xml
          subPath: users.xml
        - name: data
          mountPath: /var/lib/clickhouse
      volumes:
      - name: config
        configMap:
          name: clickhouse-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 2Ti
```

## Phase 2: Application Components

### Step 5: Metrics Gateway Deployment

```python
# metrics_gateway.py
from flask import Flask, request, jsonify
import asyncio
import json
import logging
from kafka import KafkaProducer
from datetime import datetime
import hashlib

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

# Kafka producer configuration
producer = KafkaProducer(
    bootstrap_servers=['metrics-kafka-kafka-bootstrap.monitoring.svc.cluster.local:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    key_serializer=lambda v: v.encode('utf-8') if v else None,
    compression_type='snappy',
    batch_size=16384,
    linger_ms=10,
    buffer_memory=33554432,
    max_request_size=1048576,
    retries=3,
    acks='all'
)

@app.route('/health', methods=['GET'])
def health_check():
    return jsonify({"status": "healthy", "timestamp": datetime.utcnow().isoformat()})

@app.route('/metrics', methods=['POST'])
def receive_metrics():
    try:
        data = request.get_json()
        if not data or 'metrics' not in data:
            return jsonify({"error": "Invalid payload format"}), 400
            
        metrics = data['metrics']
        success_count = 0
        
        for metric in metrics:
            # Validate metric structure
            required_fields = ['metric_name', 'timestamp', 'value', 'labels']
            if not all(field in metric for field in required_fields):
                logging.warning(f"Skipping invalid metric: {metric}")
                continue
                
            # Enrich with processing metadata
            metric['received_at'] = datetime.utcnow().isoformat()
            metric['gateway_id'] = 'gateway-1'  # From environment
            
            # Determine partition key for balanced distribution
            partition_key = f"{metric['metric_name']}:{metric['labels'].get('host', 'unknown')}"
            
            # Send to Kafka
            future = producer.send(
                'metrics-raw',
                key=partition_key,
                value=metric
            )
            
            success_count += 1
            
        # Ensure all messages are sent
        producer.flush()
        
        return jsonify({
            "status": "success",
            "metrics_received": len(metrics),
            "metrics_processed": success_count
        }), 200
        
    except Exception as e:
        logging.error(f"Error processing metrics: {e}")
        return jsonify({"error": "Internal server error"}), 500

@app.route('/metrics/batch', methods=['POST'])
def receive_metrics_batch():
    """Handle large batch uploads with streaming"""
    try:
        # For large batches, process in chunks
        data = request.get_json()
        metrics = data.get('metrics', [])
        
        chunk_size = 1000
        total_processed = 0
        
        for i in range(0, len(metrics), chunk_size):
            chunk = metrics[i:i + chunk_size]
            
            for metric in chunk:
                if all(field in metric for field in ['metric_name', 'timestamp', 'value', 'labels']):
                    partition_key = f"{metric['metric_name']}:{metric['labels'].get('host', 'unknown')}"
                    producer.send('metrics-raw', key=partition_key, value=metric)
                    total_processed += 1
                    
            # Flush every chunk
            producer.flush()
            
        return jsonify({
            "status": "success",
            "metrics_processed": total_processed
        }), 200
        
    except Exception as e:
        logging.error(f"Error processing batch: {e}")
        return jsonify({"error": "Internal server error"}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, debug=False)
```

```yaml
# metrics-gateway-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-gateway
  namespace: monitoring
spec:
  replicas: 3
  selector:
    matchLabels:
      app: metrics-gateway
  template:
    metadata:
      labels:
        app: metrics-gateway
    spec:
      containers:
      - name: metrics-gateway
        image: metrics-gateway:latest
        ports:
        - containerPort: 8080
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "metrics-kafka-kafka-bootstrap.monitoring.svc.cluster.local:9092"
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "4Gi"
            cpu: "2"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-gateway-service
  namespace: monitoring
spec:
  selector:
    app: metrics-gateway
  ports:
  - port: 8080
    targetPort: 8080
  type: LoadBalancer
```

### Step 6: Metrics Storage Writer

```python
# storage_writer.py
import asyncio
import asyncpg
import json
import logging
from kafka import KafkaConsumer
from datetime import datetime
import redis.asyncio as redis
from contextlib import asynccontextmanager

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MetricsStorageWriter:
    def __init__(self, db_config, redis_config, kafka_config):
        self.db_config = db_config
        self.redis_config = redis_config
        self.kafka_config = kafka_config
        self.db_pool = None
        self.redis_client = None
        
    async def initialize(self):
        """Initialize database connections"""
        
        # Create TimescaleDB connection pool
        self.db_pool = await asyncpg.create_pool(
            host=self.db_config['host'],
            port=self.db_config['port'],
            database=self.db_config['database'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            min_size=10,
            max_size=20,
            command_timeout=60
        )
        
        # Create Redis connection
        self.redis_client = redis.Redis(
            host=self.redis_config['host'],
            port=self.redis_config['port'],
            db=0,
            decode_responses=True
        )
        
        logger.info("Storage writer initialized")
        
    async def start_consuming(self):
        """Start consuming metrics from Kafka"""
        
        consumer = KafkaConsumer(
            'metrics-raw',
            bootstrap_servers=self.kafka_config['bootstrap_servers'],
            group_id='storage-writer',
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='latest',
            enable_auto_commit=True,
            max_poll_records=1000,
            fetch_max_wait_ms=1000
        )
        
        batch_size = 1000
        batch = []
        
        try:
            for message in consumer:
                metric = message.value
                batch.append(metric)
                
                if len(batch) >= batch_size:
                    await self.write_metrics_batch(batch)
                    batch = []
                    
        except Exception as e:
            logger.error(f"Error consuming messages: {e}")
            if batch:
                await self.write_metrics_batch(batch)
                
    async def write_metrics_batch(self, metrics):
        """Write batch of metrics to TimescaleDB and Redis"""
        
        if not metrics:
            return
            
        try:
            # Prepare batch for TimescaleDB
            db_batch = []
            redis_batch = []
            
            for metric in metrics:
                # TimescaleDB insert data
                timestamp = datetime.fromisoformat(metric['timestamp'].replace('Z', '+00:00'))
                db_batch.append((
                    timestamp,
                    metric['metric_name'],
                    metric['value'],
                    json.dumps(metric['labels']),
                    metric['labels'].get('host'),
                    metric['labels'].get('region'),
                    metric['labels'].get('environment')
                ))
                
                # Redis cache data for latest values
                cache_key = f"latest:{metric['metric_name']}:{hash(str(sorted(metric['labels'].items())))}"
                cache_value = {
                    'value': metric['value'],
                    'timestamp': metric['timestamp'],
                    'labels': metric['labels']
                }
                redis_batch.append((cache_key, json.dumps(cache_value)))
            
            # Write to TimescaleDB using COPY
            async with self.db_pool.acquire() as conn:
                await conn.copy_records_to_table(
                    'metrics_hot',
                    records=db_batch,
                    columns=['time', 'metric_name', 'value', 'labels', 'host', 'region', 'environment']
                )
            
            # Update Redis cache
            pipe = self.redis_client.pipeline()
            for cache_key, cache_value in redis_batch:
                pipe.setex(cache_key, 300, cache_value)  # 5-minute TTL
            await pipe.execute()
            
            logger.info(f"Successfully wrote {len(metrics)} metrics")
            
        except Exception as e:
            logger.error(f"Error writing metrics batch: {e}")
            raise

async def main():
    # Configuration
    db_config = {
        'host': 'timescaledb-0.timescaledb.monitoring.svc.cluster.local',
        'port': 5432,
        'database': 'metrics',
        'user': 'metrics_user',
        'password': 'your_password'
    }
    
    redis_config = {
        'host': 'redis-cluster.monitoring.svc.cluster.local',
        'port': 6379
    }
    
    kafka_config = {
        'bootstrap_servers': ['metrics-kafka-kafka-bootstrap.monitoring.svc.cluster.local:9092']
    }
    
    # Start storage writer
    writer = MetricsStorageWriter(db_config, redis_config, kafka_config)
    await writer.initialize()
    await writer.start_consuming()

if __name__ == "__main__":
    asyncio.run(main())
```

### Step 7: Aggregation Service

```python
# aggregation_service.py
import asyncio
import asyncpg
import clickhouse_connect
import logging
from datetime import datetime, timedelta
import json

logger = logging.getLogger(__name__)

class AggregationService:
    def __init__(self, timescaledb_config, clickhouse_config):
        self.timescaledb_config = timescaledb_config
        self.clickhouse_config = clickhouse_config
        self.timescaledb_pool = None
        self.clickhouse_client = None
        
    async def initialize(self):
        """Initialize database connections"""
        
        # TimescaleDB connection
        self.timescaledb_pool = await asyncpg.create_pool(
            **self.timescaledb_config,
            min_size=5,
            max_size=10
        )
        
        # ClickHouse connection
        self.clickhouse_client = clickhouse_connect.get_client(
            host=self.clickhouse_config['host'],
            port=self.clickhouse_config['port'],
            database=self.clickhouse_config['database'],
            compress=True
        )
        
        # Create ClickHouse table if not exists
        await self.create_clickhouse_tables()
        
    async def create_clickhouse_tables(self):
        """Create ClickHouse tables for warm storage"""
        
        create_table_sql = """
        CREATE TABLE IF NOT EXISTS metrics_warm (
            time DateTime,
            metric_name LowCardinality(String),
            labels String,
            host LowCardinality(String),
            region LowCardinality(String),
            environment LowCardinality(String),
            min_value Float64,
            max_value Float64,
            avg_value Float64,
            sum_value Float64,
            count_value UInt64,
            p50_value Float64,
            p95_value Float64,
            p99_value Float64
        ) ENGINE = MergeTree()
        PARTITION BY toYYYYMM(time)
        ORDER BY (metric_name, host, time)
        SETTINGS index_granularity = 8192
        """
        
        self.clickhouse_client.command(create_table_sql)
        
        # Set TTL for automatic cleanup
        ttl_sql = "ALTER TABLE metrics_warm MODIFY TTL time + INTERVAL 30 DAY"
        try:
            self.clickhouse_client.command(ttl_sql)
        except:
            pass  # TTL might already be set
            
    async def run_continuous_aggregation(self):
        """Run continuous aggregation from hot to warm storage"""
        
        while True:
            try:
                # Aggregate data that's 5 minutes old to allow for late arrivals
                end_time = datetime.utcnow() - timedelta(minutes=5)
                start_time = end_time - timedelta(minutes=1)
                
                await self.aggregate_minute_data(start_time, end_time)
                
                # Wait 1 minute before next aggregation
                await asyncio.sleep(60)
                
            except Exception as e:
                logger.error(f"Aggregation error: {e}")
                await asyncio.sleep(60)
                
    async def aggregate_minute_data(self, start_time: datetime, end_time: datetime):
        """Aggregate one minute of data from hot to warm storage"""
        
        aggregation_query = """
        SELECT 
            date_trunc('minute', time) as time_bucket,
            metric_name,
            labels,
            host,
            region,
            environment,
            MIN(value) as min_value,
            MAX(value) as max_value,
            AVG(value) as avg_value,
            SUM(value) as sum_value,
            COUNT(*) as count_value,
            percentile_cont(0.50) WITHIN GROUP (ORDER BY value) as p50_value,
            percentile_cont(0.95) WITHIN GROUP (ORDER BY value) as p95_value,
            percentile_cont(0.99) WITHIN GROUP (ORDER BY value) as p99_value
        FROM metrics_hot
        WHERE time >= $1 AND time < $2
        GROUP BY time_bucket, metric_name, labels, host, region, environment
        ORDER BY time_bucket
        """
        
        async with self.timescaledb_pool.acquire() as conn:
            rows = await conn.fetch(aggregation_query, start_time, end_time)
            
        if not rows:
            return
            
        # Prepare data for ClickHouse
        clickhouse_data = []
        for row in rows:
            clickhouse_data.append([
                row['time_bucket'],
                row['metric_name'],
                row['labels'],
                row['host'] or '',
                row['region'] or '',
                row['environment'] or '',
                row['min_value'],
                row['max_value'],
                row['avg_value'],
                row['sum_value'],
                row['count_value'],
                row['p50_value'],
                row['p95_value'],
                row['p99_value']
            ])
        
        # Insert into ClickHouse
        if clickhouse_data:
            self.clickhouse_client.insert(
                'metrics_warm',
                clickhouse_data,
                column_names=[
                    'time', 'metric_name', 'labels', 'host', 'region', 'environment',
                    'min_value', 'max_value', 'avg_value', 'sum_value', 'count_value',
                    'p50_value', 'p95_value', 'p99_value'
                ]
            )
            
        logger.info(f"Aggregated {len(clickhouse_data)} minute-level data points for {start_time}")

# Deployment configuration would be similar to storage writer
async def main():
    timescaledb_config = {
        'host': 'timescaledb-0.timescaledb.monitoring.svc.cluster.local',
        'port': 5432,
        'database': 'metrics',
        'user': 'metrics_user',
        'password': 'your_password'
    }
    
    clickhouse_config = {
        'host': 'clickhouse-0.clickhouse.monitoring.svc.cluster.local',
        'port': 9000,
        'database': 'default'
    }
    
    service = AggregationService(timescaledb_config, clickhouse_config)
    await service.initialize()
    await service.run_continuous_aggregation()

if __name__ == "__main__":
    asyncio.run(main())
```

## Phase 3: Alerting System

### Step 8: Alert Engine Deployment

```python
# Complete alert engine implementation would go here
# (Using the AlertEngine class from alerting-design.md)
```

```yaml
# alert-engine-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alert-engine
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: alert-engine
  template:
    metadata:
      labels:
        app: alert-engine
    spec:
      containers:
      - name: alert-engine
        image: alert-engine:latest
        env:
        - name: TIMESCALEDB_HOST
          value: "timescaledb-0.timescaledb.monitoring.svc.cluster.local"
        - name: CLICKHOUSE_HOST
          value: "clickhouse-0.clickhouse.monitoring.svc.cluster.local"
        - name: REDIS_HOST
          value: "redis-cluster.monitoring.svc.cluster.local"
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "4Gi"
            cpu: "2"
        volumeMounts:
        - name: alert-rules
          mountPath: /app/config/alert-rules.yml
          subPath: alert-rules.yml
      volumes:
      - name: alert-rules
        configMap:
          name: alert-rules-config
```

## Phase 4: Configuration and Testing

### Step 9: Configuration Management

```yaml
# alert-rules-configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-rules-config
  namespace: monitoring
data:
  alert-rules.yml: |
    alert_rules:
      - name: "high_cpu_usage"
        metric: "cpu_usage_percent"
        condition:
          operator: ">"
          threshold: 80
          duration: "5m"
          aggregation: "avg"
        labels:
          severity: "warning"
          team: "infrastructure"
        annotations:
          summary: "High CPU usage on {{ $labels.host }}"
          
      - name: "memory_exhaustion"
        metric: "memory_usage_percent"
        condition:
          operator: ">"
          threshold: 90
          duration: "2m"
          aggregation: "avg"
        labels:
          severity: "critical"
          team: "infrastructure"
        annotations:
          summary: "Memory exhaustion on {{ $labels.host }}"
```

### Step 10: Testing and Validation

```bash
# test-deployment.sh
#!/bin/bash

echo "Testing metrics ingestion..."

# Test metrics gateway
curl -X POST http://metrics-gateway-service.monitoring.svc.cluster.local:8080/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "metrics": [
      {
        "metric_name": "cpu_usage_percent",
        "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
        "value": 85.5,
        "labels": {
          "host": "test-server-01",
          "region": "us-east-1",
          "environment": "test"
        }
      }
    ]
  }'

echo "Waiting for metric to be processed..."
sleep 30

# Query TimescaleDB to verify storage
kubectl exec -it timescaledb-0 -n monitoring -- psql -U metrics_user -d metrics -c "
SELECT COUNT(*) FROM metrics_hot WHERE metric_name = 'cpu_usage_percent' AND host = 'test-server-01';
"

echo "Testing complete!"
```

## Phase 5: Monitoring the Monitoring System

### Step 11: Self-Monitoring Setup

```yaml
# monitoring-stack-monitoring.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: self-monitoring-config
  namespace: monitoring
data:
  alert-rules.yml: |
    alert_rules:
      - name: "kafka_consumer_lag"
        metric: "kafka_consumer_lag"
        condition:
          operator: ">"
          threshold: 10000
          duration: "2m"
        labels:
          severity: "warning"
          team: "platform"
          
      - name: "timescaledb_connection_failure"
        metric: "timescaledb_up"
        condition:
          operator: "=="
          threshold: 0
          duration: "1m"
        labels:
          severity: "critical"
          team: "platform"
          
      - name: "clickhouse_insert_errors"
        metric: "clickhouse_insert_error_rate"
        condition:
          operator: ">"
          threshold: 0.05
          duration: "5m"
        labels:
          severity: "warning"
          team: "platform"
```

## Deployment Checklist

### Pre-deployment Verification

- [ ] Kubernetes cluster is running and accessible
- [ ] Storage classes for persistent volumes are configured
- [ ] Network policies allow communication between components
- [ ] SSL certificates are ready for external endpoints
- [ ] DNS resolution is working for service discovery

### Deployment Steps

- [ ] Deploy Kafka cluster and create topics
- [ ] Deploy TimescaleDB and initialize schema
- [ ] Deploy ClickHouse cluster and create tables
- [ ] Deploy Redis cluster for caching
- [ ] Deploy metrics gateway and test ingestion
- [ ] Deploy storage writer and verify data flow
- [ ] Deploy aggregation service and test warm storage
- [ ] Deploy alert engine and configure rules
- [ ] Set up notification channels
- [ ] Configure monitoring dashboards
- [ ] Implement backup and disaster recovery
- [ ] Set up log aggregation for the monitoring system
- [ ] Configure autoscaling for application components

### Post-deployment Tasks

- [ ] Load test the system with realistic traffic
- [ ] Validate alert firing and resolution
- [ ] Test notification delivery to all channels
- [ ] Verify data retention policies are working
- [ ] Set up operational runbooks
- [ ] Train operations team on system management
- [ ] Implement capacity planning monitoring
- [ ] Schedule regular backup testing

This implementation guide provides a comprehensive roadmap for deploying the metrics monitoring and alerting system in a production environment with high availability and scalability considerations.
