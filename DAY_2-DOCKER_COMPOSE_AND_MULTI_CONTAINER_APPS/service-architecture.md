# Service Architecture Documentation
## Day 2: Docker Compose Multi-Container Application

---

## System Overview

This is a full-stack application consisting of three services orchestrated with Docker Compose:
- **Frontend**: React application (served by NGINX)
- **Backend**: Node.js/Express API server
- **Database**: MongoDB for data persistence

---

## Architecture Diagram
┌─────────────────────────────────────────────────────────┐
│                    Docker Host                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │            Frontend Network (Bridge)            │   │
│  │                                                 │   │
│  │   ┌─────────────┐      ┌──────────────┐       │   │
│  │   │   Client    │──────│   Server     │       │   │
│  │   │   (React)   │      │   (Node.js)  │       │   │
│  │   │   :80       │      │   :3000      │       │   │
│  │   └─────────────┘      └──────────────┘       │   │
│  │         │                      │               │   │
│  └─────────│──────────────────────│───────────────┘   │
│            │                      │                   │
│            │  ┌───────────────────┘                   │
│            │  │                                       │
│  ┌─────────│──│───────────────────────────────────┐  │
│  │         │  │   Backend Network (Bridge)        │  │
│  │         │  │                                   │  │
│  │         │  │   ┌──────────────┐               │  │
│  │         │  └───│    Server    │               │  │
│  │             │  │   (Node.js)  │               │  │
│  │             │  │   :3000      │               │  │
│  │             │  └──────┬───────┘               │  │
│  │             │         │                       │  │
│  │             │         │                       │  │
│  │             │  ┌──────▼───────┐               │  │
│  │             │  │   MongoDB    │               │  │
│  │             │  │   :27017     │               │  │
│  │             │  └──────────────┘               │  │
│  │             │         │                       │  │
│  └─────────────│─────────│───────────────────────┘  │
│                │         │                          │
│  ┌─────────────│─────────│───────────────────────┐  │
│  │             │         │    Volumes            │  │
│  │             │         │                       │  │
│  │             │    ┌────▼──────┐                │  │
│  │             │    │mongo-data │                │  │
│  │             │    └───────────┘                │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  Port Mappings:                                       │
│  Host:8080 → Container:80  (Frontend)                 │
│  Host:3000 → Container:3000 (Backend)                 │
└───────────────────────────────────────────────────────┘

---

## Services

### 1. Frontend (React + NGINX)

**Container Name:** `day2-frontend`  
**Image:** Custom (built from ./client/Dockerfile)  
**Ports:** 8080:80  
**Networks:** frontend  

**Responsibilities:**
- Serve React application
- Provide user interface
- Make API calls to backend

**Technology Stack:**
- React 18
- NGINX (production server)
- Multi-stage Docker build

**Environment Variables:**
- `REACT_APP_API_URL`: Backend API endpoint

**Health:** NGINX built-in health

---

### 2. Backend (Node.js API)

**Container Name:** `day2-backend`  
**Image:** Custom (built from ./server/Dockerfile)  
**Ports:** 3000:3000  
**Networks:** frontend, backend  

**Responsibilities:**
- RESTful API endpoints
- Business logic
- Database operations
- CORS handling

**Technology Stack:**
- Node.js 18
- Express.js
- MongoDB driver

**Environment Variables:**
- `NODE_ENV`: development/production
- `PORT`: 3000
- `MONGODB_URI`: mongodb://mongo:27017/mydb

**API Endpoints:**
- `GET /` - API information
- `GET /health` - Health check
- `GET /api/items` - List all items
- `POST /api/items` - Create new item
- `DELETE /api/items/:id` - Delete item

**Health Check:**
- Interval: 30s
- Timeout: 3s
- Start Period: 10s
- Test: HTTP GET /health

---

### 3. Database (MongoDB)

**Container Name:** `day2-mongodb`  
**Image:** mongo:7  
**Ports:** None (internal only)  
**Networks:** backend  

**Responsibilities:**
- Data persistence
- NoSQL database operations

**Technology Stack:**
- MongoDB 7

**Environment Variables:**
- `MONGO_INITDB_DATABASE`: mydb

**Volumes:**
- `mongo-data:/data/db` (persistent data)
- `mongo-config:/data/configdb` (configuration)

**Health Check:**
- Interval: 10s
- Timeout: 5s
- Retries: 5
- Start Period: 20s
- Test: mongosh ping command

---

## Networks

### Frontend Network
**Type:** Bridge  
**Purpose:** Communication between client and server  
**Services:** client, server  
**Internet Access:** Yes  

### Backend Network
**Type:** Bridge  
**Purpose:** Communication between server and database  
**Services:** server, mongo  
**Internet Access:** No (internal=false but isolated from frontend)  

**Network Isolation Benefits:**
- Frontend cannot directly access database
- Backend acts as gateway
- Improved security posture

---

## Volumes

### mongo-data
**Type:** Named volume  
**Driver:** local  
**Purpose:** Persistent MongoDB data  
**Mount Point:** /data/db  
**Lifecycle:** Independent of containers  

### mongo-config
**Type:** Named volume  
**Driver:** local  
**Purpose:** MongoDB configuration  
**Mount Point:** /data/configdb  

**Data Persistence:**
- Data survives container restarts
- Data survives container deletion
- Can be backed up independently
- Managed by Docker

---

## Service Communication

### Client → Server
**Method:** HTTP requests  
**Network:** frontend  
**URL:** http://server:3000 (internal) or http://localhost:3000 (from host)  
**Protocol:** REST API  

### Server → MongoDB
**Method:** MongoDB protocol  
**Network:** backend  
**URL:** mongodb://mongo:27017/mydb  
**Driver:** mongodb npm package  

**DNS Resolution:**
- Docker provides automatic DNS
- Service names resolve to container IPs
- No need for IP addresses in config

---

## Deployment

### Start All Services
```bash
docker compose up -d
```

### View Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f server
docker compose logs -f mongo
docker compose logs -f client
```

### Check Status
```bash
docker compose ps
```

### Stop All Services
```bash
docker compose down
```

### Stop and Remove Volumes
```bash
docker compose down -v
```

---

## Verification Checklist

### Service Connectivity

**Test Backend API:**
```bash
curl http://localhost:3000
curl http://localhost:3000/health
curl http://localhost:3000/api/items
```

**Test Frontend:**
```bash
# Open in browser
http://localhost:8080
```

### MongoDB Connection

**Verify from Backend:**
```bash
# Check backend logs for MongoDB connection
docker compose logs server | grep MongoDB

# Expected: "Connected to MongoDB successfully!"
```

**Direct MongoDB Access:**
```bash
docker compose exec mongo mongosh mydb
> db.items.find()
```

### Volume Persistence

**Test Data Persistence:**
```bash
# 1. Create some data via frontend

# 2. Stop containers
docker compose down

# 3. Start again
docker compose up -d

# 4. Check data is still there
# Data should persist because of volumes
```

### Network Isolation

**Verify Network Setup:**
```bash
# List networks
docker network ls | grep day2

# Inspect frontend network
docker network inspect day2-fullstack-app_frontend

# Inspect backend network
docker network inspect day2-fullstack-app_backend
```

### Logs Exposure

**All Logs Accessible:**
```bash
# Frontend logs
docker compose logs client

# Backend logs
docker compose logs server

# Database logs
docker compose logs mongo

# Combined logs
docker compose logs
```

---

## Data Flow

### Creating an Item

1. **User Action:**
   - User fills form in React app
   - Clicks "Add Item"

2. **Frontend:**
   - React makes POST request to http://localhost:3000/api/items
   - Request includes item data (name, description)

3. **Backend:**
   - Express receives request
   - Validates data
   - Connects to MongoDB via mongodb://mongo:27017/mydb

4. **Database:**
   - MongoDB stores document in `items` collection
   - Returns inserted document ID

5. **Response:**
   - Backend sends success response to frontend
   - Frontend refreshes item list
   - User sees new item

---

## Environment Variables

### Development (.env)
```bash
NODE_ENV=development
API_PORT=3000
CLIENT_PORT=8080
MONGO_INITDB_DATABASE=mydb
```

### Production (.env.production)
```bash
NODE_ENV=production
API_PORT=3000
CLIENT_PORT=80
MONGO_INITDB_DATABASE=mydb
```

---

## Troubleshooting

### Container Not Starting
```bash
# Check logs
docker compose logs [service-name]

# Rebuild image
docker compose build --no-cache [service-name]

# Start with dependency check
docker compose up --force-recreate
```

### Cannot Connect to MongoDB
```bash
# Check if MongoDB is healthy
docker compose ps

# Check MongoDB logs
docker compose logs mongo

# Verify network
docker compose exec server ping mongo
```

### Frontend Cannot Reach Backend
```bash
# Check CORS settings in server.js
# Verify API_URL in client
# Check browser console for errors
```

### Volume Data Lost
```bash
# List volumes
docker volume ls | grep day2

# Inspect volume
docker volume inspect day2-fullstack-app_mongo-data

# Ensure not using docker compose down -v
```

---

## Best Practices Implemented

- **Multi-stage Builds:** Client uses build stage for React, production stage with NGINX  
- **Health Checks:** Both server and database have health checks  
- **Non-root User:** Server runs as nodejs user  
- **Network Segmentation:** Frontend and backend networks separated  
- **Volume Persistence:** Database data persists across restarts  
- **Environment Variables:** Configuration via .env file  
- **Dependency Management:** depends_on with health conditions  
- **Restart Policies:** unless-stopped for automatic recovery  
- **Logging:** All services log to stdout (Docker captures)  

---

## Performance Considerations

- **Frontend:** NGINX serves static files efficiently
- **Backend:** Node.js with connection pooling
- **Database:** MongoDB indexes on frequently queried fields
- **Network:** Bridge networks have minimal overhead
- **Volumes:** Local driver for best performance

---

## Security Considerations

- Database not exposed to host
- Backend validates all inputs
- CORS configured in backend
- Non-root users in containers
- Environment variables for sensitive data
- Network isolation between layers

---

## Scaling Possibilities
```bash
# Scale backend servers
docker compose up -d --scale server=3

# Add load balancer (NGINX)
# Add Redis for caching
# Use external MongoDB (cloud)
```

---

## Maintenance

### Backup
```bash
# Backup MongoDB data
docker compose exec mongo mongodump --out=/backup

# Copy backup from container
docker cp day2-mongodb:/backup ./mongodb-backup
```

### Updates
```bash
# Pull latest images
docker compose pull

# Rebuild custom images
docker compose build

# Restart with new images
docker compose up -d
```

---
