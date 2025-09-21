# 📁 Repository Structure

## Overview
This repository contains the complete SNI JFR capture solution organized into logical directories for production use.

## Directory Structure

```
sni-jfr-events/                           # 🏠 Repository root (git repository)
├── 📄 README.md                          # Main repository documentation
├── 📄 .gitignore                         # Git ignore patterns
├── 📄 pom.xml                            # Maven build configuration
├── 📄 REPOSITORY-STRUCTURE.md            # This file
├──
├── 📁 src/main/java/com/kafka/jfr/sni/   # 🔧 Source code
│   ├── ProductionSNIAgent.java           # 🏭 Main production agent
│   ├── SNIHandshakeEvent.java            # 📊 Custom JFR event definition
│   ├── SNIInterceptor.java               # 🔌 SSL instrumentation utilities
│   ├── SNIProgrammaticAgent.java         # 🧪 Development/test agent
│   └── ...                               # Supporting classes
├──
├── 📁 target/                             # 🎯 Build artifacts (generated)
│   └── sni-jfr-agent-1.0.0.jar          # Production-ready agent JAR
├──
├── 📁 docs/                              # 📚 Production documentation
│   ├── PRODUCTION-DEPLOYMENT-GUIDE.md    # 🚀 Complete deployment guide
│   ├── CONFIGURATION-TEMPLATES.md        # ⚙️ Configuration examples
│   ├── MONITORING-TROUBLESHOOTING.md     # 🔍 Operations & troubleshooting
│   └── FINAL-PRODUCTION-SUMMARY.md       # 🏆 Executive summary
├──
├── 📁 scripts/                           # 🔧 Utilities and test scripts
│   ├── test-production-agent.sh          # Production agent testing
│   ├── test-programmatic-jfr.sh          # JFR functionality testing
│   ├── test-sni-direct.sh                # Direct SNI testing
│   ├── run-kafka-with-sni-jfr.sh         # Kafka with JFR setup
│   ├── generate-kafka-certs-with-sans.sh # SSL certificate generation
│   └── ...                               # Additional utility scripts
├──
├── 📁 evidence/                          # ✅ Validation and proof
│   ├── SNI-JFR-SUCCESS-EVIDENCE.md      # Technical validation proof
│   ├── TEST-PASSED-EVIDENCE.md          # Test results evidence
│   ├── JFR-CUSTOM-TEST-SUCCESS.md       # Custom JFR validation
│   ├── VERIFICATION-COMMANDS.md         # Verification procedures
│   └── ...                               # Additional evidence files
├──
└── 📁 examples/                          # 💡 Usage examples and guides
    ├── PRODUCTION-PACKAGE-README.md      # Package overview
    ├── SNI-JFR-DEPLOYMENT.md            # Deployment examples
    └── ...                               # Additional examples
```

## Key Files

### 🎯 Production Artifacts
- **`target/sni-jfr-agent-1.0.0.jar`** - Ready-to-deploy agent JAR (821KB)
- **`src/main/java/.../ProductionSNIAgent.java`** - Production agent implementation

### 📚 Essential Documentation
- **`README.md`** - Main repository introduction and quick start
- **`docs/PRODUCTION-DEPLOYMENT-GUIDE.md`** - Complete deployment instructions
- **`docs/CONFIGURATION-TEMPLATES.md`** - Configuration for all scenarios
- **`docs/MONITORING-TROUBLESHOOTING.md`** - Operations guide

### 🔧 Key Scripts
- **`scripts/test-production-agent.sh`** - Test production agent
- **`scripts/test-programmatic-jfr.sh`** - Validate JFR functionality
- **`scripts/generate-kafka-certs-with-sans.sh`** - SSL certificate setup

### ✅ Validation Evidence
- **`evidence/SNI-JFR-SUCCESS-EVIDENCE.md`** - Technical proof of concept
- **`evidence/TEST-PASSED-EVIDENCE.md`** - Comprehensive test results

## File Categories

### Source Code (7 files)
- Core agent implementations
- JFR event definitions
- SSL instrumentation utilities

### Documentation (10 files)
- Production deployment guides
- Configuration templates
- Troubleshooting procedures
- Validation evidence

### Scripts (11 files)
- Test automation scripts
- Deployment utilities
- Certificate generation tools

### Build Configuration
- Maven POM with all dependencies
- Git ignore patterns
- Build artifact management

## Getting Started

1. **For Users**: Start with `README.md`
2. **For Deployment**: See `docs/PRODUCTION-DEPLOYMENT-GUIDE.md`
3. **For Development**: Check `src/main/java/` and build with `mvn clean package`
4. **For Testing**: Use scripts in `scripts/` directory
5. **For Validation**: Review evidence in `evidence/` directory

## Repository Status

- ✅ **Complete**: All production materials included
- ✅ **Organized**: Logical directory structure
- ✅ **Documented**: Comprehensive documentation
- ✅ **Tested**: Evidence of successful testing
- ✅ **Production Ready**: Ready for immediate deployment

This repository structure provides everything needed for production deployment and ongoing maintenance of the SNI JFR capture solution.