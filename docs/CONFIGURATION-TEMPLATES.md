# ⚙️ Configuration Templates & Examples

## Environment Configuration Templates

### 1. Production Kafka Broker
```bash
#!/bin/bash
# /opt/kafka/config/broker-with-sni-agent.env

# Core agent configuration
export KAFKA_OPTS="$KAFKA_OPTS -javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar"

# SNI JFR Configuration
export KAFKA_OPTS="$KAFKA_OPTS -Dsni.jfr.output=/var/log/kafka/broker-sni-$(hostname)-$(date +%Y%m%d).jfr"
export KAFKA_OPTS="$KAFKA_OPTS -Dsni.jfr.debug=false"
export KAFKA_OPTS="$KAFKA_OPTS -Dsni.jfr.maxSize=200000000"  # 200MB

# JFR Performance tuning
export KAFKA_OPTS="$KAFKA_OPTS -XX:FlushInterval=60"
export KAFKA_OPTS="$KAFKA_OPTS -XX:StartFlightRecording=maxsize=100m,maxage=24h"

# Memory settings for agent
export KAFKA_HEAP_OPTS="-Xmx4g -Xms4g"
```

### 2. Development Environment
```bash
#!/bin/bash
# /opt/kafka/config/dev-with-sni-agent.env

# Agent with debug enabled
export KAFKA_OPTS="$KAFKA_OPTS -javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar"
export KAFKA_OPTS="$KAFKA_OPTS -Dsni.jfr.output=./dev-sni-capture.jfr"
export KAFKA_OPTS="$KAFKA_OPTS -Dsni.jfr.debug=true"
export KAFKA_OPTS="$KAFKA_OPTS -Dsni.jfr.maxSize=50000000"  # 50MB for dev

# Additional JFR events for debugging
export KAFKA_OPTS="$KAFKA_OPTS -XX:StartFlightRecording=duration=60s,filename=dev-startup.jfr"
```

### 3. Client Application Template
```bash
#!/bin/bash
# client-app-with-sni.sh

APP_NAME="kafka-client-app"
LOG_DIR="/var/log/apps/${APP_NAME}"

# Ensure log directory exists
mkdir -p ${LOG_DIR}

# Client-side SNI capture
export JAVA_OPTS="-javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar"
export JAVA_OPTS="$JAVA_OPTS -Dsni.jfr.output=${LOG_DIR}/client-sni-$(date +%Y%m%d_%H%M%S).jfr"
export JAVA_OPTS="$JAVA_OPTS -Dsni.jfr.debug=false"

# Start application
java $JAVA_OPTS -jar ${APP_NAME}.jar
```

## Systemd Service Templates

### 1. Kafka Broker with SNI Agent
```ini
# /etc/systemd/system/kafka-with-sni.service
[Unit]
Description=Apache Kafka with SNI JFR Agent
Documentation=https://kafka.apache.org/
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=kafka
Group=kafka
Environment="KAFKA_OPTS=-javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar -Dsni.jfr.output=/var/log/kafka/sni-capture.jfr"
Environment="KAFKA_HEAP_OPTS=-Xmx4g -Xms4g"
Environment="KAFKA_JVM_PERFORMANCE_OPTS=-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true"
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 2. Kafka Client Service
```ini
# /etc/systemd/system/kafka-client-with-sni.service
[Unit]
Description=Kafka Client with SNI Monitoring
After=network.target

[Service]
Type=simple
User=kafka-client
Group=kafka-client
WorkingDirectory=/opt/kafka-client
Environment="JAVA_OPTS=-javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar"
Environment="JAVA_OPTS=-Dsni.jfr.output=/var/log/kafka-client/sni-capture.jfr"
ExecStart=/usr/bin/java $JAVA_OPTS -jar kafka-client-app.jar
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## Docker Configuration

### 1. Kafka Broker Dockerfile
```dockerfile
# Dockerfile.kafka-sni
FROM confluentinc/cp-kafka:7.6.6

# Copy SNI agent
COPY sni-jfr-agent-1.0.0.jar /opt/kafka/lib/

# Create log directory
RUN mkdir -p /var/log/kafka && chown appuser:appuser /var/log/kafka

# Set environment for SNI capture
ENV KAFKA_OPTS="-javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar -Dsni.jfr.output=/var/log/kafka/sni-capture.jfr"

# Expose JFR files as volume
VOLUME ["/var/log/kafka"]
```

### 2. Docker Compose Configuration
```yaml
# docker-compose.yml
version: '3.8'
services:
  kafka-with-sni:
    build:
      context: .
      dockerfile: Dockerfile.kafka-sni
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: SSL://kafka:9093
      KAFKA_SSL_KEYSTORE_LOCATION: /etc/kafka/secrets/keystore.jks
      KAFKA_SSL_TRUSTSTORE_LOCATION: /etc/kafka/secrets/truststore.jks
      # SNI Agent Configuration
      KAFKA_OPTS: >-
        -javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar
        -Dsni.jfr.output=/var/log/kafka/sni-capture.jfr
        -Dsni.jfr.debug=false
        -Dsni.jfr.maxSize=100000000
    volumes:
      - ./logs:/var/log/kafka
      - ./secrets:/etc/kafka/secrets
    ports:
      - "9093:9093"
```

## Kubernetes Deployment

### 1. ConfigMap for Agent Configuration
```yaml
# kafka-sni-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-sni-config
data:
  sni-agent.env: |
    KAFKA_OPTS=-javaagent:/opt/kafka/lib/sni-jfr-agent-1.0.0.jar -Dsni.jfr.output=/var/log/kafka/sni-capture.jfr -Dsni.jfr.debug=false
```

### 2. Kafka StatefulSet with SNI Agent
```yaml
# kafka-with-sni.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka-with-sni
spec:
  serviceName: kafka
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      initContainers:
      - name: install-sni-agent
        image: busybox
        command: ['sh', '-c', 'cp /agent/sni-jfr-agent-1.0.0.jar /opt/kafka/lib/']
        volumeMounts:
        - name: sni-agent
          mountPath: /agent
        - name: kafka-libs
          mountPath: /opt/kafka/lib
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.6.6
        envFrom:
        - configMapRef:
            name: kafka-sni-config
        volumeMounts:
        - name: kafka-logs
          mountPath: /var/log/kafka
        - name: kafka-libs
          mountPath: /opt/kafka/lib
      volumes:
      - name: sni-agent
        configMap:
          name: sni-agent-jar
      - name: kafka-libs
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: kafka-logs
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Application Properties

### 1. Spring Boot Configuration
```properties
# application.properties
# Kafka client with SNI monitoring
spring.kafka.bootstrap-servers=kafka-cluster.example.com:9093
spring.kafka.security.protocol=SSL
spring.kafka.ssl.trust-store-location=classpath:truststore.jks
spring.kafka.ssl.key-store-location=classpath:keystore.jks

# JVM options for SNI agent
spring.application.jvm-args=-javaagent:lib/sni-jfr-agent-1.0.0.jar,-Dsni.jfr.output=./client-sni.jfr
```

### 2. Java Application JVM Arguments
```bash
# Application startup with SNI monitoring
java -javaagent:lib/sni-jfr-agent-1.0.0.jar \
     -Dsni.jfr.output=./app-sni-$(date +%Y%m%d).jfr \
     -Dsni.jfr.debug=false \
     -Dsni.jfr.maxSize=50000000 \
     -jar your-application.jar
```

## Monitoring Configuration

### 1. Prometheus Metrics (Custom Extension)
```java
// Custom metrics exporter for SNI data
@Component
public class SNIMetricsExporter {

    @EventListener
    public void handleSNICapture(SNIHandshakeEvent event) {
        Counter.builder("kafka.sni.captures")
            .tag("hostname", event.sniHostname)
            .tag("thread", event.threadName)
            .register(Metrics.globalRegistry)
            .increment();
    }
}
```

### 2. Grafana Dashboard Query Templates
```promql
# SNI capture rate
rate(kafka_sni_captures_total[5m])

# Unique SNI hostnames
count by (hostname) (kafka_sni_captures_total)

# SNI captures by thread
sum by (thread) (kafka_sni_captures_total)
```

## Analysis Scripts

### 1. SNI Analysis Script
```bash
#!/bin/bash
# analyze-sni.sh

JFR_FILE="${1:-/var/log/kafka/sni-capture.jfr}"

echo "=== SNI Analysis Report ==="
echo "File: $JFR_FILE"
echo "Generated: $(date)"
echo

# Total captures
total=$(jfr print --events kafka.sni.Handshake "$JFR_FILE" | grep -c "sniHostname")
echo "Total SNI captures: $total"
echo

# Unique hostnames
echo "Unique SNI hostnames:"
jfr print --events kafka.sni.Handshake "$JFR_FILE" | \
    grep "sniHostname" | \
    awk '{print $3}' | \
    sort | uniq -c | sort -nr
echo

# Time range
echo "Time range:"
jfr print --events kafka.sni.Handshake "$JFR_FILE" | \
    grep "startTime" | \
    head -1 | awk '{print "Start:", $3}'
jfr print --events kafka.sni.Handshake "$JFR_FILE" | \
    grep "startTime" | \
    tail -1 | awk '{print "End:  ", $3}'
```

### 2. Real-time Monitoring
```bash
#!/bin/bash
# monitor-sni.sh

LOG_FILE="/var/log/kafka/server.log"

echo "Monitoring SNI captures in real-time..."
echo "Press Ctrl+C to stop"

tail -f "$LOG_FILE" | grep --line-buffered "PROD-SNI.*SNI CAPTURED" | while read line; do
    timestamp=$(echo "$line" | awk '{print $1, $2}')
    hostname=$(echo "$line" | grep -o 'SNI CAPTURED: [^(]*' | sed 's/SNI CAPTURED: //')
    count=$(echo "$line" | grep -o 'count: [0-9]*' | sed 's/count: //')

    printf "[%s] %-30s (Total: %s)\n" "$timestamp" "$hostname" "$count"
done
```

These configuration templates provide comprehensive coverage for deploying the SNI JFR agent across different environments and platforms.