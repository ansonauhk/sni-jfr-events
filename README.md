# SNI JFR Events

[![Maven Build](https://github.com/ansonauhk/sni-jfr-events/actions/workflows/maven-build.yml/badge.svg)](https://github.com/ansonauhk/sni-jfr-events/actions/workflows/maven-build.yml)

A Java agent that captures Server Name Indication (SNI) values during SSL/TLS handshakes in Apache Kafka brokers and records them using Java Flight Recorder (JFR) custom events.

## Problem Statement

Standard JFR TLS events only show the resolved hostname (IP address or DNS-resolved name) but not the actual SNI value sent by clients. This makes it difficult to track which client alias or hostname was actually used in the SSL handshake.

## Solution

This Java agent uses bytecode instrumentation to intercept SSL/TLS handshake operations and emit custom JFR events containing the actual SNI hostname values.

## Features

- 🔍 **Real SNI Capture**: Captures actual SNI values sent by clients
- 📊 **JFR Integration**: Custom JFR events for seamless monitoring
- 🚀 **Zero Code Changes**: Works as a Java agent, no Kafka modifications needed
- 📝 **Production Ready**: SLF4J logging, error handling, and performance optimized
- 🔧 **Configurable**: Debug mode, output file configuration, and more

## Quick Start

### Prerequisites

- Java 17+
- Apache Kafka 2.8+
- Maven 3.6+ (for building)

### Building

```bash
git clone https://github.com/ansonauhk/sni-jfr-events.git
cd sni-jfr-events
mvn clean package
```

### Usage

1. **Add the agent to Kafka broker**:
```bash
export KAFKA_OPTS="-javaagent:/path/to/sni-jfr-agent-1.0.0.jar"
bin/kafka-server-start.sh config/server.properties
```

2. **Configure the agent** (optional):
```bash
export KAFKA_OPTS="-javaagent:/path/to/sni-jfr-agent-1.0.0.jar \
                    -Dsni.jfr.output=kafka-sni.jfr \
                    -Dsni.jfr.debug=true"
```

3. **View captured SNI events**:
```bash
jfr print --events kafka.sni.Handshake kafka-sni.jfr
```

## Example Output

```
kafka.sni.Handshake {
  startTime = 10:45:23.456
  sniHostname = "kafka-client-alias.example.com"
  resolvedHost = "192.168.1.100"
  connectionType = "CLIENT_SNI_CAPTURE"
  peerPort = 9093
  protocolVersion = "TLSv1.3"
  cipherSuite = "TLS_AES_256_GCM_SHA384"
}
```

## Configuration Options

| System Property | Default | Description |
|-----------------|---------|-------------|
| `sni.jfr.output` | `kafka-sni-capture.jfr` | Output JFR file path |
| `sni.jfr.debug` | `false` | Enable debug logging |
| `sni.jfr.maxSize` | `100000000` | Max recording size (bytes) |

## Documentation

- [Developer Guide](docs/DEVELOPER-GUIDE.md) - How the project works
- [Production Deployment](docs/PRODUCTION-DEPLOYMENT.md) - Production setup guide
- [Monitoring & Troubleshooting](docs/MONITORING-TROUBLESHOOTING.md) - Debug and monitor
- [Logging Best Practices](docs/LOGGING-BEST-PRACTICES.md) - Logging configuration

## Project Structure

```
sni-jfr-events/
├── src/main/java/          # Java source code
│   └── com/kafka/jfr/sni/  # Agent and event classes
├── docs/                    # Documentation
├── examples/                # Configuration examples
├── scripts/                 # Test and utility scripts
└── pom.xml                 # Maven configuration
```

## How It Works

1. **Agent loads at JVM startup** via `-javaagent` flag
2. **Instruments SSL classes** using bytecode manipulation
3. **Intercepts SNI setting** in `SSLParameters.setServerNames()`
4. **Emits custom JFR events** with captured SNI values
5. **Records to JFR file** for analysis

## Building from Source

```bash
# Clone repository
git clone https://github.com/ansonauhk/sni-jfr-events.git
cd sni-jfr-events

# Build with Maven
mvn clean package

# Run tests
mvn test

# The agent JAR will be at:
# target/sni-jfr-agent-1.0.0.jar
```

## Testing

```bash
# Test with Kafka broker
./scripts/test-with-kafka.sh

# Verify SNI capture
./scripts/verify-sni-capture.sh
```

## Contributing

Contributions are welcome! Please see the [Developer Guide](docs/DEVELOPER-GUIDE.md) for details on:
- Architecture overview
- Adding new instrumentation points
- Testing guidelines
- Code style

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues, questions, or suggestions:
- Open an issue on [GitHub](https://github.com/ansonauhk/sni-jfr-events/issues)
- Check the [Troubleshooting Guide](docs/MONITORING-TROUBLESHOOTING.md)

## Acknowledgments

- Built for Apache Kafka SSL/TLS monitoring
- Uses Java Flight Recorder for event recording
- Leverages Javassist for bytecode instrumentation