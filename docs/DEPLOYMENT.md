# Deployment Guide

This guide walks through deploying the logging stack to production on a university server.

## Deployment Phases

1. **Phase 1 (Week 1)**: Initial Setup & Testing
2. **Phase 2 (Week 2-3)**: Service Integration
3. **Phase 3 (Week 4)**: Production Validation

## Phase 1: Initial Setup & Testing

### Step 1: Server Preparation

```bash
# SSH into production server
ssh ubuntu@logs.universidad.edu

# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sh

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify Docker
docker --version
docker run hello-world
```

### Step 2: Clone Repository

```bash
# Create application directory
sudo mkdir -p /opt/logging-stack
sudo chown $USER:$USER /opt/logging-stack

# Clone repository
cd /opt/logging-stack
git clone https://github.com/StudentUCO/logging-stack-docker.git .

# Verify structure
ls -la
# Should show: docker-compose.yml, .env.example, loki/, grafana/, etc.
```

### Step 3: Create Data Directories

```bash
# Create volumes with correct permissions
mkdir -p volumes/loki-data volumes/grafana-data
mkdir -p certs backups

# Set permissions (secure)
chmod 700 volumes/
chmod 700 certs/
chmod 700 backups/

# Verify
ls -la volumes/
```

### Step 4: Environment Configuration

```bash
# Copy template
cp .env.example .env

# Edit for production
nano .env

# Update these values:
# - GRAFANA_ADMIN_PASSWORD=SecureP@ssw0rd2024!
# - LOKI_RETENTION_DAYS=90
# - GRAFANA_ROOT_URL=https://logs.universidad.edu
# - LOGGING_NETWORK_SUBNET=172.25.0.0/16
```

### Step 5: Deploy Stack

```bash
# Start services
docker-compose up -d

# Wait for startup
echo "Waiting for services to be healthy..."
sleep 30

# Check status
docker-compose ps

# Should show:
# loki       Up  (healthy)
# grafana    Up  (healthy)
```

### Step 6: Initial Testing

```bash
# Test Loki API
curl http://localhost:3100/ready
# Should respond: "ready"

# Test Grafana
curl -s http://localhost:3000/api/health | jq '.status'
# Should respond: "ok"

# Check logs
docker-compose logs loki | tail -20
docker-compose logs grafana | tail -20
```

### Step 7: Grafana Initial Setup

```bash
# Access Grafana
# http://localhost:3000
# Login: admin / (password from .env)

# Steps in UI:
# 1. Change admin password
# 2. Go to Configuration → Data Sources
# 3. Verify Loki datasource is configured
# 4. Go to Explore
# 5. Select Loki
# 6. Run test query: {job="docker"}
```

## Phase 2: Service Integration

### Step 1: Update External Services

For each existing service (Keycloak, Nextcloud, etc.):

```bash
# Navigate to service directory
cd /opt/keycloak-docker-deploy

# Get logging network name
echo "logging-stack-docker_logging-net"
```

### Step 2: Modify docker-compose.yml

See [Integration Guide](INTEGRATION.md) for detailed steps.

```bash
# Edit service docker-compose.yml
nano docker-compose.yml

# Add logging config to each service
# Add networks connection
# Add external network declaration
```

### Step 3: Restart Services with Logging

```bash
# Apply changes
docker-compose down
docker-compose up -d

# Verify logging
docker-compose ps
```

### Step 4: Verify Logs in Grafana

```bash
# Access Grafana
# http://localhost:3000 → Explore

# Test queries:
# {service="keycloak"}
# {service="nextcloud"}
# {service="nginx"}

# Should show logs from services
```

## Phase 3: Production Validation

### Step 1: Security Hardening

Follow [Security Guide](SECURITY.md):

```bash
# Get SSL certificate
sudo certbot certonly --standalone -d logs.universidad.edu

# Copy to certs directory
sudo cp /etc/letsencrypt/live/logs.universidad.edu/fullchain.pem certs/server.crt
sudo cp /etc/letsencrypt/live/logs.universidad.edu/privkey.pem certs/server.key
sudo chown 472:472 certs/*  # Grafana user
```

### Step 2: Reverse Proxy Setup (Nginx)

```bash
# Install Nginx
sudo apt-get install -y nginx

# Create config
sudo nano /etc/nginx/sites-available/logs
```

**Nginx Configuration**:

```nginx
upstream grafana {
  server localhost:3000;
}

upstream loki {
  server localhost:3100;
}

# Redirect HTTP to HTTPS
server {
  listen 80;
  listen [::]:80;
  server_name logs.universidad.edu;
  return 301 https://$server_name$request_uri;
}

# HTTPS server
server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name logs.universidad.edu;
  
  # SSL certificates
  ssl_certificate /etc/letsencrypt/live/logs.universidad.edu/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/logs.universidad.edu/privkey.pem;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  
  # Security headers
  add_header Strict-Transport-Security "max-age=31536000" always;
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  
  # Grafana
  location / {
    proxy_pass http://grafana;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
  
  # Loki API (with authentication)
  location /loki/api/ {
    auth_basic "Loki API";
    auth_basic_user_file /etc/nginx/htpasswd/loki;
    
    proxy_pass http://loki;
    proxy_set_header Host $host;
  }
}
```

```bash
# Create htpasswd for Loki API
sudo htpasswd -c /etc/nginx/htpasswd/loki api-user

# Enable configuration
sudo ln -s /etc/nginx/sites-available/logs /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Step 3: Firewall Configuration

```bash
# Enable firewall
sudo ufw enable

# Allow SSH
sudo ufw allow 22/tcp

# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow from university network only (optional)
sudo ufw allow from 200.10.0.0/16 to any port 3000
sudo ufw allow from 200.10.0.0/16 to any port 3100

# Status
sudo ufw status
```

### Step 4: Health Monitoring

Create health check script:

```bash
# health-check.sh
#!/bin/bash

echo "Checking logging stack health..."

# Check Loki
if curl -f http://localhost:3100/ready > /dev/null 2>&1; then
  echo "✅ Loki: OK"
else
  echo "❌ Loki: FAILED"
  exit 1
fi

# Check Grafana
if curl -f http://localhost:3000/api/health > /dev/null 2>&1; then
  echo "✅ Grafana: OK"
else
  echo "❌ Grafana: FAILED"
  exit 1
fi

# Check disk space
USAGE=$(du -sh volumes/loki-data | awk '{print $1}')
echo "✅ Loki storage: $USAGE"

echo "All checks passed!"
```

```bash
# Make executable
chmod +x health-check.sh

# Add to crontab for monitoring
crontab -e

# Add line:
# 0 * * * * /opt/logging-stack/health-check.sh >> /var/log/logging-health.log 2>&1
```

### Step 5: Backup Configuration

```bash
# Create backup script
cat > backup-logging.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/opt/logging-stack/backups"

# Backup Loki data
echo "Backing up Loki data..."
tar czf $BACKUP_DIR/loki-backup-$DATE.tar.gz \
  -C /opt/logging-stack volumes/loki-data

# Backup Grafana data
echo "Backing up Grafana data..."
tar czf $BACKUP_DIR/grafana-backup-$DATE.tar.gz \
  -C /opt/logging-stack volumes/grafana-data

# Keep only last 7 backups
find $BACKUP_DIR -name "*-backup-*.tar.gz" -mtime +7 -delete

echo "Backup completed: $DATE"
EOF

chmod +x backup-logging.sh

# Schedule daily backups
crontab -e
# Add: 2 2 * * * /opt/logging-stack/backup-logging.sh
```

### Step 6: Create Systemd Service (Optional)

For easier management:

```bash
sudo tee /etc/systemd/system/logging-stack.service << 'EOF'
[Unit]
Description=Logging Stack Docker Compose Service
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/logging-stack

ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable logging-stack
sudo systemctl start logging-stack
```

## Post-Deployment Validation

### Checklist

- [ ] All services running (`docker-compose ps`)
- [ ] Loki responds to health check (`curl http://localhost:3100/ready`)
- [ ] Grafana accessible and configured
- [ ] External services sending logs
- [ ] Queries return expected results
- [ ] HTTPS working
- [ ] Firewall configured
- [ ] Backups scheduled
- [ ] Monitoring active
- [ ] Team trained on access and queries

### Performance Baseline

Document baseline metrics:

```bash
# Memory usage
docker stats --no-stream loki grafana

# Disk usage
du -sh /opt/logging-stack/volumes/

# Network I/O
docker stats --format "table {{.Container}}\t{{.NetIO}}"
```

## Rollback Procedure

If issues arise:

```bash
# Stop services
docker-compose down

# Restore from backup
tar xzf backups/loki-backup-YYYYMMDD_HHMMSS.tar.gz
tar xzf backups/grafana-backup-YYYYMMDD_HHMMSS.tar.gz

# Restart
docker-compose up -d

# Verify
curl http://localhost:3100/ready
```

## Support & Troubleshooting

- [Troubleshooting Guide](TROUBLESHOOTING.md) - Common issues
- [Security Guide](SECURITY.md) - Security configuration
- [Monitoring Guide](MONITORING.md) - Set up alerting

## Next Steps

1. Train IT staff on logging queries
2. Create dashboards for specific services
3. Set up alerting for critical events
4. Schedule monthly security reviews
5. Plan scalability for future growth
