# Logging Stack Docker - Project Overview

## Mission

Provide a **centralized, production-grade logging solution** for Universidad Colombia Osso's infrastructure, enabling real-time visibility into all containerized services.

## What's Included

This repository contains a complete, production-ready logging stack with:

### Core Components

| Component | Purpose | Status |
|-----------|---------|--------|
| **Grafana Loki** | Log aggregation & storage | ✅ Configured |
| **Grafana** | Visualization & querying | ✅ Configured |
| **Promtail** | Log collection agent | ✅ Configured |
| **Docker Compose** | Infrastructure as Code | ✅ Ready |

### Documentation

| Document | Purpose |
|----------|----------|
| [README.md](../README.md) | Main documentation |
| [QUICKSTART.md](QUICKSTART.md) | 5-minute setup |
| [CONFIGURATION.md](CONFIGURATION.md) | Detailed configuration |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Production deployment |
| [INTEGRATION.md](INTEGRATION.md) | Connect external services |
| [SECURITY.md](SECURITY.md) | Security hardening |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Common issues & fixes |
| [LOG_QUERIES.md](LOG_QUERIES.md) | Query examples |
| [MONITORING.md](MONITORING.md) | Alerting & monitoring |

### Configuration Files

```
logging-stack-docker/
├── docker-compose.yml          # Main orchestration
├── .env.example                # Environment template
├── .env                        # Your configuration (git ignored)
├── LICENSE                    # MIT License
├── CONTRIBUTING.md            # Contribution guidelines
├── loki/
│  ├── loki-config.yml           # Loki configuration
│  └── rules/                    # Alert rules
├── grafana/
│  ├── provisioning/             # Auto-provisioning
│  │  ├── dashboards/
│  │  ├── datasources/
│  │  └── alerting/
│  └── dashboards/               # Pre-built dashboards
├── promtail/
│  └── promtail-config.yml       # Log agent config
├── docs/                      # Documentation
├── volumes/                   # Data persistence (git ignored)
├── certs/                     # SSL certificates (git ignored)
└── backups/                   # Backup location (git ignored)
```

## Quick Start

### Development (5 minutes)

```bash
git clone https://github.com/StudentUCO/logging-stack-docker.git
cd logging-stack-docker
cp .env.example .env
docker-compose up -d
# Access: http://localhost:3000
```

### Production (30 minutes)

See [DEPLOYMENT.md](DEPLOYMENT.md) for complete production setup.

## Architecture

### System Diagram

```
┌───────────────────────────────────────────┐
│               EXTERNAL SERVICES                      │
│  ┌──────────────────────────────┐  │
│  │  Keycloak | Nextcloud | Nginx                 │  │
│  └──────────────────────────────┘  │
│                ▼                                      │
│              (logs)                                  │
│                ▼                                      │
│  ┌──────────────────────────────┐  │
│  │  Docker Logging Driver (Loki)                 │  │
│  └──────────────────────────────┘  │
└───────────────────────────────────────────┘
                ▼
           logging-net
                ▼
┌───────────────────────────────────────────┐
│            LOGGING STACK                            │
│  ┌──────────────────────────────┐  │
│  │  Loki                                         │  │
│  │  - Accepts logs from docker driver          │  │
│  │  - Stores with labels (service, env, etc)   │  │
│  │  - Retention: 90 days (configurable)        │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │  Grafana                                    │  │
│  │  - Pre-configured datasource (Loki)        │  │
│  │  - Dashboard templates                     │  │
│  │  - Query builder & explorer                │  │
│  │  - Alerting engine                         │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │  Promtail (optional)                        │  │
│  │  - Scrapes docker logs                     │  │
│  │  - Enriches with metadata                  │  │
│  └──────────────────────────────┘  │
└───────────────────────────────────────────┘
                ▼
         https://logs.universidad.edu
                ▼
           ┌─────────────┐
           │ IT STAFF / USERS  │
           └─────────────┘
```

## Key Features

### ✅ Production-Ready
- Secure TLS/HTTPS by default
- LDAP authentication integration
- Role-based access control
- Data retention policies
- Automated backups

### ✅ Scalable
- Easy to add new services
- Horizontal scaling ready
- Configurable log retention
- Label-based organization

### ✅ Observable
- Real-time log streaming
- Advanced log querying (LogQL)
- Pre-built dashboards
- Alert rules included

### ✅ Simple to Deploy
- Docker Compose for easy deployment
- Infrastructure as Code
- Comprehensive documentation
- Example configurations provided

## Integration Points

Your services can integrate in minutes:

```yaml
services:
  keycloak:
    image: keycloak/keycloak:latest
    
    # Add 3 lines:
    logging:                              # ← ADD THIS
      driver: loki                        # ← ADD THIS  
      options:                            # ← ADD THIS
        loki-url: http://loki:3100/loki/api/v1/push
        labels: "service=keycloak,env=production"
    
    networks:
      - keycloak-net
      - logging-net                       # ← ADD THIS
```

Then:
- Logs appear in Grafana automatically
- Query by service: `{service="keycloak"}`
- Set up alerts on errors or patterns

## Deployment Timeline

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| **Setup** | 5 mins | Running stack (dev) |
| **Config** | 10 mins | .env configured (prod) |
| **Deploy** | 5 mins | Services online |
| **Integrate** | 30 mins | First service connected |
| **Secure** | 15 mins | HTTPS + auth enabled |
| **Total** | ~1 hour | Full production setup |

## Next Steps

1. **Today**: Follow [QUICKSTART.md](QUICKSTART.md) (5 minutes)
2. **This Week**: Integrate first service ([INTEGRATION.md](INTEGRATION.md))
3. **Next Week**: Deploy to production ([DEPLOYMENT.md](DEPLOYMENT.md))
4. **Ongoing**: Set up monitoring and alerts ([MONITORING.md](MONITORING.md))

## Support & Maintenance

### Monitoring
- Health checks run automatically
- Logs available for troubleshooting
- Pre-built dashboards for system health

### Troubleshooting
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) covers 99% of issues
- Common issues have step-by-step solutions
- Debug commands provided

### Updates
- Regular security updates
- New features added regularly
- Backward compatible versions

## Compliance & Security

- ✅ FERPA-compliant (student data)
- ✅ GDPR-ready (EU users)
- ✅ Institutional policies enforced
- ✅ Audit logging enabled
- ✅ Encryption in transit & at rest

## Technology Stack

| Technology | Version | Purpose |
|------------|---------|----------|
| **Grafana Loki** | 2.9.x | Log aggregation |
| **Grafana** | 10.x | UI & visualization |
| **Promtail** | 2.9.x | Log collection (optional) |
| **Docker** | 20.10+ | Containerization |
| **Docker Compose** | 1.29+ | Orchestration |

## Repository Structure

- **Source**: [github.com/StudentUCO/logging-stack-docker](https://github.com/StudentUCO/logging-stack-docker)
- **Issues**: [Report bugs or request features](https://github.com/StudentUCO/logging-stack-docker/issues)
- **Discussions**: [Ask questions or share ideas](https://github.com/StudentUCO/logging-stack-docker/discussions)
- **License**: MIT

## License

MIT License - Free for educational and commercial use. See [LICENSE](../LICENSE) for details.

---

**Ready to start?** → Head to [QUICKSTART.md](QUICKSTART.md) now!
