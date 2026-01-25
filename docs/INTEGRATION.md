# Integration Guide

This guide explains how to connect external Docker Compose services (Keycloak, Nextcloud, Nginx, etc.) to the centralized Loki logging stack.

## Overview

The logging stack creates a shared Docker network. Your services can connect to this network and send logs to Loki through Docker's logging driver.

```
┌─────────────────────────────────┐
│   Logging Stack (isolated)      │
│  ├─ Loki (3100)                │
│  └─ Grafana (3000)             │
│  Network: logging-net          │
└─────────────────────────────────┘
          ▲
          │
     logging-net
          │
┌─────────────────────────────────┐
│   External Services             │
│  ├─ Nginx                       │
│  ├─ Keycloak                    │
│  ├─ Nextcloud                   │
│  └─ Custom Apps                 │
└─────────────────────────────────┘
```

## Prerequisites

1. Logging stack deployed and running
2. Knowledge of your external service's docker-compose.yml
3. Docker Compose v1.29 or later

## Step 1: Identify Logging Network Name

First, find the exact name of the logging network:

```bash
# Start logging stack if not running
cd logging-stack-docker
docker-compose up -d

# List Docker networks
docker network ls | grep logging

# Output example:
# 6a1b2c3d  logging-stack-docker_logging-net  bridge  local

# Full network name: logging-stack-docker_logging-net
```

## Step 2: Configure External Service

For each service in your external docker-compose.yml, add three things:

### 2A. Add Logging Driver

Add to each service container:

```yaml
services:
  nginx:
    image: nginx:latest
    # ... existing config ...
    
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-batch-size: "400"
        loki-batch-timeout: "1s"
        labels: "service=nginx,env=production"
        tag: "{{.Name}}"
```

**Key options**:
- `loki-url`: Must point to Loki via the shared network
- `labels`: Metadata for filtering logs in Grafana
- `loki-batch-size`: Logs batched before sending (1-1000)
- `tag`: Container name format in logs

### 2B. Add Network Connection

Add the logging network to each service:

```yaml
services:
  nginx:
    image: nginx:latest
    # ... existing config ...
    
    networks:
      - app-net        # Your existing network
      - logging-net    # ← ADD THIS LINE
```

### 2C. Declare External Network

At the end of docker-compose.yml, declare the external network:

```yaml
networks:
  app-net:
    driver: bridge
  
  logging-net:           # ← ADD THIS
    external: true
    name: logging-stack-docker_logging-net  # Match exact network name
```

## Complete Example: Keycloak

```yaml
version: '3.8'

services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    container_name: keycloak
    
    ports:
      - "8080:8080"
    
    environment:
      - KEYCLOAK_ADMIN=admin
      - KEYCLOAK_ADMIN_PASSWORD=changeme
      - KC_DB=postgres
      - KC_DB_URL=jdbc:postgresql://postgres:5432/keycloak
      - KC_DB_USERNAME=keycloak
      - KC_DB_PASSWORD=keycloak
    
    # ← ADD LOGGING CONFIG
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-batch-size: "400"
        labels: "service=keycloak,env=production"
        tag: "{{.Name}}"
    
    # ← ADD BOTH NETWORKS
    networks:
      - keycloak-net
      - logging-net
    
    depends_on:
      - postgres
  
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    
    environment:
      - POSTGRES_PASSWORD=postgres
    
    # ← ADD LOGGING CONFIG
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-batch-size: "200"
        labels: "service=postgres,env=production"
        tag: "{{.Name}}"
    
    # ← ADD BOTH NETWORKS
    networks:
      - keycloak-net
      - logging-net
    
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:

networks:
  keycloak-net:
    driver: bridge
  
  # ← ADD EXTERNAL NETWORK
  logging-net:
    external: true
    name: logging-stack-docker_logging-net
```

## Step 3: Deploy and Test

```bash
# Navigate to external service directory
cd /path/to/keycloak-docker-deploy

# Start/update services
docker-compose up -d

# Verify services are running
docker-compose ps

# Check if services can reach Loki
docker-compose exec keycloak ping loki
# Expected: 64 bytes from loki (172.25.x.x): ...
```

## Step 4: Verify Logs in Grafana

1. Open Grafana: http://localhost:3000
2. Go to: Explore (left panel)
3. Select Loki as datasource
4. Enter query: `{service="keycloak"}`
5. Should see logs from Keycloak

## Logging Configuration Reference

### Labels Examples

```yaml
# Basic
labels: "service=keycloak"

# With environment
labels: "service=keycloak,env=production,team=security"

# Dynamic labels
labels: "service={{.ImageName}},env=prod"
```

### Batch Size Tuning

- **Small services**: 100-200 (faster delivery)
- **Normal services**: 300-500 (balanced)
- **High-volume services**: 1000+ (better compression)

### Tag Formats

```yaml
# Container name only
tag: "{{.Name}}"

# With image name
tag: "{{.ImageName}}-{{.Name}}"

# With container ID
tag: "{{.ID}}"
```

## Common Issues

### Issue 1: "Network not found"

```
Error: Network logging-stack-docker_logging-net not found
```

**Solution**:
```bash
# Check exact network name
docker network ls | grep logging

# Update docker-compose.yml with correct name
```

### Issue 2: "Can't connect to Loki"

```
Error: Failed to connect to http://loki:3100
```

**Solution**:
```bash
# Verify Loki is running
cd logging-stack-docker
docker-compose ps loki

# Check network connectivity
docker network inspect logging-stack-docker_logging-net

# Verify container is on network
docker inspect your-service | grep -A 5 "Networks"
```

### Issue 3: "No logs appearing"

```bash
# Check Loki is receiving data
curl http://localhost:3100/loki/api/v1/label/service/values

# Should return: ["keycloak", "postgres", ...]
```

### Issue 4: "Host not found: loki"

Containers must be on the same network. Verify:

```bash
docker network inspect logging-stack-docker_logging-net | grep Containers

# Should list both loki and your service
```

## Advanced: Multiple External Projects

You can integrate multiple Docker Compose projects:

```
Project 1: Keycloak
Project 2: Nextcloud  
Project 3: Custom APIs
       ↓
  All → logging-net → Loki
```

Each project just needs:
1. Logging driver on each service
2. Connection to logging-net
3. External network declaration

## Log Query Examples

After integration, query logs in Grafana:

```logql
# All Keycloak logs
{service="keycloak"}

# Keycloak errors only
{service="keycloak"} | json | level="ERROR"

# Keycloak + Postgres logs
{service=~"keycloak|postgres"}

# All production services
{env="production"}

# All logs in last hour
{service="keycloak"} | since(1h)
```

## Next Steps

- [Log Query Guide](LOG_QUERIES.md) - Advanced query examples
- [Monitoring & Alerts](MONITORING.md) - Set up alerting
- [Deployment Guide](DEPLOYMENT.md) - Production deployment

## Support

If integration fails:

1. Check exact network name: `docker network ls`
2. Verify containers are running: `docker-compose ps`
3. Test connectivity: `docker-compose exec service ping loki`
4. Review logs: `docker-compose logs service`
5. Check Loki health: `curl http://localhost:3100/ready`
