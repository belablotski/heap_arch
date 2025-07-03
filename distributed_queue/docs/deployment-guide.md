# Deployment Guide

## Overview

This guide provides comprehensive deployment instructions for the distributed message queue system across different environments and platforms.

## Prerequisites

### System Requirements

#### Minimum Hardware Requirements
- **CPU**: 8 cores @ 2.4GHz
- **Memory**: 16 GB RAM
- **Storage**: 100 GB SSD (for logs) + 500 GB HDD (for retention)
- **Network**: 1 Gbps network interface

#### Recommended Production Hardware
- **CPU**: 16+ cores @ 3.0GHz+
- **Memory**: 64+ GB RAM
- **Storage**: 500 GB NVMe SSD (for active segments) + 2 TB SSD (for retention)
- **Network**: 10 Gbps network interface

#### Operating System Requirements
- **Linux**: Ubuntu 20.04 LTS, CentOS 8, RHEL 8, Amazon Linux 2
- **Java**: OpenJDK 11 or 17
- **Python**: 3.8+ (for operational scripts)

### Network Configuration

#### Port Requirements
- **9092**: Broker client port (PLAINTEXT)
- **9093**: Broker client port (SSL)
- **9094**: Broker internal replication port
- **8080**: HTTP metrics and health check port
- **2181**: Zookeeper client port
- **2888**: Zookeeper peer port
- **3888**: Zookeeper leader election port

#### Firewall Configuration
```bash
# Allow message queue ports
sudo ufw allow 9092:9094/tcp
sudo ufw allow 8080/tcp

# Allow Zookeeper ports
sudo ufw allow 2181/tcp
sudo ufw allow 2888/tcp
sudo ufw allow 3888/tcp

# Allow SSH
sudo ufw allow 22/tcp

# Enable firewall
sudo ufw enable
```

## Deployment Options

### 1. Docker Deployment

#### Docker Compose Configuration

**docker-compose.yml**
```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    hostname: zookeeper
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log
    networks:
      - mq-network
    restart: unless-stopped

  broker-1:
    image: message-queue/broker:latest
    hostname: broker-1
    container_name: broker-1
    depends_on:
      - zookeeper
    environment:
      BROKER_ID: 1
      ZOOKEEPER_CONNECT: zookeeper:2181
      LISTENERS: PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093
      ADVERTISED_LISTENERS: PLAINTEXT://broker-1:9092,SSL://broker-1:9093
      SECURITY_INTER_BROKER_PROTOCOL: PLAINTEXT
      LOG_DIRS: /var/lib/kafka/data
      NUM_NETWORK_THREADS: 8
      NUM_IO_THREADS: 8
      LOG_RETENTION_HOURS: 168
      LOG_SEGMENT_BYTES: 1073741824
      DEFAULT_REPLICATION_FACTOR: 3
      MIN_INSYNC_REPLICAS: 2
    volumes:
      - broker-1-data:/var/lib/kafka/data
      - ./ssl:/etc/kafka/ssl
      - ./config:/etc/kafka/config
    ports:
      - "9092:9092"
      - "9093:9093"
      - "8080:8080"
    networks:
      - mq-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  broker-2:
    image: message-queue/broker:latest
    hostname: broker-2
    container_name: broker-2
    depends_on:
      - zookeeper
    environment:
      BROKER_ID: 2
      ZOOKEEPER_CONNECT: zookeeper:2181
      LISTENERS: PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093
      ADVERTISED_LISTENERS: PLAINTEXT://broker-2:9092,SSL://broker-2:9093
      SECURITY_INTER_BROKER_PROTOCOL: PLAINTEXT
      LOG_DIRS: /var/lib/kafka/data
      NUM_NETWORK_THREADS: 8
      NUM_IO_THREADS: 8
      LOG_RETENTION_HOURS: 168
      LOG_SEGMENT_BYTES: 1073741824
      DEFAULT_REPLICATION_FACTOR: 3
      MIN_INSYNC_REPLICAS: 2
    volumes:
      - broker-2-data:/var/lib/kafka/data
      - ./ssl:/etc/kafka/ssl
      - ./config:/etc/kafka/config
    ports:
      - "9095:9092"
      - "9096:9093"
      - "8081:8080"
    networks:
      - mq-network
    restart: unless-stopped

  broker-3:
    image: message-queue/broker:latest
    hostname: broker-3
    container_name: broker-3
    depends_on:
      - zookeeper
    environment:
      BROKER_ID: 3
      ZOOKEEPER_CONNECT: zookeeper:2181
      LISTENERS: PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093
      ADVERTISED_LISTENERS: PLAINTEXT://broker-3:9092,SSL://broker-3:9093
      SECURITY_INTER_BROKER_PROTOCOL: PLAINTEXT
      LOG_DIRS: /var/lib/kafka/data
      NUM_NETWORK_THREADS: 8
      NUM_IO_THREADS: 8
      LOG_RETENTION_HOURS: 168
      LOG_SEGMENT_BYTES: 1073741824
      DEFAULT_REPLICATION_FACTOR: 3
      MIN_INSYNC_REPLICAS: 2
    volumes:
      - broker-3-data:/var/lib/kafka/data
      - ./ssl:/etc/kafka/ssl
      - ./config:/etc/kafka/config
    ports:
      - "9097:9092"
      - "9098:9093"
      - "8082:8080"
    networks:
      - mq-network
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - mq-network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana:/etc/grafana/provisioning
    networks:
      - mq-network
    restart: unless-stopped

volumes:
  zookeeper-data:
  zookeeper-logs:
  broker-1-data:
  broker-2-data:
  broker-3-data:
  prometheus-data:
  grafana-data:

networks:
  mq-network:
    driver: bridge
```

#### Broker Dockerfile

```dockerfile
FROM openjdk:17-jre-slim

# Install required packages
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Create kafka user and group
RUN groupadd -r kafka && useradd -r -g kafka kafka

# Set up directories
RUN mkdir -p /opt/kafka /var/lib/kafka/data /etc/kafka/config /var/log/kafka
RUN chown -R kafka:kafka /opt/kafka /var/lib/kafka /etc/kafka /var/log/kafka

# Copy application JAR
COPY target/message-queue-broker-*.jar /opt/kafka/broker.jar
COPY scripts/start-broker.sh /opt/kafka/start-broker.sh
COPY config/broker.properties /etc/kafka/config/broker.properties

# Make scripts executable
RUN chmod +x /opt/kafka/start-broker.sh

# Switch to kafka user
USER kafka

# Set working directory
WORKDIR /opt/kafka

# Expose ports
EXPOSE 9092 9093 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Start the broker
CMD ["./start-broker.sh"]
```

#### Startup Script

**scripts/start-broker.sh**
```bash
#!/bin/bash

set -e

# Wait for Zookeeper to be ready
echo "Waiting for Zookeeper..."
while ! nc -z ${ZOOKEEPER_HOST:-zookeeper} ${ZOOKEEPER_PORT:-2181}; do
    sleep 1
done
echo "Zookeeper is ready"

# Generate broker configuration
envsubst < /etc/kafka/config/broker.properties.template > /etc/kafka/config/broker.properties

# Start JVM with optimized settings
export KAFKA_HEAP_OPTS="-Xmx4G -Xms4G"
export KAFKA_JVM_PERFORMANCE_OPTS="-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true"
export KAFKA_JVM_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=9999"

echo "Starting Message Queue Broker..."
exec java $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_JVM_OPTS \
    -cp /opt/kafka/broker.jar \
    com.mq.broker.BrokerServer \
    /etc/kafka/config/broker.properties
```

### 2. Kubernetes Deployment

#### Namespace and ConfigMap

**k8s/namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: message-queue
```

**k8s/configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: broker-config
  namespace: message-queue
data:
  broker.properties: |
    num.network.threads=8
    num.io.threads=8
    socket.send.buffer.bytes=102400
    socket.receive.buffer.bytes=102400
    socket.request.max.bytes=104857600
    log.segment.bytes=1073741824
    log.retention.hours=168
    log.cleanup.policy=delete
    default.replication.factor=3
    min.insync.replicas=2
    unclean.leader.election.enable=false
    auto.create.topics.enable=false
    delete.topic.enable=true
    compression.type=snappy
    log4j.properties: |
    log4j.rootLogger=INFO, stdout, kafkaAppender
    log4j.appender.stdout=org.apache.log4j.ConsoleAppender
    log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
    log4j.appender.stdout.layout.ConversionPattern=[%d] %p %m (%c)%n
```

#### Zookeeper StatefulSet

**k8s/zookeeper.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zookeeper
  namespace: message-queue
spec:
  serviceName: zookeeper
  replicas: 3
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
      - name: zookeeper
        image: confluentinc/cp-zookeeper:7.4.0
        ports:
        - containerPort: 2181
        - containerPort: 2888
        - containerPort: 3888
        env:
        - name: ZOOKEEPER_SERVER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['statefulset.kubernetes.io/pod-name']
        - name: ZOOKEEPER_CLIENT_PORT
          value: "2181"
        - name: ZOOKEEPER_TICK_TIME
          value: "2000"
        - name: ZOOKEEPER_INIT_LIMIT
          value: "5"
        - name: ZOOKEEPER_SYNC_LIMIT
          value: "2"
        - name: ZOOKEEPER_SERVERS
          value: "zookeeper-0.zookeeper:2888:3888;zookeeper-1.zookeeper:2888:3888;zookeeper-2.zookeeper:2888:3888"
        volumeMounts:
        - name: zookeeper-data
          mountPath: /var/lib/zookeeper/data
        - name: zookeeper-logs
          mountPath: /var/lib/zookeeper/log
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: zookeeper-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
  - metadata:
      name: zookeeper-logs
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: message-queue
spec:
  ports:
  - port: 2181
    name: client
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zookeeper
```

#### Broker StatefulSet

**k8s/broker.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: broker
  namespace: message-queue
spec:
  serviceName: broker
  replicas: 3
  selector:
    matchLabels:
      app: broker
  template:
    metadata:
      labels:
        app: broker
    spec:
      containers:
      - name: broker
        image: message-queue/broker:latest
        ports:
        - containerPort: 9092
        - containerPort: 9093
        - containerPort: 8080
        env:
        - name: BROKER_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['statefulset.kubernetes.io/pod-name']
        - name: ZOOKEEPER_CONNECT
          value: "zookeeper-0.zookeeper:2181,zookeeper-1.zookeeper:2181,zookeeper-2.zookeeper:2181"
        - name: LISTENERS
          value: "PLAINTEXT://0.0.0.0:9092,SSL://0.0.0.0:9093"
        - name: ADVERTISED_LISTENERS
          value: "PLAINTEXT://$(POD_NAME).broker:9092,SSL://$(POD_NAME).broker:9093"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: broker-data
          mountPath: /var/lib/kafka/data
        - name: broker-config
          mountPath: /etc/kafka/config
        resources:
          requests:
            memory: "4Gi"
            cpu: "2000m"
          limits:
            memory: "8Gi"
            cpu: "4000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
      volumes:
      - name: broker-config
        configMap:
          name: broker-config
  volumeClaimTemplates:
  - metadata:
      name: broker-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi

---
apiVersion: v1
kind: Service
metadata:
  name: broker
  namespace: message-queue
spec:
  ports:
  - port: 9092
    name: plaintext
  - port: 9093
    name: ssl
  - port: 8080
    name: metrics
  clusterIP: None
  selector:
    app: broker

---
apiVersion: v1
kind: Service
metadata:
  name: broker-external
  namespace: message-queue
spec:
  type: LoadBalancer
  ports:
  - port: 9092
    targetPort: 9092
    name: plaintext
  - port: 9093
    targetPort: 9093
    name: ssl
  selector:
    app: broker
```

### 3. Bare Metal / VM Deployment

#### Installation Script

**scripts/install.sh**
```bash
#!/bin/bash

set -e

# Configuration
KAFKA_USER="kafka"
KAFKA_HOME="/opt/kafka"
KAFKA_DATA="/var/lib/kafka"
KAFKA_LOGS="/var/log/kafka"
ZOOKEEPER_DATA="/var/lib/zookeeper"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] WARNING: $1${NC}"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR: $1${NC}"
    exit 1
}

# Check if running as root
if [[ $EUID -eq 0 ]]; then
    error "This script should not be run as root"
fi

# Update system packages
log "Updating system packages..."
sudo apt-get update

# Install Java 17
log "Installing Java 17..."
sudo apt-get install -y openjdk-17-jdk

# Verify Java installation
java -version || error "Java installation failed"

# Create kafka user
log "Creating kafka user..."
sudo useradd -r -s /bin/false kafka || warn "User kafka already exists"

# Create directories
log "Creating directories..."
sudo mkdir -p $KAFKA_HOME $KAFKA_DATA $KAFKA_LOGS $ZOOKEEPER_DATA
sudo chown -R kafka:kafka $KAFKA_HOME $KAFKA_DATA $KAFKA_LOGS $ZOOKEEPER_DATA

# Download and install application
log "Installing Message Queue application..."
sudo cp target/message-queue-broker-*.jar $KAFKA_HOME/broker.jar
sudo cp -r config/* /etc/kafka/
sudo cp -r scripts/* $KAFKA_HOME/scripts/
sudo chown -R kafka:kafka $KAFKA_HOME /etc/kafka

# Create systemd service files
log "Creating systemd services..."

# Zookeeper service
sudo tee /etc/systemd/system/zookeeper.service > /dev/null <<EOF
[Unit]
Description=Zookeeper
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
ExecStart=/usr/bin/java -Xmx1G -Xms1G -cp $KAFKA_HOME/broker.jar com.mq.zookeeper.ZookeeperServer /etc/kafka/zookeeper.properties
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Broker service
sudo tee /etc/systemd/system/kafka-broker.service > /dev/null <<EOF
[Unit]
Description=Message Queue Broker
After=network.target zookeeper.service
Requires=zookeeper.service

[Service]
Type=simple
User=kafka
Group=kafka
Environment=KAFKA_HEAP_OPTS=-Xmx4G -Xms4G
Environment=KAFKA_JVM_PERFORMANCE_OPTS=-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35
ExecStart=/usr/bin/java \$KAFKA_HEAP_OPTS \$KAFKA_JVM_PERFORMANCE_OPTS -cp $KAFKA_HOME/broker.jar com.mq.broker.BrokerServer /etc/kafka/broker.properties
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Reload systemd
sudo systemctl daemon-reload

# Configure log rotation
log "Configuring log rotation..."
sudo tee /etc/logrotate.d/kafka > /dev/null <<EOF
$KAFKA_LOGS/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
EOF

# Set up monitoring
log "Setting up monitoring..."
# Install Prometheus node exporter
wget -O /tmp/node_exporter.tar.gz https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz
tar -xf /tmp/node_exporter.tar.gz -C /tmp
sudo mv /tmp/node_exporter-1.6.0.linux-amd64/node_exporter /usr/local/bin/
sudo chown root:root /usr/local/bin/node_exporter

# Node exporter service
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

log "Installation completed successfully!"
log "To start services:"
log "  sudo systemctl enable zookeeper kafka-broker"
log "  sudo systemctl start zookeeper"
log "  sudo systemctl start kafka-broker"
```

#### Cluster Setup Script

**scripts/setup-cluster.sh**
```bash
#!/bin/bash

# Cluster configuration
BROKERS=("broker1.example.com" "broker2.example.com" "broker3.example.com")
ZOOKEEPERS=("zk1.example.com" "zk2.example.com" "zk3.example.com")

# SSH key for remote access
SSH_KEY="~/.ssh/id_rsa"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Function to execute command on remote host
remote_exec() {
    local host=$1
    local command=$2
    ssh -i $SSH_KEY ubuntu@$host "$command"
}

# Deploy to all Zookeeper nodes
log "Setting up Zookeeper cluster..."
for i in "${!ZOOKEEPERS[@]}"; do
    host=${ZOOKEEPERS[$i]}
    broker_id=$((i + 1))
    
    log "Configuring Zookeeper on $host (ID: $broker_id)"
    
    # Generate Zookeeper configuration
    zk_config="
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper/data
clientPort=2181
"
    
    for j in "${!ZOOKEEPERS[@]}"; do
        zk_id=$((j + 1))
        zk_config="${zk_config}server.${zk_id}=${ZOOKEEPERS[$j]}:2888:3888
"
    done
    
    # Deploy configuration
    echo "$zk_config" | ssh -i $SSH_KEY ubuntu@$host "sudo tee /etc/kafka/zookeeper.properties > /dev/null"
    remote_exec $host "echo $broker_id | sudo tee /var/lib/zookeeper/data/myid > /dev/null"
    remote_exec $host "sudo systemctl enable zookeeper && sudo systemctl start zookeeper"
done

# Wait for Zookeeper cluster to be ready
log "Waiting for Zookeeper cluster to be ready..."
sleep 30

# Deploy to all broker nodes
log "Setting up broker cluster..."
for i in "${!BROKERS[@]}"; do
    host=${BROKERS[$i]}
    broker_id=$((i + 1))
    
    log "Configuring broker on $host (ID: $broker_id)"
    
    # Generate broker configuration
    zk_connect=$(IFS=:2181,; echo "${ZOOKEEPERS[*]}:2181")
    
    broker_config="
broker.id=$broker_id
listeners=PLAINTEXT://:9092,SSL://:9093
advertised.listeners=PLAINTEXT://$host:9092,SSL://$host:9093
zookeeper.connect=$zk_connect
log.dirs=/var/lib/kafka/data
num.network.threads=8
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.segment.bytes=1073741824
log.retention.hours=168
log.cleanup.policy=delete
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
auto.create.topics.enable=false
delete.topic.enable=true
"
    
    # Deploy configuration
    echo "$broker_config" | ssh -i $SSH_KEY ubuntu@$host "sudo tee /etc/kafka/broker.properties > /dev/null"
    remote_exec $host "sudo systemctl enable kafka-broker && sudo systemctl start kafka-broker"
done

log "Cluster setup completed!"
log "Verifying cluster health..."

# Wait for brokers to start
sleep 60

# Check cluster health
for host in "${BROKERS[@]}"; do
    log "Checking health of $host..."
    if remote_exec $host "curl -s http://localhost:8080/health | grep -q healthy"; then
        log "✓ $host is healthy"
    else
        log "✗ $host is not healthy"
    fi
done

log "Cluster deployment completed!"
```

## Configuration Management

### Environment-Specific Configurations

#### Development Environment

**config/dev/broker.properties**
```properties
# Development environment configuration
broker.id=1
listeners=PLAINTEXT://:9092
zookeeper.connect=localhost:2181

# Reduced settings for development
log.dirs=/tmp/kafka-logs
log.segment.bytes=268435456
log.retention.hours=24
num.network.threads=3
num.io.threads=3

# Faster startup
unclean.leader.election.enable=true
auto.create.topics.enable=true
default.replication.factor=1
min.insync.replicas=1
```

#### Staging Environment

**config/staging/broker.properties**
```properties
# Staging environment configuration
listeners=PLAINTEXT://:9092,SSL://:9093
security.inter.broker.protocol=SSL
ssl.keystore.location=/etc/kafka/ssl/broker.keystore.jks
ssl.keystore.password=${SSL_KEYSTORE_PASSWORD}
ssl.truststore.location=/etc/kafka/ssl/broker.truststore.jks
ssl.truststore.password=${SSL_TRUSTSTORE_PASSWORD}

# Production-like settings
log.segment.bytes=1073741824
log.retention.hours=168
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
auto.create.topics.enable=false
```

#### Production Environment

**config/prod/broker.properties**
```properties
# Production environment configuration
listeners=SSL://:9093
security.inter.broker.protocol=SSL
ssl.client.auth=required
ssl.keystore.location=/etc/kafka/ssl/broker.keystore.jks
ssl.keystore.password=${SSL_KEYSTORE_PASSWORD}
ssl.truststore.location=/etc/kafka/ssl/broker.truststore.jks
ssl.truststore.password=${SSL_TRUSTSTORE_PASSWORD}

# Security
authorizer.class.name=com.mq.security.SimpleAclAuthorizer
super.users=User:admin

# Performance optimized settings
num.network.threads=16
num.io.threads=16
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
log.segment.bytes=1073741824
log.retention.hours=168
compression.type=snappy

# High availability
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
auto.create.topics.enable=false
delete.topic.enable=false
```

### SSL Certificate Generation

**scripts/generate-ssl-certs.sh**
```bash
#!/bin/bash

set -e

# Configuration
CERT_DIR="/etc/kafka/ssl"
KEYSTORE_PASSWORD="changeit"
TRUSTSTORE_PASSWORD="changeit"
VALIDITY_DAYS=3650

# Create certificate directory
sudo mkdir -p $CERT_DIR
cd $CERT_DIR

# Generate CA private key
openssl genrsa -out ca-key.pem 2048

# Generate CA certificate
openssl req -new -x509 -key ca-key.pem -out ca-cert.pem -days $VALIDITY_DAYS \
    -subj "/C=US/ST=CA/L=San Francisco/O=MessageQueue/OU=Engineering/CN=MessageQueue-CA"

# Generate broker private key
openssl genrsa -out broker-key.pem 2048

# Generate broker certificate signing request
openssl req -new -key broker-key.pem -out broker-csr.pem \
    -subj "/C=US/ST=CA/L=San Francisco/O=MessageQueue/OU=Engineering/CN=broker"

# Generate broker certificate signed by CA
openssl x509 -req -in broker-csr.pem -CA ca-cert.pem -CAkey ca-key.pem \
    -out broker-cert.pem -days $VALIDITY_DAYS -CAcreateserial

# Create broker keystore
openssl pkcs12 -export -in broker-cert.pem -inkey broker-key.pem \
    -out broker.p12 -name broker -password pass:$KEYSTORE_PASSWORD

keytool -importkeystore -deststorepass $KEYSTORE_PASSWORD \
    -destkeypass $KEYSTORE_PASSWORD -destkeystore broker.keystore.jks \
    -srckeystore broker.p12 -srcstoretype PKCS12 -srcstorepass $KEYSTORE_PASSWORD

# Create truststore
keytool -import -alias ca -file ca-cert.pem -keystore broker.truststore.jks \
    -storepass $TRUSTSTORE_PASSWORD -noprompt

# Set permissions
sudo chown -R kafka:kafka $CERT_DIR
sudo chmod 600 $CERT_DIR/*.pem $CERT_DIR/*.jks

echo "SSL certificates generated successfully!"
echo "Keystore password: $KEYSTORE_PASSWORD"
echo "Truststore password: $TRUSTSTORE_PASSWORD"
```

## Monitoring and Alerting Setup

### Prometheus Configuration

**monitoring/prometheus.yml**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'message-queue-brokers'
    static_configs:
      - targets: 
        - 'broker-1:8080'
        - 'broker-2:8080'  
        - 'broker-3:8080'
    metrics_path: /metrics
    scrape_interval: 15s

  - job_name: 'zookeeper'
    static_configs:
      - targets:
        - 'zookeeper-1:8080'
        - 'zookeeper-2:8080'
        - 'zookeeper-3:8080'

  - job_name: 'node-exporter'
    static_configs:
      - targets:
        - 'broker-1:9100'
        - 'broker-2:9100'
        - 'broker-3:9100'
```

### Alert Rules

**monitoring/alert_rules.yml**
```yaml
groups:
- name: message-queue-alerts
  rules:
  - alert: BrokerDown
    expr: up{job="message-queue-brokers"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Message Queue broker is down"
      description: "Broker {{ $labels.instance }} has been down for more than 1 minute."

  - alert: HighProducerLatency
    expr: histogram_quantile(0.95, rate(produce_request_time_ms_bucket[5m])) > 100
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High producer latency detected"
      description: "95th percentile producer latency is {{ $value }}ms on {{ $labels.instance }}"

  - alert: HighConsumerLag
    expr: consumer_lag_max > 100000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High consumer lag detected"
      description: "Consumer group {{ $labels.group }} has lag of {{ $value }} messages"

  - alert: DiskSpaceHigh
    expr: (node_filesystem_size_bytes{mountpoint="/var/lib/kafka"} - node_filesystem_free_bytes{mountpoint="/var/lib/kafka"}) / node_filesystem_size_bytes{mountpoint="/var/lib/kafka"} > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Disk space usage is high"
      description: "Disk usage on {{ $labels.instance }} is {{ $value | humanizePercentage }}"

  - alert: UnderReplicatedPartitions
    expr: under_replicated_partitions > 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Under-replicated partitions detected"
      description: "{{ $value }} partitions are under-replicated on {{ $labels.instance }}"
```

This deployment guide provides comprehensive instructions for deploying the distributed message queue system across different environments and platforms, with proper configuration management, security setup, and monitoring integration.
