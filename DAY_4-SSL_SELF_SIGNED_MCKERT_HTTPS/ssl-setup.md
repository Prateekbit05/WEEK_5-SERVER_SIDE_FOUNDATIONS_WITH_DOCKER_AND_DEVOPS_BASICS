# SSL/TLS Setup Documentation
## Day 4: HTTPS with mkcert and NGINX

---

## Overview

This project demonstrates HTTPS configuration using:
- mkcert for local SSL certificates
- NGINX for SSL termination
- HTTP to HTTPS automatic redirect
- TLS 1.2 and 1.3 support

---

## Certificate Information

### Generation Method: mkcert

**Why mkcert?**
- Creates locally-trusted certificates
- No browser security warnings
- Perfect for development
- Automatic CA installation

**Generated Certificates:**
- `localhost.pem` - Public certificate
- `localhost-key.pem` - Private key

**Valid For:**
- localhost
- 127.0.0.1
- ::1 (IPv6 localhost)

**Certificate Authority:**
- Local CA created by mkcert
- Installed in system trust store
- Installed in Firefox (if present)

### Certificate Generation Steps
```bash
# 1. Install mkcert
brew install mkcert  # macOS
choco install mkcert # Windows

# 2. Create local CA
mkcert -install

# 3. Generate certificates
mkcert -cert-file nginx/ssl/localhost.pem \
       -key-file nginx/ssl/localhost-key.pem \
       localhost 127.0.0.1 ::1
```

---

## NGINX HTTPS Configuration

### SSL Configuration Sections

**1. Certificate Location:**
```nginx
ssl_certificate /etc/nginx/ssl/localhost.pem;
ssl_certificate_key /etc/nginx/ssl/localhost-key.pem;
```

**2. SSL Protocols:**
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```
- TLS 1.0 and 1.1 disabled (deprecated)
- TLS 1.2 and 1.3 enabled (secure)

**3. Cipher Suites:**
```nginx
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256...';
```
- Forward secrecy enabled
- Strong encryption algorithms
- Modern cipher preference

**4. Session Management:**
```nginx
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
```
- Improves performance
- Reduces handshake overhead

---

## HTTP to HTTPS Redirect

### Configuration:
```nginx
server {
    listen 80;
    server_name localhost;
    return 301 https://$server_name$request_uri;
}
```

**How it works:**
1. Client connects to http://localhost
2. NGINX returns 301 redirect
3. Browser follows to https://localhost
4. Secure connection established

**Testing:**
```bash
# Should redirect to HTTPS
curl -I http://localhost

# Response:
HTTP/1.1 301 Moved Permanently
Location: https://localhost/
```

---

## Security Headers

### HSTS (HTTP Strict Transport Security)
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```
- Forces HTTPS for 1 year
- Applies to all subdomains
- Prevents downgrade attacks

### X-Frame-Options
```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```
- Prevents clickjacking
- Allows framing from same origin only

### X-Content-Type-Options
```nginx
add_header X-Content-Type-Options "nosniff" always;
```
- Prevents MIME type sniffing
- Forces declared content type

### X-XSS-Protection
```nginx
add_header X-XSS-Protection "1; mode=block" always;
```
- Enables XSS filter
- Blocks page if attack detected

---

## SSL Termination

### What is SSL Termination?

SSL termination = decrypting HTTPS traffic at the proxy layer

**Flow:**
Client ←--HTTPS--→ NGINX ←--HTTP--→ Backend
(encrypted)        (unencrypted)

**Benefits:**
- Backend doesn't handle SSL/TLS
- Centralized certificate management
- Better performance
- Simpler backend configuration

**Headers Added:**
```nginx
X-Forwarded-Proto: https
X-Real-IP: client_ip
X-Forwarded-For: client_ip
```

---

## Testing HTTPS

### Browser Testing

**1. Access Application:**
https://localhost

**2. Verify Lock Icon:**
- Lock icon visible in address bar
- "Connection is secure" message
- No warnings

**3. Check Certificate:**
- Click lock icon
- View certificate details
- Verify issued by mkcert

**4. Test HTTP Redirect:**
http://localhost  →  https://localhost

### Command Line Testing

**Test HTTPS Connection:**
```bash
curl https://localhost
```

**Verbose Output:**
```bash
curl -v https://localhost
```

**Check Certificate:**
```bash
openssl s_client -connect localhost:443 -servername localhost
```

**View Certificate Details:**
```bash
openssl x509 -in nginx/ssl/localhost.pem -text -noout
```

---

## Verification Checklist

### Certificate Generation
```bash
# Check certificates exist
ls -la nginx/ssl/

# Expected files:
# localhost.pem
# localhost-key.pem
```

###  HTTPS Working
```bash
# Test HTTPS
curl -I https://localhost

# Expected: HTTP/2 200
```

###  HTTP Redirect
```bash
# Test redirect
curl -I http://localhost

# Expected: 301 Moved Permanently
# Location: https://localhost/
```

###  Browser Trust
- Open https://localhost in browser
- No security warnings
- Lock icon visible
- Certificate trusted

###  Backend Communication
```bash
# Check backend logs
docker compose logs backend

# Should show HTTPS in logs from X-Forwarded-Proto header
```

---

## Troubleshooting

### Problem: Browser Shows "Not Secure"

**Solution:**
```bash
# Reinstall mkcert CA
mkcert -install

# Restart browser completely
# Clear browser cache
# Try in private/incognito mode
```

### Problem: Certificate Error

**Solution:**
```bash
# Regenerate certificates
mkcert -cert-file nginx/ssl/localhost.pem \
       -key-file nginx/ssl/localhost-key.pem \
       localhost 127.0.0.1 ::1

# Restart NGINX
docker compose restart nginx
```

### Problem: HTTP Doesn't Redirect

**Check nginx.conf:**
```nginx
# Ensure HTTP server block has redirect
server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

### Problem: Mixed Content Warnings

**Solution:**
- Ensure all resources loaded via HTTPS
- Check API calls use relative URLs
- Update hardcoded http:// to https://

---

## Production Considerations

### For Production Use:

**1. Real Certificates:**
- Use Let's Encrypt (free, automated)
- Use commercial CA
- NOT self-signed certificates

**2. Certificate Management:**
```bash
# Auto-renewal with certbot
certbot renew --nginx
```

**3. Strong Security:**
- Enable OCSP stapling
- Use HSTS preload
- Implement Certificate Transparency

**4. Monitoring:**
- Certificate expiration alerts
- SSL Labs testing (ssllabs.com/ssltest/)
- Regular security audits

---

## File Structure
nginx/ssl/
├── localhost.pem          # Public certificate
└── localhost-key.pem      # Private key (NEVER commit!)

**Security Notes:**
-  NEVER commit private keys to git
-  Add `*.key` and `*-key.pem` to .gitignore
-  Set proper permissions: `chmod 600 *.key`

---

## Environment-Specific Certificates

### Development (localhost):
```bash
mkcert localhost 127.0.0.1 ::1
```

### Custom Domain (myapp.local):
```bash
# Add to /etc/hosts:
# 127.0.0.1 myapp.local

# Generate certificate:
mkcert myapp.local "*.myapp.local"
```

### Production (example.com):
```bash
# Use Let's Encrypt
certbot certonly --nginx -d example.com -d www.example.com
```

---

## Key Learnings

### SSL/TLS Concepts:
-  Encryption in transit
-  Certificate-based authentication
-  Public key infrastructure (PKI)
-  Certificate authorities

### mkcert Benefits:
-  Local development certificates
-  Trusted by browser
-  No security warnings
-  Easy to use

### NGINX SSL:
-  SSL termination
-  Protocol configuration
-  Security headers
-  HTTP redirect

### Production Ready:
-  TLS 1.2/1.3 only
-  Strong ciphers
-  HSTS enabled
-  Security headers

---
