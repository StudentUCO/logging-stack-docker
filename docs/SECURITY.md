# Security Guide

This guide covers security hardening for production deployment of the logging stack in a university environment.

## Overview

The logging stack handles sensitive institutional data. This guide ensures:

- ✅ Authentication and authorization
- ✅ Encrypted communications
- ✅ Network isolation
- ✅ Access control
- ✅ Audit logging
- ✅ Backup security

## Important Updates (2026)

> ⚠️ **CRITICAL CHANGE**: Ports for Loki (`3100`) and Promtail (`9080`) are now bound to `127.0.0.1` (localhost) by default in `docker-compose.yml`. You MUST use a reverse proxy (like Nginx) or an SSH tunnel to access them from outside the server.

## Table of Contents

1. [Pre-Deployment Security](#pre-deployment-security)
2. [Authentication](#authentication)
3. [Encryption](#encryption)
4. [Network Security](#network-security)
5. [Access Control](#access-control)
6. [Monitoring & Auditing](#monitoring--auditing)
7. [Compliance](#compliance)

## Pre-Deployment Security

### 1. Change Default Credentials

**Critical**: Always change default passwords before deployment.

```bash
# Edit .env
nano .env
```

Set strong passwords:

```env
GRAFANA_ADMIN_PASSWORD=YourVery$trongP@ssw0rd2024!
```

**Password Requirements**:
- Minimum 16 characters
- Uppercase letters (A-Z)
- Lowercase letters (a-z)
- Numbers (0-9)
- Special characters (!@#$%^&*)
- No dictionary words
- No personal information

### 2. Update All Images

Use specific versions to ensure security updates:

```yaml
services:
  loki:
    image: grafana/loki:3.3.2  # ✓ Specific version
    # NOT: image: grafana/loki:latest  # ✗ Latest can change
  
  grafana:
    image: grafana/grafana:11.4.0  # ✓ Specific version
```

### 3. Verify Image Integrity

```bash
# Pull and verify images
docker pull grafana/loki:3.3.2

# Check SHA256
docker inspect grafana/loki:3.3.2 | grep Digest

# Compare with official repository
# https://hub.docker.com/r/grafana/loki/tags
```

## Authentication

### Grafana Authentication

#### 1. Admin User Management

Create non-default admin account:

```bash
# Access Grafana
curl -X POST http://localhost:3000/api/admin/users \
  -H "Content-Type: application/json" \
  -u admin:YourVery$trongP@ssw0rd2024! \
  -d '{
    "name": "It Administrator",
    "email": "admin@universidad.edu",
    "login": "it-admin",
    "password": "NewAdminP@ss2024!",
    "isAdmin": true
  }'
```

#### 2. LDAP Integration (Recommended)

Integrate with university's Active Directory:

```yaml
# Edit docker-compose.yml
environment:
  - GF_AUTH_LDAP_ENABLED=true
  - GF_AUTH_LDAP_CONFIG_FILE=/etc/grafana/ldap.toml

volumes:
  - ./grafana/ldap.toml:/etc/grafana/ldap.toml:ro
```

**Create `grafana/ldap.toml`**:

```toml
[[servers]]
host = "ldap.universidad.edu"
port = 389
use_ssl = true
start_tls = true
ssl_skip_verify = false

bind_dn = "cn=grafana,ou=users,dc=universidad,dc=edu"
bind_password = """${LDAP_PASSWORD}"""

search_filter = "(uid=%s)"
search_base_dns = [
  "ou=users,dc=universidad,dc=edu"
]

[servers.attributes]
name = "cn"
surname = "sn"
username = "uid"
member_of = "memberOf"
email = "mail"
```

#### 3. OAuth2/SAML (Advanced)

For enterprise SSO:

```yaml
environment:
  - GF_AUTH_GENERIC_OAUTH_ENABLED=true
  - GF_AUTH_GENERIC_OAUTH_CLIENT_ID=grafana
  - GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET=${OAUTH_SECRET}
  - GF_AUTH_GENERIC_OAUTH_SCOPES=openid profile email
  - GF_AUTH_GENERIC_OAUTH_AUTH_URL=https://sso.universidad.edu/auth
  - GF_AUTH_GENERIC_OAUTH_TOKEN_URL=https://sso.universidad.edu/token
  - GF_AUTH_GENERIC_OAUTH_API_URL=https://sso.universidad.edu/userinfo
```

### Loki Authentication

Loki doesn't have built-in auth. Secure with reverse proxy:

```yaml
# Docker Compose: Ports are bound to 127.0.0.1 by default
ports:
  - "127.0.0.1:3100:3100"  # ✓ Secure default

# Route through Nginx with auth
```

**Nginx reverse proxy example**:

```nginx
upstream loki {
  server 127.0.0.1:3100; # Connect to localhost
}

server {
  listen 443 ssl http2;
  server_name logs-api.universidad.edu;
  
  auth_basic "Loki API";
  auth_basic_user_file /etc/nginx/htpasswd/loki;
  
  location / {
    proxy_pass http://loki;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

## Encryption

### TLS/HTTPS for Grafana

#### 1. Obtain Certificate

```bash
# Using Let's Encrypt
sudo certbot certonly --standalone \
  -d logs.universidad.edu \
  -m it-admin@universidad.edu

# Copy to certs directory
sudo cp /etc/letsencrypt/live/logs.universidad.edu/fullchain.pem ./certs/server.crt
sudo cp /etc/letsencrypt/live/logs.universidad.edu/privkey.pem ./certs/server.key
sudo chown -R 472:472 ./certs/  # Grafana user
```

#### 2. Configure Grafana for HTTPS

```yaml
services:
  grafana:
    environment:
      - GF_SERVER_PROTOCOL=https
      - GF_SERVER_CERT_FILE=/etc/grafana/certs/server.crt
      - GF_SERVER_CERT_KEY=/etc/grafana/certs/server.key
    
    volumes:
      - ./certs/server.crt:/etc/grafana/certs/server.crt:ro
      - ./certs/server.key:/etc/grafana/certs/server.key:ro
```

#### 3. HTTP to HTTPS Redirect

```nginx
server {
  listen 80;
  server_name logs.universidad.edu;
  return 301 https://$server_name$request_uri;
}
```

### Encrypted Docker Socket

For Promtail security:

```yaml
# Only expose Docker socket to Promtail via bind mount
promtail:
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro  # Read-only
```

## Network Security

### 1. Network Isolation

```yaml
networks:
  logging-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.25.0.0/16
    # Disable inter-container communication outside of network
    driver_opts:
      "com.docker.network.driver.mtu": "1500"
```

### 2. Firewall Rules

```bash
# Allow only necessary ports
sudo ufw allow ssh                   # Port 22
sudo ufw allow http                  # Port 80
sudo ufw allow https                 # Port 443

# Deny direct access to internal ports (redundant if bound to 127.0.0.1 but good practice)
sudo ufw deny 3000
sudo ufw deny 3100
sudo ufw deny 9080

# Deny all others
sudo ufw default deny incoming
```

### 3. IP Whitelisting

```nginx
# Nginx: Allow specific IPs only
server {
  listen 443 ssl http2;
  server_name logs.universidad.edu;
  
  # Allow university network only
  allow 200.10.0.0/16;     # Universidad IP range
  allow 203.20.0.0/16;     # VPN range
  deny all;
  
  # ... rest of config
}
```

## Access Control

### 1. Grafana RBAC

```bash
# Create roles via API
curl -X POST http://localhost:3000/api/roles \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "name": "Log Viewer",
    "description": "Can view logs but not modify",
    "permissions": [
      {"action": "datasources:query", "scope": "datasources:*"}
    ]
  }'
```

### 2. Loki Label-Based Access

Use labels to restrict log visibility:

```logql
# Query only faculty's logs
{faculty="engineering", env="production"}

# Admin can query all
{env="production"}
```

### 3. Service Account Management

For automated integrations:

```bash
# Create service account for monitoring
curl -X POST http://localhost:3000/api/serviceaccounts \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "name": "Monitoring Service",
    "role": "Viewer"
  }'
```

## Monitoring & Auditing

### 1. Enable Audit Logging

```yaml
environment:
  - GF_AUDIT_LOGGING_ENABLED=true
  - GF_AUDIT_LOG_LEVEL=info
```

### 2. Log Access Attempts

```bash
# Monitor Grafana access logs
docker logs grafana | grep "authentication"

# Monitor Loki requests
docker logs loki | grep "loki_api"
```

### 3. Alerts for Security Events

Set up alerts in Grafana:

```logql
# Alert: Failed login attempts
{service="grafana"} | json | status="unauthorized" | count() > 5

# Alert: Large data export
{service="grafana"} | json | action="export" | size > 1000000
```

### 4. Regular Backups

```bash
#!/bin/bash
# backup-logs.sh
DATE=$(date +%Y%m%d_%H%M%S)

# Backup Grafana configuration
docker exec grafana tar czf /tmp/grafana-backup-$DATE.tar.gz /var/lib/grafana
docker cp grafana:/tmp/grafana-backup-$DATE.tar.gz ./backups/

# Backup Loki data
sudo tar czf ./backups/loki-backup-$DATE.tar.gz ./volumes/loki-data/

# Encrypt backups
echo 'Backup encryption password (from secure storage):'
read -s BACKUP_PASSWORD
tar cz ./backups/loki-backup-$DATE.tar.gz | openssl enc -aes-256-cbc -salt -k $BACKUP_PASSWORD > ./backups/loki-backup-$DATE.tar.gz.enc
rm ./backups/loki-backup-$DATE.tar.gz

# Upload to secure location
rsync -avz ./backups/ backup-server:/backups/logging-stack/
```

## Compliance

### FERPA (Student Records)

If logging student data:

- ✅ Encrypt at rest and in transit
- ✅ Access control (staff only)
- ✅ Audit all access
- ✅ Retention policies (delete after period)
- ✅ Regular backups to secure location

### GDPR (EU Data)

If university has EU users:

- ✅ Data Processing Agreement (DPA)
- ✅ Consent documentation
- ✅ Right to erasure (implement log deletion)
- ✅ Data residency (keep in EU)
- ✅ Breach notification (72 hours)

### Institutional Policy

Create policy documents:

```markdown
# Logging Stack Security Policy

## Access Control
- Only IT staff and designated administrators
- LDAP authentication required
- Multi-factor authentication for admin accounts

## Data Retention
- Production logs: 90 days
- Security audit logs: 1 year
- User access logs: 6 months

## Encryption
- TLS 1.3 for all external connections
- AES-256 for backups
- Certificate renewal: quarterly

## Monitoring
- Daily security review
- Monthly access audit
- Quarterly penetration testing
```

## Deployment Checklist

Before going to production:

- [ ] All default passwords changed
- [ ] HTTPS/TLS configured
- [ ] LDAP integration tested
- [ ] Firewall rules applied
- [ ] Backup script tested
- [ ] Audit logging enabled
- [ ] Security documentation created
- [ ] Staff trained on access control
- [ ] Incident response plan documented
- [ ] Regular penetration testing scheduled

## Next Steps

- [Deployment Guide](DEPLOYMENT.md) - Deploy securely
- [Monitoring & Alerts](MONITORING.md) - Set up monitoring
- [Troubleshooting](TROUBLESHOOTING.md) - Security troubleshooting
