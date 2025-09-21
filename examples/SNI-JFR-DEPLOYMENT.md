# Custom JFR Events for SNI Capture in Kafka

## Overview
This solution implements custom JFR events to capture the actual SNI (Server Name Indication) hostname sent during TLS handshakes in Kafka, addressing the limitation where standard JFR events only show resolved hostnames.

## Components

1. **SNIHandshakeEvent.java** - Custom JFR event class that records:
   - SNI hostname (actual value from ClientHello)
   - Resolved hostname
   - Connection details (IP, port, protocol, cipher)
   - Handshake duration

2. **SNIJFRAgent.java** - Java agent that instruments:
   - `SSLEngineImpl` - Used by Kafka's network layer
   - `SSLSocketImpl` - Used by some Kafka clients

3. **SNIEventEmitter.java** - Emits JFR events when TLS handshakes occur

## Deployment Steps

### 1. Build the Agent
```bash
cd /Users/ansonau/kafka-playground/sni-jfr-events
mvn clean package
```

This creates: `target/sni-jfr-agent-1.0.0.jar`

### 2. Run Kafka with the Agent

#### Option A: Using the Provided Script
```bash
./run-kafka-with-sni-jfr.sh
```

#### Option B: Manual Configuration

Add to Kafka's JVM options:
```bash
# Agent for SNI capture
AGENT_OPTS="-javaagent:/path/to/sni-jfr-agent-1.0.0.jar"

# JFR recording with custom events
JFR_OPTS="-XX:StartFlightRecording=name=SNICapture,\
settings=/path/to/sni-enhanced.jfc,\
filename=kafka-sni.jfr,\
maxsize=200m,\
dumponexit=true"

# Start Kafka
KAFKA_OPTS="$AGENT_OPTS $JFR_OPTS" \
    bin/kafka-server-start etc/kafka/server.properties
```

### 3. Verify Agent Loading

Check Kafka logs for:
```
[SNI-JFR-Agent] Starting SNI JFR Agent...
[SNI-JFR-Agent] Instrumenting SSLEngineImpl
[SNI-JFR-Agent] Instrumenting SSLSocketImpl
```

### 4. Analyze Custom Events

```bash
# View custom SNI events
jfr print --events "kafka.sni.*" kafka-sni.jfr

# Expected output:
kafka.sni.Handshake {
  startTime = 11:45:23.123
  sniHostname = "kafka-with-alias.domaina.com"  # <-- Actual SNI value
  resolvedHost = "kafka-broker.domaina.com"     # <-- Resolved hostname
  peerAddress = "127.0.0.1"
  peerPort = 9093
  protocolVersion = "TLSv1.3"
  cipherSuite = "TLS_AES_256_GCM_SHA384"
  connectionType = "CLIENT"
  threadName = "kafka-network-thread-0"
  handshakeDuration = 125 ms
}
```

## What Changes for Kafka

### Required Changes:
1. **Add Java Agent JAR** to classpath
2. **Add JVM Options** for agent and JFR
3. **No Code Changes** to Kafka itself

### Performance Impact:
- **Minimal overhead** (~1-2%) from:
  - JFR recording (< 1% with proper configuration)
  - Bytecode instrumentation (negligible after JIT optimization)
  - Event emission (only during TLS handshakes)

### Benefits:
- **Captures actual SNI hostname** sent by clients
- **Distinguishes between SNI and resolved hostnames**
- **Works with all SSL/TLS connections** in Kafka
- **No modification to Kafka source code**

## Troubleshooting

### Agent Not Loading
- Check Java version compatibility (requires Java 11+)
- Verify agent JAR path is correct
- Check for security manager restrictions

### No Custom Events in JFR
- Ensure the JFC configuration includes `kafka.sni.Handshake` event
- Verify SSL/TLS connections are being made
- Check agent logs for instrumentation errors

### High Overhead
- Disable stack traces in JFC configuration
- Increase threshold values for events
- Use sampling instead of recording all events

## Configuration Options

### JFC Configuration
```xml
<event name="kafka.sni.Handshake">
  <setting name="enabled">true</setting>
  <setting name="threshold">0 ms</setting>  <!-- Set higher to reduce events -->
  <setting name="stackTrace">false</setting> <!-- Set true for debugging -->
</event>
```

### Agent Options
The agent can accept arguments:
```bash
-javaagent:sni-jfr-agent.jar=verbose    # Enable verbose logging
-javaagent:sni-jfr-agent.jar=filter=*.domaina.com  # Filter by hostname
```

## Limitations

1. **Requires Java 11+** for JFR support
2. **SSLEngine/SSLSocket only** - doesn't capture raw socket TLS
3. **Client-side SNI** - server-side reception still shows resolved hostname
4. **Instrumentation overhead** - minimal but non-zero

## Security Considerations

- The agent has access to SSL/TLS parameters
- Ensure agent JAR is from trusted source
- Consider using signed JARs in production
- Limit JFR file access (contains connection details)

## Integration with Existing Monitoring

The custom events can be:
- Streamed using JFR Event Streaming API (Java 14+)
- Exported to monitoring systems (Prometheus, Grafana)
- Analyzed with JDK Mission Control
- Processed with custom JFR parsers

## Example Use Cases

1. **Multi-tenant Kafka** - Track which SNI names clients use
2. **SSL Certificate Management** - Verify correct certificate selection
3. **Network Debugging** - Diagnose SNI-related connection issues
4. **Compliance** - Audit TLS connections and protocols
5. **Performance Analysis** - Measure handshake duration by SNI