# Kafka SNI/JFR Test - Verification Commands

## Quick Pass/Fail Check

### One-Line Test Verification
```bash
# This single command tells you if the test passed:
grep -q 'Consumed extension: server_name' kafka-jfr.log && echo '✅ TEST PASSED' || echo '❌ TEST FAILED'
```

## Detailed Verification Commands

### 1. Server-Side SNI Reception (PRIMARY)
```bash
# Count how many times server consumed SNI extension
grep -c "Consumed extension: server_name" kafka-jfr.log

# Expected: Number > 0 (typically 30-50 for a short test)
# If 0: TEST FAILED - Server didn't receive SNI
```

### 2. Verify JFR Recording
```bash
# List JFR files
ls -lh *.jfr

# Expected: One or more .jfr files with size > 0
# Example: sni-test-success.jfr (1.0M)
```

### 3. Analyze JFR File Content

#### Option A: If jfr command is available
```bash
# Find jfr command
JFR_CMD=$(find /Library/Java/JavaVirtualMachines -name jfr 2>/dev/null | head -1)

# Or if JAVA_HOME is set:
JFR_CMD="$JAVA_HOME/bin/jfr"

# Analyze TLS events
$JFR_CMD print --events jdk.TLSHandshake sni-test-success.jfr | grep -c "peerHost"
# Expected: > 0 if TLS events were captured

$JFR_CMD print --events jdk.X509Certificate sni-test-success.jfr | grep -c "subject"
# Expected: > 0 if certificate events were captured

# View detailed TLS handshake info
$JFR_CMD print --events jdk.TLSHandshake sni-test-success.jfr | head -20

# Export to JSON for analysis
$JFR_CMD print --json sni-test-success.jfr > jfr-analysis.json
grep "TLSHandshake\|X509Certificate" jfr-analysis.json
```

#### Option B: Using jcmd (Always available)
```bash
# Get Kafka PID
KAFKA_PID=$(pgrep -f kafka.Kafka)

# Check JFR status
jcmd $KAFKA_PID JFR.check

# Dump recording if still running
jcmd $KAFKA_PID JFR.dump name=SNITest filename=sni-test-success.jfr
```

### 4. Verify Certificate Configuration
```bash
# Check certificate CN
openssl x509 -in ssl/broker-cert-signed -subject -noout | grep "CN="
# Expected: CN=kafka-broker.domaina.com

# Check SANs
openssl x509 -in ssl/broker-cert-signed -text -noout | grep -A1 "Subject Alternative Name"
# Expected: First SAN should be kafka-broker.domaina.com
```

### 5. Test Live Connection
```bash
# Test connection with the hostname
confluent-7.6.6/bin/kafka-topics \
    --bootstrap-server kafka-broker.domaina.com:9093 \
    --command-config admin-ssl.properties --list

# Expected: Lists topics without errors
```

## Automated Verification Script

Create this script as `verify-test.sh`:

```bash
#!/bin/bash

echo "=== SNI/JFR Test Verification ==="

# Check server-side SNI
SNI_COUNT=$(grep -c "Consumed extension: server_name" kafka-jfr.log 2>/dev/null || echo "0")
echo "Server consumed SNI: $SNI_COUNT times"

# Check JFR files
JFR_COUNT=$(ls *.jfr 2>/dev/null | wc -l)
echo "JFR files found: $JFR_COUNT"

# Check certificate CN
CN=$(openssl x509 -in ssl/broker-cert-signed -subject -noout 2>/dev/null | grep -o "CN=[^,]*" | cut -d= -f2)
echo "Certificate CN: $CN"

# Test connection
kafka-topics --bootstrap-server kafka-broker.domaina.com:9093 \
    --command-config admin-ssl.properties --list &>/dev/null
CONNECTION=$?
echo "Connection test: $([ $CONNECTION -eq 0 ] && echo "SUCCESS" || echo "FAILED")"

# Final verdict
if [ $SNI_COUNT -gt 0 ] && [ "$CN" = "kafka-broker.domaina.com" ] && [ $CONNECTION -eq 0 ]; then
    echo -e "\n✅ ALL TESTS PASSED"
else
    echo -e "\n❌ SOME TESTS FAILED"
    [ $SNI_COUNT -eq 0 ] && echo "  - Server didn't receive SNI"
    [ "$CN" != "kafka-broker.domaina.com" ] && echo "  - Certificate CN mismatch"
    [ $CONNECTION -ne 0 ] && echo "  - Connection failed"
fi
```

## Understanding the Results

### PASS Criteria
- ✅ Server logs show `Consumed extension: server_name` (at least once)
- ✅ JFR file exists and has size > 0
- ✅ Certificate CN = kafka-broker.domaina.com
- ✅ Successful SSL connection to broker

### FAIL Indicators
- ❌ No "Consumed extension: server_name" in broker logs
- ❌ JFR file missing or empty
- ❌ Certificate CN doesn't match hostname
- ❌ Connection errors

## Common Issues

| Problem | Check Command | Solution |
|---------|---------------|----------|
| No SNI in logs | `grep -c "Consumed extension: server_name" kafka-jfr.log` | Enable SSL debug in broker |
| JFR not found | `ls *.jfr` | Check JFR options in startup |
| jfr command missing | `which jfr` | Use jcmd instead |
| Connection fails | `kafka-topics --list` | Check certificates and config |

## Quick Troubleshooting

```bash
# Check if Kafka is running
pgrep -f kafka.Kafka

# Check last 50 lines of broker log for errors
tail -50 kafka-jfr.log | grep -E "ERROR\|FATAL"

# Verify hostname mapping
grep "kafka-broker.domaina.com" /etc/hosts

# Check if SSL port is listening
netstat -an | grep 9093
```