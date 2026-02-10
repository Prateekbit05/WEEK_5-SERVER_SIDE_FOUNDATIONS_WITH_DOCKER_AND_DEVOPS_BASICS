# Linux Internals Exploration - Day 1 Docker Exercise

## Container Information

**Container Name:** node1
**Image:** day1-node-app:latest  
**Base OS:** Alpine Linux 3.18  
---

## 1. File System Structure

### Working Directory: `/app`
```
/app/
├── node_modules/
├── package.json
├── server.js
```

**Observations:**
- Application runs from `/app` directory
- Files owned by `nodejs` user (UID 1001)
- Permissions are restrictive (non-root user)

### Root Directory: `/`
```
/
├── bin/       # Essential command binaries
├── etc/       # Configuration files
├── home/      # User home directories
├── lib/       # Shared libraries
├── proc/      # Process information
├── root/      # Root user home
├── tmp/       # Temporary files
├── usr/       # User programs
├── var/       # Variable data
└── app/       # Our application (custom)
```

**Key Findings:**
- Alpine Linux minimal filesystem (~5MB base)
- No unnecessary packages installed
- Standard Linux FHS (Filesystem Hierarchy Standard)

---

## 2. Process Management

### Running Processes (ps aux):
```
PID   USER     COMMAND
1     nodejs   node server.js
15    nodejs   /bin/sh
23    nodejs   ps aux
```

**Observations:**
- Node.js server runs as PID 1 (init process)
- Container has minimal processes (efficient)
- All processes run as `nodejs` user (security)
- When PID 1 exits, container stops

### Process Details:
- **Main Process:** `node server.js` (PID 1)
- **Memory Usage:** ~50MB
- **CPU Usage:** <1%
- **Process State:** Running (R)

---

## 3. Users and Permissions

### Current User:
```bash
$ whoami
nodejs

$ id
uid=1001(nodejs) gid=1001(nodejs) groups=1001(nodejs)
```

### User Setup:
- **Username:** nodejs
- **UID:** 1001
- **GID:** 1001
- **Home:** /home/nodejs
- **Shell:** /bin/sh

**Security Notes:**
- Running as non-root user (best practice)
- Limited privileges
- Cannot modify system files
- Isolated from host system

### File Permissions:
```
-rw-r--r--  1 nodejs nodejs  123 Jan 1 12:00 package.json
-rw-r--r--  1 nodejs nodejs 2048 Jan 1 12:00 server.js
drwxr-xr-x  3 nodejs nodejs 4096 Jan 1 12:00 node_modules
```

**Permission Breakdown:**
- Owner (nodejs): read, write
- Group (nodejs): read
- Others: read
- Directories: execute permission for traversal

---

## 4. Network Configuration

### Network Interface:
```
eth0: 172.17.0.2/16
lo:   127.0.0.1/8
```

**Network Details:**
- **IP Address:** 172.17.0.2 (Docker bridge network)
- **Gateway:** 172.17.0.1
- **Container Hostname:** f3a2d1c9b8e7 (container ID)
- **Listening Port:** 3000 (Node.js server)

### Connectivity:
- Can reach external internet
- Can be accessed from host via port mapping
- DNS resolution working

---

## 5. System Resources

### Memory:
```
Total Memory: 2048 MB
Used:         512 MB
Free:         1536 MB
App Usage:    ~50 MB
```

### CPU:
```
CPUs Available: 4
Architecture:   x86_64
Model:         Intel Core i7
```

### Disk:
```
Filesystem      Size  Used  Avail Use% Mounted on
overlay         100G   15G    85G  15% /
tmpfs            1G     0     1G   0% /dev
```

**Observations:**
- Container shares host kernel
- Memory limits not set (uses host memory)
- CPU sharing with host
- Overlay filesystem (layered)

---

## 6. Operating System

### OS Information:
```
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.18.4
PRETTY_NAME="Alpine Linux v3.18"
```

**Characteristics:**
- Minimal Linux distribution
- BusyBox utilities (lightweight)
- musl libc (instead of glibc)
- APK package manager
- Very small footprint (~5MB)

---

## 7. Installed Software

### Key Packages:
- Node.js v18.18.0
- npm 9.8.1
- Alpine Package Manager (apk)
- Basic Unix utilities (ls, ps, top, etc.)

### Missing Tools (compared to Ubuntu):
- bash (only sh available)
- systemd
- Many GNU tools
- Lightweight alternatives included

---

## 8. Environment Variables
```bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOME=/home/nodejs
NODE_VERSION=18.18.0
HOSTNAME=f3a2d1c9b8e7
PWD=/app
```

**Key Variables:**
- `NODE_VERSION`: Ensures Node.js version consistency
- `PATH`: Command lookup path
- `HOSTNAME`: Container ID (changes each run)
- `PWD`: Current working directory

---

## 9. Logging

### Application Logs:
- **Method:** stdout/stderr (console.log)
- **Captured by:** Docker daemon
- **Access:** `docker logs day1-container`

**Log Sample:**
```
Server started on port 3000
Running in Docker container: f3a2d1c9b8e7
[2024-01-15T10:30:45.123Z] GET /
[2024-01-15T10:30:50.456Z] GET /health
```

**Logging Best Practices Observed:**
- Logs to stdout (Docker best practice)
- Structured log format
- Timestamps included
- No file-based logging in container

---

## 10. Container vs Virtual Machine

### Key Differences Observed:

| Aspect | Container | Virtual Machine |
|--------|-----------|-----------------|
| Boot Time | Instant | Minutes |
| Size | ~50MB | ~GBs |
| Isolation | Process-level | Hardware-level |
| Kernel | Shared with host | Separate |
| Resource Usage | Minimal | High |
| Startup | PID 1 starts | Full OS boots |

---

## 11. Security Observations

**Good Practices Found:**
- Running as non-root user
- Minimal attack surface (Alpine)
- Limited installed packages
- Read-only application files
- Process isolation

**Potential Improvements:**
- Could add resource limits (memory, CPU)
- Could make root filesystem read-only
- Could drop additional capabilities

---

## 12. Commands Used for Exploration
```bash
# File system
ls -la
pwd
du -sh /app
cat /etc/os-release
df -h

# Processes
ps aux
ps -ef
top
ps auxf

# Users
whoami
id
cat /etc/passwd

# Network
ip addr show
hostname
cat /etc/hosts
netstat -tuln

# Resources
free -m
nproc
uptime
cat /proc/cpuinfo

# Environment
env
echo $PATH

# Package info
apk list --installed
node --version
```

---

## 13. Key Learnings

1. **Containers are Lightweight:**
   - Only essential files included
   - Shares host kernel
   - Fast startup (seconds)

2. **Process Isolation:**
   - Container has own process namespace
   - PID 1 is application (not init system)
   - Process tree isolated from host

3. **File System Layering:**
   - Read-only image layers
   - Writable container layer
   - Efficient storage with layer sharing

4. **Networking:**
   - Virtual network interface
   - Bridge network by default
   - Port mapping for external access

5. **Security:**
   - Non-root user execution
   - Minimal packages = smaller attack surface
   - Namespace isolation

6. **Ephemeral Nature:**
   - Filesystem changes lost on deletion
   - Need volumes for persistence
   - Stateless by design

---

## 14. Comparison: Inside Container vs Host

| Aspect | Inside Container | On Host |
|--------|------------------|---------|
| User | nodejs (UID 1001) | Your username |
| Processes | Only container processes | All system processes |
| Filesystem | Isolated, minimal | Full system |
| Network | Virtual interface | Physical/virtual interfaces |
| Permissions | Limited | Full (or sudo) |
| OS | Alpine Linux | Your OS |

---

## 15. Conclusion

This exercise demonstrated how Docker containers provide:
- **Isolation:** Separate namespace for processes, files, network
- **Efficiency:** Minimal overhead, fast startup
- **Security:** Non-root user, limited capabilities
- **Portability:** Same environment anywhere Docker runs
- **Consistency:** Reproducible across machines

**Container Characteristics:**
- Lightweight (MBs not GBs)
- Fast (seconds not minutes)
- Isolated (own process tree)
- Ephemeral (stateless by default)
- Portable (run anywhere)

---

## Appendix: Raw Command Outputs

### ps aux
```
PID   USER     TIME  COMMAND
    1 nodejs   0:00 node server.js
   15 nodejs   0:00 /bin/sh
   23 nodejs   0:00 ps aux
```

### df -h
```
Filesystem      Size  Used Avail Use% Mounted on
overlay         100G   15G   85G  15% /
tmpfs            64M     0   64M   0% /dev
tmpfs          1.0G     0  1.0G   0% /sys/fs/cgroup
```

### free -m
```
              total        used        free      shared  buff/cache   available
Mem:           2048         512        1200          10         336        1450
Swap:          1024           0        1024
```

---

