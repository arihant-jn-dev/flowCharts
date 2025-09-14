# Complete Proxy Guide: Types, Usage & Flow Diagrams

## Overview
This comprehensive guide explains different types of proxies, their use cases, and how they work in real-world scenarios. Proxies act as intermediaries between clients and servers, providing various benefits like security, performance, load balancing, and privacy.

## 🎯 **Quick Proxy Concepts Overview**

**A proxy is an intermediary server that sits between a client and a target server, forwarding requests and responses.**

**Key Benefits:**
- 🔒 **Security** - Hide client identity, filter malicious content
- ⚡ **Performance** - Caching, load balancing, compression
- 🌐 **Access Control** - Geographic restrictions, content filtering
- 📊 **Monitoring** - Traffic analysis, logging, debugging
- 🔄 **Load Distribution** - Distribute traffic across multiple servers

## Table of Contents
- [Forward Proxy vs Reverse Proxy](#forward-proxy-vs-reverse-proxy)
- [Nginx Reverse Proxy](#nginx-reverse-proxy)
- [Load Balancer Proxy](#load-balancer-proxy)
- [API Gateway Proxy](#api-gateway-proxy)
- [CDN (Content Delivery Network)](#cdn-content-delivery-network)
- [SSL/TLS Termination Proxy](#ssltls-termination-proxy)
- [Caching Proxy](#caching-proxy)
- [Security Proxy (WAF)](#security-proxy-waf)
- [Real-World Examples](#real-world-examples)

---

## Forward Proxy vs Reverse Proxy

### 🔄 **Proxy Types Overview**

```mermaid
graph TB
    subgraph "Forward Proxy Flow"
        Client1[👨‍💻 Client] --> FProxy[🔄 Forward Proxy<br/>Squid, Corporate Proxy]
        FProxy --> Internet[🌐 Internet/External Servers]
        Internet --> FProxy
        FProxy --> Client1
    end
    
    subgraph "Reverse Proxy Flow"
        Client2[👨‍💻 Client] --> Internet2[🌐 Internet]
        Internet2 --> RProxy[🔄 Reverse Proxy<br/>Nginx, HAProxy, Cloudflare]
        RProxy --> Server1[🖥️ Server 1]
        RProxy --> Server2[🖥️ Server 2]
        RProxy --> Server3[🖥️ Server 3]
    end
    
    style FProxy fill:#e1f5fe
    style RProxy fill:#e8f5e8
    style Client1 fill:#fff3e0
    style Client2 fill:#fff3e0
```

### **📊 Forward Proxy vs Reverse Proxy Comparison**

| Aspect | Forward Proxy | Reverse Proxy |
|--------|---------------|---------------|
| **Purpose** | Hides client identity from servers | Hides server details from clients |
| **Who Controls** | Client or organization | Server or service provider |
| **Traffic Direction** | Client → Proxy → Server | Client → Proxy → Multiple Servers |
| **Use Cases** | Corporate filtering, anonymity | Load balancing, SSL termination |
| **Examples** | Squid, Corporate firewalls | Nginx, HAProxy, Cloudflare |

### **🔒 Forward Proxy Example**

**Corporate Network Setup:**
```bash
# Squid proxy configuration
# /etc/squid/squid.conf

# Allow local network
acl localnet src 192.168.1.0/24

# Block social media during work hours
acl social_media dstdomain .facebook.com .twitter.com .instagram.com
acl work_hours time MTWHF 09:00-17:00
http_access deny social_media work_hours

# Allow HTTPS traffic
acl SSL_ports port 443
acl CONNECT method CONNECT
http_access allow CONNECT SSL_ports

# Default proxy port
http_port 3128

# Cache configuration
cache_dir ufs /var/spool/squid 100 16 256
```

**Client Configuration:**
```bash
# Configure client to use corporate proxy
export http_proxy=http://proxy.company.com:3128
export https_proxy=http://proxy.company.com:3128

# Or in browser settings:
# HTTP Proxy: proxy.company.com:3128
# HTTPS Proxy: proxy.company.com:3128
```

---

## Nginx Reverse Proxy

### 🌐 **Nginx as Reverse Proxy**

```mermaid
graph TD
    Internet[🌐 Internet] --> Nginx[🔄 Nginx Reverse Proxy<br/>Port 80/443]
    
    Nginx --> |/api/*| Backend[🖥️ Backend API<br/>Node.js:3000]
    Nginx --> |/static/*| Static[📁 Static Files<br/>Port 8080]
    Nginx --> |/*| Frontend[⚛️ React App<br/>Port 3000]
    
    Nginx --> |Load Balance| App1[🖥️ App Instance 1<br/>Port 3001]
    Nginx --> |Load Balance| App2[🖥️ App Instance 2<br/>Port 3002]
    Nginx --> |Load Balance| App3[🖥️ App Instance 3<br/>Port 3003]
    
    style Nginx fill:#e8f5e8
    style Backend fill:#e1f5fe
    style Frontend fill:#f3e5f5
    style Static fill:#fff3e0
```

### **⚙️ Nginx Configuration Examples**

#### **Basic Reverse Proxy:**
```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name myapp.com;
    
    # Proxy all requests to backend
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### **Advanced Multi-Service Proxy:**
```nginx
# /etc/nginx/sites-available/microservices
upstream backend_api {
    server localhost:3001 weight=3;
    server localhost:3002 weight=2;
    server localhost:3003 weight=1;
    server localhost:3004 backup;
}

upstream frontend_app {
    server localhost:3000;
}

server {
    listen 80;
    server_name myapp.com;
    
    # API requests
    location /api/ {
        proxy_pass http://backend_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # API-specific settings
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        # Enable proxy buffering for better performance
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Static files
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # Frontend application
    location / {
        proxy_pass http://frontend_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

#### **Docker Compose with Nginx Proxy:**
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - frontend
      - backend
    
  frontend:
    image: react-app:latest
    expose:
      - "3000"
    
  backend:
    image: node-api:latest
    expose:
      - "3001"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    
  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

## Load Balancer Proxy

### ⚖️ **Load Balancing Strategies**

```mermaid
graph TD
    Client[👨‍💻 Client Request] --> LB[⚖️ Load Balancer<br/>HAProxy/Nginx]
    
    LB --> |Round Robin| Server1[🖥️ Server 1<br/>CPU: 30%]
    LB --> |Least Connections| Server2[🖥️ Server 2<br/>CPU: 45%]
    LB --> |Weighted| Server3[🖥️ Server 3<br/>CPU: 60%]
    LB --> |Health Check| Server4[🖥️ Server 4<br/>❌ Down]
    
    subgraph "Load Balancing Algorithms"
        RR[🔄 Round Robin<br/>Sequential distribution]
        LC[📊 Least Connections<br/>Route to server with fewest active connections]
        WRR[⚖️ Weighted Round Robin<br/>Distribute based on server capacity]
        IP[🎯 IP Hash<br/>Same client → same server]
    end
    
    style LB fill:#e8f5e8
    style Server4 fill:#ffebee
    style Server1 fill:#e8f5e8
    style Server2 fill:#fff3e0
    style Server3 fill:#f3e5f5
```

### **🔧 HAProxy Configuration**

#### **Production Load Balancer Setup:**
```haproxy
# /etc/haproxy/haproxy.cfg
global
    daemon
    user haproxy
    group haproxy
    
    # SSL/TLS configuration
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    option redispatch
    retries 3

# Frontend - receives requests
frontend web_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/myapp.pem
    
    # Redirect HTTP to HTTPS
    redirect scheme https code 301 if !{ ssl_fc }
    
    # Route API requests
    acl is_api path_beg /api/
    use_backend api_servers if is_api
    
    # Route static files
    acl is_static path_beg /static/
    use_backend static_servers if is_static
    
    # Default to web servers
    default_backend web_servers

# Backend pools
backend web_servers
    balance roundrobin
    option httpchk GET /health
    
    server web1 192.168.1.10:3000 check inter 30s
    server web2 192.168.1.11:3000 check inter 30s
    server web3 192.168.1.12:3000 check inter 30s

backend api_servers
    balance leastconn
    option httpchk GET /api/health
    
    server api1 192.168.1.20:3001 check inter 15s weight 3
    server api2 192.168.1.21:3001 check inter 15s weight 2
    server api3 192.168.1.22:3001 check inter 15s weight 1

backend static_servers
    balance source
    option httpchk GET /health
    
    server static1 192.168.1.30:8080 check inter 60s
    server static2 192.168.1.31:8080 check inter 60s

# Statistics page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
```

### **📊 Load Balancer with Health Checks**

```bash
# Health check script for application
#!/bin/bash
# /usr/local/bin/health_check.sh

# Check if application is responding
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)

if [ $response -eq 200 ]; then
    echo "Application healthy"
    exit 0
else
    echo "Application unhealthy - HTTP $response"
    exit 1
fi
```

---

## API Gateway Proxy

### 🚪 **API Gateway Architecture**

```mermaid
graph TD
    Mobile[📱 Mobile App] --> Gateway[🚪 API Gateway<br/>Kong/AWS API Gateway]
    Web[🌐 Web App] --> Gateway
    Partner[🤝 Partner API] --> Gateway
    
    Gateway --> |Authentication| Auth[🔐 Auth Service]
    Gateway --> |Rate Limiting| Rate[📊 Rate Limiter]
    Gateway --> |Logging| Log[📝 Logging Service]
    
    Gateway --> |/users/*| UserAPI[👤 User Service<br/>Port 3001]
    Gateway --> |/products/*| ProductAPI[📦 Product Service<br/>Port 3002]
    Gateway --> |/orders/*| OrderAPI[📋 Order Service<br/>Port 3003]
    Gateway --> |/payments/*| PaymentAPI[💳 Payment Service<br/>Port 3004]
    
    subgraph "Cross-Cutting Concerns"
        Auth
        Rate
        Log
        Transform[🔄 Data Transformation]
        Cache[💾 Response Caching]
        Monitor[📊 Monitoring]
    end
    
    style Gateway fill:#e8f5e8
    style Auth fill:#e1f5fe
    style Rate fill:#fff3e0
    style Log fill:#f3e5f5
```

### **⚙️ Kong API Gateway Configuration**

#### **Kong Setup with Docker:**
```yaml
# docker-compose.yml for Kong
version: '3.8'

services:
  kong-database:
    image: postgres:13
    environment:
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: kong
      POSTGRES_DB: kong
    volumes:
      - kong_data:/var/lib/postgresql/data

  kong-migration:
    image: kong:latest
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kong
    depends_on:
      - kong-database

  kong:
    image: kong:latest
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    ports:
      - "8000:8000"  # Proxy port
      - "8443:8443"  # Proxy SSL port
      - "8001:8001"  # Admin API port
    depends_on:
      - kong-migration

volumes:
  kong_data:
```

#### **Kong Service Configuration:**
```bash
# Add a service
curl -i -X POST http://localhost:8001/services/ \
  --data "name=user-service" \
  --data "url=http://user-service:3001"

# Add a route
curl -i -X POST http://localhost:8001/services/user-service/routes \
  --data "paths[]=/users"

# Add authentication plugin
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=jwt"

# Add rate limiting
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=rate-limiting" \
  --data "config.minute=100" \
  --data "config.hour=1000"

# Add request transformation
curl -i -X POST http://localhost:8001/services/user-service/plugins \
  --data "name=request-transformer" \
  --data "config.add.headers[]=X-Service-Name:user-service"
```

### **🔧 Express.js API Gateway Example**

```javascript
// api-gateway/server.js
const express = require('express');
const httpProxy = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');

const app = express();

// Rate limiting middleware
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP'
});

// JWT authentication middleware
const authenticateToken = (req, res, next) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1];
  
  if (!token) {
    return res.sendStatus(401);
  }
  
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.sendStatus(403);
    req.user = user;
    next();
  });
};

// Apply rate limiting to all routes
app.use(limiter);

// Logging middleware
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} - ${req.method} ${req.path}`);
  next();
});

// Service proxies
const serviceProxies = {
  '/api/users': {
    target: 'http://user-service:3001',
    changeOrigin: true,
    pathRewrite: { '^/api/users': '' }
  },
  '/api/products': {
    target: 'http://product-service:3002',
    changeOrigin: true,
    pathRewrite: { '^/api/products': '' }
  },
  '/api/orders': {
    target: 'http://order-service:3003',
    changeOrigin: true,
    pathRewrite: { '^/api/orders': '' }
  }
};

// Apply authentication to protected routes
app.use('/api/users', authenticateToken);
app.use('/api/orders', authenticateToken);

// Setup proxy routes
Object.keys(serviceProxies).forEach(path => {
  app.use(path, httpProxy(serviceProxies[path]));
});

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`API Gateway running on port ${PORT}`);
});
```

---

## CDN (Content Delivery Network)

### 🌍 **CDN Global Distribution**

```mermaid
graph TD
    Origin[🏠 Origin Server<br/>New York]
    
    subgraph "CDN Edge Locations"
        LA[🌴 Los Angeles<br/>CDN Edge]
        London[🏰 London<br/>CDN Edge]
        Tokyo[🗼 Tokyo<br/>CDN Edge]
        Sydney[🦘 Sydney<br/>CDN Edge]
    end
    
    User1[👨‍💻 User<br/>California] --> LA
    User2[👨‍💻 User<br/>UK] --> London
    User3[👨‍💻 User<br/>Japan] --> Tokyo
    User4[👨‍💻 User<br/>Australia] --> Sydney
    
    LA --> |Cache Miss| Origin
    London --> |Cache Miss| Origin
    Tokyo --> |Cache Miss| Origin
    Sydney --> |Cache Miss| Origin
    
    subgraph "CDN Benefits"
        Speed[⚡ Faster Load Times<br/>Content served from nearest edge]
        Scale[📈 Scalability<br/>Distribute load globally]
        Reliability[🛡️ Reliability<br/>Multiple edge locations]
        Cost[💰 Cost Reduction<br/>Reduce origin server load]
    end
    
    style Origin fill:#e8f5e8
    style LA fill:#e1f5fe
    style London fill:#f3e5f5
    style Tokyo fill:#fff3e0
    style Sydney fill:#fce4ec
```

### **☁️ CloudFlare CDN Configuration**

#### **CloudFlare Workers (Edge Computing):**
```javascript
// cloudflare-worker.js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)
  
  // Cache static assets for 1 year
  if (url.pathname.match(/\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2)$/)) {
    const response = await fetch(request)
    const newResponse = new Response(response.body, response)
    
    newResponse.headers.set('Cache-Control', 'public, max-age=31536000')
    newResponse.headers.set('CDN-Cache-Control', 'max-age=31536000')
    
    return newResponse
  }
  
  // API requests - add security headers
  if (url.pathname.startsWith('/api/')) {
    const response = await fetch(request)
    const newResponse = new Response(response.body, response)
    
    newResponse.headers.set('X-Content-Type-Options', 'nosniff')
    newResponse.headers.set('X-Frame-Options', 'DENY')
    newResponse.headers.set('X-XSS-Protection', '1; mode=block')
    
    return newResponse
  }
  
  // Default response
  return fetch(request)
}
```

#### **AWS CloudFront Configuration:**
```json
{
  "Origins": {
    "Items": [
      {
        "Id": "myapp-origin",
        "DomainName": "myapp.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only"
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "myapp-origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "managed-caching-optimized",
    "Compress": true,
    "TrustedSigners": {
      "Enabled": false
    }
  },
  "CacheBehaviors": {
    "Items": [
      {
        "PathPattern": "/api/*",
        "TargetOriginId": "myapp-origin",
        "ViewerProtocolPolicy": "https-only",
        "CachePolicyId": "managed-caching-disabled",
        "TTL": 0
      },
      {
        "PathPattern": "/static/*",
        "TargetOriginId": "myapp-origin",
        "ViewerProtocolPolicy": "allow-all",
        "CachePolicyId": "managed-caching-optimized",
        "TTL": 31536000
      }
    ]
  }
}
```

---

## SSL/TLS Termination Proxy

### 🔒 **SSL Termination Architecture**

```mermaid
graph TD
    Client[👨‍💻 Client<br/>HTTPS Request] --> |SSL/TLS Encrypted| Proxy[🔒 SSL Termination Proxy<br/>Nginx/HAProxy]
    
    Proxy --> |Decrypt SSL| Internal[🔓 Internal Network<br/>HTTP Traffic]
    
    Internal --> App1[🖥️ App Server 1<br/>Port 3000]
    Internal --> App2[🖥️ App Server 2<br/>Port 3001]
    Internal --> App3[🖥️ App Server 3<br/>Port 3002]
    
    subgraph "SSL Benefits"
        Centralized[🎯 Centralized SSL Management<br/>Single point for certificates]
        Performance[⚡ Better Performance<br/>Offload SSL processing from apps]
        Security[🔒 Enhanced Security<br/>Latest SSL/TLS protocols]
        Compliance[📋 Compliance<br/>PCI DSS, HIPAA requirements]
    end
    
    style Proxy fill:#e8f5e8
    style Client fill:#e1f5fe
    style Internal fill:#f3e5f5
```

### **🔧 SSL Termination Configuration**

#### **Nginx SSL Termination:**
```nginx
# /etc/nginx/sites-available/ssl-termination
server {
    listen 80;
    server_name myapp.com www.myapp.com;
    
    # Redirect all HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;
    
    # SSL Configuration
    ssl_certificate /etc/ssl/certs/myapp.com.crt;
    ssl_certificate_key /etc/ssl/private/myapp.com.key;
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 10m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
    
    # Proxy to backend (HTTP - SSL terminated)
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
    }
}

# Backend server pool (HTTP only)
upstream backend_servers {
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;
}
```

#### **Let's Encrypt SSL Automation:**
```bash
#!/bin/bash
# ssl-setup.sh - Automated SSL certificate management

# Install Certbot
sudo apt-get update
sudo apt-get install certbot python3-certbot-nginx

# Obtain SSL certificate
sudo certbot --nginx -d myapp.com -d www.myapp.com

# Setup auto-renewal
echo "0 12 * * * /usr/bin/certbot renew --quiet" | sudo crontab -

# Test renewal
sudo certbot renew --dry-run
```

#### **Docker SSL Termination:**
```yaml
# docker-compose.yml with SSL termination
version: '3.8'

services:
  nginx-ssl:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx-ssl.conf:/etc/nginx/nginx.conf
      - ./ssl-certs:/etc/ssl/certs:ro
      - ./ssl-private:/etc/ssl/private:ro
    depends_on:
      - app1
      - app2
      - app3
    
  app1:
    image: myapp:latest
    expose:
      - "3000"
    environment:
      - NODE_ENV=production
      - TRUST_PROXY=true
    
  app2:
    image: myapp:latest
    expose:
      - "3000"
    environment:
      - NODE_ENV=production
      - TRUST_PROXY=true
    
  app3:
    image: myapp:latest
    expose:
      - "3000"
    environment:
      - NODE_ENV=production
      - TRUST_PROXY=true
```

---

## Caching Proxy

### 💾 **Caching Proxy Architecture**

```mermaid
graph TD
    Client[👨‍💻 Client Request] --> Cache[💾 Caching Proxy<br/>Nginx/Varnish/Redis]
    
    Cache --> |Cache Hit| FastResponse[⚡ Fast Response<br/>Served from cache]
    Cache --> |Cache Miss| Origin[🏠 Origin Server]
    Origin --> |Store in Cache| Cache
    Cache --> Client
    
    subgraph "Cache Types"
        Browser[🌐 Browser Cache<br/>Client-side caching]
        CDN[🌍 CDN Cache<br/>Edge-level caching]
        Reverse[🔄 Reverse Proxy Cache<br/>Server-side caching]
        Database[🗄️ Database Cache<br/>Query result caching]
    end
    
    subgraph "Cache Strategies"
        TTL[⏰ Time-based (TTL)<br/>Expire after time]
        LRU[📊 LRU (Least Recently Used)<br/>Remove old unused data]
        Manual[🎛️ Manual Invalidation<br/>Explicit cache clearing]
        Event[📢 Event-driven<br/>Invalidate on data changes]
    end
    
    style Cache fill:#e8f5e8
    style FastResponse fill:#e8f5e8
    style Origin fill:#e1f5fe
```

### **⚙️ Varnish Caching Configuration**

#### **Varnish Cache Setup:**
```vcl
# /etc/varnish/default.vcl
vcl 4.0;

# Backend servers
backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .connect_timeout = 60s;
    .first_byte_timeout = 60s;
    .between_bytes_timeout = 60s;
}

backend app1 {
    .host = "10.0.1.10";
    .port = "3000";
}

backend app2 {
    .host = "10.0.1.11";
    .port = "3000";
}

# Load balancing
sub vcl_init {
    new lb = directors.round_robin();
    lb.add_backend(app1);
    lb.add_backend(app2);
}

sub vcl_recv {
    # Use load balancer
    set req.backend_hint = lb.backend();
    
    # Don't cache requests with cookies (user-specific)
    if (req.http.Cookie) {
        return (pass);
    }
    
    # Don't cache POST requests
    if (req.method != "GET" && req.method != "HEAD") {
        return (pass);
    }
    
    # Cache static files for long time
    if (req.url ~ "\.(css|js|png|jpg|jpeg|gif|ico|svg)$") {
        unset req.http.Cookie;
        return (hash);
    }
    
    # Don't cache API requests by default
    if (req.url ~ "^/api/") {
        return (pass);
    }
}

sub vcl_backend_response {
    # Cache static files for 1 hour
    if (bereq.url ~ "\.(css|js|png|jpg|jpeg|gif|ico|svg)$") {
        set beresp.ttl = 1h;
        set beresp.http.Cache-Control = "public, max-age=3600";
    }
    
    # Cache HTML for 5 minutes
    if (beresp.http.Content-Type ~ "text/html") {
        set beresp.ttl = 5m;
    }
    
    # Don't cache errors
    if (beresp.status >= 400) {
        set beresp.ttl = 0s;
    }
}

sub vcl_deliver {
    # Add cache hit/miss header
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
    
    # Add cache age
    set resp.http.X-Cache-Age = obj.age;
}
```

#### **Redis Caching with Node.js:**
```javascript
// cache-service.js
const redis = require('redis');
const client = redis.createClient({
  host: 'localhost',
  port: 6379
});

class CacheService {
  constructor() {
    this.defaultTTL = 3600; // 1 hour
  }
  
  async get(key) {
    try {
      const data = await client.get(key);
      return data ? JSON.parse(data) : null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }
  
  async set(key, data, ttl = this.defaultTTL) {
    try {
      await client.setex(key, ttl, JSON.stringify(data));
      return true;
    } catch (error) {
      console.error('Cache set error:', error);
      return false;
    }
  }
  
  async del(key) {
    try {
      await client.del(key);
      return true;
    } catch (error) {
      console.error('Cache delete error:', error);
      return false;
    }
  }
  
  async flush() {
    try {
      await client.flushall();
      return true;
    } catch (error) {
      console.error('Cache flush error:', error);
      return false;
    }
  }
}

// Express middleware for caching
const cache = new CacheService();

const cacheMiddleware = (ttl = 3600) => {
  return async (req, res, next) => {
    const key = `cache:${req.originalUrl}`;
    
    // Try to get from cache
    const cachedData = await cache.get(key);
    if (cachedData) {
      return res.json(cachedData);
    }
    
    // Override res.json to cache response
    const originalJson = res.json;
    res.json = function(data) {
      cache.set(key, data, ttl);
      return originalJson.call(this, data);
    };
    
    next();
  };
};

// Usage in Express app
const express = require('express');
const app = express();

// Cache product list for 10 minutes
app.get('/api/products', cacheMiddleware(600), async (req, res) => {
  const products = await getProductsFromDatabase();
  res.json(products);
});

// Cache user profile for 5 minutes
app.get('/api/user/:id', cacheMiddleware(300), async (req, res) => {
  const user = await getUserFromDatabase(req.params.id);
  res.json(user);
});

module.exports = { CacheService, cacheMiddleware };
```

---

## Security Proxy (WAF)

### 🛡️ **Web Application Firewall (WAF) Architecture**

```mermaid
graph TD
    Internet[🌐 Internet Traffic] --> WAF[🛡️ Web Application Firewall<br/>CloudFlare/AWS WAF/ModSecurity]
    
    WAF --> |Filter Malicious Traffic| Blocked[🚫 Blocked Requests<br/>SQL Injection, XSS, etc.]
    WAF --> |Allow Clean Traffic| Clean[✅ Clean Requests]
    
    Clean --> RateLimit[📊 Rate Limiting]
    RateLimit --> GeoBlock[🌍 Geo-blocking]
    GeoBlock --> BotProtection[🤖 Bot Protection]
    BotProtection --> Backend[🖥️ Backend Servers]
    
    subgraph "Security Rules"
        SQLInjection[💉 SQL Injection Protection]
        XSS[🔓 XSS Prevention]
        CSRF[🔒 CSRF Protection]
        DDoS[⚡ DDoS Mitigation]
        FileUpload[📁 File Upload Scanning]
    end
    
    subgraph "Monitoring"
        Logs[📝 Security Logs]
        Alerts[🚨 Security Alerts]
        Analytics[📊 Traffic Analytics]
        Forensics[🔍 Attack Forensics]
    end
    
    style WAF fill:#e8f5e8
    style Blocked fill:#ffebee
    style Clean fill:#e8f5e8
    style Backend fill:#e1f5fe
```

### **🔧 ModSecurity WAF Configuration**

#### **ModSecurity with Nginx:**
```nginx
# /etc/nginx/nginx.conf
load_module modules/ngx_http_modsecurity_module.so;

http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
    
    server {
        listen 80;
        server_name myapp.com;
        
        # Enable ModSecurity for this location
        location / {
            modsecurity on;
            modsecurity_rules '
                SecRuleEngine On
                SecDefaultAction "phase:1,log,auditlog,pass"
                SecDefaultAction "phase:2,log,auditlog,pass"
            ';
            
            proxy_pass http://backend;
        }
        
        # API protection with stricter rules
        location /api/ {
            modsecurity on;
            modsecurity_rules '
                SecRuleEngine On
                SecRule ARGS "@detectSQLi" "id:1001,phase:2,block,msg:SQL Injection Attack"
                SecRule ARGS "@detectXSS" "id:1002,phase:2,block,msg:XSS Attack"
            ';
            
            proxy_pass http://backend;
        }
    }
}
```

#### **ModSecurity Rules:**
```apache
# /etc/nginx/modsec/main.conf
Include /etc/nginx/modsec/modsecurity.conf

# Core Rule Set (CRS)
Include /usr/share/modsecurity-crs/crs-setup.conf
Include /usr/share/modsecurity-crs/rules/*.conf

# Custom rules
SecRule REQUEST_URI "@beginsWith /admin" \
    "id:2001,phase:1,block,msg:'Admin panel access blocked',logdata:'IP: %{REMOTE_ADDR}'"

SecRule REQUEST_METHOD "@streq POST" \
    "id:2002,phase:1,chain,msg:'POST request size limit'"
SecRule REQUEST_BODY "@gt 1048576" \
    "block,msg:'Request body too large'"

# Rate limiting rule
SecAction "id:2003,phase:1,nolog,pass,setvar:ip.requests=+1,expirevar:ip.requests=60"
SecRule IP:REQUESTS "@gt 30" \
    "id:2004,phase:1,block,msg:'Rate limit exceeded: %{ip.requests} requests in 60 seconds'"

# Bot protection
SecRule REQUEST_HEADERS:User-Agent "@contains bot" \
    "id:2005,phase:1,log,msg:'Bot detected',setvar:tx.bot_score=+1"
SecRule TX:BOT_SCORE "@gt 5" \
    "id:2006,phase:2,block,msg:'Bot score too high'"
```

#### **AWS WAF Configuration:**
```json
{
  "Name": "MyWebACL",
  "Scope": "CLOUDFRONT",
  "DefaultAction": {
    "Allow": {}
  },
  "Rules": [
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {
        "None": {}
      },
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSetMetric"
      }
    },
    {
      "Name": "RateLimitRule",
      "Priority": 2,
      "Action": {
        "Block": {}
      },
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimitMetric"
      }
    },
    {
      "Name": "IPWhitelistRule",
      "Priority": 3,
      "Action": {
        "Allow": {}
      },
      "Statement": {
        "IPSetReferenceStatement": {
          "ARN": "arn:aws:wafv2:us-east-1:123456789012:global/ipset/trusted-ips"
        }
      }
    }
  ]
}
```

---

## Real-World Examples

### 🏢 **Enterprise E-commerce Platform**

```mermaid
graph TD
    subgraph "External Traffic"
        Users[👥 Users]
        Mobile[📱 Mobile Apps]
        Partners[🤝 Partners]
    end
    
    subgraph "Edge Layer"
        CDN[🌍 CloudFlare CDN<br/>Global Distribution]
        WAF[🛡️ Web Application Firewall<br/>Security Filtering]
    end
    
    subgraph "Load Balancer Layer"
        ALB[⚖️ Application Load Balancer<br/>AWS ALB/F5]
        HealthCheck[💚 Health Checks]
    end
    
    subgraph "Proxy Layer"
        Gateway[🚪 API Gateway<br/>Kong/AWS API Gateway]
        Auth[🔐 Authentication]
        RateLimit[📊 Rate Limiting]
    end
    
    subgraph "Application Layer"
        Web[🌐 Web Servers<br/>Nginx + React]
        API[🔧 API Servers<br/>Node.js/Java]
        Admin[👨‍💼 Admin Panel<br/>Internal Tools]
    end
    
    subgraph "Data Layer"
        Cache[💾 Redis Cache]
        DB[🗄️ PostgreSQL]
        Files[📁 File Storage<br/>S3/MinIO]
    end
    
    Users --> CDN
    Mobile --> CDN
    Partners --> CDN
    
    CDN --> WAF
    WAF --> ALB
    ALB --> Gateway
    
    Gateway --> Auth
    Gateway --> RateLimit
    Gateway --> Web
    Gateway --> API
    Gateway --> Admin
    
    API --> Cache
    API --> DB
    Web --> Files
    
    style CDN fill:#e8f5e8
    style WAF fill:#ffebee
    style ALB fill:#e1f5fe
    style Gateway fill:#f3e5f5
```

#### **Complete Docker Compose Setup:**
```yaml
# docker-compose.production.yml
version: '3.8'

services:
  # Load Balancer
  nginx-lb:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/ssl:ro
    depends_on:
      - api-gateway

  # API Gateway
  api-gateway:
    image: kong:latest
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    ports:
      - "8000:8000"
      - "8001:8001"
    depends_on:
      - kong-db
      - web-app
      - api-server

  # Web Application
  web-app:
    image: myapp/frontend:latest
    deploy:
      replicas: 3
    environment:
      - NODE_ENV=production
      - API_URL=http://api-server:3001
    depends_on:
      - api-server

  # API Server
  api-server:
    image: myapp/backend:latest
    deploy:
      replicas: 3
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  # Cache Layer
  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data
    deploy:
      replicas: 1

  # Database
  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # Kong Database
  kong-db:
    image: postgres:13
    environment:
      POSTGRES_DB: kong
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: kong
    volumes:
      - kong_data:/var/lib/postgresql/data

  # Monitoring
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  postgres_data:
  kong_data:
  redis_data:
  grafana_data:

networks:
  default:
    driver: bridge
```

### 🏥 **Healthcare Application with High Security**

```mermaid
graph TD
    subgraph "DMZ (Demilitarized Zone)"
        Internet[🌐 Internet] --> Firewall[🔥 Network Firewall]
        Firewall --> WAF[🛡️ WAF + DDoS Protection]
        WAF --> LB[⚖️ Load Balancer<br/>SSL Termination]
    end
    
    subgraph "Application Zone"
        LB --> Auth[🔐 Authentication Proxy<br/>SAML/OAuth2]
        Auth --> Gateway[🚪 API Gateway<br/>FHIR Compliance]
        Gateway --> WebServers[🏥 Healthcare Apps]
    end
    
    subgraph "Secure Backend"
        WebServers --> AppServers[🖥️ Application Servers<br/>HIPAA Compliant]
        AppServers --> AuditProxy[📝 Audit Logging Proxy]
        AuditProxy --> Database[🔒 Encrypted Database<br/>PHI Data]
    end
    
    subgraph "Compliance Features"
        Encryption[🔐 End-to-end Encryption]
        Audit[📋 HIPAA Audit Logs]
        Access[👨‍⚕️ Role-based Access Control]
        Backup[💾 Encrypted Backups]
    end
    
    style WAF fill:#ffebee
    style Auth fill:#e8f5e8
    style Database fill:#e1f5fe
    style AuditProxy fill:#fff3e0
```

#### **HIPAA-Compliant Proxy Configuration:**
```nginx
# /etc/nginx/sites-available/healthcare-app
server {
    listen 443 ssl http2;
    server_name healthcare.example.com;
    
    # Strong SSL configuration for HIPAA compliance
    ssl_certificate /etc/ssl/certs/healthcare.crt;
    ssl_certificate_key /etc/ssl/private/healthcare.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # HIPAA security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-Frame-Options DENY always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Audit logging for all requests
    access_log /var/log/nginx/healthcare_access.log audit_format;
    error_log /var/log/nginx/healthcare_error.log;
    
    # Authentication required for all endpoints
    location / {
        auth_request /auth;
        
        proxy_pass http://healthcare_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Audit headers
        proxy_set_header X-User-ID $user_id;
        proxy_set_header X-Session-ID $session_id;
        proxy_set_header X-Request-ID $request_id;
    }
    
    # Authentication endpoint
    location = /auth {
        internal;
        proxy_pass http://auth_service/validate;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
    }
    
    # PHI data endpoints with extra protection
    location /api/patient/ {
        auth_request /auth;
        
        # Additional rate limiting for sensitive data
        limit_req zone=patient_data burst=10 nodelay;
        
        proxy_pass http://healthcare_backend;
        proxy_set_header X-PHI-Access "true";
        
        # Enhanced audit logging
        access_log /var/log/nginx/phi_access.log phi_audit_format;
    }
}

# Custom log formats for HIPAA compliance
log_format audit_format '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       'user_id="$user_id" session_id="$session_id" '
                       'request_id="$request_id" response_time=$request_time';

log_format phi_audit_format '$remote_addr - $remote_user [$time_local] '
                           '"$request" $status $bytes_sent '
                           'user_id="$user_id" patient_id="$patient_id" '
                           'access_type="PHI" response_time=$request_time';

upstream healthcare_backend {
    server 10.0.1.10:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:3000 backup;
}

upstream auth_service {
    server 10.0.2.10:3001;
    server 10.0.2.11:3001;
}
```

---

## Conclusion

Proxies are essential components in modern web architecture, providing:

### **🎯 Key Benefits**:
1. **🔒 Security** - WAF, DDoS protection, access control
2. **⚡ Performance** - Caching, load balancing, SSL termination
3. **📊 Scalability** - Horizontal scaling, traffic distribution
4. **🔍 Observability** - Logging, monitoring, analytics
5. **🛠️ Flexibility** - A/B testing, canary deployments, API versioning

### **🏗️ Common Proxy Patterns**:
- **Forward Proxy** - Client-side anonymity and filtering
- **Reverse Proxy** - Server-side load balancing and security
- **API Gateway** - Microservices orchestration and management
- **CDN** - Global content distribution and performance
- **WAF** - Application-level security and protection

### **📋 Implementation Checklist**:
- ✅ Choose appropriate proxy type for your use case
- ✅ Configure SSL/TLS termination properly
- ✅ Implement health checks and monitoring
- ✅ Set up proper logging and alerting
- ✅ Plan for high availability and disaster recovery
- ✅ Regular security updates and maintenance

Understanding these proxy patterns and their proper implementation enables teams to build robust, scalable, and secure web applications that can handle modern traffic demands while maintaining excellent performance and security standards.
