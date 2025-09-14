# Docker Complete Guide with Flowcharts & Examples

## Overview
This comprehensive guide explains Docker terminology, concepts, and workflows using visual flowcharts and real-world examples. Docker is a containerization platform that packages applications and their dependencies into lightweight, portable containers.

## 🎯 **Quick Docker Concepts Overview**

**Docker workflow involves 5 key components:**

1. **📦 Dockerfile** → **Defines** how to build your application container
2. **🏗️ Docker Image** → **Template** for creating containers (like a blueprint)
3. **🚀 Docker Container** → **Running instance** of an image (actual application)
4. **🏪 Docker Registry** → **Storage** for sharing images (Docker Hub, private registries)
5. **🎛️ Docker Engine** → **Runtime** that manages containers on your machine

**🎯 Simple Analogy**: Think of Docker like shipping containers:
- **Dockerfile** = Instructions for packing a shipping container
- **Image** = Sealed, ready-to-ship container template
- **Container** = Container in transit/at destination (running)
- **Registry** = Container warehouse/port
- **Engine** = Shipping infrastructure (trucks, cranes, etc.)

## Table of Contents
- [Docker Architecture Flowchart](#docker-architecture-flowchart)
- [Core Docker Terms Explained](#core-docker-terms-explained)
- [Docker Development Workflow](#docker-development-workflow)
- [Real-World Project Examples](#real-world-project-examples)
- [Docker Commands Flowchart](#docker-commands-flowchart)
- [Container Lifecycle](#container-lifecycle)
- [Docker Networking & Volumes](#docker-networking--volumes)
- [Best Practices](#best-practices)

---

## Docker Architecture Flowchart

```mermaid
graph TD
    Dev[👨‍💻 Developer] --> Dockerfile[📄 Dockerfile]
    
    Dockerfile --> |docker build| Image[🏗️ Docker Image]
    Image --> |docker run| Container[🚀 Running Container]
    
    Image --> |docker push| Registry[🏪 Docker Registry<br/>Docker Hub, ECR, etc.]
    Registry --> |docker pull| Image2[🏗️ Docker Image<br/>on different machine]
    Image2 --> |docker run| Container2[🚀 Running Container<br/>Production/Testing]
    
    Container --> |docker stop| StoppedContainer[⏹️ Stopped Container]
    StoppedContainer --> |docker start| Container
    StoppedContainer --> |docker rm| Removed[🗑️ Removed]
    
    Engine[🎛️ Docker Engine] --> |manages| Container
    Engine --> |manages| Container2
    Engine --> |manages| Images[📚 Local Images]
    
    Host[💻 Host OS] --> Engine
    
    style Dockerfile fill:#e1f5fe
    style Image fill:#f3e5f5
    style Container fill:#e8f5e8
    style Registry fill:#fff3e0
    style Engine fill:#fce4ec
```

### **🔍 Architecture Components Explained**

| Component | Role | Real Example |
|-----------|------|--------------|
| **Dockerfile** | Build instructions | Recipe to install Node.js, copy code, set port |
| **Image** | Immutable template | Ubuntu + Node.js + your app code (frozen state) |
| **Container** | Running process | Your web app actually serving traffic on port 3000 |
| **Registry** | Image storage | Docker Hub storing your "myapp:v1.0" image |
| **Engine** | Container runtime | Docker daemon managing all containers on your laptop |

---

## Core Docker Terms Explained

### 📄 **Dockerfile**
**🎯 MAIN PURPOSE**: Text file with instructions to build a Docker image
**Think of it as**: Recipe or blueprint for your application environment

**Real Example - Node.js Web App**:
```dockerfile
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory inside container
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application source code
COPY . .

# Expose port that app runs on
EXPOSE 3000

# Define command to run application
CMD ["npm", "start"]
```

**What each instruction does**:
- `FROM`: Base operating system + runtime
- `WORKDIR`: Set folder to work in
- `COPY`: Transfer files from your computer to container
- `RUN`: Execute commands during build (install packages)
- `EXPOSE`: Document which port app uses
- `CMD`: Command to start your application

---

### 🏗️ **Docker Image**
**🎯 MAIN PURPOSE**: Read-only template used to create containers
**Think of it as**: Snapshot of your application + environment

**Real Example - Building an Image**:
```bash
# Build image from Dockerfile
$ docker build -t mywebapp:v1.0 .

# List local images
$ docker images
REPOSITORY    TAG    IMAGE ID       CREATED         SIZE
mywebapp      v1.0   abc123def456   5 minutes ago   150MB
node          18     def456ghi789   2 weeks ago     120MB
```

**Image Characteristics**:
- **Immutable**: Cannot be changed after creation
- **Layered**: Built in layers for efficiency
- **Portable**: Runs same way on any Docker host
- **Versioned**: Tagged with versions (v1.0, latest, etc.)

**Image Layers Example**:
```
┌─────────────────────────────┐
│ Your app code (Layer 4)     │ ← Your specific application
├─────────────────────────────┤
│ npm dependencies (Layer 3)  │ ← Node modules
├─────────────────────────────┤
│ Node.js runtime (Layer 2)   │ ← Node.js installation
├─────────────────────────────┤
│ Alpine Linux (Layer 1)      │ ← Base operating system
└─────────────────────────────┘
```

---

### 🚀 **Docker Container**
**🎯 MAIN PURPOSE**: Running instance of a Docker image
**Think of it as**: Your application actually running and serving users

**Real Example - Running a Container**:
```bash
# Run container from image
$ docker run -d -p 3000:3000 --name my-running-app mywebapp:v1.0

# Check running containers
$ docker ps
CONTAINER ID   IMAGE           COMMAND       CREATED         STATUS         PORTS                    NAMES
xyz789abc123   mywebapp:v1.0   "npm start"   2 minutes ago   Up 2 minutes   0.0.0.0:3000->3000/tcp   my-running-app
```

**Container Characteristics**:
- **Isolated**: Own filesystem, network, processes
- **Ephemeral**: Can be stopped/started/deleted easily
- **Stateless**: Data is lost when container is removed (unless using volumes)
- **Lightweight**: Shares host OS kernel

**Container vs Image**:
```
Image (Template)     →     Container (Running Instance)
─────────────────    →     ──────────────────────────────
📋 Class definition  →     🏃 Object instance
🏠 House blueprint   →     🏘️ Actual house people live in
💿 Software CD       →     💻 Installed, running program
```

---

### 🏪 **Docker Registry**
**🎯 MAIN PURPOSE**: Central repository to store and share Docker images
**Think of it as**: App store or library for Docker images

**Real Examples**:

**Docker Hub (Public Registry)**:
```bash
# Push your image to Docker Hub
$ docker tag mywebapp:v1.0 yourusername/mywebapp:v1.0
$ docker push yourusername/mywebapp:v1.0

# Pull image from Docker Hub
$ docker pull nginx:latest
$ docker pull postgres:13
$ docker pull redis:alpine
```

**Private Registry Examples**:
```bash
# AWS Elastic Container Registry (ECR)
$ docker push 123456789012.dkr.ecr.us-west-2.amazonaws.com/myapp:v1.0

# Google Container Registry (GCR)
$ docker push gcr.io/my-project/myapp:v1.0

# Private Harbor Registry
$ docker push registry.mycompany.com/myapp:v1.0
```

**Registry Benefits**:
- **Collaboration**: Team shares same images
- **Deployment**: Production pulls exact same image
- **Versioning**: Multiple versions stored and tagged
- **Security**: Private registries for proprietary code

---

### 🎛️ **Docker Engine**
**🎯 MAIN PURPOSE**: Runtime that manages Docker containers on host system
**Think of it as**: Operating system for containers

**Engine Components**:
```
┌─────────────────────────────────────┐
│           Docker Client             │ ← Commands you type (docker run, etc.)
├─────────────────────────────────────┤
│           Docker API                │ ← Communication layer
├─────────────────────────────────────┤
│         Docker Daemon              │ ← Main engine process
├─────────────────────────────────────┤
│        containerd                  │ ← Container runtime
├─────────────────────────────────────┤
│           runc                     │ ← Low-level container execution
├─────────────────────────────────────┤
│          Host OS Kernel            │ ← Linux/Windows kernel
└─────────────────────────────────────┘
```

**What Docker Engine Does**:
- **Image Management**: Build, store, delete images
- **Container Lifecycle**: Create, start, stop, monitor containers
- **Resource Management**: CPU, memory, network allocation
- **Security**: Container isolation and permissions
- **Networking**: Container-to-container communication
- **Storage**: Volume and bind mount management

---

## Docker Development Workflow

### 🔄 **Complete Development Lifecycle**

```mermaid
flowchart TD
    Start([🚀 Start Development]) --> Code[💻 Write Code]
    Code --> Dockerfile[📄 Create/Update Dockerfile]
    
    Dockerfile --> Build[🏗️ docker build -t app:dev .]
    Build --> Test[🧪 docker run app:dev]
    
    Test --> Works{✅ Works?}
    Works -->|No| Debug[🐛 Debug & Fix]
    Debug --> Code
    
    Works -->|Yes| Tag[🏷️ docker tag app:dev app:v1.0]
    Tag --> Push[📤 docker push app:v1.0]
    
    Push --> Deploy[🚀 Deploy to Production]
    Deploy --> Prod[🏭 docker run app:v1.0]
    
    Prod --> Monitor[📊 Monitor & Logs]
    Monitor --> Update{🔄 Need Updates?}
    Update -->|Yes| Code
    Update -->|No| Monitor
    
    style Start fill:#e8f5e8
    style Deploy fill:#fff3e0
    style Prod fill:#f3e5f5
    style Works fill:#e1f5fe
```

### **📋 Step-by-Step Workflow Example**

**Scenario**: Building a React web application

#### **Step 1: Development Setup**
```bash
# 1. Create project structure
mkdir my-react-app
cd my-react-app
npm create react-app .

# 2. Create Dockerfile
touch Dockerfile
```

#### **Step 2: Create Dockerfile**
```dockerfile
# Dockerfile for React App
FROM node:18-alpine

WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm install

# Copy source code
COPY . .

# Build the app
RUN npm run build

# Install serve to run the built app
RUN npm install -g serve

# Expose port
EXPOSE 3000

# Start the application
CMD ["serve", "-s", "build", "-l", "3000"]
```

#### **Step 3: Build and Test**
```bash
# Build the image
$ docker build -t my-react-app:dev .

# Run for testing
$ docker run -d -p 3000:3000 --name react-test my-react-app:dev

# Test the application
$ curl http://localhost:3000
# Or open browser to http://localhost:3000

# Check logs
$ docker logs react-test

# Stop test container
$ docker stop react-test && docker rm react-test
```

#### **Step 4: Production Deployment**
```bash
# Tag for production
$ docker tag my-react-app:dev my-react-app:v1.0

# Push to registry
$ docker push yourusername/my-react-app:v1.0

# Deploy on production server
$ docker run -d -p 80:3000 --name production-app my-react-app:v1.0
```

---

## Real-World Project Examples

### 🌐 **Example 1: Full-Stack Web Application**

**Project**: E-commerce website with React frontend, Node.js API, and PostgreSQL database

#### **Project Structure**:
```
ecommerce-app/
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── database/
│   └── init.sql
└── docker-compose.yml
```

#### **Frontend Dockerfile** (React):
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "build", "-l", "3000"]
```

#### **Backend Dockerfile** (Node.js API):
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 5000
CMD ["npm", "start"]
```

#### **Docker Compose** (Orchestrate all services):
```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://user:password@database:5432/ecommerce
    depends_on:
      - database

  database:
    image: postgres:13
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=ecommerce
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

#### **How to Run**:
```bash
# Start entire application stack
$ docker-compose up -d

# View running services
$ docker-compose ps

# View logs
$ docker-compose logs frontend
$ docker-compose logs backend

# Stop everything
$ docker-compose down
```

---

### 🔬 **Example 2: Microservices Architecture**

**Project**: Online store with separate services for users, products, orders, and payments

```mermaid
graph TD
    Client[👨‍💻 Client/Browser] --> Gateway[🚪 API Gateway<br/>nginx:alpine]
    
    Gateway --> Users[👤 User Service<br/>node:18-alpine]
    Gateway --> Products[📦 Product Service<br/>python:3.9-slim]
    Gateway --> Orders[📋 Order Service<br/>node:18-alpine]
    Gateway --> Payments[💳 Payment Service<br/>java:11-jre-slim]
    
    Users --> UserDB[(👤 User Database<br/>postgres:13)]
    Products --> ProductDB[(📦 Product Database<br/>mongodb:5)]
    Orders --> OrderDB[(📋 Order Database<br/>postgres:13)]
    Payments --> PaymentDB[(💳 Payment Database<br/>redis:alpine)]
    
    style Client fill:#e8f5e8
    style Gateway fill:#fff3e0
    style Users fill:#e1f5fe
    style Products fill:#f3e5f5
    style Orders fill:#fce4ec
    style Payments fill:#e8f5e8
```

#### **Service Structure**:
```
microservices-store/
├── api-gateway/
│   ├── Dockerfile
│   └── nginx.conf
├── user-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── product-service/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
├── order-service/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── payment-service/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
└── docker-compose.yml
```

#### **User Service Dockerfile** (Node.js):
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3001
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3001/health || exit 1

CMD ["npm", "start"]
```

#### **Product Service Dockerfile** (Python):
```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 3002
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3002/health || exit 1

CMD ["python", "app.py"]
```

#### **Docker Compose for Microservices**:
```yaml
version: '3.8'

services:
  api-gateway:
    build: ./api-gateway
    ports:
      - "80:80"
    depends_on:
      - user-service
      - product-service
      - order-service
      - payment-service

  user-service:
    build: ./user-service
    environment:
      - DATABASE_URL=postgresql://user:password@user-db:5432/users
    depends_on:
      - user-db
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  product-service:
    build: ./product-service
    environment:
      - MONGO_URL=mongodb://product-db:27017/products
    depends_on:
      - product-db

  order-service:
    build: ./order-service
    environment:
      - DATABASE_URL=postgresql://user:password@order-db:5432/orders
    depends_on:
      - order-db

  payment-service:
    build: ./payment-service
    environment:
      - REDIS_URL=redis://payment-cache:6379
    depends_on:
      - payment-cache

  # Databases
  user-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=users
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - user_data:/var/lib/postgresql/data

  product-db:
    image: mongodb:5
    volumes:
      - product_data:/data/db

  order-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=orders
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - order_data:/var/lib/postgresql/data

  payment-cache:
    image: redis:alpine
    volumes:
      - payment_data:/data

volumes:
  user_data:
  product_data:
  order_data:
  payment_data:
```

---

### 🧪 **Example 3: Development Environment**

**Project**: Complete development environment for a team working on a Django application

#### **Development Docker Setup**:
```dockerfile
# Dockerfile.dev for Django development
FROM python:3.9

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt

# Copy project
COPY . .

# Expose port
EXPOSE 8000

# Development command with auto-reload
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

#### **Development Docker Compose**:
```yaml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app                    # Mount source code for live editing
      - /app/node_modules         # Preserve node_modules
    ports:
      - "8000:8000"
    environment:
      - DEBUG=1
      - DATABASE_URL=postgresql://dev:devpass@db:5432/devdb
    depends_on:
      - db
      - redis
    command: python manage.py runserver 0.0.0.0:8000

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=devdb
      - POSTGRES_USER=dev
      - POSTGRES_PASSWORD=devpass
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"              # Expose for DB tools

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"              # Expose for Redis tools

  mailhog:                       # Email testing
    image: mailhog/mailhog
    ports:
      - "1025:1025"              # SMTP
      - "8025:8025"              # Web interface

volumes:
  postgres_dev_data:
```

#### **Developer Workflow**:
```bash
# Start development environment
$ docker-compose up -d

# View logs in real-time
$ docker-compose logs -f web

# Run migrations
$ docker-compose exec web python manage.py migrate

# Create superuser
$ docker-compose exec web python manage.py createsuperuser

# Run tests
$ docker-compose exec web python manage.py test

# Install new Python package
$ docker-compose exec web pip install new-package
$ docker-compose exec web pip freeze > requirements-dev.txt

# Access database directly
$ docker-compose exec db psql -U dev -d devdb

# Stop environment
$ docker-compose down
```

---

## Docker Commands Flowchart

### 🔧 **Essential Docker Commands Workflow**

```mermaid
flowchart TD
    Start([🚀 Start Docker Project]) --> CheckDocker{🔍 Docker installed?}
    
    CheckDocker -->|No| Install[📥 Install Docker Desktop]
    Install --> CheckDocker
    
    CheckDocker -->|Yes| CreateDockerfile[📄 Create Dockerfile]
    
    CreateDockerfile --> Build[🏗️ docker build -t name:tag .]
    Build --> BuildSuccess{✅ Build successful?}
    
    BuildSuccess -->|No| FixDockerfile[🔧 Fix Dockerfile errors]
    FixDockerfile --> Build
    
    BuildSuccess -->|Yes| Run[🚀 docker run -d -p 8080:80 name:tag]
    Run --> RunSuccess{✅ Container running?}
    
    RunSuccess -->|No| Debug[🐛 Check logs: docker logs container]
    Debug --> FixCode[🔧 Fix application code]
    FixCode --> Build
    
    RunSuccess -->|Yes| Test[🧪 Test application]
    Test --> Works{✅ Working correctly?}
    
    Works -->|No| Stop[⏹️ docker stop container]
    Stop --> Remove[🗑️ docker rm container]
    Remove --> FixCode
    
    Works -->|Yes| Tag[🏷️ docker tag name:tag name:v1.0]
    Tag --> Push[📤 docker push name:v1.0]
    
    Push --> Deploy[🚀 Deploy to production]
    
    style Start fill:#e8f5e8
    style Deploy fill:#fff3e0
    style BuildSuccess fill:#e1f5fe
    style Works fill:#f3e5f5
```

### **📋 Command Categories**

#### **🏗️ Image Management**
```bash
# Build image from Dockerfile
$ docker build -t myapp:v1.0 .

# List local images
$ docker images

# Remove image
$ docker rmi myapp:v1.0

# Pull image from registry
$ docker pull nginx:latest

# Push image to registry
$ docker push myapp:v1.0

# Tag image
$ docker tag myapp:v1.0 myapp:latest

# View image history/layers
$ docker history myapp:v1.0

# Inspect image details
$ docker inspect myapp:v1.0
```

#### **🚀 Container Management**
```bash
# Run container
$ docker run -d -p 8080:80 --name webapp nginx

# List running containers
$ docker ps

# List all containers (including stopped)
$ docker ps -a

# Stop container
$ docker stop webapp

# Start stopped container
$ docker start webapp

# Restart container
$ docker restart webapp

# Remove container
$ docker rm webapp

# Remove running container (force)
$ docker rm -f webapp

# Execute command in running container
$ docker exec -it webapp bash
```

#### **📊 Monitoring & Debugging**
```bash
# View container logs
$ docker logs webapp

# Follow logs in real-time
$ docker logs -f webapp

# View container resource usage
$ docker stats

# Inspect container details
$ docker inspect webapp

# View container processes
$ docker top webapp

# Copy files from container to host
$ docker cp webapp:/app/file.txt ./file.txt

# Copy files from host to container
$ docker cp ./file.txt webapp:/app/file.txt
```

#### **🧹 Cleanup Commands**
```bash
# Remove all stopped containers
$ docker container prune

# Remove all unused images
$ docker image prune

# Remove all unused volumes
$ docker volume prune

# Remove all unused networks
$ docker network prune

# Remove everything (careful!)
$ docker system prune -a

# View disk usage
$ docker system df
```

---

## Container Lifecycle

### 🔄 **Container States and Transitions**

```mermaid
stateDiagram-v2
    [*] --> Created : docker create
    Created --> Running : docker start
    [*] --> Running : docker run
    
    Running --> Paused : docker pause
    Paused --> Running : docker unpause
    
    Running --> Stopped : docker stop
    Stopped --> Running : docker start
    
    Running --> Killed : docker kill
    Killed --> [*] : docker rm
    
    Stopped --> [*] : docker rm
    Created --> [*] : docker rm
    
    note right of Running
        Container is actively
        executing processes
    end note
    
    note right of Stopped
        Container has finished
        but still exists
    end note
    
    note right of Created
        Container exists but
        never started
    end note
```

### **📊 Container State Examples**

#### **Creating and Managing Container States**:
```bash
# Create container without starting
$ docker create --name webapp -p 8080:80 nginx
# State: Created

# Start the created container
$ docker start webapp
# State: Running

# Pause all processes in container
$ docker pause webapp
# State: Paused

# Resume paused container
$ docker unpause webapp
# State: Running

# Gracefully stop container (sends SIGTERM)
$ docker stop webapp
# State: Stopped

# Force kill container (sends SIGKILL)
$ docker kill webapp
# State: Killed

# Remove container
$ docker rm webapp
# State: Removed (no longer exists)
```

#### **Real Example - Web Server Lifecycle**:
```bash
# 1. Start a web server
$ docker run -d --name my-nginx -p 8080:80 nginx
2f5b8c7a1e4d

# 2. Check it's running
$ curl http://localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...

# 3. View container status
$ docker ps
CONTAINER ID   IMAGE   COMMAND                  CREATED          STATUS          PORTS                  NAMES
2f5b8c7a1e4d   nginx   "/docker-entrypoint.…"   30 seconds ago   Up 29 seconds   0.0.0.0:8080->80/tcp   my-nginx

# 4. Stop the server gracefully
$ docker stop my-nginx
my-nginx

# 5. Container still exists but stopped
$ docker ps -a
CONTAINER ID   IMAGE   COMMAND                  CREATED         STATUS                     PORTS   NAMES
2f5b8c7a1e4d   nginx   "/docker-entrypoint.…"   2 minutes ago   Exited (0) 30 seconds ago          my-nginx

# 6. Restart the same container
$ docker start my-nginx
my-nginx

# 7. Remove when done
$ docker stop my-nginx && docker rm my-nginx
```

---

## Docker Networking & Volumes

### 🌐 **Docker Networking Types**

```mermaid
graph TD
    Host[💻 Host Machine] --> Bridge[🌉 Bridge Network<br/>Default]
    Host --> HostNet[🔗 Host Network<br/>Direct host access]
    Host --> None[🚫 None Network<br/>No networking]
    Host --> Custom[⚙️ Custom Network<br/>User-defined]
    
    Bridge --> Container1[📦 Container A<br/>172.17.0.2]
    Bridge --> Container2[📦 Container B<br/>172.17.0.3]
    
    Custom --> WebApp[🌐 Web App<br/>web-network]
    Custom --> Database[🗄️ Database<br/>web-network]
    
    Container1 <-.-> Container2
    WebApp <-.-> Database
    
    style Bridge fill:#e1f5fe
    style Custom fill:#e8f5e8
    style HostNet fill:#fff3e0
    style None fill:#ffebee
```

#### **Network Examples**:

**Bridge Network (Default)**:
```bash
# Containers can communicate by IP
$ docker run -d --name web nginx
$ docker run -d --name app node:alpine

# Find container IPs
$ docker inspect web | grep IPAddress
"IPAddress": "172.17.0.2"

$ docker inspect app | grep IPAddress  
"IPAddress": "172.17.0.3"

# App can reach web at 172.17.0.2
```

**Custom Network (Recommended)**:
```bash
# Create custom network
$ docker network create my-app-network

# Run containers on custom network
$ docker run -d --name web --network my-app-network nginx
$ docker run -d --name app --network my-app-network node:alpine

# Containers can communicate by name
$ docker exec app ping web  # Works! DNS resolution by name
```

**Real Example - Multi-tier Application**:
```yaml
# docker-compose.yml with custom networks
version: '3.8'

services:
  frontend:
    image: nginx
    networks:
      - web-tier
    ports:
      - "80:80"

  backend:
    image: node:alpine
    networks:
      - web-tier
      - data-tier

  database:
    image: postgres
    networks:
      - data-tier
    environment:
      POSTGRES_PASSWORD: secret

networks:
  web-tier:
    driver: bridge
  data-tier:
    driver: bridge
    internal: true  # No external access
```

### 💾 **Docker Volumes & Data Persistence**

```mermaid
graph TD
    Container[📦 Container] --> VolTypes{Volume Types}
    
    VolTypes --> Named[📁 Named Volume<br/>Docker managed]
    VolTypes --> Bind[🔗 Bind Mount<br/>Host directory]
    VolTypes --> Tmpfs[💾 tmpfs Mount<br/>Memory only]
    
    Named --> VolStorage[🗄️ /var/lib/docker/volumes/]
    Bind --> HostDir[💻 Host Directory<br/>/home/user/data]
    Tmpfs --> Memory[🧠 RAM Storage<br/>Non-persistent]
    
    Database[🗄️ Database Container] --> Named
    Development[💻 Dev Container] --> Bind
    Cache[⚡ Cache Container] --> Tmpfs
    
    style Named fill:#e8f5e8
    style Bind fill:#e1f5fe
    style Tmpfs fill:#fff3e0
```

#### **Volume Examples**:

**Named Volume (Database Data)**:
```bash
# Create named volume
$ docker volume create postgres-data

# Use volume with database
$ docker run -d \
  --name postgres-db \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:13

# Data persists even if container is removed
$ docker rm -f postgres-db
$ docker run -d \
  --name postgres-db-new \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:13
# Same data is available!
```

**Bind Mount (Development)**:
```bash
# Mount current directory for live editing
$ docker run -d \
  --name dev-app \
  -v $(pwd):/app \
  -p 3000:3000 \
  node:alpine \
  sh -c "cd /app && npm start"

# Changes to local files immediately reflect in container
```

**Real Example - WordPress with Database**:
```yaml
# docker-compose.yml with volumes
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: secret
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html

  db:
    image: mysql:5.7
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - db_data:/var/lib/mysql

volumes:
  wordpress_data:
  db_data:
```

---

## Best Practices

### 🏗️ **Dockerfile Best Practices**

#### **✅ Optimized Dockerfile Example**:
```dockerfile
# Use specific version tags
FROM node:18.17-alpine3.18

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Set working directory
WORKDIR /app

# Copy package files first (for better caching)
COPY package*.json ./

# Install dependencies as root
RUN npm ci --only=production && \
    npm cache clean --force

# Copy source code
COPY --chown=nextjs:nodejs . .

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
```

#### **🚫 Common Dockerfile Mistakes**:
```dockerfile
# ❌ BAD: Using latest tag
FROM node:latest

# ✅ GOOD: Using specific version
FROM node:18.17-alpine3.18

# ❌ BAD: Running as root
USER root

# ✅ GOOD: Using non-root user
USER nodejs

# ❌ BAD: Installing unnecessary packages
RUN apt-get update && apt-get install -y \
    curl vim nano git python3 build-essential

# ✅ GOOD: Minimal install
RUN apt-get update && apt-get install -y \
    curl && \
    rm -rf /var/lib/apt/lists/*

# ❌ BAD: Copying everything first
COPY . .
RUN npm install

# ✅ GOOD: Copy package.json first for better caching
COPY package*.json ./
RUN npm install
COPY . .
```

### 🔒 **Security Best Practices**

#### **Container Security Checklist**:
```bash
# 1. Use non-root user
RUN useradd -r -s /bin/false myapp
USER myapp

# 2. Use minimal base images
FROM alpine:3.18
FROM distroless/nodejs18-debian11

# 3. Scan images for vulnerabilities
$ docker scout cves myapp:latest

# 4. Use secrets management
$ docker run -d \
  --secret my-secret \
  myapp:latest

# 5. Limit container resources
$ docker run -d \
  --memory=512m \
  --cpus="1.5" \
  --pids-limit 100 \
  myapp:latest

# 6. Use read-only filesystem
$ docker run -d \
  --read-only \
  --tmpfs /tmp \
  myapp:latest

# 7. Drop unnecessary capabilities
$ docker run -d \
  --cap-drop=ALL \
  --cap-add=NET_ADMIN \
  myapp:latest
```

### 📊 **Performance Best Practices**

#### **Multi-stage Build Example**:
```dockerfile
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production

WORKDIR /app

# Copy only production dependencies and built app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

USER node
EXPOSE 3000
CMD ["npm", "start"]
```

#### **Resource Optimization**:
```yaml
# docker-compose.yml with resource limits
version: '3.8'

services:
  web:
    image: myapp:latest
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### 🏷️ **Image Tagging Strategy**:
```bash
# Semantic versioning
myapp:1.0.0          # Specific version
myapp:1.0            # Minor version
myapp:1              # Major version
myapp:latest         # Latest stable

# Environment tags
myapp:dev            # Development
myapp:staging        # Staging
myapp:prod           # Production

# Git-based tags
myapp:main           # Main branch
myapp:feature-auth   # Feature branch
myapp:commit-abc123  # Specific commit
```

### 📝 **Documentation Template**:
```markdown
# MyApp Docker Image

## Quick Start
```bash
docker run -d -p 3000:3000 myapp:latest
```

## Environment Variables
- `PORT`: Application port (default: 3000)
- `DATABASE_URL`: Database connection string
- `NODE_ENV`: Environment (development/production)

## Volumes
- `/app/uploads`: User uploaded files
- `/app/logs`: Application logs

## Health Check
- Endpoint: `GET /health`
- Healthy response: `200 OK`

## Example Docker Compose
```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    volumes:
      - ./uploads:/app/uploads
```
```

---

## Conclusion

Docker transforms how we develop, deploy, and manage applications by providing:

1. **🏗️ Consistency**: Same environment everywhere (dev, test, prod)
2. **📦 Portability**: Run anywhere Docker is installed  
3. **⚡ Efficiency**: Lightweight containers vs heavy VMs
4. **🔧 Scalability**: Easy horizontal scaling and orchestration
5. **🔒 Isolation**: Secure application boundaries
6. **🚀 DevOps**: Streamlined CI/CD pipelines

### **🎯 Key Takeaways**:
- **Dockerfile** = Instructions to build your app environment
- **Image** = Immutable template of your application
- **Container** = Running instance serving real traffic
- **Registry** = Central storage for sharing images
- **Engine** = Runtime managing everything locally

Understanding these core concepts and following best practices enables teams to leverage Docker's full potential for modern application development and deployment.
