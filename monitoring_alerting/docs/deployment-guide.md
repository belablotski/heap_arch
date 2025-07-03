# Deployment Guide

## Overview

This guide provides comprehensive instructions for deploying the metrics monitoring and alerting system in production environments, including prerequisites, deployment strategies, and operational considerations.

## Pre-deployment Requirements

### Infrastructure Prerequisites

```yaml
infrastructure_requirements:
  kubernetes_cluster:
    version: ">=1.21"
    nodes: 15
    node_types:
      - control_plane: 3 nodes (4 CPU, 16GB RAM)
      - kafka_nodes: 3 nodes (8 CPU, 32GB RAM, 1TB SSD)
      - database_nodes: 6 nodes (16 CPU, 64GB RAM, 2TB SSD)
      - application_nodes: 3 nodes (8 CPU, 16GB RAM, 500GB SSD)
    
  network_requirements:
    bandwidth: ">=10Gbps"
    latency: "<=1ms between nodes"
    load_balancer: "Layer 4 TCP/UDP support"
    
  storage_requirements:
    storage_classes:
      - fast_ssd: "NVMe SSD with >=10K IOPS"
      - standard_ssd: "Standard SSD with >=3K IOPS"
    backup_storage: "Object storage (S3/GCS) for cold storage"
    
  security_requirements:
    certificates: "TLS certificates for external endpoints"
    secrets_management: "Kubernetes secrets or external vault"
    network_policies: "Pod-to-pod communication rules"
    rbac: "Role-based access control configuration"
```

### Software Dependencies

```bash
# Required software and versions
kubectl >= 1.21
helm >= 3.7
docker >= 20.10

# Optional but recommended
terraform >= 1.0  # For infrastructure as code
ansible >= 4.0    # For configuration management
prometheus >= 2.35  # For monitoring the monitoring system
grafana >= 8.5    # For visualization
```

## Deployment Environments

### Environment Strategy

```yaml
deployment_environments:
  development:
    purpose: "Development and testing"
    scale: "10% of production"
    components:
      - single_node_kafka
      - single_node_timescaledb
      - single_node_clickhouse
      - minimal_alerting
    
  staging:
    purpose: "Pre-production testing and validation"
    scale: "50% of production"
    components:
      - multi_node_kafka_cluster
      - timescaledb_primary_replica
      - clickhouse_cluster_minimal
      - full_alerting_pipeline
    
  production:
    purpose: "Full production deployment"
    scale: "100% target capacity"
    components:
      - high_availability_kafka
      - timescaledb_cluster_ha
      - clickhouse_cluster_distributed
      - complete_monitoring_stack
```

## Infrastructure as Code

### Terraform Configuration

```hcl
# infrastructure/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

# VPC and Networking
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "monitoring-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = true
  
  tags = {
    Environment = var.environment
    Project     = "monitoring-system"
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "monitoring-cluster"
  cluster_version = "1.21"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  node_groups = {
    kafka_nodes = {
      instance_types = ["m5.2xlarge"]
      min_size       = 3
      max_size       = 6
      desired_size   = 3
      
      k8s_labels = {
        workload = "kafka"
      }
      
      taints = [{
        key    = "workload"
        value  = "kafka"
        effect = "NO_SCHEDULE"
      }]
    }
    
    database_nodes = {
      instance_types = ["m5.4xlarge"]
      min_size       = 6
      max_size       = 12
      desired_size   = 6
      
      k8s_labels = {
        workload = "database"
      }
      
      taints = [{
        key    = "workload"
        value  = "database"
        effect = "NO_SCHEDULE"
      }]
    }
    
    application_nodes = {
      instance_types = ["m5.xlarge"]
      min_size       = 3
      max_size       = 10
      desired_size   = 3
      
      k8s_labels = {
        workload = "application"
      }
    }
  }
  
  tags = {
    Environment = var.environment
    Project     = "monitoring-system"
  }
}

# Storage Classes
resource "kubernetes_storage_class" "fast_ssd" {
  metadata {
    name = "fast-ssd"
  }
  
  storage_provisioner    = "ebs.csi.aws.com"
  volume_binding_mode    = "WaitForFirstConsumer"
  allow_volume_expansion = true
  
  parameters = {
    type      = "gp3"
    iops      = "10000"
    throughput = "500"
    fsType    = "ext4"
  }
}

# S3 Bucket for Cold Storage
resource "aws_s3_bucket" "cold_storage" {
  bucket = "monitoring-cold-storage-${var.environment}"
  
  tags = {
    Environment = var.environment
    Purpose     = "metrics-cold-storage"
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "cold_storage_lifecycle" {
  bucket = aws_s3_bucket.cold_storage.id
  
  rule {
    id     = "cold_storage_lifecycle"
    status = "Enabled"
    
    transition {
      days          = 180
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 2555  # 7 years
    }
  }
}
```

### Ansible Playbooks

```yaml
# ansible/deploy-monitoring.yml
---
- name: Deploy Monitoring System
  hosts: localhost
  gather_facts: false
  vars:
    environment: "{{ env | default('staging') }}"
    namespace: monitoring
    
  tasks:
    - name: Create monitoring namespace
      kubernetes.core.k8s:
        name: "{{ namespace }}"
        api_version: v1
        kind: Namespace
        state: present
        
    - name: Install cert-manager
      kubernetes.core.helm:
        name: cert-manager
        chart_ref: jetstack/cert-manager
        release_namespace: cert-manager
        create_namespace: true
        values:
          installCRDs: true
          
    - name: Install Strimzi Kafka Operator
      kubernetes.core.helm:
        name: strimzi-kafka-operator
        chart_ref: strimzi/strimzi-kafka-operator
        release_namespace: "{{ namespace }}"
        
    - name: Deploy Kafka Cluster
      kubernetes.core.k8s:
        definition: "{{ lookup('file', 'kafka-cluster.yml') | from_yaml_all | list }}"
        namespace: "{{ namespace }}"
        
    - name: Wait for Kafka cluster to be ready
      kubernetes.core.k8s_info:
        api_version: kafka.strimzi.io/v1beta2
        kind: Kafka
        name: metrics-kafka
        namespace: "{{ namespace }}"
        wait: true
        wait_condition:
          type: Ready
          status: "True"
        wait_timeout: 600
        
    - name: Deploy TimescaleDB
      kubernetes.core.helm:
        name: timescaledb
        chart_ref: timescale/timescaledb-single
        release_namespace: "{{ namespace }}"
        values:
          replicaCount: 2
          image:
            tag: "pg14-latest"
          resources:
            requests:
              memory: 32Gi
              cpu: 8
            limits:
              memory: 32Gi
              cpu: 8
          persistence:
            enabled: true
            size: 1Ti
            storageClass: fast-ssd
            
    - name: Deploy ClickHouse
      kubernetes.core.k8s:
        definition: "{{ lookup('file', 'clickhouse-cluster.yml') | from_yaml_all | list }}"
        namespace: "{{ namespace }}"
        
    - name: Deploy Application Components
      kubernetes.core.k8s:
        definition: "{{ lookup('file', item) | from_yaml_all | list }}"
        namespace: "{{ namespace }}"
      loop:
        - metrics-gateway.yml
        - storage-writer.yml
        - aggregation-service.yml
        - alert-engine.yml
```

## Helm Charts

### Main Monitoring Chart

```yaml
# helm/monitoring-system/Chart.yaml
apiVersion: v2
name: monitoring-system
description: Comprehensive monitoring and alerting system
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: kafka
    version: "0.20.8"
    repository: "https://charts.bitnami.com/bitnami"
    condition: kafka.enabled
    
  - name: postgresql
    version: "11.6.12"
    repository: "https://charts.bitnami.com/bitnami"
    condition: timescaledb.enabled
    
  - name: redis
    version: "16.13.2"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```yaml
# helm/monitoring-system/values.yaml
global:
  environment: production
  namespace: monitoring
  storageClass: fast-ssd
  
kafka:
  enabled: true
  replicaCount: 3
  heapOpts: "-Xmx8g -Xms8g"
  resources:
    requests:
      memory: 16Gi
      cpu: 4
    limits:
      memory: 16Gi
      cpu: 4
  persistence:
    enabled: true
    size: 500Gi
  zookeeper:
    replicaCount: 3
    
timescaledb:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      memory: 32Gi
      cpu: 8
    limits:
      memory: 32Gi
      cpu: 8
  persistence:
    enabled: true
    size: 1Ti
    
clickhouse:
  enabled: true
  shards: 3
  replicas: 2
  resources:
    requests:
      memory: 64Gi
      cpu: 16
    limits:
      memory: 64Gi
      cpu: 16
  persistence:
    enabled: true
    size: 2Ti
    
redis:
  enabled: true
  architecture: cluster
  redis:
    replicaCount: 6
  resources:
    requests:
      memory: 16Gi
      cpu: 4
    limits:
      memory: 16Gi
      cpu: 4
      
metricsGateway:
  replicaCount: 3
  image:
    repository: monitoring/metrics-gateway
    tag: "1.0.0"
  resources:
    requests:
      memory: 4Gi
      cpu: 2
    limits:
      memory: 4Gi
      cpu: 2
  service:
    type: LoadBalancer
    port: 8080
    
storageWriter:
  replicaCount: 2
  image:
    repository: monitoring/storage-writer
    tag: "1.0.0"
  resources:
    requests:
      memory: 8Gi
      cpu: 4
    limits:
      memory: 8Gi
      cpu: 4
      
aggregationService:
  replicaCount: 2
  image:
    repository: monitoring/aggregation-service
    tag: "1.0.0"
  resources:
    requests:
      memory: 4Gi
      cpu: 2
    limits:
      memory: 4Gi
      cpu: 2
      
alertEngine:
  replicaCount: 2
  image:
    repository: monitoring/alert-engine
    tag: "1.0.0"
  resources:
    requests:
      memory: 4Gi
      cpu: 2
    limits:
      memory: 4Gi
      cpu: 2
```

## Container Images

### Docker Build Pipeline

```dockerfile
# metrics-gateway/Dockerfile
FROM python:3.9-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd --create-home --shell /bin/bash app
USER app

EXPOSE 8080

CMD ["python", "metrics_gateway.py"]
```

```yaml
# .github/workflows/build-and-push.yml
name: Build and Push Container Images

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        component:
          - metrics-gateway
          - storage-writer
          - aggregation-service
          - alert-engine
          
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
        
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.component }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: ./${{ matrix.component }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Configuration Management

### ConfigMaps and Secrets

```yaml
# configs/alert-rules-configmap.yml
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
          runbook_url: "https://runbooks.company.com/high-cpu"
          
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
          runbook_url: "https://runbooks.company.com/memory-exhaustion"
```

```yaml
# configs/notification-config.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notification-config
  namespace: monitoring
data:
  notification-config.yml: |
    notification_config:
      teams:
        infrastructure:
          warning_channels:
            - type: email
              to_emails: ["infra-team@company.com"]
              from_email: "alerts@company.com"
              smtp_host: "smtp.company.com"
              smtp_port: 587
              use_tls: true
              
          critical_channels:
            - type: email
              to_emails: ["infra-team@company.com"]
              from_email: "alerts@company.com"
              smtp_host: "smtp.company.com"
              smtp_port: 587
              use_tls: true
              
            - type: pagerduty
              integration_key_secret: "pagerduty-integration-key"
              
        backend:
          warning_channels:
            - type: webhook
              url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
              auth_header_secret: "slack-webhook-token"
              
          critical_channels:
            - type: email
              to_emails: ["backend-team@company.com", "oncall@company.com"]
              
            - type: pagerduty
              integration_key_secret: "backend-pagerduty-key"
```

```yaml
# secrets/database-secrets.yml
apiVersion: v1
kind: Secret
metadata:
  name: timescaledb-secret
  namespace: monitoring
type: Opaque
data:
  username: bWV0cmljc191c2Vy  # base64 encoded
  password: <base64-encoded-password>
  
---
apiVersion: v1
kind: Secret
metadata:
  name: clickhouse-secret
  namespace: monitoring
type: Opaque
data:
  username: ZGVmYXVsdA==  # base64 encoded "default"
  password: <base64-encoded-password>
```

## Deployment Procedures

### Blue-Green Deployment Strategy

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

set -euo pipefail

ENVIRONMENT=${1:-staging}
VERSION=${2:-latest}
NAMESPACE="monitoring"

echo "Starting blue-green deployment for version $VERSION in $ENVIRONMENT"

# Step 1: Deploy new version (green)
echo "Deploying green environment..."
helm upgrade monitoring-system-green ./helm/monitoring-system \
  --namespace "$NAMESPACE" \
  --values "environments/$ENVIRONMENT/values.yml" \
  --set global.version="$VERSION" \
  --set global.environment="green" \
  --install \
  --wait \
  --timeout=30m

# Step 2: Run health checks on green environment
echo "Running health checks on green environment..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=monitoring-system-green \
  --namespace="$NAMESPACE" \
  --timeout=600s

# Run application-level health checks
./scripts/health-check.sh "green" "$NAMESPACE"

if [ $? -ne 0 ]; then
  echo "Health checks failed for green environment"
  exit 1
fi

# Step 3: Switch traffic to green environment
echo "Switching traffic to green environment..."
kubectl patch service metrics-gateway-service \
  --namespace="$NAMESPACE" \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/selector/app.kubernetes.io~1instance", "value": "monitoring-system-green"}]'

# Step 4: Wait and monitor
echo "Monitoring green environment for 5 minutes..."
sleep 300

# Step 5: Remove blue environment
echo "Removing blue environment..."
helm uninstall monitoring-system-blue --namespace="$NAMESPACE" || true

echo "Blue-green deployment completed successfully"
```

### Rollback Procedure

```bash
#!/bin/bash
# scripts/rollback.sh

set -euo pipefail

NAMESPACE="monitoring"
PREVIOUS_VERSION=${1:-$(helm list -n $NAMESPACE -o json | jq -r '.[0].app_version')}

echo "Rolling back to version $PREVIOUS_VERSION"

# Step 1: Deploy previous version as blue
helm upgrade monitoring-system-blue ./helm/monitoring-system \
  --namespace "$NAMESPACE" \
  --values "environments/production/values.yml" \
  --set global.version="$PREVIOUS_VERSION" \
  --set global.environment="blue" \
  --install \
  --wait \
  --timeout=15m

# Step 2: Health check
./scripts/health-check.sh "blue" "$NAMESPACE"

# Step 3: Switch traffic back
kubectl patch service metrics-gateway-service \
  --namespace="$NAMESPACE" \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/selector/app.kubernetes.io~1instance", "value": "monitoring-system-blue"}]'

echo "Rollback completed to version $PREVIOUS_VERSION"
```

## Operational Procedures

### Backup and Recovery

```bash
#!/bin/bash
# scripts/backup.sh

set -euo pipefail

DATE=$(date +%Y%m%d_%H%M%S)
NAMESPACE="monitoring"
BACKUP_LOCATION="s3://monitoring-backups"

echo "Starting backup at $DATE"

# Backup TimescaleDB
echo "Backing up TimescaleDB..."
kubectl exec -n "$NAMESPACE" timescaledb-0 -- pg_dump -U metrics_user metrics | \
  gzip | aws s3 cp - "$BACKUP_LOCATION/timescaledb/backup_$DATE.sql.gz"

# Backup ClickHouse
echo "Backing up ClickHouse..."
kubectl exec -n "$NAMESPACE" clickhouse-0 -- clickhouse-client --query "BACKUP DATABASE default TO S3('$BACKUP_LOCATION/clickhouse/backup_$DATE/', 'AWS_ACCESS_KEY', 'AWS_SECRET_KEY')"

# Backup configurations
echo "Backing up configurations..."
kubectl get configmaps,secrets -n "$NAMESPACE" -o yaml | \
  gzip | aws s3 cp - "$BACKUP_LOCATION/configs/configs_$DATE.yml.gz"

echo "Backup completed successfully"
```

### Health Check Script

```bash
#!/bin/bash
# scripts/health-check.sh

set -euo pipefail

ENVIRONMENT=${1:-blue}
NAMESPACE=${2:-monitoring}

echo "Running health checks for $ENVIRONMENT environment in $NAMESPACE namespace"

# Check pod health
echo "Checking pod health..."
kubectl get pods -n "$NAMESPACE" -l app.kubernetes.io/instance="monitoring-system-$ENVIRONMENT" \
  --field-selector=status.phase!=Running && exit 1 || echo "All pods are running"

# Check service endpoints
echo "Checking service endpoints..."
GATEWAY_IP=$(kubectl get service metrics-gateway-service -n "$NAMESPACE" -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

if [ -z "$GATEWAY_IP" ]; then
  echo "Failed to get gateway IP"
  exit 1
fi

# Test metrics ingestion
echo "Testing metrics ingestion..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST "http://$GATEWAY_IP:8080/health")

if [ "$HTTP_STATUS" != "200" ]; then
  echo "Health check failed with status $HTTP_STATUS"
  exit 1
fi

# Test database connectivity
echo "Testing database connectivity..."
kubectl exec -n "$NAMESPACE" timescaledb-0 -- pg_isready -U metrics_user -d metrics
kubectl exec -n "$NAMESPACE" clickhouse-0 -- clickhouse-client --query "SELECT 1"

echo "All health checks passed"
```

### Monitoring and Alerting for the Monitoring System

```yaml
# self-monitoring/prometheus-rules.yml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: monitoring-system-alerts
  namespace: monitoring
spec:
  groups:
    - name: monitoring-system
      rules:
        - alert: KafkaConsumerLag
          expr: kafka_consumer_lag_sum > 10000
          for: 5m
          labels:
            severity: warning
            component: kafka
          annotations:
            summary: "High consumer lag in Kafka"
            description: "Consumer lag is {{ $value }} messages"
            
        - alert: TimescaleDBConnectionFailure
          expr: up{job="timescaledb"} == 0
          for: 1m
          labels:
            severity: critical
            component: timescaledb
          annotations:
            summary: "TimescaleDB is down"
            description: "TimescaleDB connection check failed"
            
        - alert: ClickHouseQueryErrors
          expr: rate(clickhouse_query_errors_total[5m]) > 0.1
          for: 5m
          labels:
            severity: warning
            component: clickhouse
          annotations:
            summary: "High query error rate in ClickHouse"
            description: "Query error rate is {{ $value }} errors/second"
```

This deployment guide provides a comprehensive approach to deploying and operating the monitoring system in production environments with proper automation, monitoring, and operational procedures.
