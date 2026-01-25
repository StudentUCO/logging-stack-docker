# Logging Stack Docker - Grafana + Loki + Promtail

[![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![Docker Compose](https://img.shields.io/badge/docker%20compose-v3.8+-blue)](https://docs.docker.com/compose/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

> **Centralized logging infrastructure for Docker Compose environments**
> 
> Production-ready solution for university infrastructure combining Grafana for visualization, Loki for log aggregation, and Promtail for log collection.

## 📋 Features

- ✅ **Lightweight & Efficient**: Loki uses label-based indexing (50% less resources than ELK)
- ✅ **Zero-Trust Architecture**: TLS encryption, RBAC, network segmentation
- ✅ **Production-Ready**: Health checks, automatic restarts, volume persistence
- ✅ **Multi-Service Support**: Easily connect existing Docker Compose stacks
- ✅ **Professional Dashboards**: Pre-configured Grafana with example queries
- ✅ **Easy Integration**: Simple Docker log driver configuration
- ✅ **Scalable**: Horizontal scaling ready with Promtail agents
- ✅ **University-Focused**: Cost-effective, low overhead (2-4 GB RAM)

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    LOGGING STACK                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Loki (Log Storage & Query Engine)                          │
│  ├─ Label-based indexing (efficient storage)               │
│  ├─ Chunk storage (compressed)                             │
│  └─ HTTP API (3100)                                        │
│                                                              │
│  Grafana (Visualization & Dashboards)                       │
│  ├─ Professional UI (3000)                                 │
│  ├─ Pre-configured Loki datasource                         │
│  └─ Example dashboards & queries                           │
│                                                              │
│  Promtail (Log Collector - Optional)                        │
│  └─ File-based log collection for non-Docker sources       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
         ▲
         │ (Logs via Docker log driver or HTTP API)
         │
    ┌────┴─────────────────────────────────────┐
    │  External Docker Compose Services        │
    ├─────────────────────────────────────────┤
    │ • Nginx Reverse Proxy                   │
    │ • Keycloak                              │
    │ • Nextcloud                             │
    │ • PostgreSQL                            │
    │ • Redis                                 │
    │ • Custom APIs & Applications            │
    └─────────────────────────────────────────┘
```

## 🚀 Quick Start

### Prerequisites

- Docker Engine 20.10+
- Docker Compose 1.29+
- 3-4 GB available RAM
- 30 GB available disk space

### 1. Clone the Repository

```bash
git clone https://github.com/StudentUCO/logging-stack-docker.git
cd logging-stack-docker
```

### 2. Configure Environment Variables

```bash
cp .env.example .env
# Edit .env with your settings (passwords, retention policies, etc.)
nano .env
```

### 3. Deploy the Stack

```bash
# Deploy
docker-compose up -d

# Wait for services to be healthy
watch docker-compose ps

# View logs
docker-compose logs -f loki
```

### 4. Access Services

- **Grafana**: http://localhost:3000 (admin / password from .env)
- **Loki API**: http://localhost:3100
- **Health Check**: http://localhost:3100/ready

## 📚 Documentation

### For Configuration Details

- [Configuration Guide](docs/CONFIGURATION.md) - Environment variables, retention policies
- [Security Setup](docs/SECURITY.md) - TLS, RBAC, authentication
- [Integration Guide](docs/INTEGRATION.md) - Connect external Docker Compose services

### For Operations

- [Deployment Guide](docs/DEPLOYMENT.md) - Production deployment steps
- [Monitoring & Alerts](docs/MONITORING.md) - Health checks and alerting setup
- [Troubleshooting](docs/TROUBLESHOOTING.md) - Common issues and solutions

### For Development

- [Contributing](CONTRIBUTING.md) - How to contribute improvements
- [Architecture Deep Dive](docs/ARCHITECTURE.md) - Technical details

## 🔗 Integration with External Services

### Quick Integration Example

To integrate an external Docker Compose service:

```yaml
version: '3.8'

services:
  your-service:
    image: your-image:latest
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-batch-size: "400"
        labels: "service=your-service,env=production"
    networks:
      - your-network
      - logging-net  # ← Connect to logging network

networks:
  your-network:
    driver: bridge
  logging-net:
    external: true
    name: logging-stack-docker_logging-net  # Match this name
```

See [Integration Guide](docs/INTEGRATION.md) for detailed instructions.

## 🔐 Security Considerations

### Production Deployment

- ✅ All passwords changed from defaults
- ✅ TLS/HTTPS configured for external access
- ✅ Network isolation using Docker networks
- ✅ Health checks and monitoring enabled
- ✅ Automatic log rotation and retention
- ✅ Access logs stored and audited

See [Security Guide](docs/SECURITY.md) for detailed security configuration.

## 📊 Resource Requirements

### Minimum

- **RAM**: 2 GB
- **CPU**: 2 cores
- **Disk**: 20 GB (90 days at 100k logs/day)

### Recommended (University Scale)

- **RAM**: 4 GB
- **CPU**: 4 cores
- **Disk**: 50 GB (90 days at 300k logs/day)

### Storage Efficiency

- Raw logs: 100 GB
- Compressed by Loki: ~10 GB
- **Compression ratio**: ~10:1

## 📈 Scalability Path

### Phase 1: Single Node (Current)
- Loki + Grafana + Promtail
- Up to 500k logs/day
- Single server deployment

### Phase 2: Distributed Collectors (Future)
- Add Promtail agents on other servers
- Collect logs from multiple locations
- Centralized Loki instance

### Phase 3: Loki Clustering (Advanced)
- Loki instance replication
- High availability
- Multi-region support

### Phase 4: Extended Observability (Long-term)
- Add Prometheus for metrics
- Integrate with Jaeger for tracing
- Full observability stack

## 🛠️ Command Reference

```bash
# Start the stack
docker-compose up -d

# View status
docker-compose ps

# View logs
docker-compose logs -f loki
docker-compose logs -f grafana

# Stop the stack
docker-compose down

# Remove all data (WARNING: deletes logs)
docker-compose down -v

# Scale services (if applicable)
docker-compose up -d --scale promtail=3

# Health check
curl http://localhost:3100/ready
```

## 📝 Log Query Examples

### Query Logs from Specific Service

```logql
# All Keycloak logs
{service="keycloak"}

# Keycloak errors only
{service="keycloak"} | json | level="ERROR"

# Nginx 4xx and 5xx responses
{service="nginx"} | json | status >= 400

# Backend API requests with duration > 1s
{service="backend-api"} | json | duration > 1000
```

### Query by Time Range

```logql
# Last 1 hour
{service="database"} | since(1h)

# Last 30 minutes
{service="auth"}
```

### Aggregate and Count

```logql
# Count logs by service
sum(count_over_time({job=~".+"}[5m])) by (service)

# Error rate
count(count_over_time({level="ERROR"}[5m]))
```

See [Log Query Guide](docs/LOG_QUERIES.md) for more examples.

## 🐛 Troubleshooting

### Services Not Starting

```bash
# Check logs
docker-compose logs loki

# Verify ports are not in use
sudo netstat -tlnp | grep 3000
sudo netstat -tlnp | grep 3100
```

### No Logs Appearing in Grafana

```bash
# Check if Loki is receiving logs
curl http://localhost:3100/loki/api/v1/label/service/values

# Check Docker network connectivity
docker network inspect logging-stack-docker_logging-net
```

### High Memory Usage

- Check retention policy in `loki/loki-config.yaml`
- Reduce `max_cache_freshness_period`
- Increase disk space allocation

See [Troubleshooting Guide](docs/TROUBLESHOOTING.md) for more solutions.

## 📞 Support & Community

- 📖 [Documentation](docs/)
- 🐛 [Report Issues](https://github.com/StudentUCO/logging-stack-docker/issues)
- 💬 [Discussions](https://github.com/StudentUCO/logging-stack-docker/discussions)

## 📄 License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [Grafana](https://grafana.com/) - Visualization platform
- [Loki](https://grafana.com/oss/loki/) - Log aggregation system
- [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/) - Log collection agent
- University infrastructure team

## 📌 Version History

- **v1.0.0** (2026-01-25) - Initial production-ready release
  - Loki 2.9.8
  - Grafana 10.2.1
  - Pre-configured for multi-service integration
  - Security hardened
  - Complete documentation

---

**Last Updated**: January 25, 2026
**Status**: ✅ Production Ready
