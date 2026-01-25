# Configuration Guide

This guide covers all configuration options for the Logging Stack Docker deployment.

## Table of Contents

1. [Environment Variables](#environment-variables)
2. [Loki Configuration](#loki-configuration)
3. [Grafana Configuration](#grafana-configuration)
4. [Promtail Configuration](#promtail-configuration)
5. [Network Configuration](#network-configuration)
6. [Performance Tuning](#performance-tuning)

## Environment Variables

All configuration is managed through the `.env` file. Copy `.env.example` to `.env` and modify as needed:

```bash
cp .env.example .env
nano .env
```

### Critical Variables (Change in Production)

#### Grafana Admin Password

```env
GRAFANA_ADMIN_PASSWORD=YourSecurePassword123!
```

**Security**: Use a strong password with uppercase, lowercase, numbers, and special characters.

#### Loki Retention

```env
LOKI_RETENTION_DAYS=90
```

**Recommendation**: 
- Development: 7-14 days
- Production: 30-90 days (depending on log volume and storage)
- Compliance: 365+ days (if required)

#### Storage Path

```env
LOKI_DATA_PATH=./volumes/loki-data
GRAFANA_DATA_PATH=./volumes/grafana-data
```

**Important**: Ensure the paths have sufficient disk space.

### Network Configuration

```env
LOGGING_NETWORK_SUBNET=172.25.0.0/16
```

This subnet should not conflict with other Docker networks. Services use this network to communicate.

## Loki Configuration

Loki is configured via `loki/loki-config.yaml`. Key sections:

### Ingestion Rate Limiting

```yaml
limits_config:
  ingestion_rate_mb: 50  # Max 50 MB/s
  ingestion_burst_size_mb: 100  # Burst capacity
```

Adjust based on your log volume:
- Light traffic: 10-20 MB/s
- Medium traffic: 20-50 MB/s
- Heavy traffic: 50-100 MB/s

### Retention Policy

```yaml
limits_config:
  retention_period: 2160h  # 90 days
```

Converting to hours:
- 7 days = 168 hours
- 30 days = 720 hours
- 90 days = 2160 hours
- 365 days = 8760 hours

### Chunk Configuration

```yaml
ingester:
  max_chunk_age: 6h  # How long before flushing
  chunk_idle_period: 3m  # Idle time before flush
```

**Tuning**:
- Longer periods = fewer small chunks = better compression
- Shorter periods = faster query results = more I/O

### Query Limits

```yaml
limits_config:
  max_entries_limit_per_second: 100000
  max_line_size: 256000  # 256 KB per line
```

## Grafana Configuration

Grafana environment variables in `.env`:

### Admin User

```env
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=ChangeMe123!
```

The admin user can create other users and manage permissions.

### Root URL

```env
GRAFANA_ROOT_URL=http://localhost:3000
```

For reverse proxy setups:

```env
GRAFANA_ROOT_URL=https://logs.universidad.edu
```

### Security Settings

Enabled in `docker-compose.yml`:

```yaml
- GF_SECURITY_COOKIE_SECURE=true
- GF_SECURITY_COOKIE_HTTPONLY=true
- GF_SECURITY_COOKIE_SAMESITE=Strict
```

## Promtail Configuration

Promtail is configured via `promtail/promtail-config.yaml`.

### Adding New Scrape Configs

```yaml
scrape_configs:
  - job_name: my_app
    static_configs:
      - targets:
          - localhost
        labels:
          job: my_app
          __path__: /var/log/my_app/*.log
          service: my_app
```

### Processing Pipelines

Add processing stages:

```yaml
pipeline_stages:
  - json:  # Parse JSON logs
      expressions:
        level: level
        message: message
  - timestamp:  # Extract timestamp
      source: timestamp
      format: "2006-01-02T15:04:05.000000000Z07:00"
  - labels:  # Create labels
      level:
```

## Network Configuration

### Docker Network

The logging stack creates an isolated Docker network:

```bash
docker network ls | grep logging
# Shows: logging-stack-docker_logging-net
```

### Connecting External Services

To connect another Docker Compose project:

```yaml
networks:
  app-net:
    driver: bridge
  logging-net:
    external: true
    name: logging-stack-docker_logging-net

services:
  my-service:
    networks:
      - app-net
      - logging-net
```

## Performance Tuning

### For High Log Volume (>500k logs/day)

```env
LOKI_RETENTION_DAYS=30  # Reduce retention
```

```yaml
# In loki-config.yaml
ingester:
  max_chunk_age: 10m  # Faster flushing
  chunk_idle_period: 1m

limits_config:
  ingestion_rate_mb: 100  # Higher limit
  ingestion_burst_size_mb: 200
```

### For Limited Resources (<4GB RAM)

```yaml
limits_config:
  ingestion_rate_mb: 10
  max_cache_freshness_period: 2m  # Shorter cache

querier:
  max_concurrent: 5  # Fewer parallel queries
```

### For Long-term Retention (365+ days)

```env
LOKI_RETENTION_DAYS=365
```

Ensure sufficient disk space: ~400 GB for 1 year at 1 million logs/day.

## Common Scenarios

### Scenario 1: Small Deployment (1 faculty)

```env
LOKI_PORT=3100
GRAFANA_PORT=3000
LOKI_RETENTION_DAYS=30
LOKI_DATA_PATH=./volumes/loki-data  # 10 GB sufficient
```

### Scenario 2: Medium Deployment (University-wide)

```env
LOKI_RETENTION_DAYS=90
LOKI_DATA_PATH=/data/loki  # 50 GB
```

```yaml
# Loki: Increase ingestion rate
ingester:
  rate_limit_max_bytes_per_second: 20000000  # 20 MB/s
```

### Scenario 3: High-Security Deployment

```env
ENABLE_TLS=true
```

```yaml
auth_enabled: true  # Enable authentication in Loki
```

## Verification

After configuration, verify:

```bash
# Check Loki health
curl http://localhost:3100/ready

# Check Grafana
curl http://localhost:3000/api/health

# Verify network connectivity
docker network inspect logging-stack-docker_logging-net
```

## Next Steps

- [Deployment Guide](DEPLOYMENT.md) - Deploy to production
- [Integration Guide](INTEGRATION.md) - Connect external services
- [Security Guide](SECURITY.md) - Harden for production
