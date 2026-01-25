# Quick Start Guide

**Get logging stack running in 5 minutes** (development) or **30 minutes** (production).

## Development (Local Testing) - 5 Minutes

### Step 1: Clone Repository

```bash
git clone https://github.com/StudentUCO/logging-stack-docker.git
cd logging-stack-docker
```

### Step 2: Create Configuration

```bash
cp .env.example .env
# Default values work for development
```

### Step 3: Start Services

```bash
docker-compose up -d
```

### Step 4: Access Services

- **Grafana**: http://localhost:3000 (admin / ChangeMe123!)
- **Loki API**: http://localhost:3100

### Step 5: Test with Sample Logs

```bash
# Create test logs
docker run --rm --log-driver=loki --log-opt loki-url=http://loki:3100/loki/api/v1/push --log-opt labels=service=test alpine echo "Test log entry"

# View in Grafana
# Explore > Loki > Query: {service="test"}
```

**Done!** Your logging stack is running.

---

## Production (University Server) - 30 Minutes

### Step 1: Server Setup (5 minutes)

```bash
# SSH to server
ssh ubuntu@your-server.edu

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Clone repo
cd /opt
git clone https://github.com/StudentUCO/logging-stack-docker.git logging-stack
cd logging-stack
```

### Step 2: Configuration (5 minutes)

```bash
# Copy template
cp .env.example .env

# Edit for production (nano or your editor)
nano .env

# Critical changes:
# - GRAFANA_ADMIN_PASSWORD=Your$trongP@ssw0rd!
# - GRAFANA_ROOT_URL=https://logs.universidad.edu
# - LOKI_RETENTION_DAYS=90
```

### Step 3: Create Data Directories (2 minutes)

```bash
# Create volume directories
mkdir -p volumes/{loki-data,grafana-data} certs backups
chmod 700 volumes certs backups
```

### Step 4: Deploy (3 minutes)

```bash
# Start services
docker-compose up -d

# Verify
docker-compose ps
curl http://localhost:3100/ready  # Should return: ready
```

### Step 5: SSL Certificate (10 minutes)

```bash
# Get certificate
sudo certbot certonly --standalone -d logs.universidad.edu

# Copy to certs
sudo cp /etc/letsencrypt/live/logs.universidad.edu/fullchain.pem certs/server.crt
sudo cp /etc/letsencrypt/live/logs.universidad.edu/privkey.pem certs/server.key
sudo chown 472:472 certs/*
```

### Step 6: Nginx Reverse Proxy (5 minutes)

```bash
# Install Nginx
sudo apt-get install -y nginx

# Create config
sudo tee /etc/nginx/sites-available/logs-proxy << 'EOF'
server {
  listen 80;
  server_name logs.universidad.edu;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl http2;
  server_name logs.universidad.edu;
  
  ssl_certificate /etc/letsencrypt/live/logs.universidad.edu/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/logs.universidad.edu/privkey.pem;
  
  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
EOF

# Enable
sudo ln -s /etc/nginx/sites-available/logs-proxy /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Done! ✅

Your logging stack is now:
- ✅ Running and secured
- ✅ Accessible via HTTPS
- ✅ Ready to receive logs

---

## Next Steps

### 1. Integrate External Services

See [Integration Guide](INTEGRATION.md) for connecting:
- Nginx Reverse Proxy
- Keycloak
- Nextcloud
- Custom APIs

**Quick example for one service**:

```bash
# Edit your service's docker-compose.yml
cd ../keycloak-docker-deploy  # Navigate to service
nano docker-compose.yml

# Add to Keycloak service:
#     logging:
#       driver: loki
#       options:
#         loki-url: "http://loki:3100/loki/api/v1/push"
#         labels: "service=keycloak,env=production"
#     networks:
#       - keycloak-net
#       - logging-net

# At bottom, add:
# logging-net:
#   external: true
#   name: logging-stack-docker_logging-net

# Restart
docker-compose down && docker-compose up -d
```

### 2. Create Queries and Dashboards

In Grafana (http://logs.universidad.edu):

1. Go to **Explore**
2. Select **Loki** datasource
3. Try queries:
   ```logql
   # All Keycloak logs
   {service="keycloak"}
   
   # Errors only
   {service="keycloak"} | json | level="ERROR"
   
   # Last hour
   {service="keycloak"} | since(1h)
   ```

### 3. Set Up Alerts (Optional)

Create alerts for critical events:

```bash
# Example: Alert on high error rate
# {service="keycloak"} | json | level="ERROR" | count() > 10
```

---

## Common Issues

### Services won't start

```bash
# Check logs
docker-compose logs loki
docker-compose logs grafana

# Ensure ports are free
lsof -i :3000
lsof -i :3100

# If in use, kill the process or use different ports
```

### Can't access Grafana

```bash
# Verify running
docker-compose ps

# Verify port mapping
docker port grafana

# Check logs
curl http://localhost:3000  # Should respond
```

### No logs appearing

```bash
# Check if services are connected to logging network
docker network inspect logging-stack-docker_logging-net

# Should see both loki and your service containers
```

---

## Cheat Sheet

```bash
# Start/stop
docker-compose up -d
docker-compose down

# View logs
docker-compose logs -f loki
docker-compose logs -f grafana

# Check status
docker-compose ps

# Test Loki
curl http://localhost:3100/ready

# Test Grafana
curl http://localhost:3000/api/health

# Access Grafana
# Open browser to http://localhost:3000
# Login: admin / (password from .env)

# Reload configuration
docker-compose restart

# Remove all data
docker-compose down -v
```

---

## Documentation Links

- **Detailed Configuration**: [CONFIGURATION.md](CONFIGURATION.md)
- **Integration with Services**: [INTEGRATION.md](INTEGRATION.md)
- **Security Hardening**: [SECURITY.md](SECURITY.md)
- **Production Deployment**: [DEPLOYMENT.md](DEPLOYMENT.md)
- **Troubleshooting**: [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- **Log Queries**: [LOG_QUERIES.md](LOG_QUERIES.md)

---

**Need Help?**
- 📖 Read the [main README](../README.md)
- 🐛 [Report issues](https://github.com/StudentUCO/logging-stack-docker/issues)
- 💬 [Start discussions](https://github.com/StudentUCO/logging-stack-docker/discussions)

**Happy Logging!** 🚀
