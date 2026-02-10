# Production Deployment Guide

## Overview
This guide covers deploying a full-stack Docker application with HTTPS, reverse proxy, health checks, and automated deployment.

## Architecture
```
Internet → Nginx (443) → Frontend (80) → Backend (5000)
                     ↓
                  SSL/TLS
```

## Prerequisites
- Docker 20.10+
- Docker Compose 2.0+
- Domain name (for production SSL)
- Minimum 2GB RAM, 2 CPU cores

## Quick Start

### 1. Initial Setup
```bash
# Make scripts executable
chmod +x deploy.sh setup-ssl.sh docker-healthcheck.sh

# Run deployment script
./deploy.sh
```

### 2. Configuration
Edit `.env` file with your settings:
```bash
ENVIRONMENT=production
SECRET_KEY=your-secret-key-here
DOMAIN=yourdomain.com
SSL_EMAIL=admin@yourdomain.com
```

### 3. Deploy
```bash
# Production deployment
docker-compose -f docker-compose.prod.yml up -d --build

# Development deployment
docker-compose up -d --build
```

## SSL/HTTPS Setup

### Self-Signed (Development)
```bash
./setup-ssl.sh
```

### Let's Encrypt (Production)
```bash
# Install certbot
sudo apt-get install certbot

# Generate certificate
sudo certbot certonly --standalone -d yourdomain.com

# Copy to nginx directory
sudo cp /etc/letsencrypt/live/yourdomain.com/fullchain.pem nginx/ssl/cert.pem
sudo cp /etc/letsencrypt/live/yourdomain.com/privkey.pem nginx/ssl/key.pem

# Restart nginx
docker-compose -f docker-compose.prod.yml restart nginx
```

## Health Checks

All services include health checks:
- **Backend**: HTTP GET to `/health`
- **Frontend**: HTTP GET to root
- **Nginx**: HTTP GET to `/health`

Check status:
```bash
docker-compose -f docker-compose.prod.yml ps
```

## Monitoring

### View Logs
```bash
# All services
docker-compose -f docker-compose.prod.yml logs -f

# Specific service
docker-compose -f docker-compose.prod.yml logs -f backend

# Last 100 lines
docker-compose -f docker-compose.prod.yml logs --tail=100
```

### Health Check
```bash
# Backend health
curl https://yourdomain.com/api/status

# Frontend
curl https://yourdomain.com

# Nginx
curl https://yourdomain.com/health
```

## Maintenance

### Update Application
```bash
# Pull latest changes
git pull

# Rebuild and restart
docker-compose -f docker-compose.prod.yml up -d --build
```

### Backup
```bash
# Backup volumes
docker run --rm -v backend-logs:/data -v $(pwd)/backups:/backup \
  alpine tar czf /backup/backend-logs-$(date +%Y%m%d).tar.gz /data

# Backup database (if applicable)
docker-compose exec backend pg_dump -U user myapp > backup.sql
```

### Restore
```bash
# Restore volumes
docker run --rm -v backend-logs:/data -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/backup.tar.gz -C /data
```

## Scaling

### Increase Backend Workers
Edit `docker-compose.prod.yml`:
```yaml
deploy:
  replicas: 3
```

### Resource Limits
Already configured in `docker-compose.prod.yml`:
- CPU: 1 core limit, 0.5 core reserved
- Memory: 512MB limit, 256MB reserved

## Security Checklist

- [x] HTTPS enabled
- [x] Security headers configured
- [x] Rate limiting enabled
- [x] Non-root containers
- [x] Secrets in .env (not committed)
- [x] Resource limits set
- [x] Health checks configured
- [x] Log rotation configured

## Troubleshooting

### Services not starting
```bash
# Check logs
docker-compose -f docker-compose.prod.yml logs

# Check container status
docker ps -a

# Restart services
docker-compose -f docker-compose.prod.yml restart
```

### SSL errors
```bash
# Verify certificates
openssl x509 -in nginx/ssl/cert.pem -text -noout

# Check nginx config
docker-compose exec nginx nginx -t
```

### Performance issues
```bash
# Check resource usage
docker stats

# Check logs for errors
docker-compose logs -f
```

## CI/CD Integration

### GitHub Actions Example
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to production
        run: |
          ssh user@server 'cd /app && git pull && ./deploy.sh'
```

## Environment Variables Reference

| Variable | Description | Example |
|----------|-------------|---------|
| ENVIRONMENT | Deployment environment | production |
| SECRET_KEY | Application secret | random-string |
| DATABASE_URL | Database connection | postgresql://... |
| DOMAIN | Your domain name | example.com |
| SSL_EMAIL | Let's Encrypt email | admin@example.com |

## Performance Optimization

1. **Enable Gzip**: Already configured in nginx
2. **Cache static assets**: Configured in frontend nginx
3. **CDN**: Consider CloudFlare for static assets
4. **Database**: Add connection pooling
5. **Monitoring**: Add Prometheus + Grafana
