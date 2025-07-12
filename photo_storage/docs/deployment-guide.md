# Deployment Guide for Photo Storage System

## Overview

This guide provides comprehensive instructions for deploying the photo storage system in production environments, including infrastructure setup, configuration management, and operational procedures.

## Infrastructure Requirements

### Minimum System Requirements

#### Production Environment
```yaml
compute_nodes:
  api_servers:
    count: 10
    instance_type: "c5.2xlarge"  # 8 vCPU, 16GB RAM
    storage: "100GB SSD"
    
  processing_workers:
    count: 20
    instance_type: "g4dn.2xlarge"  # 8 vCPU, 32GB RAM, T4 GPU
    storage: "200GB SSD"
    
  search_nodes:
    count: 6
    instance_type: "r5.xlarge"  # 4 vCPU, 32GB RAM
    storage: "500GB SSD"

database_cluster:
  primary:
    instance_type: "r5.4xlarge"  # 16 vCPU, 128GB RAM
    storage: "2TB SSD"
    iops: 10000
    
  replicas:
    count: 3
    instance_type: "r5.2xlarge"  # 8 vCPU, 64GB RAM
    storage: "2TB SSD"

cache_cluster:
  redis_nodes:
    count: 6
    instance_type: "r5.xlarge"  # 4 vCPU, 32GB RAM
    memory: "26GB available for Redis"

message_queue:
  kafka_brokers:
    count: 5
    instance_type: "m5.xlarge"  # 4 vCPU, 16GB RAM
    storage: "1TB SSD"

object_storage:
  primary_region: "us-west-2"
  backup_regions: ["us-east-1", "eu-west-1"]
  storage_classes: ["STANDARD", "IA", "GLACIER"]
```

### Network Architecture
```yaml
vpc_configuration:
  cidr: "10.0.0.0/16"
  
subnets:
  public:
    - cidr: "10.0.1.0/24"  # Load balancers
    - cidr: "10.0.2.0/24"  # NAT gateways
  private:
    - cidr: "10.0.10.0/24"  # API servers
    - cidr: "10.0.11.0/24"  # Processing workers
    - cidr: "10.0.12.0/24"  # Database cluster
  isolated:
    - cidr: "10.0.20.0/24"  # Admin/management

security_groups:
  load_balancer:
    ingress:
      - port: 443
        protocol: tcp
        source: "0.0.0.0/0"
      - port: 80
        protocol: tcp
        source: "0.0.0.0/0"
        
  api_servers:
    ingress:
      - port: 8080
        protocol: tcp
        source: "load_balancer_sg"
        
  database:
    ingress:
      - port: 5432
        protocol: tcp
        source: "api_servers_sg"
```

## Container Orchestration

### Kubernetes Deployment

#### Namespace Configuration
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: photo-storage
  labels:
    app: photo-storage
    environment: production
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: photo-storage-quota
  namespace: photo-storage
spec:
  hard:
    requests.cpu: "200"
    requests.memory: "400Gi"
    limits.cpu: "400"
    limits.memory: "800Gi"
    persistentvolumeclaims: "50"
```

#### API Service Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: photo-api
  namespace: photo-storage
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3
      maxUnavailable: 1
  selector:
    matchLabels:
      app: photo-api
  template:
    metadata:
      labels:
        app: photo-api
    spec:
      containers:
      - name: api-server
        image: photo-storage/api:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        - name: S3_BUCKET
          value: "photos-primary-prod"
        - name: LOG_LEVEL
          value: "info"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: photo-api-config
---
apiVersion: v1
kind: Service
metadata:
  name: photo-api-service
  namespace: photo-storage
spec:
  selector:
    app: photo-api
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```

#### Processing Worker Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: photo-processor
  namespace: photo-storage
spec:
  replicas: 20
  selector:
    matchLabels:
      app: photo-processor
  template:
    metadata:
      labels:
        app: photo-processor
    spec:
      nodeSelector:
        accelerator: nvidia-tesla-t4
      containers:
      - name: processor
        image: photo-storage/processor:v1.2.3
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
            nvidia.com/gpu: 1
          limits:
            cpu: 4000m
            memory: 8Gi
            nvidia.com/gpu: 1
        env:
        - name: KAFKA_BROKERS
          value: "kafka-cluster:9092"
        - name: WORKER_TYPE
          value: "cv-processor"
        - name: GPU_ENABLED
          value: "true"
        volumeMounts:
        - name: tmp-storage
          mountPath: /tmp
      volumes:
      - name: tmp-storage
        emptyDir:
          sizeLimit: 10Gi
```

### Helm Charts

#### Values Configuration
```yaml
# values.yaml
global:
  environment: production
  region: us-west-2
  
api:
  replicaCount: 10
  image:
    repository: photo-storage/api
    tag: v1.2.3
  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  autoscaling:
    enabled: true
    minReplicas: 10
    maxReplicas: 50
    targetCPUUtilizationPercentage: 70

processor:
  replicaCount: 20
  image:
    repository: photo-storage/processor
    tag: v1.2.3
  gpu:
    enabled: true
    type: nvidia-tesla-t4
  resources:
    requests:
      cpu: 2000m
      memory: 4Gi
      nvidia.com/gpu: 1

database:
  host: postgres-primary.internal
  port: 5432
  name: photos_production
  ssl: require

redis:
  cluster:
    enabled: true
    nodes: 6
  memory: 26Gi

kafka:
  brokers: 5
  topics:
    photoUpload:
      partitions: 128
      replicationFactor: 3
```

#### Installation Commands
```bash
# Add Helm repository
helm repo add photo-storage https://charts.photo-storage.com
helm repo update

# Install in production
helm install photo-storage photo-storage/photo-storage \
  --namespace photo-storage \
  --create-namespace \
  --values values-production.yaml \
  --wait \
  --timeout 10m

# Upgrade deployment
helm upgrade photo-storage photo-storage/photo-storage \
  --namespace photo-storage \
  --values values-production.yaml \
  --wait
```

## Database Setup

### PostgreSQL Cluster Configuration

#### Primary Database Setup
```sql
-- Create database and user
CREATE DATABASE photos_production;
CREATE USER photos_app WITH ENCRYPTED PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE photos_production TO photos_app;

-- Connect to photos database
\c photos_production;

-- Create extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- Create tables
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'active'
);

CREATE TABLE photos (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id),
    filename VARCHAR(255) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(50) NOT NULL,
    width INTEGER NOT NULL,
    height INTEGER NOT NULL,
    taken_at TIMESTAMP WITH TIME ZONE,
    uploaded_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    storage_path VARCHAR(500) NOT NULL,
    checksum VARCHAR(64) NOT NULL,
    processing_status VARCHAR(20) DEFAULT 'pending',
    metadata JSONB,
    tags TEXT[],
    deleted_at TIMESTAMP WITH TIME ZONE
);

-- Create indexes
CREATE INDEX idx_photos_user_date ON photos (user_id, uploaded_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_photos_processing ON photos (processing_status) WHERE processing_status IN ('pending', 'processing');
CREATE INDEX idx_photos_checksum ON photos (checksum);
CREATE INDEX idx_photos_tags ON photos USING gin (tags);
CREATE INDEX idx_photos_metadata ON photos USING gin (metadata);
```

#### Read Replica Configuration
```bash
#!/bin/bash
# setup-read-replica.sh

# Configure read replica
docker run -d \
  --name postgres-replica \
  --network photo-storage-network \
  -e POSTGRES_DB=photos_production \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=replica_password \
  -e PGUSER=postgres \
  -v replica-data:/var/lib/postgresql/data \
  -v ./postgresql-replica.conf:/etc/postgresql/postgresql.conf \
  postgres:14 \
  postgres -c config_file=/etc/postgresql/postgresql.conf

# Setup streaming replication
echo "host replication replica_user 10.0.0.0/8 md5" >> /var/lib/postgresql/data/pg_hba.conf

# Restart primary to apply replication settings
systemctl restart postgresql
```

### Database Migration

#### Migration Scripts
```go
package migrations

import (
    "database/sql"
    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(db *sql.DB) error {
    driver, err := postgres.WithInstance(db, &postgres.Config{})
    if err != nil {
        return err
    }
    
    m, err := migrate.NewWithDatabaseInstance(
        "file://migrations",
        "postgres", driver)
    if err != nil {
        return err
    }
    
    return m.Up()
}
```

## Configuration Management

### Environment Configuration
```yaml
# config/production.yaml
server:
  host: "0.0.0.0"
  port: 8080
  read_timeout: 30s
  write_timeout: 30s
  idle_timeout: 120s

database:
  host: "postgres-primary.internal"
  port: 5432
  name: "photos_production"
  user: "photos_app"
  password: "${DATABASE_PASSWORD}"
  ssl_mode: "require"
  max_open_conns: 100
  max_idle_conns: 25
  conn_max_lifetime: "5m"

redis:
  cluster_endpoints:
    - "redis-node-1:6379"
    - "redis-node-2:6379"
    - "redis-node-3:6379"
  password: "${REDIS_PASSWORD}"
  pool_size: 100

kafka:
  brokers:
    - "kafka-1:9092"
    - "kafka-2:9092"
    - "kafka-3:9092"
  consumer_group: "photo-processors"

storage:
  provider: "s3"
  bucket: "photos-primary-prod"
  region: "us-west-2"
  access_key: "${AWS_ACCESS_KEY_ID}"
  secret_key: "${AWS_SECRET_ACCESS_KEY}"

logging:
  level: "info"
  format: "json"
  output: "stdout"

monitoring:
  metrics_port: 9090
  health_check_path: "/health"
  jaeger_endpoint: "http://jaeger-collector:14268/api/traces"
```

### Secret Management
```yaml
# secrets.yaml (encrypted with SOPS or similar)
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: photo-storage
type: Opaque
data:
  url: cG9zdGdyZXM6Ly9waG90b3NfYXBwOnNlY3VyZV9wYXNzd29yZEBwb3N0Z3Jlcy1wcmltYXJ5LmludGVybmFsOjU0MzIvcGhvdG9zX3Byb2R1Y3Rpb24/c3NsbW9kZT1yZXF1aXJl
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: photo-storage
type: Opaque
data:
  access-key-id: QUtJQUlPU0ZPRE5ON0VYQU1QTEU=
  secret-access-key: d0phbHJYVXRuRkVNSS9LN01ERU5HL2JQeFJmaUNZRVhBTVBMRUtFWQ==
```

## Load Balancer Configuration

### HAProxy Configuration
```
# haproxy.cfg
global
    daemon
    maxconn 4096
    log stdout local0
    
defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog
    option dontlognull
    retries 3

frontend photo_api_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/photo-storage.pem
    redirect scheme https if !{ ssl_fc }
    
    # Rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request reject if { sc_http_req_rate(0) gt 20 }
    
    # Health check
    acl health_check path_beg /health
    use_backend health_backend if health_check
    
    default_backend photo_api_backend

backend photo_api_backend
    balance roundrobin
    option httpchk GET /health
    
    server api-1 10.0.10.10:8080 check
    server api-2 10.0.10.11:8080 check
    server api-3 10.0.10.12:8080 check
    server api-4 10.0.10.13:8080 check
    server api-5 10.0.10.14:8080 check

backend health_backend
    server health localhost:8080 check
```

### NGINX Configuration
```nginx
# nginx.conf
upstream photo_api {
    least_conn;
    server 10.0.10.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.10.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.10.12:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

server {
    listen 80;
    listen 443 ssl http2;
    server_name api.photos.example.com;
    
    # SSL configuration
    ssl_certificate /etc/ssl/certs/photo-storage.crt;
    ssl_certificate_key /etc/ssl/private/photo-storage.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;
    
    # Gzip compression
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;
    
    location / {
        proxy_pass http://photo_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
    
    location /health {
        access_log off;
        proxy_pass http://photo_api;
    }
}
```

## Monitoring Setup

### Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "photo_storage_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'photo-api'
    static_configs:
      - targets: ['photo-api-1:9090', 'photo-api-2:9090']
    metrics_path: /metrics
    scrape_interval: 10s
    
  - job_name: 'photo-processor'
    static_configs:
      - targets: ['processor-1:9090', 'processor-2:9090']
    metrics_path: /metrics
    scrape_interval: 15s
    
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
    
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

### Grafana Dashboard
```json
{
  "dashboard": {
    "title": "Photo Storage System",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{service}}"
          }
        ]
      },
      {
        "title": "Response Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95 Latency"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m])",
            "legendFormat": "Error Rate"
          }
        ]
      }
    ]
  }
}
```

## Backup and Recovery

### Automated Backup Script
```bash
#!/bin/bash
# backup.sh

set -e

BACKUP_DIR="/backups/$(date +%Y%m%d)"
S3_BUCKET="photo-storage-backups"
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

# Database backup
echo "Creating database backup..."
pg_dump -h postgres-primary.internal \
        -U postgres \
        -d photos_production \
        --no-password \
        --format=custom \
        --compress=9 \
        > "$BACKUP_DIR/database.dump"

# Upload to S3
echo "Uploading backup to S3..."
aws s3 cp "$BACKUP_DIR/database.dump" \
          "s3://$S3_BUCKET/database/$(date +%Y%m%d)/database.dump"

# Cleanup old backups
echo "Cleaning up old backups..."
find /backups -type d -mtime +$RETENTION_DAYS -exec rm -rf {} +

aws s3 ls "s3://$S3_BUCKET/database/" | \
    awk '{print $2}' | \
    while read -r folder; do
        folder_date=$(echo "$folder" | tr -d '/')
        if [[ $(date -d "$folder_date" +%s) -lt $(date -d "$RETENTION_DAYS days ago" +%s) ]]; then
            aws s3 rm "s3://$S3_BUCKET/database/$folder" --recursive
        fi
    done

echo "Backup completed successfully"
```

### Disaster Recovery Procedure
```bash
#!/bin/bash
# disaster_recovery.sh

BACKUP_DATE=${1:-$(date +%Y%m%d)}
S3_BUCKET="photo-storage-backups"
RECOVERY_DIR="/recovery"

echo "Starting disaster recovery for date: $BACKUP_DATE"

# Download backup from S3
mkdir -p "$RECOVERY_DIR"
aws s3 cp "s3://$S3_BUCKET/database/$BACKUP_DATE/database.dump" \
          "$RECOVERY_DIR/database.dump"

# Restore database
echo "Restoring database..."
pg_restore -h postgres-primary.internal \
           -U postgres \
           -d photos_production \
           --clean \
           --if-exists \
           --no-owner \
           --no-privileges \
           "$RECOVERY_DIR/database.dump"

# Verify restoration
echo "Verifying restoration..."
psql -h postgres-primary.internal \
     -U postgres \
     -d photos_production \
     -c "SELECT COUNT(*) FROM photos;"

echo "Disaster recovery completed"
```

This deployment guide provides a comprehensive approach to deploying and operating the photo storage system in production environments with high availability, scalability, and reliability.
