# NGINX Reverse Proxy + Load Balancing Documentation
## Day 3: Docker Exercise

---

## Overview

This project demonstrates NGINX as a reverse proxy with load balancing across multiple backend instances.

**Architecture:**
- 1 NGINX reverse proxy (load balancer)
- 2 Backend API instances (Node.js)
- 1 Frontend application
- Round-robin load balancing enabled

---

## Architecture Diagram
```
┌──────────────────────────────────────────────────────┐
│                    Client Browser                     │
└────────────────────┬─────────────────────────────────┘
                     │ HTTP :80
                     ↓
┌────────────────────────────────────────────────────────┐
│                  NGINX (Reverse Proxy)                 │
│                  Round-Robin Load Balancer             │
│                                                        │
│  Routes:                                               │
│  /      → frontend:8080                                │
│  /api/* → backend_servers (load balanced)              │
└─────────┬──────────────────────────────┬───────────────┘
          │                              │
          │ Frontend Network             │ Backend Network
          │                              │
┌─────────▼──────────┐        ┌─────────▼──────────┐
│     Frontend       │        │   Backend Pool     │
│   (Node.js)        │        │                    │
│    :8080          │        │  ┌──────────────┐  │
└────────────────────┘        │  │  backend-1   │  │
                              │  │    :3000     │  │
                              │  └──────────────┘  │
                              │         │          │
                              │  ┌──────▼───────┐  │
                              │  │  backend-2   │  │
                              │  │    :3000     │  │
                              │  └──────────────┘  │
                              └────────────────────┘
```

---

## Components

### 1. NGINX Reverse Proxy

**Function:**
- Receives all incoming HTTP requests on port 80
- Routes requests based on URL path
- Distributes API requests across backend instances
- Acts as single entry point for entire application

**Configuration:** `nginx/nginx.conf`

**Key Features:**
- Reverse proxy for frontend
- Load balancing for backend API
- Health check endpoint
- Request logging
- Timeout configuration

### 2. Backend Instances

**Instances:** 2 (backend-1, backend-2)

**Identification:**
- Each instance has unique INSTANCE_ID environment variable
- Reports which instance handled each request
- Enables verification of load distribution

**Endpoints:**
- `GET /` - Basic info
- `GET /api` - API response
- `GET /api/info` - Detailed instance information
- `GET /api/test` - Load balancing test endpoint
- `GET /health` - Health check

### 3. Frontend Application

**Function:**
- User interface for testing load balancer
- Displays which backend instance handled request
- Shows load distribution statistics

**Features:**
- Single request testing
- Multiple request testing (10, 20 requests)
- Real-time statistics
- Visual instance identification

---

## Load Balancing

### Round-Robin Algorithm

**How it works:**
1. NGINX receives request to `/api`
2. Forwards to next backend in rotation
3. backend-1 → backend-2 → backend-1 → backend-2...
4. Ensures even distribution

**Configuration in nginx.conf:**
```nginx
upstream backend_servers {
    server backend-1:3000 weight=1;
    server backend-2:3000 weight=1;
}
```

**Parameters:**
- `weight=1` - Equal weight for both servers
- `max_fails=3` - Mark unhealthy after 3 failures
- `fail_timeout=30s` - Retry after 30 seconds

### Load Balancing Algorithms Available

**1. Round Robin (Default - Implemented):**
```nginx
upstream backend_servers {
    server backend-1:3000;
    server backend-2:3000;
}
```
- Equal distribution
- Simple and effective
- No configuration needed

**2. Least Connections:**
```nginx
upstream backend_servers {
    least_conn;
    server backend-1:3000;
    server backend-2:3000;
}
```
- Sends to server with fewest active connections
- Good for long-lived connections

**3. IP Hash:**
```nginx
upstream backend_servers {
    ip_hash;
    server backend-1:3000;
    server backend-2:3000;
}
```
- Same client always goes to same server
- Session persistence
- Sticky sessions

**4. Weighted Round Robin:**
```nginx
upstream backend_servers {
    server backend-1:3000 weight=3;
    server backend-2:3000 weight=1;
}
```
- backend-1 gets 75% of requests
- backend-2 gets 25% of requests
- Useful for different server capacities

---

## Request Flow

### Frontend Request (/):
```
Client → NGINX:80 → Frontend:8080 → Response
```

### API Request (/api/test):
```
Client → NGINX:80 → backend_servers → backend-1:3000 → Response
Client → NGINX:80 → backend_servers → backend-2:3000 → Response
Client → NGINX:80 → backend_servers → backend-1:3000 → Response
...
```

---

## Deployment

### Start All Services
```bash
docker compose up -d --build
```

### View Logs
```bash
# All services
docker compose logs -f

# NGINX only
docker compose logs -f nginx

# Specific backend
docker compose logs -f backend-1
docker compose logs -f backend-2
```

### Check Status
```bash
docker compose ps
```

### Scale Backend (Add more instances)
```bash
# Scale to 3 instances
docker compose up -d --scale backend=3

# Scale to 5 instances
docker compose up -d --scale backend=5
```

**Note:** When scaling, update nginx.conf to include new instances.

---

## Testing

### 1. Access Frontend
```bash
# Open in browser
http://localhost

# You'll see the load balancing demo interface
```

### 2. Test Load Balancing Manually
```bash
# Send 10 requests and observe distribution
for i in {1..10}; do
  curl http://localhost/api/test
  echo ""
done

# Each request should alternate between instances
```

### 3. Test with curl
```bash
# Single request
curl http://localhost/api/info | jq

# Multiple requests to see distribution
for i in {1..20}; do
  echo "Request $i:"
  curl -s http://localhost/api/test | jq -r '.instance'
done
```

**Expected Output:**
```
Request 1: backend-1
Request 2: backend-2
Request 3: backend-1
Request 4: backend-2
Request 5: backend-1
...
```

### 4. View NGINX Status
```bash
curl http://localhost/nginx_status
```

---

## Verification Checklist

### NGINX Running
```bash
docker compose ps nginx
# Status should be "Up"
```

### Both Backends Running
```bash
docker compose ps | grep backend
# Both backend-1 and backend-2 should be "Up (healthy)"
```

### Load Balancing Working
```bash
# Send multiple requests
for i in {1..10}; do
  curl -s http://localhost/api/test | jq -r '.instance'
done

# Should see alternating: backend-1, backend-2, backend-1, backend-2...
```

### Even Distribution
```bash
# Send 100 requests and count distribution
for i in {1..100}; do
  curl -s http://localhost/api/test | jq -r '.instance'
done | sort | uniq -c

# Expected output (approximately):
#   50 backend-1
#   50 backend-2
```

### Failover Working
```bash
# Stop one backend instance
docker compose stop backend-1

# Send requests - all should go to backend-2
for i in {1..5}; do
  curl -s http://localhost/api/test | jq -r '.instance'
done

# Restart backend-1
docker compose start backend-1

# Load balancing should resume
```

---

## Monitoring

### NGINX Access Logs
```bash
# Real-time access logs
docker compose logs -f nginx

# View logged requests
docker compose exec nginx tail -f /var/log/nginx/access.log
```

### Backend Logs
```bash
# See which instance handled each request
docker compose logs backend-1 | grep "GET /api/test"
docker compose logs backend-2 | grep "GET /api/test"
```

### Request Distribution
```bash
# Count requests per backend
docker compose logs backend-1 | grep -c "GET /api/test"
docker compose logs backend-2 | grep -c "GET /api/test"
```

---

## Configuration Files

### nginx.conf Key Sections

**1. Upstream Definition:**
```nginx
upstream backend_servers {
    server backend-1:3000 weight=1;
    server backend-2:3000 weight=1;
    keepalive 32;
}
```

**2. Proxy Configuration:**
```nginx
location /api {
    proxy_pass http://backend_servers;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

**3. Headers:**
- `X-Real-IP`: Client's actual IP
- `X-Forwarded-For`: Proxy chain
- `X-Forwarded-Proto`: Original protocol (http/https)

---

## Troubleshooting

### Problem: Requests Always Go to Same Backend

**Solution:**
```bash
# Check nginx.conf for ip_hash directive
# Remove ip_hash to use round-robin

# Restart NGINX
docker compose restart nginx
```

### Problem: Backend Unreachable

**Solution:**
```bash
# Check backend health
docker compose ps

# Check network connectivity
docker compose exec nginx ping backend-1
docker compose exec nginx ping backend-2

# Check nginx error logs
docker compose logs nginx | grep error
```

### Problem: Uneven Distribution

**Solution:**
```bash
# Check weights in nginx.conf
# Ensure both have weight=1

# Check for stuck connections
docker compose exec nginx cat /var/run/nginx.pid
```

---

## Advanced Configuration

### Add More Backend Instances

**1. Update docker-compose.yml:**
```yaml
  backend-3:
    build:
      context: ./backend
    container_name: day3-backend-3
    environment:
      INSTANCE_ID: backend-3
    networks:
      - backend-net
```

**2. Update nginx.conf:**
```nginx
upstream backend_servers {
    server backend-1:3000;
    server backend-2:3000;
    server backend-3:3000;  # Add new instance
}
```

**3. Restart:**
```bash
docker compose up -d
```

### Enable Health Checks

Already implemented in docker-compose.yml:
```yaml
healthcheck:
  test: ["CMD", "node", "-e", "require('http').get(...)"]
  interval: 10s
  timeout: 3s
  retries: 3
```

### Session Persistence (Sticky Sessions)

Add to nginx.conf upstream block:
```nginx
upstream backend_servers {
    ip_hash;  # Enable sticky sessions
    server backend-1:3000;
    server backend-2:3000;
}
```

---

## Performance Considerations

### Keep-Alive Connections
```nginx
upstream backend_servers {
    keepalive 32;  # Pool of 32 connections
}
```

**Benefits:**
- Reduces TCP handshake overhead
- Improves response time
- Better resource utilization

### Timeouts
```nginx
proxy_connect_timeout 60s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

### Buffering
```nginx
proxy_buffering off;  # For real-time responses
```

---

## Security Considerations

### Limit Request Rate
```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api {
    limit_req zone=api burst=20;
    proxy_pass http://backend_servers;
}
```

### Hide NGINX Version
```nginx
http {
    server_tokens off;
}
```

### Add Security Headers
```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
```

---

## Cleanup
```bash
# Stop all services
docker compose down

# Remove volumes
docker compose down -v

# Remove images
docker compose down --rmi all
```

---

## Key Learnings

1. **Reverse Proxy:**
   - Single entry point for all requests
   - Hides internal architecture
   - Enables routing and load balancing

2. **Load Balancing:**
   - Distributes load across multiple servers
   - Improves availability and scalability
   - Round-robin ensures fair distribution

3. **NGINX:**
   - Lightweight and performant
   - Flexible configuration
   - Production-ready

4. **Docker Networking:**
   - Service discovery via DNS
   - Network isolation
   - Container-to-container communication

---
