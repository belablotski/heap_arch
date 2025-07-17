# Deployment Guide - Agentic Device Fleet Monitoring System

## Prerequisites

### Infrastructure Requirements

- **Kubernetes Cluster**: v1.24+ with at least 3 nodes
- **Storage**: Persistent volumes for Trino and metadata
- **Networking**: Load balancer for external access
- **Resources**: Minimum 16GB RAM, 8 CPU cores per node

### External Dependencies

- **Kafka Cluster**: Existing Kafka infrastructure for device events
- **Object Storage**: S3-compatible storage for data lake
- **Database**: Access to existing MSSQL database for real-time queries
- **AI Provider**: OpenAI API key or Azure OpenAI endpoint

## Environment Setup

### 1. Configuration Management

Create configuration files for each environment:

**config/production.yaml**
```yaml
environment: production
ai:
  provider: openai
  model: gpt-4
  api_key: ${OPENAI_API_KEY}
  temperature: 0.1
  max_tokens: 2000

trino:
  coordinator_host: trino-coordinator
  port: 8080
  catalog: datalake
  schema: fleet
  
kafka:
  brokers: 
    - kafka-broker-1:9092
    - kafka-broker-2:9092
    - kafka-broker-3:9092
  topics:
    device_events: device-status-events
  consumer_group: device-fleet-agent

storage:
  type: s3
  bucket: device-fleet-data-lake
  region: us-west-2
  access_key_id: ${AWS_ACCESS_KEY_ID}
  secret_access_key: ${AWS_SECRET_ACCESS_KEY}

database:
  mssql:
    host: ${MSSQL_HOST}
    port: 1433
    database: DeviceManagement
    username: ${MSSQL_USERNAME}
    password: ${MSSQL_PASSWORD}

security:
  jwt_secret: ${JWT_SECRET}
  cors_origins: 
    - https://fleet-dashboard.company.com
  rate_limits:
    queries_per_minute: 100
    custom_sql_per_minute: 20
```

### 2. Secrets Management

```bash
# Create Kubernetes secrets
kubectl create secret generic device-fleet-secrets \
  --from-literal=openai-api-key=${OPENAI_API_KEY} \
  --from-literal=aws-access-key-id=${AWS_ACCESS_KEY_ID} \
  --from-literal=aws-secret-access-key=${AWS_SECRET_ACCESS_KEY} \
  --from-literal=mssql-username=${MSSQL_USERNAME} \
  --from-literal=mssql-password=${MSSQL_PASSWORD} \
  --from-literal=jwt-secret=${JWT_SECRET}
```

## Container Images

### 1. AI Agent Service

**Dockerfile**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/
COPY config/ ./config/

# Set environment
ENV PYTHONPATH=/app/src
ENV FLASK_APP=src.main:create_app

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "src.main:create_app()"]
```

**requirements.txt**
```
flask==2.3.2
gunicorn==21.2.0
openai==0.27.8
pandas==2.0.3
sqlalchemy==2.0.19
trino==0.324.0
kafka-python==2.0.2
boto3==1.28.25
pydantic==2.1.1
prometheus-client==0.17.1
```

### 2. MCP Server

**Dockerfile**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source code
COPY src/ ./src/
COPY config/ ./config/

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD node src/health-check.js || exit 1

EXPOSE 3000

USER node

CMD ["node", "src/server.js"]
```

### 3. Data Pipeline

**Dockerfile**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements-pipeline.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Copy pipeline code
COPY pipeline/ ./pipeline/
COPY config/ ./config/

ENV PYTHONPATH=/app

# Health check
HEALTHCHECK --interval=60s --timeout=15s --start-period=60s --retries=3 \
  CMD python pipeline/health_check.py || exit 1

CMD ["python", "pipeline/consumer.py"]
```

## Kubernetes Deployment

### 1. Namespace and RBAC

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: device-fleet
  labels:
    name: device-fleet

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: device-fleet-sa
  namespace: device-fleet

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: device-fleet-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: device-fleet-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: device-fleet-role
subjects:
- kind: ServiceAccount
  name: device-fleet-sa
  namespace: device-fleet
```

### 2. ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-fleet-config
  namespace: device-fleet
data:
  app-config.yaml: |
    environment: production
    trino:
      coordinator_host: trino-coordinator
      port: 8080
      catalog: datalake
      schema: fleet
    kafka:
      brokers: 
        - kafka-broker-1:9092
        - kafka-broker-2:9092
      topics:
        device_events: device-status-events
    logging:
      level: INFO
      format: json
```

### 3. AI Agent Deployment

```yaml
# ai-agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent
  namespace: device-fleet
  labels:
    app: ai-agent
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ai-agent
  template:
    metadata:
      labels:
        app: ai-agent
    spec:
      serviceAccountName: device-fleet-sa
      containers:
      - name: ai-agent
        image: device-fleet/ai-agent:1.0.0
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: CONFIG_PATH
          value: /config/app-config.yaml
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: device-fleet-secrets
              key: openai-api-key
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: device-fleet-secrets
              key: aws-access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: device-fleet-secrets
              key: aws-secret-access-key
        volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: device-fleet-config

---
apiVersion: v1
kind: Service
metadata:
  name: ai-agent-service
  namespace: device-fleet
spec:
  selector:
    app: ai-agent
  ports:
  - port: 80
    targetPort: 8000
    name: http
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-agent-hpa
  namespace: device-fleet
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-agent
  minReplicas: 3
  maxReplicas: 10
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
```

### 4. MCP Server Deployment

```yaml
# mcp-server-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server
  namespace: device-fleet
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mcp-server
  template:
    metadata:
      labels:
        app: mcp-server
    spec:
      containers:
      - name: mcp-server
        image: device-fleet/mcp-server:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: production
        - name: TRINO_HOST
          value: trino-coordinator
        - name: TRINO_PORT
          value: "8080"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"

---
apiVersion: v1
kind: Service
metadata:
  name: mcp-server-service
  namespace: device-fleet
spec:
  selector:
    app: mcp-server
  ports:
  - port: 3000
    targetPort: 3000
```

### 5. Trino Deployment

```yaml
# trino-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trino-coordinator
  namespace: device-fleet
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trino-coordinator
  template:
    metadata:
      labels:
        app: trino-coordinator
    spec:
      containers:
      - name: trino
        image: trinodb/trino:latest
        ports:
        - containerPort: 8080
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: device-fleet-secrets
              key: aws-access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: device-fleet-secrets
              key: aws-secret-access-key
        volumeMounts:
        - name: trino-config
          mountPath: /etc/trino
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
      volumes:
      - name: trino-config
        configMap:
          name: trino-config

---
apiVersion: v1
kind: Service
metadata:
  name: trino-coordinator
  namespace: device-fleet
spec:
  selector:
    app: trino-coordinator
  ports:
  - port: 8080
    targetPort: 8080
```

## Load Balancer and Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: device-fleet-ingress
  namespace: device-fleet
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.device-fleet.company.com
    secretName: device-fleet-tls
  rules:
  - host: api.device-fleet.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ai-agent-service
            port:
              number: 80
```

## Monitoring and Observability

### 1. Prometheus Monitoring

```yaml
# monitoring.yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: device-fleet-metrics
  namespace: device-fleet
spec:
  selector:
    matchLabels:
      app: ai-agent
  endpoints:
  - port: http
    path: /metrics
    interval: 30s

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: device-fleet-alerts
  namespace: device-fleet
spec:
  groups:
  - name: device-fleet
    rules:
    - alert: HighQueryLatency
      expr: avg(query_duration_seconds) > 5
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High query latency detected"
        description: "Average query latency is {{ $value }}s"
    
    - alert: ErrorRateHigh
      expr: rate(error_total[5m]) > 0.1
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
```

### 2. Logging

```yaml
# logging.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: device-fleet
data:
  fluent-bit.conf: |
    [INPUT]
        Name tail
        Path /var/log/containers/*device-fleet*.log
        Parser json
        Tag kube.*
        
    [OUTPUT]
        Name elasticsearch
        Match kube.*
        Host elasticsearch.logging.svc.cluster.local
        Port 9200
        Index device-fleet-logs
```

## Deployment Process

### 1. Pre-deployment Checklist

- [ ] Verify Kafka connectivity
- [ ] Confirm S3 bucket access
- [ ] Test MSSQL database connection
- [ ] Validate OpenAI API key
- [ ] Check resource quotas
- [ ] Review security policies

### 2. Deployment Steps

```bash
#!/bin/bash
# deploy.sh

set -e

NAMESPACE="device-fleet"
VERSION="${1:-latest}"

echo "Deploying Device Fleet Agent v${VERSION}..."

# Apply base resources
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secrets.yaml

# Deploy data layer
kubectl apply -f trino-deployment.yaml
kubectl wait --for=condition=available --timeout=300s deployment/trino-coordinator -n $NAMESPACE

# Deploy MCP server
kubectl apply -f mcp-server-deployment.yaml
kubectl wait --for=condition=available --timeout=180s deployment/mcp-server -n $NAMESPACE

# Deploy AI agent
kubectl apply -f ai-agent-deployment.yaml
kubectl wait --for=condition=available --timeout=180s deployment/ai-agent -n $NAMESPACE

# Configure networking
kubectl apply -f ingress.yaml

# Setup monitoring
kubectl apply -f monitoring.yaml

echo "Deployment complete!"
echo "Health check: https://api.device-fleet.company.com/health"
```

### 3. Post-deployment Verification

```bash
#!/bin/bash
# verify.sh

NAMESPACE="device-fleet"
BASE_URL="https://api.device-fleet.company.com"

# Health checks
echo "Checking system health..."
curl -f "$BASE_URL/health" || exit 1

# Test query
echo "Testing query functionality..."
curl -X POST "$BASE_URL/query" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TEST_TOKEN" \
  -d '{"query": "How many devices are active?"}' || exit 1

echo "Verification complete!"
```

## Rollback Strategy

```bash
#!/bin/bash
# rollback.sh

NAMESPACE="device-fleet"
PREVIOUS_VERSION="${1}"

if [ -z "$PREVIOUS_VERSION" ]; then
  echo "Usage: ./rollback.sh <previous_version>"
  exit 1
fi

echo "Rolling back to version $PREVIOUS_VERSION..."

kubectl rollout undo deployment/ai-agent -n $NAMESPACE --to-revision=$PREVIOUS_VERSION
kubectl rollout undo deployment/mcp-server -n $NAMESPACE --to-revision=$PREVIOUS_VERSION

kubectl rollout status deployment/ai-agent -n $NAMESPACE
kubectl rollout status deployment/mcp-server -n $NAMESPACE

echo "Rollback complete!"
```

## Maintenance

### Regular Tasks

- **Daily**: Monitor error rates and query performance
- **Weekly**: Review resource utilization and scaling metrics
- **Monthly**: Update dependencies and security patches
- **Quarterly**: Performance optimization and capacity planning

### Backup Strategy

- **Configuration**: Store in version control
- **Secrets**: Backup to secure vault
- **Trino Metadata**: Regular database backups
- **Query Logs**: Retain for 90 days for analysis
