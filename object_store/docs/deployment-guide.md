# Production Deployment Guide

## Table of Contents
1. [Infrastructure Requirements](#infrastructure-requirements)
2. [Environment Setup](#environment-setup)
3. [Configuration Management](#configuration-management)
4. [Deployment Process](#deployment-process)
5. [Monitoring and Observability](#monitoring-and-observability)
6. [Scaling and Maintenance](#scaling-and-maintenance)
7. [Disaster Recovery](#disaster-recovery)
8. [Troubleshooting](#troubleshooting)

## Infrastructure Requirements

### Minimum Hardware Specifications

#### API Gateway Nodes
```yaml
api_gateway:
  instance_type: "c5.2xlarge"
  cpu: "8 vCPUs"
  memory: "16 GB"
  network: "Up to 10 Gbps"
  storage: "100 GB SSD"
  min_nodes: 3
  max_nodes: 20
  zones: 3
```

#### Storage Nodes
```yaml
storage_nodes:
  instance_type: "i3.4xlarge"
  cpu: "16 vCPUs"
  memory: "122 GB"
  storage: "2 x 1.9 TB NVMe SSD"
  network: "Up to 10 Gbps"
  min_nodes: 9  # For 6+3 erasure coding
  recommended_nodes: 18
  zones: 3
```

#### Metadata Database
```yaml
metadata_db:
  primary:
    instance_type: "r5.4xlarge"
    cpu: "16 vCPUs"
    memory: "128 GB"
    storage: "1 TB SSD"
    iops: "3000"
    
  read_replicas:
    count: 3
    instance_type: "r5.2xlarge"
    cpu: "8 vCPUs"
    memory: "64 GB"
    storage: "500 GB SSD"
```

### Network Requirements

```yaml
network:
  vpc:
    cidr: "10.0.0.0/16"
    dns_hostnames: true
    dns_support: true
    
  bandwidth:
    inter_region: "10 Gbps"
    intra_region: "100 Gbps"
    cdn_global: "Multi-Tbps"
    
  latency:
    intra_zone: "<1ms"
    inter_zone: "<5ms"
    inter_region: "<100ms"
    
  load_balancers:
    application_lb:
      type: "Application Load Balancer"
      scheme: "internet-facing"
      cross_zone: true
      
    network_lb:
      type: "Network Load Balancer"
      scheme: "internal"
      preserve_source_ip: true
```

## Environment Setup

### Terraform Infrastructure

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "object-store"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# VPC Configuration
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "objectstore-${var.environment}"
  cidr = "10.0.0.0/16"
  
  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  database_subnets = ["10.0.201.0/24", "10.0.202.0/24", "10.0.203.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = true
  enable_dns_hostnames = true
  enable_dns_support = true
  
  tags = {
    Environment = var.environment
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "objectstore-${var.environment}"
  cluster_version = "1.28"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  eks_managed_node_groups = {
    api_nodes = {
      instance_types = ["c5.2xlarge"]
      min_size       = 3
      max_size       = 20
      desired_size   = 6
      
      labels = {
        role = "api"
      }
      
      taints = [{
        key    = "role"
        value  = "api"
        effect = "NO_SCHEDULE"
      }]
    }
    
    storage_nodes = {
      instance_types = ["i3.4xlarge"]
      min_size       = 9
      max_size       = 50
      desired_size   = 18
      
      labels = {
        role = "storage"
      }
      
      taints = [{
        key    = "role"
        value  = "storage"
        effect = "NO_SCHEDULE"
      }]
    }
  }
}

# RDS for Metadata
resource "aws_db_instance" "metadata" {
  identifier = "objectstore-metadata-${var.environment}"
  
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r5.4xlarge"
  
  allocated_storage     = 1000
  max_allocated_storage = 10000
  storage_type          = "gp3"
  storage_encrypted     = true
  
  db_name  = "objectstore"
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.metadata_db.id]
  db_subnet_group_name   = aws_db_subnet_group.metadata.name
  
  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  performance_insights_enabled = true
  monitoring_interval         = 60
  
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "objectstore-metadata-${var.environment}-final"
  
  tags = {
    Name = "objectstore-metadata-${var.environment}"
  }
}

# ElastiCache for Caching
resource "aws_elasticache_replication_group" "cache" {
  replication_group_id       = "objectstore-cache-${var.environment}"
  description                = "Object Store Cache"
  
  node_type          = "cache.r6g.2xlarge"
  port               = 6379
  parameter_group_name = "default.redis7"
  
  num_cache_clusters = 3
  
  subnet_group_name  = aws_elasticache_subnet_group.cache.name
  security_group_ids = [aws_security_group.cache.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
  
  snapshot_retention_limit = 7
  snapshot_window         = "03:00-05:00"
  
  tags = {
    Name = "objectstore-cache-${var.environment}"
  }
}
```

### Kubernetes Deployment

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: objectstore
  labels:
    name: objectstore
    environment: production

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: objectstore-config
  namespace: objectstore
data:
  config.yaml: |
    server:
      port: 8080
      read_timeout: 30s
      write_timeout: 30s
      
    database:
      host: metadata-db.objectstore.local
      port: 5432
      name: objectstore
      max_connections: 100
      max_idle: 10
      max_lifetime: 1h
      
    storage:
      erasure_coding:
        data_shards: 6
        parity_shards: 3
        chunk_size: 67108864  # 64MB
      
      replication:
        factor: 3
        cross_region: true
        
    cache:
      redis:
        endpoints:
          - cache-001.objectstore.local:6379
          - cache-002.objectstore.local:6379
          - cache-003.objectstore.local:6379
        pool_size: 50
        
    security:
      tls:
        cert_file: /etc/tls/tls.crt
        key_file: /etc/tls/tls.key
      encryption:
        kms_endpoint: https://kms.us-west-1.amazonaws.com
        key_rotation_period: 2160h  # 90 days

---
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: objectstore-api
  namespace: objectstore
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: objectstore-api
  template:
    metadata:
      labels:
        app: objectstore-api
    spec:
      nodeSelector:
        role: api
      tolerations:
        - key: role
          value: api
          effect: NoSchedule
      containers:
        - name: objectstore-api
          image: objectstore/api:v1.0.0
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 8443
              name: https
          env:
            - name: CONFIG_FILE
              value: /etc/config/config.yaml
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: objectstore-secrets
                  key: db-password
          volumeMounts:
            - name: config
              mountPath: /etc/config
            - name: tls
              mountPath: /etc/tls
          resources:
            requests:
              cpu: 2000m
              memory: 4Gi
            limits:
              cpu: 4000m
              memory: 8Gi
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
      volumes:
        - name: config
          configMap:
            name: objectstore-config
        - name: tls
          secret:
            secretName: objectstore-tls

---
# storage-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: objectstore-storage
  namespace: objectstore
spec:
  selector:
    matchLabels:
      app: objectstore-storage
  template:
    metadata:
      labels:
        app: objectstore-storage
    spec:
      nodeSelector:
        role: storage
      tolerations:
        - key: role
          value: storage
          effect: NoSchedule
      containers:
        - name: objectstore-storage
          image: objectstore/storage:v1.0.0
          ports:
            - containerPort: 9090
              name: storage
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: STORAGE_PATH
              value: /data
          volumeMounts:
            - name: storage
              mountPath: /data
            - name: config
              mountPath: /etc/config
          resources:
            requests:
              cpu: 4000m
              memory: 8Gi
            limits:
              cpu: 8000m
              memory: 16Gi
      volumes:
        - name: storage
          hostPath:
            path: /mnt/objectstore
            type: DirectoryOrCreate
        - name: config
          configMap:
            name: objectstore-config

---
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: objectstore-api-hpa
  namespace: objectstore
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: objectstore-api
  minReplicas: 6
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
```

## Configuration Management

### Environment-Specific Configurations

```yaml
# environments/production.yaml
environment: production
region: us-west-1

api:
  replicas: 12
  resources:
    requests:
      cpu: 2000m
      memory: 4Gi
    limits:
      cpu: 4000m
      memory: 8Gi
  
storage:
  nodes: 36
  replication_factor: 3
  erasure_coding:
    data_shards: 6
    parity_shards: 3
  
database:
  instance_class: db.r5.4xlarge
  storage: 2000
  backup_retention: 30
  read_replicas: 3
  
cache:
  node_type: cache.r6g.2xlarge
  num_nodes: 6
  
monitoring:
  retention_days: 30
  alerting_enabled: true
  
security:
  encryption_at_rest: true
  encryption_in_transit: true
  key_rotation_days: 90
  
compliance:
  audit_logging: true
  soc2_compliance: true
  gdpr_compliance: true
```

### Helm Configuration

```yaml
# values.yaml
global:
  imageRegistry: "your-registry.com"
  imageTag: "v1.0.0"
  environment: "production"

api:
  replicaCount: 12
  image:
    repository: objectstore/api
    pullPolicy: Always
  
  service:
    type: ClusterIP
    port: 8080
    targetPort: 8080
  
  ingress:
    enabled: true
    className: "nginx"
    annotations:
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    hosts:
      - host: api.objectstore.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: objectstore-tls
        hosts:
          - api.objectstore.com

storage:
  image:
    repository: objectstore/storage
    pullPolicy: Always
  
  persistence:
    enabled: true
    storageClass: "fast-ssd"
    size: 4Ti
    mountPath: /data

metadata:
  postgresql:
    enabled: false  # Using external RDS
    external:
      host: "metadata-db.cluster-xxx.us-west-1.rds.amazonaws.com"
      port: 5432
      database: "objectstore"
      username: "objectstore"
      existingSecret: "objectstore-db-secret"

redis:
  enabled: false  # Using external ElastiCache
  external:
    endpoints:
      - cache-001.xxx.cache.amazonaws.com:6379
      - cache-002.xxx.cache.amazonaws.com:6379
      - cache-003.xxx.cache.amazonaws.com:6379
    auth:
      enabled: true
      existingSecret: "objectstore-redis-secret"

monitoring:
  prometheus:
    enabled: true
    retention: 30d
  grafana:
    enabled: true
    adminPassword: "secure-password"
  alertmanager:
    enabled: true
```

## Deployment Process

### CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    tags:
      - 'v*'

env:
  AWS_REGION: us-west-1
  EKS_CLUSTER_NAME: objectstore-production

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t objectstore:${{ github.sha }} .
        
      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: objectstore:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          
      - name: Upload security scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: trivy-results.sarif

  deploy:
    needs: security-scan
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
          
      - name: Login to ECR
        id: ecr-login
        uses: aws-actions/amazon-ecr-login@v2
        
      - name: Build and push image
        env:
          ECR_REGISTRY: ${{ steps.ecr-login.outputs.registry }}
          ECR_REPOSITORY: objectstore
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          
      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME
          
          # Update image in deployment
          kubectl set image deployment/objectstore-api \
            objectstore-api=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -n objectstore
            
          kubectl set image daemonset/objectstore-storage \
            objectstore-storage=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -n objectstore
            
          # Wait for rollout
          kubectl rollout status deployment/objectstore-api -n objectstore --timeout=600s
          kubectl rollout status daemonset/objectstore-storage -n objectstore --timeout=600s
          
      - name: Run post-deployment tests
        run: |
          kubectl apply -f tests/smoke-tests.yaml
          kubectl wait --for=condition=complete job/smoke-tests --timeout=300s
          
      - name: Notify deployment status
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#deployments'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

set -euo pipefail

NAMESPACE="objectstore"
NEW_VERSION=$1
CURRENT_COLOR=$(kubectl get service objectstore-api -n $NAMESPACE -o jsonpath='{.spec.selector.color}')

if [ "$CURRENT_COLOR" = "blue" ]; then
    NEW_COLOR="green"
else
    NEW_COLOR="blue"
fi

echo "Current color: $CURRENT_COLOR"
echo "Deploying to: $NEW_COLOR"

# Deploy new version
envsubst < deployment-template.yaml | \
  sed "s/{{COLOR}}/$NEW_COLOR/g" | \
  sed "s/{{VERSION}}/$NEW_VERSION/g" | \
  kubectl apply -f -

# Wait for deployment to be ready
kubectl rollout status deployment/objectstore-api-$NEW_COLOR -n $NAMESPACE

# Run health checks
echo "Running health checks..."
for i in {1..30}; do
    if kubectl exec -n $NAMESPACE deployment/objectstore-api-$NEW_COLOR -- curl -f http://localhost:8080/health; then
        echo "Health check passed"
        break
    fi
    sleep 10
done

# Run integration tests
echo "Running integration tests..."
kubectl apply -f tests/integration-tests-$NEW_COLOR.yaml
kubectl wait --for=condition=complete job/integration-tests-$NEW_COLOR --timeout=300s

TEST_RESULT=$(kubectl get job integration-tests-$NEW_COLOR -n $NAMESPACE -o jsonpath='{.status.conditions[0].type}')
if [ "$TEST_RESULT" != "Complete" ]; then
    echo "Integration tests failed"
    exit 1
fi

# Switch traffic to new version
echo "Switching traffic to $NEW_COLOR"
kubectl patch service objectstore-api -n $NAMESPACE -p '{"spec":{"selector":{"color":"'$NEW_COLOR'"}}}'

# Monitor for errors
echo "Monitoring for 5 minutes..."
sleep 300

# Check error rates
ERROR_RATE=$(kubectl exec -n monitoring deployment/prometheus -- \
  promtool query instant 'rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])' | \
  tail -1 | awk '{print $2}')

if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
    echo "Error rate too high: $ERROR_RATE"
    echo "Rolling back..."
    kubectl patch service objectstore-api -n $NAMESPACE -p '{"spec":{"selector":{"color":"'$CURRENT_COLOR'"}}}'
    exit 1
fi

# Cleanup old version
echo "Deployment successful. Cleaning up $CURRENT_COLOR"
kubectl delete deployment objectstore-api-$CURRENT_COLOR -n $NAMESPACE

echo "Deployment completed successfully"
```

## Monitoring and Observability

### Prometheus Configuration

```yaml
# prometheus-config.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "objectstore-rules.yml"

scrape_configs:
  - job_name: 'objectstore-api'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - objectstore
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: objectstore-api
        action: keep
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

  - job_name: 'objectstore-storage'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - objectstore
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        regex: objectstore-storage
        action: keep

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

### Grafana Dashboards

```json
{
  "dashboard": {
    "id": null,
    "title": "Object Store Metrics",
    "tags": ["objectstore"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (method, status)",
            "legendFormat": "{{method}} {{status}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec",
            "min": 0
          }
        ]
      },
      {
        "id": 2,
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "50th percentile"
          }
        ]
      },
      {
        "id": 3,
        "title": "Storage Utilization",
        "type": "singlestat",
        "targets": [
          {
            "expr": "sum(storage_used_bytes) / sum(storage_total_bytes) * 100",
            "legendFormat": "Storage Used %"
          }
        ],
        "valueMaps": [
          {
            "value": "null",
            "text": "N/A"
          }
        ],
        "thresholds": "70,85"
      }
    ],
    "refresh": "5s",
    "time": {
      "from": "now-1h",
      "to": "now"
    }
  }
}
```

### Log Aggregation

```yaml
# fluent-bit-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: objectstore
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Path              /var/log/containers/objectstore*.log
        Parser            docker
        Tag               objectstore.*
        Refresh_Interval  5

    [FILTER]
        Name                kubernetes
        Match               objectstore.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

    [OUTPUT]
        Name  es
        Match objectstore.*
        Host  elasticsearch.logging.svc.cluster.local
        Port  9200
        Index objectstore-logs
        Type  _doc
        
  parsers.conf: |
    [PARSER]
        Name   docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep On
```

## Scaling and Maintenance

### Auto-Scaling Configuration

```yaml
# cluster-autoscaler.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-status
  namespace: kube-system
data:
  nodes.max: "100"
  nodes.min: "20"
  scale-down-delay-after-add: "10m"
  scale-down-unneeded-time: "10m"
  skip-nodes-with-local-storage: "false"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/objectstore-production
          env:
            - name: AWS_REGION
              value: us-west-1
```

### Maintenance Scripts

```bash
#!/bin/bash
# maintenance.sh

set -euo pipefail

NAMESPACE="objectstore"
MAINTENANCE_MODE=${1:-"enable"}

enable_maintenance() {
    echo "Enabling maintenance mode..."
    
    # Scale down API pods
    kubectl scale deployment objectstore-api --replicas=2 -n $NAMESPACE
    
    # Add maintenance annotation
    kubectl annotate service objectstore-api maintenance.objectstore.com/enabled=true -n $NAMESPACE
    
    # Update load balancer health check
    kubectl patch service objectstore-api -n $NAMESPACE -p '{"spec":{"selector":{"maintenance":"true"}}}'
    
    echo "Maintenance mode enabled"
}

disable_maintenance() {
    echo "Disabling maintenance mode..."
    
    # Remove maintenance annotation
    kubectl annotate service objectstore-api maintenance.objectstore.com/enabled- -n $NAMESPACE
    
    # Restore normal service selector
    kubectl patch service objectstore-api -n $NAMESPACE -p '{"spec":{"selector":{"app":"objectstore-api"}}}'
    
    # Scale up API pods
    kubectl scale deployment objectstore-api --replicas=12 -n $NAMESPACE
    
    # Wait for pods to be ready
    kubectl rollout status deployment/objectstore-api -n $NAMESPACE
    
    echo "Maintenance mode disabled"
}

case $MAINTENANCE_MODE in
    "enable")
        enable_maintenance
        ;;
    "disable")
        disable_maintenance
        ;;
    *)
        echo "Usage: $0 [enable|disable]"
        exit 1
        ;;
esac
```

## Disaster Recovery

### Backup Strategy

```bash
#!/bin/bash
# backup.sh

set -euo pipefail

BACKUP_TYPE=${1:-"incremental"}
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
S3_BUCKET="objectstore-backups"
REGION="us-west-1"

backup_metadata() {
    echo "Backing up metadata database..."
    
    kubectl exec -n objectstore deployment/postgres-backup -- \
        pg_dump -h metadata-db.cluster-xxx.us-west-1.rds.amazonaws.com \
                -U objectstore \
                -d objectstore \
                --format=custom \
                --no-owner \
                --no-privileges \
                --verbose \
                --file=/backup/metadata-$TIMESTAMP.dump
                
    kubectl cp objectstore/postgres-backup-xxx:/backup/metadata-$TIMESTAMP.dump \
        ./metadata-$TIMESTAMP.dump
        
    aws s3 cp metadata-$TIMESTAMP.dump \
        s3://$S3_BUCKET/metadata/metadata-$TIMESTAMP.dump \
        --region $REGION
        
    rm metadata-$TIMESTAMP.dump
    echo "Metadata backup completed"
}

backup_storage_manifest() {
    echo "Backing up storage manifest..."
    
    kubectl exec -n objectstore deployment/objectstore-api -- \
        /app/tools/export-manifest --output=/tmp/manifest-$TIMESTAMP.json
        
    kubectl cp objectstore/objectstore-api-xxx:/tmp/manifest-$TIMESTAMP.json \
        ./manifest-$TIMESTAMP.json
        
    aws s3 cp manifest-$TIMESTAMP.json \
        s3://$S3_BUCKET/manifests/manifest-$TIMESTAMP.json \
        --region $REGION
        
    rm manifest-$TIMESTAMP.json
    echo "Storage manifest backup completed"
}

case $BACKUP_TYPE in
    "metadata")
        backup_metadata
        ;;
    "manifest")
        backup_storage_manifest
        ;;
    "full")
        backup_metadata
        backup_storage_manifest
        ;;
    *)
        echo "Usage: $0 [metadata|manifest|full]"
        exit 1
        ;;
esac
```

### Recovery Procedures

```bash
#!/bin/bash
# recover.sh

set -euo pipefail

RECOVERY_TYPE=${1:-"metadata"}
BACKUP_DATE=${2:-"latest"}
S3_BUCKET="objectstore-backups"
REGION="us-west-1"

recover_metadata() {
    echo "Recovering metadata from backup..."
    
    if [ "$BACKUP_DATE" = "latest" ]; then
        BACKUP_FILE=$(aws s3 ls s3://$S3_BUCKET/metadata/ --region $REGION | \
                     sort | tail -n 1 | awk '{print $4}')
    else
        BACKUP_FILE="metadata-$BACKUP_DATE.dump"
    fi
    
    echo "Using backup file: $BACKUP_FILE"
    
    aws s3 cp s3://$S3_BUCKET/metadata/$BACKUP_FILE ./$BACKUP_FILE --region $REGION
    
    # Create temporary postgres pod for restore
    kubectl run postgres-restore -n objectstore \
        --image=postgres:15 \
        --restart=Never \
        --rm -i --tty \
        --overrides='{"spec":{"containers":[{"name":"postgres","image":"postgres:15","env":[{"name":"PGPASSWORD","valueFrom":{"secretKeyRef":{"name":"objectstore-secrets","key":"db-password"}}}],"volumeMounts":[{"name":"backup","mountPath":"/backup"}]}],"volumes":[{"name":"backup","emptyDir":{}}]}}' \
        -- bash
    
    # Copy backup file to pod
    kubectl cp ./$BACKUP_FILE objectstore/postgres-restore:/backup/$BACKUP_FILE
    
    # Restore database
    kubectl exec -n objectstore postgres-restore -- \
        pg_restore -h metadata-db.cluster-xxx.us-west-1.rds.amazonaws.com \
                  -U objectstore \
                  -d objectstore \
                  --clean \
                  --if-exists \
                  --verbose \
                  /backup/$BACKUP_FILE
    
    rm ./$BACKUP_FILE
    echo "Metadata recovery completed"
}

case $RECOVERY_TYPE in
    "metadata")
        recover_metadata
        ;;
    *)
        echo "Usage: $0 [metadata] [backup-date]"
        exit 1
        ;;
esac
```

## Troubleshooting

### Common Issues and Solutions

#### High Memory Usage

```bash
# Check memory usage by pod
kubectl top pods -n objectstore --sort-by=memory

# Check memory limits
kubectl describe pods -n objectstore | grep -A 5 "Limits:"

# Increase memory limits if necessary
kubectl patch deployment objectstore-api -n objectstore -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"objectstore-api","resources":{"limits":{"memory":"16Gi"}}}]}}}}'
```

#### Storage Node Issues

```bash
# Check storage node health
kubectl get nodes -l role=storage

# Check storage usage
kubectl exec -n objectstore daemonset/objectstore-storage -- df -h

# Check storage node logs
kubectl logs -n objectstore -l app=objectstore-storage --tail=100

# Restart problematic storage node
kubectl delete pod -n objectstore -l app=objectstore-storage,node=problematic-node
```

#### Database Connection Issues

```bash
# Check database connectivity
kubectl exec -n objectstore deployment/objectstore-api -- \
  pg_isready -h metadata-db.cluster-xxx.us-west-1.rds.amazonaws.com -p 5432

# Check connection pool
kubectl exec -n objectstore deployment/objectstore-api -- \
  curl -s http://localhost:8080/debug/db-stats

# Restart API pods if necessary
kubectl rollout restart deployment/objectstore-api -n objectstore
```

This comprehensive deployment guide provides all the necessary components and procedures for successfully deploying and maintaining a production-ready distributed object store system.
