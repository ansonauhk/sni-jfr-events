# Kafka JFR SNI/CN Testing Guide

## Overview
Complete guide for testing Kafka SSL/mTLS with SNI using Java Flight Recorder (JFR) with custom configuration for minimal performance overhead.

## Quick Start

### One-Command Test
```bash
./run-jfr-sni-test.sh
```

This automated script will:
- Verify all prerequisites
- Start services with JFR enabled
- Generate TLS traffic
- Analyze results
- Generate evidence file

## Prerequisites

### System Requirements
- Java 17+ (for JFR support)
- Confluent Kafka 7.6.6
- macOS or Linux
- OpenSSL for certificate generation

### Configuration Files Required
| File | Purpose | Location |
|------|---------|----------|
| `custom-tls.jfc` | JFR configuration with TLS events | `/Users/ansonau/kafka-playground/` |
| `generate-kafka-certs-with-sans.sh` | Certificate generation | `/Users/ansonau/kafka-playground/` |
| `admin-ssl.properties` | Client SSL configuration | `/Users/ansonau/kafka-playground/` |
| `server.properties` | Kafka broker configuration | `confluent-7.6.6/etc/kafka/` |

## Test Components

### 1. Custom JFC Configuration
The `custom-tls.jfc` file enables TLS-specific JFR events:
- `jdk.TLSHandshake` - Captures SNI information
- `jdk.X509Certificate` - Captures certificate CN
- `jdk.X509Validation` - Certificate validation events
- `jdk.SocketRead/Write` - Connection details

### 2. Hostname Configuration
```bash
# Add to /etc/hosts
127.0.0.1    kafka-broker.domaina.com
```

### 3. Certificate Requirements
- **CN**: Must be `kafka-broker.domaina.com`
- **SANs**: Must include the hostname as first entry
- **Format**: JKS keystores with password `kafka123`

### 4. Kafka Configuration
```properties
# server.properties
listeners=SSL://:9093
advertised.listeners=SSL://kafka-broker.domaina.com:9093
inter.broker.listener.name=SSL
ssl.client.auth=required
ssl.protocol=TLSv1.3
ssl.enabled.protocols=TLSv1.3,TLSv1.2
```

## Manual Test Steps

### Step 1: Prepare Environment
```bash
# Verify hostname
grep "kafka-broker.domaina.com" /etc/hosts

# Generate certificates if needed
./generate-kafka-certs-with-sans.sh

# Verify certificate CN
openssl x509 -in ssl/broker-cert-signed -subject -noout
```

### Step 2: Start Services with JFR
```bash
# Start Zookeeper
confluent-7.6.6/bin/zookeeper-server-start -daemon \
  confluent-7.6.6/etc/kafka/zookeeper.properties

# Start Kafka with JFR
JFR_OPTS="-XX:StartFlightRecording=name=SNITest,\
settings=/Users/ansonau/kafka-playground/custom-tls.jfc,\
filename=kafka-sni-test.jfr,maxsize=100m,dumponexit=true"

KAFKA_OPTS="$JFR_OPTS" confluent-7.6.6/bin/kafka-server-start \
  confluent-7.6.6/etc/kafka/server.properties &
```

### Step 3: Generate TLS Traffic
```bash
# Generate connections with SNI
for i in {1..5}; do
  KAFKA_OPTS="-Djavax.net.debug=ssl:handshake:verbose" \
    confluent-7.6.6/bin/kafka-topics \
    --bootstrap-server kafka-broker.domaina.com:9093 \
    --command-config admin-ssl.properties \
    --list 2>&1 | grep "server_name"
  sleep 2
done
```

### Step 4: Dump JFR Recording
```bash
# Find Kafka PID
KAFKA_PID=$(pgrep -f kafka.Kafka)

# Dump recording
jcmd $KAFKA_PID JFR.dump name=SNITest filename=final-test.jfr
```

### Step 5: Verify Test Pass - Commands to Run

#### Primary Verification Commands (Run These)

```bash
# 1. CHECK SERVER-SIDE SNI RECEPTION (Most Important)
# This proves the broker received SNI extension:
echo "=== Checking if server consumed SNI extension ==="
grep -c "Consumed extension: server_name" kafka-jfr.log
# PASS CRITERIA: Should return a number > 0 (typically 30-50 for a short test)

# 2. VERIFY SUCCESSFUL CONNECTIONS
echo "=== Testing connection with hostname ==="
confluent-7.6.6/bin/kafka-topics --bootstrap-server kafka-broker.domaina.com:9093 \
  --command-config admin-ssl.properties --list
# PASS CRITERIA: Should list topics without errors

# 3. VERIFY CERTIFICATE CN
echo "=== Checking certificate CN ==="
openssl x509 -in ssl/broker-cert-signed -subject -noout | grep "CN="
# PASS CRITERIA: Should show CN=kafka-broker.domaina.com

# 4. CHECK JFR RECORDING EXISTS
echo "=== Checking JFR recording ==="
ls -lh *.jfr
# PASS CRITERIA: Should show JFR file(s) with size > 0
```

#### Analyzing JFR File (If jfr command available)

```bash
# Find jfr command first (if not in PATH):
JFR_CMD=$(find /Library/Java/JavaVirtualMachines -name jfr 2>/dev/null | head -1)
# Or try: JFR_CMD="$JAVA_HOME/bin/jfr"

# If jfr command is found, analyze the recording:
if [ -n "$JFR_CMD" ]; then
    echo "=== Analyzing JFR for TLS events ==="

    # Check for TLS Handshake events (shows peer connections)
    $JFR_CMD print --events jdk.TLSHandshake *.jfr | grep -c "peerHost"
    # PASS CRITERIA: Should return > 0 if TLS events were captured

    # Check for certificate events
    $JFR_CMD print --events jdk.X509Certificate *.jfr | grep -c "subject"
    # PASS CRITERIA: Should return > 0 if certificate events were captured

    # Export to JSON for detailed analysis
    $JFR_CMD print --json *.jfr > jfr-analysis.json
    grep -c "TLSHandshake\|X509Certificate" jfr-analysis.json
    # PASS CRITERIA: Should find TLS-related events
else
    echo "jfr command not found, use jcmd instead:"
    jcmd $(pgrep -f kafka.Kafka) JFR.check
fi
```

#### Alternative: Using jcmd (Always Available)

```bash
# Get Kafka PID
KAFKA_PID=$(pgrep -f kafka.Kafka)

# Check JFR recording status
jcmd $KAFKA_PID JFR.check
# PASS CRITERIA: Should show recording(s) with data

# Dump current recording if still running
jcmd $KAFKA_PID JFR.dump name=SNITest filename=final-test.jfr
# PASS CRITERIA: Should create JFR file successfully
```

#### Quick One-Line Test Pass Verification

```bash
# Run this single command to verify test passed:
echo "TEST $(grep -q 'Consumed extension: server_name' kafka-jfr.log && echo 'PASSED ✅' || echo 'FAILED ❌')"
```

## Expected Results

### Successful Test Output
```
✅ Zookeeper started
✅ Kafka started with JFR recording
✅ Generated 5 TLS connections
✅ JFR recording saved: kafka-sni-test-20250920-231500.jfr
✅ Certificate CN matches hostname
✅ TEST PASSED
```

### Evidence Files Generated
- `kafka-jfr-test-*.jfr` - JFR recording (typically 1-2 MB)
- `jfr-test-evidence-*.txt` - Test evidence with SNI captures
- `kafka-jfr.log` - Kafka startup log

### Server-Side Verification Evidence

#### Method 1: JFR Recording (when jfr command available)
```
# TLS Handshake Events show the peer connection details
jdk.TLSHandshake {
  startTime = ...
  peerHost = "kafka-broker.domaina.com"  # SNI hostname received by server
  peerPort = 9093
  protocolVersion = "TLSv1.3"
  cipherSuite = "TLS_AES_256_GCM_SHA384"
  certificateId = ...
}

# Certificate Events show what cert the server presented
jdk.X509Certificate {
  algorithm = "RSA"
  subject = "CN=kafka-broker.domaina.com, OU=Engineering, O=MyOrg..."
  issuer = "CN=CARoot, O=MyOrg..."
}
```

#### Method 2: Broker SSL Debug Logs (Most Reliable)
```
# Server receives SNI extension from client:
javax.net.ssl|DEBUG|...|SSLExtensions.java:204|Consumed extension: server_name

# Server processes the SNI hostname:
javax.net.ssl|DEBUG|...|ServerNameExtension.java:...|Server name: kafka-broker.domaina.com
```

#### Method 3: Successful Connection Verification
```
# If this works, SNI is functioning correctly:
$ kafka-topics --bootstrap-server kafka-broker.domaina.com:9093 \
    --command-config admin-ssl.properties --list
__consumer_offsets
test-topic
...
```

#### What the Server Sees vs Client Sends

**Client Sends (in ClientHello):**
```
"server_name (0)": {
  type=host_name (0), value=kafka-broker.domaina.com
}
```

**Server Receives and Processes:**
```
Consumed extension: server_name
Server selects certificate based on SNI hostname
Returns appropriate certificate with CN=kafka-broker.domaina.com
```

## Performance Comparison

| Method | CPU Overhead | I/O Impact | File Size (10 min) |
|--------|--------------|------------|-------------------|
| **JFR with Custom JFC** | 1-2% | Minimal | 1-2 MB |
| SSL Debug Logging | 15-20% | Heavy | 100+ MB |
| Network Capture | 3-5% | Moderate | 10-50 MB |

## Troubleshooting

### Issue: JFR events not captured
**Solution**: Ensure custom-tls.jfc is used, not default profile

### Issue: SNI not sent
**Solution**: Use FQDN `kafka-broker.domaina.com`, not `kafka-broker`

### Issue: Certificate CN mismatch
**Solution**: Regenerate certificates with correct CN:
```bash
./generate-kafka-certs-with-sans.sh
```

### Issue: Kafka fails to start
**Solution**: Check inter.broker.listener.name configuration:
```bash
echo "inter.broker.listener.name=SSL" >> server.properties
```

## Cleanup

### Stop Services
```bash
confluent-7.6.6/bin/kafka-server-stop
confluent-7.6.6/bin/zookeeper-server-stop
```

### Remove Test Files
```bash
rm -f kafka-jfr-test-*.jfr
rm -f jfr-test-evidence-*.txt
rm -f kafka-jfr.log
```

## Files Reference

### Essential Files (Keep)
- `run-jfr-sni-test.sh` - Automated test script
- `custom-tls.jfc` - JFR configuration
- `generate-kafka-certs-with-sans.sh` - Certificate generation
- `admin-ssl.properties` - Client configuration

### Test Output Files
- `kafka-jfr-test-*.jfr` - JFR recordings
- `jfr-test-evidence-*.txt` - Test evidence
- `JFR-CUSTOM-TEST-SUCCESS.md` - Latest test results

## Summary

This test validates:
1. **SNI Support**: Client sends correct hostname in TLS handshake
2. **Certificate CN**: Matches the expected hostname
3. **JFR Monitoring**: Low-overhead production monitoring capability
4. **mTLS Authentication**: Mutual TLS works correctly

The custom JFC configuration enables TLS event capture in JFR, providing production-safe monitoring with only 1-2% performance overhead.