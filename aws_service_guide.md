# AWS Services — Complete Guide
## From Zero to Production: Every AWS Service Explained Through Building a Real App

> We'll build **ShopEase** — an e-commerce platform — layer by layer.
> Every AWS service gets introduced exactly when and why you'd need it.

---

## Table of Contents

1. [What is AWS & Why Use It](#1-what-is-aws--why-use-it)
2. [AWS Global Infrastructure — Regions, AZs, Edge](#2-aws-global-infrastructure)
3. [The App We're Building](#3-the-app-were-building)
4. [Layer 1 — Network Foundation (VPC)](#4-layer-1--network-foundation-vpc)
5. [Layer 2 — Compute (EC2, Lambda, ECS, EKS)](#5-layer-2--compute)
6. [Layer 3 — Storage (S3, EBS, EFS)](#6-layer-3--storage)
7. [Layer 4 — Databases (RDS, Aurora, DynamoDB, ElastiCache, Redshift)](#7-layer-4--databases)
8. [Layer 5 — Traffic & Access (Route 53, CloudFront, ALB, API Gateway)](#8-layer-5--traffic--access)
9. [Layer 6 — Security (IAM, Cognito, Secrets Manager, WAF, KMS)](#9-layer-6--security)
10. [Layer 7 — Messaging & Events (SQS, SNS, EventBridge, Kinesis)](#10-layer-7--messaging--events)
11. [Layer 8 — Monitoring (CloudWatch, X-Ray, CloudTrail)](#11-layer-8--monitoring)
12. [Layer 9 — CI/CD (ECR, CodePipeline, CodeBuild)](#12-layer-9--cicd)
13. [Complete Architecture — Everything Together](#13-complete-architecture)
14. [Cost & Pricing Models](#14-cost--pricing-models)
15. [Quick Reference — All Services at a Glance](#15-quick-reference)

---

## 1. What is AWS & Why Use It

**AWS (Amazon Web Services)** is a cloud platform — instead of buying and managing physical servers, you rent computing resources from Amazon and pay only for what you use.

**Before AWS (Old way):**
```
ShopEase team wants to launch:
  → Buy 10 physical servers ($20,000)
  → Rent a data center ($5,000/month)
  → Hire a sysadmin team ($200,000/year)
  → Wait 6 weeks for hardware delivery
  → Traffic spike crashes everything (no easy scaling)
  → Hardware failure = data loss

Total to get started: ~$250,000 + 6 weeks
```

**After AWS (Cloud way):**
```
ShopEase team wants to launch:
  → Create AWS account (10 minutes)
  → Launch a server with a few clicks (5 minutes)
  → Pay $50/month to start
  → Traffic spike? Auto-scale in 2 minutes
  → Hardware failure? AWS handles it automatically

Total to get started: $50/month, live in 1 day
```

**Core benefits:**
```
PAY AS YOU GO   → No upfront cost. Pay per hour/request/GB
SCALE INSTANTLY → 1 server to 1,000 servers in minutes
GLOBAL          → 34 regions worldwide, serve users locally
RELIABLE        → 99.99% uptime SLA on most services
MANAGED         → AWS handles hardware, OS patches, backups
```

---

## 2. AWS Global Infrastructure

Before touching any service, understand WHERE your resources live.

### Regions

A **Region** is a geographic area with multiple data centers. When you create anything on AWS, you choose which region it lives in.

```
AWS REGIONS (selected, 2025):
───────────────────────────────────────────────────────────
us-east-1       → Northern Virginia (most services launch here first)
us-west-2       → Oregon
eu-west-1       → Ireland
ap-south-1      → Mumbai (use this for India-facing apps)
ap-southeast-1  → Singapore
ap-northeast-1  → Tokyo
sa-east-1       → São Paulo
me-south-1      → Bahrain
───────────────────────────────────────────────────────────

RULE: Choose the region CLOSEST to your users for lowest latency.
ShopEase (Indian users) → ap-south-1 (Mumbai)
```

### Availability Zones (AZs)

Each region has **2–6 Availability Zones** — physically separate data centers within the same region, connected by high-speed fiber.

```
ap-south-1 (Mumbai Region)
├── ap-south-1a  (Data center building A)
├── ap-south-1b  (Data center building B)
└── ap-south-1c  (Data center building C)

WHY AZs MATTER:
If one data center floods or has a power failure:
  ❌ Single AZ: Your entire app goes down
  ✅ Multi-AZ:  App keeps running from other AZs

ShopEase runs in ALL 3 AZs → if one fails, users notice nothing
```

### Edge Locations

**Edge Locations** (400+ worldwide) are smaller AWS outposts closer to end users — used by CloudFront CDN to serve content faster.

```
Mumbai user accessing ShopEase:
  Without edge location: Mumbai → us-east-1 Virginia → back (200ms)
  With edge location:    Mumbai → Mumbai edge location → back (5ms)
```

---

## 3. The App We're Building

**ShopEase** — an e-commerce platform for 1 million users.

**Features:**
- User registration / login
- Product catalog with search
- Shopping cart and orders
- Image storage for products
- Email/SMS notifications
- Admin dashboard with analytics
- Mobile app + web app

We'll add AWS services one by one as we need them.

---

## 4. Layer 1 — Network Foundation (VPC)

**The very first thing before any server, database, or anything: set up your private network.**

### VPC — Virtual Private Cloud

**What it is**: Your own private, isolated section of the AWS cloud. Think of it as buying a plot of land — you decide the layout before building anything.

**Without VPC:**
```
All your servers, databases, and services are exposed to the public internet.
Anyone could try to connect to your database directly.
→ MASSIVE security risk. Never do this.
```

**ShopEase VPC Setup:**
```
VPC: 10.0.0.0/16  (gives you 65,536 private IP addresses)
Region: ap-south-1 (Mumbai)

SUBNETS — divide your VPC into sections:
────────────────────────────────────────────────────────────────

PUBLIC SUBNETS (internet-accessible):
  10.0.1.0/24  → AZ-a  (Load balancer, NAT Gateway live here)
  10.0.2.0/24  → AZ-b
  10.0.3.0/24  → AZ-c

PRIVATE SUBNETS (not directly internet-accessible):
  10.0.11.0/24 → AZ-a  (Your EC2 app servers live here)
  10.0.12.0/24 → AZ-b
  10.0.13.0/24 → AZ-c

DATABASE SUBNETS (most locked-down):
  10.0.21.0/24 → AZ-a  (RDS, ElastiCache live here)
  10.0.22.0/24 → AZ-b
  10.0.23.0/24 → AZ-c

ANALOGY:
  VPC = Your office building
  Public subnet  = Reception area (anyone can walk in)
  Private subnet = Employee workspace (only staff allowed)
  DB subnet      = Server room (only specific staff, key card only)
```

### Internet Gateway (IGW)

**What it is**: The door that connects your VPC to the public internet.

```
Internet  ←──── Internet Gateway ────→  VPC Public Subnet
                 (the front door)
```

### NAT Gateway

**What it is**: Allows servers in private subnets to access the internet (for updates, API calls) WITHOUT being reachable FROM the internet.

```
Private Subnet (App Server)
  → Wants to call Razorpay payment API (needs internet)
  → Can't go directly (it's private)
  → Goes through NAT Gateway in public subnet
  → NAT Gateway → Internet → Razorpay
  → Response comes back through NAT Gateway
  → Razorpay never knows the server's private IP

COST: ~$32/month per NAT Gateway + data transfer charges
```

### Security Groups

**What it is**: Firewall rules at the instance level — what traffic is allowed IN and OUT.

```
SHOPEASE SECURITY GROUPS:
────────────────────────────────────────────────────────────

WEB-SG (for Load Balancer):
  INBOUND:  Port 80 (HTTP) from 0.0.0.0/0 (anyone)
            Port 443 (HTTPS) from 0.0.0.0/0 (anyone)
  OUTBOUND: All traffic allowed

APP-SG (for EC2 app servers):
  INBOUND:  Port 8080 from WEB-SG only (only LB can talk to app)
  OUTBOUND: All traffic allowed

DB-SG (for RDS database):
  INBOUND:  Port 5432 (Postgres) from APP-SG only
            (ONLY app servers can reach the DB, not the internet)
  OUTBOUND: All traffic allowed
```

### Route Tables

**What it is**: Rules that determine where network traffic is directed.

```
Public Subnet Route Table:
  10.0.0.0/16 → local (traffic within VPC stays local)
  0.0.0.0/0   → Internet Gateway (everything else → internet)

Private Subnet Route Table:
  10.0.0.0/16 → local
  0.0.0.0/0   → NAT Gateway (internet access via NAT, not direct)

DB Subnet Route Table:
  10.0.0.0/16 → local
  (no 0.0.0.0/0 route → completely isolated from internet)
```

**VPC Architecture Visual:**
```
┌─────────────────────────────────────────────────────────────┐
│                    ShopEase VPC (Mumbai)                    │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  PUBLIC SUBNETS                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐      │   │
│  │  │  AZ-a      │  │  AZ-b      │  │  AZ-c      │      │   │
│  │  │  Load Bal  │  │  Load Bal  │  │  Load Bal  │      │   │
│  │  │  NAT GW    │  │  NAT GW    │  │            │      │   │
│  │  └────────────┘  └────────────┘  └────────────┘      │   │
│  └──────────────────────────────────────────────────────┘   │
│                           │ ↕                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 PRIVATE SUBNETS                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐      │   │
│  │  │  AZ-a      │  │  AZ-b      │  │  AZ-c      │      │   │
│  │  │  EC2 apps  │  │  EC2 apps  │  │  EC2 apps  │      │   │
│  │  └────────────┘  └────────────┘  └────────────┘      │   │
│  └──────────────────────────────────────────────────────┘   │
│                           │ ↕                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 DATABASE SUBNETS                     │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐      │   │
│  │  │  AZ-a      │  │  AZ-b      │  │  AZ-c      │      │   │
│  │  │  RDS       │  │  RDS       │  │            │      │   │
│  │  │  ElastiC.  │  │  (standby) │  │            │      │   │
│  │  └────────────┘  └────────────┘  └────────────┘      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
           ↕ Internet Gateway
         Internet
```

---

## 5. Layer 2 — Compute

**Compute = where your actual code runs.**

### EC2 — Elastic Compute Cloud

**What it is**: Virtual servers in the cloud. The most fundamental AWS service — a machine you fully control.

**ShopEase use**: Run the Node.js/Python API server.

```
EC2 INSTANCE TYPES:
────────────────────────────────────────────────────────────────
t3.micro   → 1 vCPU, 1GB RAM   → $8/month   → Dev/testing
t3.medium  → 2 vCPU, 4GB RAM   → $30/month  → Small apps
t3.large   → 2 vCPU, 8GB RAM   → $60/month  → Medium apps
m5.xlarge  → 4 vCPU, 16GB RAM  → $140/month → Production app servers
c5.xlarge  → 4 vCPU, 8GB RAM   → $125/month → CPU-heavy (computation)
r5.xlarge  → 4 vCPU, 32GB RAM  → $185/month → Memory-heavy (caching)
p3.2xlarge → 8 vCPU, 61GB RAM  → $2,200/mo  → GPU (ML training)

NAMING CONVENTION:
  t = "T-type" burstable (good for variable load)
  m = "M-type" general purpose
  c = "C-type" compute optimized
  r = "R-type" memory optimized
  p = "P-type" GPU instances
  
  Number = generation (higher = newer, cheaper, better)
  .micro / .small / .medium / .large / .xlarge / .2xlarge = size
```

**EC2 for ShopEase:**
```
LAUNCH TEMPLATE (blueprint for EC2):
  AMI: Amazon Linux 2023 (the OS image)
  Instance type: t3.large (2 CPU, 8GB)
  Key pair: shopease-keypair.pem (for SSH access)
  Security group: APP-SG
  Subnet: private-subnet-a
  User data script: (runs on first boot)
    #!/bin/bash
    yum update -y
    npm install -g pm2
    cd /app
    npm install
    pm2 start index.js

AUTO SCALING GROUP:
  Min instances: 2  (always have 2 running)
  Desired: 3
  Max instances: 20 (scale up to 20 during Diwali sale)
  Scale up:   when CPU > 70% for 5 minutes → add 2 instances
  Scale down: when CPU < 30% for 15 minutes → remove 1 instance
```

**EC2 Pricing Models:**
```
ON-DEMAND:
  Pay per hour, no commitment
  $0.10/hour for t3.large = $72/month
  Use for: unpredictable workloads, testing

RESERVED INSTANCES:
  Commit to 1 or 3 years → save 30-60%
  t3.large 1-year: $43/month (40% savings)
  Use for: steady production workloads

SPOT INSTANCES:
  Bid on unused AWS capacity → 70-90% cheaper
  t3.large spot: ~$0.025/hour = $18/month
  WARNING: AWS can terminate with 2-minute notice
  Use for: batch jobs, video rendering, ML training (fault-tolerant work)

SAVINGS PLANS:
  Flexible version of Reserved → commit to spending $X/hour
  Works across instance types and regions
```

### Auto Scaling Group (ASG)

**What it is**: Automatically adds or removes EC2 instances based on load.

```
NORMAL TRAFFIC (Tuesday 2pm):
  3 EC2 instances running → handles load fine

DIWALI SALE (traffic 10x):
  ASG detects: CPU > 70% on all instances
  → Launches 7 more instances (total: 10)
  → All instances register with Load Balancer automatically
  → Users never notice a slowdown

AFTER SALE (traffic drops):
  ASG detects: CPU < 30% for 15 minutes
  → Terminates 8 instances (total: 2)
  → You stop paying for idle capacity

COST IMPACT:
  Without ASG: Run 10 instances 24/7 = ₹70,000/month
  With ASG:    Run 2-10 based on load = ₹12,000/month avg
```

### Lambda — Serverless Functions

**What it is**: Run code without managing any servers. You write a function, AWS runs it when triggered, you pay only for execution time (per 100ms).

**When to use Lambda vs EC2:**
```
USE EC2 WHEN:
  ✅ Long-running processes (web server, background jobs)
  ✅ Need full control of the environment
  ✅ Consistent high traffic
  ✅ Stateful applications

USE LAMBDA WHEN:
  ✅ Event-driven tasks (image resize on upload, send email on order)
  ✅ Infrequent or unpredictable execution
  ✅ Tasks that complete in < 15 minutes
  ✅ Want zero server management
  ✅ Low traffic or bursty traffic
```

**ShopEase Lambda functions:**
```python
# EXAMPLE 1: Resize product images when uploaded to S3
def resize_image(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download original image
    image = s3.get_object(Bucket=bucket, Key=key)
    
    # Resize to 3 sizes
    for size in [(800, 600), (400, 300), (100, 100)]:
        resized = resize(image, size)
        s3.put_object(Bucket=bucket, Key=f"thumbnails/{size}/{key}", Body=resized)

# TRIGGERED BY: S3 event (every time image is uploaded)
# RUNS: only when image is uploaded (maybe 1,000 times/day)
# COST: 0.001 seconds × 1,000 × $0.0000002 = nearly free

# EXAMPLE 2: Send order confirmation email
def send_order_email(event, context):
    order = event['detail']
    ses.send_email(
        To=order['customer_email'],
        Subject=f"Order #{order['id']} confirmed!",
        Body=f"Hi {order['name']}, your order is confirmed..."
    )

# TRIGGERED BY: EventBridge (when order is placed)
```

**Lambda pricing:**
```
FREE TIER: 1 million requests/month + 400,000 GB-seconds/month
BEYOND:    $0.20 per million requests + $0.00001667 per GB-second

ShopEase image resize: 50,000 images/month × 1 second × 0.5GB
  = 25,000 GB-seconds × $0.00001667 = $0.42/month (VERY cheap)
```

### ECS — Elastic Container Service

**What it is**: AWS-managed service for running Docker containers. You package your app in Docker, ECS runs and scales it.

**EC2 vs ECS:**
```
EC2: You manage the OS, patches, scaling logic, container runtime
ECS: You just provide Docker image → AWS handles the rest

ShopEase uses ECS (Fargate mode — no EC2 to manage):

task_definition = {
    "family": "shopease-api",
    "cpu": "512",      # 0.5 vCPU
    "memory": "1024",  # 1 GB RAM
    "containers": [{
        "name": "api",
        "image": "123456789.ecr.amazonaws.com/shopease-api:v2.1",
        "portMappings": [{"containerPort": 8080}],
        "environment": [
            {"name": "DB_HOST", "value": "shopease.xyz.rds.amazonaws.com"},
            {"name": "REDIS_HOST", "value": "shopease.cache.amazonaws.com"}
        ]
    }]
}

# ECS runs 5 containers, auto-scales to 50 during Diwali
# You never touch any server
```

### EKS — Elastic Kubernetes Service

**What it is**: AWS-managed Kubernetes. If you're already using Kubernetes (k8s), EKS lets you run it on AWS without managing the control plane yourself.

```
USE ECS WHEN:
  ✅ AWS-only, don't need Kubernetes specifically
  ✅ Simpler setup, AWS-native
  ✅ Small to medium teams

USE EKS WHEN:
  ✅ Already using Kubernetes
  ✅ Need Kubernetes-specific features (Helm charts, complex networking)
  ✅ Multi-cloud strategy (want same config on AWS, GCP, Azure)
  ✅ Large engineering team with Kubernetes expertise

ShopEase: Started with ECS (simpler). Migrated to EKS when team grew.
```

### Elastic Beanstalk

**What it is**: AWS's Platform-as-a-Service. Upload your code, Beanstalk handles EC2, load balancer, auto-scaling, deployments — everything.

```
DEVELOPER EXPERIENCE:
  eb init shopease --platform node.js
  eb create production
  eb deploy

AWS automatically:
  → Creates EC2 instances
  → Sets up Load Balancer
  → Configures Auto Scaling
  → Deploys your code
  → Monitors health

USE WHEN: Small team, want to focus on code not infrastructure
LIMITATION: Less control, harder to customize
```

---

## 6. Layer 3 — Storage

**Three types of storage needs: files, disks, shared filesystems.**

### S3 — Simple Storage Service

**What it is**: Unlimited file storage. Store any file (images, videos, PDFs, backups, static websites) and access via URL. This is one of AWS's most important and widely-used services.

**Key properties:**
```
UNLIMITED STORAGE:   No limit on how much you store
99.999999999% DURABILITY: 11 nines — if you store 1 billion files,
                           you'd expect to lose 1 file in 10,000 years
GLOBALLY ACCESSIBLE: Files get a URL anyone can access
NOT A FILE SYSTEM:   Object storage — flat structure with unique keys
```

**ShopEase uses S3 for:**
```
BUCKET: shopease-images
  products/macbook-pro-2025/original.jpg
  products/macbook-pro-2025/thumbnail-800x600.jpg
  products/macbook-pro-2025/thumbnail-400x300.jpg

BUCKET: shopease-static  (serves the entire React frontend)
  index.html
  main.js
  styles.css
  (entire website served from S3 + CloudFront = no server needed)

BUCKET: shopease-backups
  db-backups/2025/11/28/postgres-dump.sql.gz
  logs/2025/11/28/app.log.gz

BUCKET: shopease-user-uploads
  invoices/user-12345/invoice-789.pdf
  avatars/user-12345/profile.jpg
```

**S3 Storage Classes (save money by tiering):**
```
STANDARD:             ₹1.84/GB/month  → Hot data (product images users see daily)
STANDARD-IA:          ₹0.99/GB/month  → Infrequent access (old order PDFs)
  (IA = Infrequent Access, retrieval fee applies)
ONE ZONE-IA:          ₹0.79/GB/month  → Can recreate if lost (thumbnails)
GLACIER INSTANT:      ₹0.30/GB/month  → Archives accessed occasionally
GLACIER FLEXIBLE:     ₹0.13/GB/month  → Archives (3-5hr retrieval, compliance logs)
GLACIER DEEP ARCHIVE: ₹0.046/GB/month → 7-year compliance storage

LIFECYCLE POLICY (automatic tiering):
  0-90 days:   Standard (hot, recent data)
  90-365 days: Standard-IA (older, less accessed)
  365+ days:   Glacier (archival, compliance)
```

**S3 Features ShopEase uses:**
```
VERSIONING:
  Keep all versions of every file
  Accidentally deleted product image? Restore previous version

PRESIGNED URLs:
  Generate temporary URLs for private files
  "User can download their invoice for the next 15 minutes"
  url = s3.generate_presigned_url('get_object',
        Params={'Bucket': 'shopease-user', 'Key': 'invoice-789.pdf'},
        ExpiresIn=900)  # 15 minutes

STATIC WEBSITE HOSTING:
  S3 can serve the entire React frontend directly
  No server needed for HTML/JS/CSS files
  Cost: ~$0.023/GB + bandwidth vs $72/month for EC2

REPLICATION:
  Automatically copy all files to another region (disaster recovery)
  Mumbai bucket → Singapore bucket (cross-region replication)

EVENT NOTIFICATIONS:
  "When an image is uploaded → trigger Lambda to resize it"
```

### EBS — Elastic Block Store

**What it is**: The hard drive for your EC2 instances. When you launch an EC2, it gets an EBS volume as its disk.

```
ANALOGY: EC2 = computer. EBS = the hard drive inside that computer.

TYPES:
  gp3 (General Purpose SSD):  ← Most common, 99% of use cases
    Performance: 3,000 IOPS, 125 MB/s throughput
    Cost: $0.08/GB/month
    Use: EC2 boot volumes, app servers

  io2 (Provisioned IOPS SSD): ← High performance, critical DBs
    Performance: Up to 64,000 IOPS
    Cost: $0.125/GB/month + IOPS cost
    Use: High-transaction databases

  st1 (Throughput HDD):       ← Sequential reads (logs, Kafka)
    Cost: $0.045/GB/month
    Use: Big data, log processing

KEY FEATURES:
  SNAPSHOTS: Point-in-time backup of the volume → stored in S3
  ENCRYPTION: Encrypted at rest using KMS keys
  MULTI-ATTACH: io2 volumes can attach to multiple EC2s (advanced)

IMPORTANT: EBS volumes are tied to ONE AZ.
  EC2 in ap-south-1a → EBS must also be in ap-south-1a
  (Unlike S3, which is regional and accessible everywhere)
```

### EFS — Elastic File System

**What it is**: A shared network file system that multiple EC2 instances can mount simultaneously — like a shared network drive.

```
PROBLEM EFS SOLVES:
  ShopEase has 5 EC2 instances running the API.
  Users upload profile avatars to EC2-1's local disk.
  User's next request goes to EC2-3 (load balanced).
  EC2-3 can't find the avatar — it's on EC2-1's disk. ❌

SOLUTION with EFS:
  All 5 EC2 instances mount the same EFS filesystem.
  Upload to EC2-1 → file appears on ALL instances instantly. ✅

USE EFS WHEN:
  ✅ Shared file access across multiple EC2 instances
  ✅ Content management systems (WordPress, Drupal)
  ✅ Machine learning training datasets

DON'T USE EFS WHEN:
  ❌ Static assets like product images → use S3 (cheaper, more scalable)
  ❌ Database files → use EBS (faster IOPS)
```

**Storage comparison:**
```
        S3              EBS             EFS
Type    Object          Block           File system
Access  Via URL/API     EC2 only (1)    EC2 only (many)
Scale   Unlimited       Up to 64TB      Unlimited (auto)
Cost    $0.023/GB       $0.08/GB        $0.30/GB
Speed   Variable        High IOPS       Medium
Use     Files, backups  EC2 disks       Shared storage
```

---

## 7. Layer 4 — Databases

**AWS offers managed databases so you never patch, backup, or failover manually.**

### RDS — Relational Database Service

**What it is**: Managed relational databases (PostgreSQL, MySQL, MariaDB, Oracle, SQL Server). AWS handles backups, patching, failover, scaling.

**ShopEase uses RDS PostgreSQL for:**
- Users table
- Products table
- Orders table
- Payments table

```
RDS CONFIGURATION for ShopEase:
──────────────────────────────────────────────────────────────
Engine:              PostgreSQL 16
Instance class:      db.r6g.xlarge (4 vCPU, 32GB RAM)
Storage:             500 GB gp3 SSD (auto-scales to 1TB)
Multi-AZ:            Enabled ← primary in AZ-a, standby in AZ-b

MULTI-AZ EXPLAINED:
  Primary DB (AZ-a) handles all reads/writes
  Standby DB (AZ-b) silently receives every transaction
  
  If AZ-a data center fails:
    → AWS promotes AZ-b standby to primary
    → DNS automatically points to new primary
    → Your app reconnects (using same endpoint URL)
    → Downtime: ~1-2 minutes (vs hours of manual recovery)
  
  YOUR CODE USES: shopease.xyz.ap-south-1.rds.amazonaws.com
  (same URL always — AWS handles failover invisibly)

BACKUPS:
  Automated daily snapshots → kept for 7 days
  Point-in-time recovery: restore to any second in last 35 days
  "Restore to exactly 2:47:23 PM yesterday before that bad migration"

READ REPLICAS:
  ShopEase has heavy read traffic (product browsing)
  Solution: Add 2 Read Replicas in same region

  Primary (write):   handles all INSERT/UPDATE/DELETE
  Read Replica 1:    handles product catalog reads
  Read Replica 2:    handles user profile reads

  Read replicas are ASYNC copies of primary
  Lag: typically < 100ms

  Code:
    primary_db = connect("shopease-primary.rds.amazonaws.com")
    read_db    = connect("shopease-readonly.rds.amazonaws.com")

    # Writes always go to primary
    primary_db.execute("INSERT INTO orders VALUES ...")
    
    # Reads can go to replica (ok if slightly behind)
    read_db.execute("SELECT * FROM products WHERE category='electronics'")
```

### Aurora — Amazon's High-Performance Database

**What it is**: Amazon's own database engine, compatible with MySQL and PostgreSQL, but 3-5x faster and more scalable.

```
AURORA vs RDS PostgreSQL:
──────────────────────────────────────────────────────────────
                    RDS PostgreSQL      Aurora PostgreSQL
Performance         Standard            3-5x faster
Storage             Manual provisioning Auto-grows (10GB → 128TB)
Read replicas       5 max               15 max
Failover            1-2 minutes         < 30 seconds
Replication lag     ~100ms              ~20ms
Cost                Lower               ~20% higher

AURORA SERVERLESS:
  No fixed instance — scales automatically from 0 to hundreds of ACUs
  
  USE CASE: ShopEase internal admin dashboard
    → Used 9am-6pm by 10 employees
    → Traffic at night: ZERO
  
  Without Aurora Serverless: Pay for db.t3.medium 24/7 = $50/month
  With Aurora Serverless:    Pay for actual usage = $8/month

ShopEase: RDS PostgreSQL for main app, Aurora Serverless for analytics
```

### DynamoDB — NoSQL at Massive Scale

**What it is**: AWS's fully managed NoSQL key-value and document database. Handles millions of requests per second with single-digit millisecond latency.

```
WHEN TO USE DynamoDB:
  ✅ Shopping cart (per-user data, key-value access)
  ✅ User sessions (quick reads/writes by session ID)
  ✅ Product reviews (flexible schema — different products need different fields)
  ✅ Real-time inventory tracking
  ✅ Millions of reads/writes per second needed
  ✅ Unpredictable traffic spikes

WHEN NOT TO USE DynamoDB:
  ❌ Complex SQL joins needed
  ❌ Strong ACID transactions required
  ❌ Ad-hoc queries / analytics
  ❌ Flexible querying patterns (DynamoDB is strict about access patterns)

SHOPEASE DYNAMODB — Shopping Cart:
  Table: shopease-carts
  Partition key: user_id
  
  Item example:
  {
    "user_id": "user-12345",
    "items": [
      {"product_id": "prod-789", "qty": 2, "price": 45999},
      {"product_id": "prod-234", "qty": 1, "price": 12999}
    ],
    "last_updated": "2025-11-28T14:23:00Z",
    "ttl": 1735344000  // auto-delete cart after 30 days
  }

  # Get cart (1ms):
  cart = dynamodb.get_item(
      TableName='shopease-carts',
      Key={'user_id': 'user-12345'}
  )

  # Update cart (1ms):
  dynamodb.update_item(
      TableName='shopease-carts',
      Key={'user_id': 'user-12345'},
      UpdateExpression='SET items = list_append(items, :new_item)',
      ExpressionAttributeValues={':new_item': [new_product]}
  )

DYNAMODB PRICING:
  On-demand: $1.25 per million reads + $6.25 per million writes
  Provisioned: Fixed capacity → cheaper if predictable load
```

### ElastiCache — Managed Redis / Memcached

**What it is**: AWS-managed in-memory cache. The fastest way to store and retrieve frequently accessed data — responses in under 1ms.

```
WITHOUT ElastiCache:
  Every request for product listing → hit PostgreSQL
  PostgreSQL query: 50-200ms
  1,000 users browsing → 1,000 DB queries/second → DB overwhelmed

WITH ElastiCache (Redis):
  First request for product listing → DB query (200ms) → store in Redis
  Next 999 requests → served from Redis (< 1ms)
  DB load reduced by 99%

SHOPEASE ElastiCache uses:
──────────────────────────────────────────────────────────────

1. PRODUCT CATALOG CACHE:
   key:   "products:category:electronics:page:1"
   value: JSON of 20 product listings
   TTL:   300 seconds (5 minutes)
   
   When product price changes:
   → Update DB
   → Delete cache key (cache invalidation)
   → Next request regenerates cache

2. SESSION STORAGE:
   key:   "session:abc123xyz"
   value: {user_id: "12345", cart_items: 3, last_active: "..."}
   TTL:   86400 seconds (24 hours)

3. RATE LIMITING:
   key:   "rate:api:ip:103.42.5.1"
   value: 47  (number of requests in this window)
   TTL:   60 seconds (per-minute rate limit)
   
   Pipeline:
   current = redis.incr("rate:api:ip:103.42.5.1")
   redis.expire("rate:api:ip:103.42.5.1", 60)
   if current > 100: return 429 Too Many Requests

4. OTP / EMAIL VERIFICATION:
   key:   "otp:user:arjun@shopease.com"
   value: "847291"
   TTL:   300 seconds (OTP expires in 5 minutes)

REDIS vs MEMCACHED:
  Use Redis when:     Data structures (lists, sets, sorted sets), persistence, pub/sub
  Use Memcached when: Pure simple key-value cache, need multi-threading

ElastiCache CLUSTER MODE:
  Multiple Redis nodes → data sharded across nodes
  ShopEase: 3-node cluster, each with 1 replica = 6 nodes total
  → 99.99% cache availability, automatic failover
```

### Redshift — Data Warehouse

**What it is**: A managed data warehouse for running complex analytical queries on massive datasets (billions of rows). NOT for your main app — for analytics and business intelligence.

```
PROBLEM: ShopEase business team wants to answer:
  "Which product categories saw highest sales growth MoM in Q3?"
  "What's our customer LTV by acquisition channel?"
  "Which seller region has highest return rate?"

These queries:
  → Join 5+ tables
  → Scan billions of rows
  → Take 2-5 minutes

Running these on the production RDS would crash it.

SOLUTION: Redshift
  → ETL pipeline copies data from RDS → Redshift nightly
  → Business team runs slow queries on Redshift
  → Production RDS is untouched

Redshift scales to petabytes. ShopEase's 100GB of data = tiny.
```

**Database selection guide:**
```
USE CASE                      → SERVICE
──────────────────────────────────────────────────────────────
Users, orders, products (SQL) → RDS PostgreSQL / Aurora
Shopping cart, sessions        → DynamoDB
Caching, rate limiting         → ElastiCache (Redis)
Search ("find red shoes")      → OpenSearch (see below)
Business analytics / reports   → Redshift
Time-series (metrics, IoT)     → Timestream
Graph data (social network)    → Neptune
Ledger / financial records     → QLDB (immutable)
```

---

## 8. Layer 5 — Traffic & Access

**Getting user requests to your app efficiently and securely.**

### Route 53 — DNS Service

**What it is**: AWS's DNS service. Translates domain names (shopease.in) to IP addresses, and routes traffic intelligently.

```
BASIC DNS:
  shopease.in → A record → 52.66.47.89 (Load Balancer IP)

ROUTING POLICIES (beyond simple DNS):
──────────────────────────────────────────────────────────────

1. LATENCY-BASED ROUTING:
   User in Mumbai → routes to ap-south-1 servers (10ms away)
   User in UK     → routes to eu-west-1 servers (15ms away)
   
   Without this: UK user hits Mumbai server (180ms extra delay)
   With this:    UK user hits Ireland server (15ms away) ✅

2. WEIGHTED ROUTING (Blue/Green Deployment):
   Version 1 (old):    10% of traffic
   Version 2 (new):    90% of traffic
   
   "Slowly shift traffic to new version while watching for errors"

3. FAILOVER ROUTING:
   Primary: Mumbai (ap-south-1) → handles all traffic
   Secondary: Singapore (ap-southeast-1) → standby
   
   If Mumbai health check fails:
   → Route 53 automatically redirects all traffic to Singapore
   → Disaster recovery, zero manual work

4. GEOLOCATION ROUTING:
   Indian users → India servers (GST compliance, data residency)
   EU users     → EU servers (GDPR compliance)
   US users     → US servers

HEALTH CHECKS:
  Route 53 pings your load balancer every 30 seconds
  If 3 consecutive checks fail → marks endpoint as unhealthy → stops routing to it
```

### CloudFront — CDN (Content Delivery Network)

**What it is**: A global network of 400+ edge locations that cache and serve your content close to users. Dramatically speeds up static assets (images, JS, CSS).

```
WITHOUT CloudFront:
  Arjun in Mumbai opens ShopEase homepage.
  Browser downloads:
    index.html (Mumbai → Mumbai server = 5ms) ✅
    main.js    (Mumbai → Mumbai server = 5ms) ✅
    product images (Mumbai → Mumbai server = 5ms) ✅
  
  Works fine if all users are in Mumbai.
  
  But: Customer in Kolkata downloads product image
  → Kolkata → Mumbai server = 45ms
  
  Customer in Chennai → Mumbai = 30ms
  Customer in Delhi → Mumbai = 25ms

WITH CloudFront:
  First request: Mumbai → Mumbai server → image cached at Mumbai edge
  Second request (and all after): served FROM the Mumbai edge location
  
  Kolkata user downloads image → from Kolkata edge location = 2ms ✅
  Chennai user → Chennai edge = 2ms ✅
  Delhi user → Delhi edge = 2ms ✅

CLOUDFRONT DISTRIBUTION for ShopEase:
  Origin: shopease-static.s3.amazonaws.com (S3 bucket)
  
  Caching:
    /static/*    → Cache for 1 year (fingerprinted filenames)
    /images/*    → Cache for 7 days
    /api/*       → Don't cache (dynamic content)
  
  Edge functions:
    → Add security headers to every response
    → Redirect HTTP → HTTPS
    → A/B test landing pages at the edge

COST: $0.0085 per GB served in India + $0.0075/10,000 HTTP requests
  Product images (1TB/month): $8.70 → massive, fast for users
  (vs serving from EC2 at $0.09/GB + server cost = much more expensive)
```

### ALB — Application Load Balancer

**What it is**: Distributes incoming web traffic across multiple EC2 instances or containers. Operates at Layer 7 (HTTP/HTTPS — understands URLs and headers).

```
ALB for ShopEase:
──────────────────────────────────────────────────────────────

LISTENER: HTTPS port 443 (SSL terminated here)

PATH-BASED ROUTING:
  /api/products/*  → Product Service target group (3 EC2 instances)
  /api/orders/*    → Order Service target group (2 EC2 instances)
  /api/users/*     → User Service target group (2 EC2 instances)
  /api/search/*    → Search Service target group (2 EC2 instances)
  /*               → Frontend target group (static, or fallback)

HOST-BASED ROUTING:
  api.shopease.in      → API servers
  admin.shopease.in    → Admin dashboard servers
  seller.shopease.in   → Seller portal servers

HEALTH CHECKS:
  ALB pings each instance every 30 seconds: GET /health
  Instance returns 200 → healthy, keep sending traffic
  Instance returns 500 or times out → unhealthy → stop sending traffic
  → Auto Scaling Group launches replacement

STICKY SESSIONS:
  If user's cart is stored in EC2 memory (bad practice, but legacy):
  → ALB can route same user to same EC2 for session duration
  → Uses cookie: AWSALB=abc123

ALB vs NLB (Network Load Balancer):
  ALB: Layer 7, HTTP-aware, path routing, SSL termination → 99% of web apps
  NLB: Layer 4, TCP/UDP, extreme performance (millions req/sec), gaming, IoT
```

### API Gateway — Managed API Layer

**What it is**: A fully managed service to create, publish, secure, and monitor APIs. Works perfectly with Lambda for serverless APIs.

```
SHOPEASE uses API Gateway for:

1. MOBILE API (different from web):
   All mobile requests → API Gateway → Lambda functions
   
   Benefits:
   → Rate limiting per mobile client (100 req/min)
   → API key management for 3rd party sellers
   → Request/response transformation
   → Caching at the API layer

2. THIRD-PARTY SELLER API:
   Sellers integrate ShopEase into their systems
   → Each seller gets an API key
   → API Gateway validates key, then routes to backend
   → Usage plans: Starter (1000 req/day), Pro (100,000 req/day)

3. SERVERLESS MICROSERVICE:
   POST /api/search → API Gateway → Lambda search function → OpenSearch
   No EC2 needed for the search service

API GATEWAY TYPES:
  REST API: Full-featured, HTTP/1.1
  HTTP API: Simpler, 70% cheaper (use for most modern APIs)
  WebSocket: Real-time bidirectional (live chat, price updates)

COST: $1.00 per million API calls (HTTP API) — extremely cheap
```

---

## 9. Layer 6 — Security

**Every service needs proper security. This layer protects everything.**

### IAM — Identity and Access Management

**What it is**: Controls WHO can access WHAT in your AWS account. The most fundamental security service. One mistake here can expose your entire infrastructure.

```
IAM CONCEPTS:
──────────────────────────────────────────────────────────────

USERS: Real people or applications that need AWS access
  arjun-admin  → Full admin access (dangerous, use sparingly)
  priya-dev    → Developer access (EC2, S3, CloudWatch)
  deploy-bot   → CI/CD access (ECR push, ECS deploy only)

GROUPS: Collections of users with same permissions
  Developers   → [priya-dev, rohit-dev, sneha-dev]
  DevOps       → [arjun-admin, vijay-devops]
  ReadOnly     → [finance-team]

ROLES: Permissions attached to AWS services (not people)
  EC2-Role     → Allows EC2 to read from S3 and write to DynamoDB
  Lambda-Role  → Allows Lambda to read SQS and send SES emails

POLICIES: JSON documents defining what's allowed
  {
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",   // can download files
      "s3:PutObject"    // can upload files
    ],
    "Resource": "arn:aws:s3:::shopease-images/*"
    // ONLY this bucket, not all S3 buckets
  }

PRINCIPLE OF LEAST PRIVILEGE:
  ❌ Bad:  Give EC2 instance admin access to all of AWS
  ✅ Good: Give EC2 ONLY the permissions it actually needs

ShopEase EC2 Role (Product Service):
  Allow: s3:GetObject on shopease-images/*
  Allow: elasticache:* on shopease-cache
  Allow: rds-db:connect on shopease-db/app_user
  Allow: cloudwatch:PutMetricData
  Deny: Everything else (implicit)

NEVER:
  ❌ Hardcode AWS access keys in your code
  ❌ Commit access keys to GitHub (bots scan for these 24/7)
  ❌ Use root account for daily operations
  ✅ Use IAM roles for applications
  ✅ Use IAM users with MFA for humans
```

### Cognito — User Authentication

**What it is**: Fully managed user authentication and authorization. Handle sign-up, sign-in, MFA, social login (Google, Facebook) — without building it yourself.

```
BUILDING AUTH YOURSELF:
  → Password hashing (bcrypt/argon2)
  → Email verification flows
  → Forgot password reset
  → MFA setup and validation
  → JWT token generation and validation
  → Social login OAuth flows
  → Session management
  → Security vulnerabilities → your responsibility

  Time: 3-4 weeks of development
  Risk: Auth bugs = account takeovers

COGNITO HANDLES ALL OF THIS:
  ShopEase Users:
  
  User Pool: shopease-users
  → Users register with email/phone
  → Cognito sends verification email automatically
  → Cognito handles forgot password
  → Cognito issues JWT tokens (ID, Access, Refresh)
  → Social sign-in: "Continue with Google" in 1 day of work
  → MFA: SMS or TOTP (Google Authenticator)
  
  Your API just verifies the JWT:
  
  @app.route('/api/orders')
  @cognito_required  # middleware verifies JWT
  def get_orders():
      user_id = request.user['sub']  # from JWT claims
      return Orders.query.filter_by(user_id=user_id).all()

IDENTITY POOL (Federated Identities):
  Allow users to directly access AWS resources:
  "Let authenticated ShopEase users upload to S3 directly from browser"
  → User gets temporary AWS credentials
  → Uploads directly to S3 (bypasses your server)
  → No server bandwidth cost for file uploads
```

### Secrets Manager

**What it is**: Securely store and automatically rotate database passwords, API keys, and other secrets. Never hardcode credentials.

```
WITHOUT Secrets Manager (dangerous):
  # In your code (gets committed to GitHub!)
  DB_PASSWORD = "MyDB@Pass123"
  RAZORPAY_SECRET = "rzp_live_abc123xyz"
  SENDGRID_API_KEY = "SG.abcdef..."

  GitHub bot finds these within minutes of push.
  Your database is compromised. Your accounts are drained.

WITH Secrets Manager:
  # At startup, fetch secrets from Secrets Manager:
  import boto3
  
  def get_secret(secret_name):
      client = boto3.client('secretsmanager', region_name='ap-south-1')
      response = client.get_secret_value(SecretId=secret_name)
      return json.loads(response['SecretString'])
  
  db_credentials = get_secret('shopease/production/db')
  # Returns: {"host": "...", "username": "app_user", "password": "Kj3#mP9$..."}
  
  razorpay = get_secret('shopease/production/razorpay')

AUTO-ROTATION:
  RDS password rotates every 30 days automatically:
  Day 1:  password = "Kj3#mP9$Qr7"
  Day 30: Secrets Manager generates new password → updates RDS → updates secret
  Day 31: Your app fetches new password on next restart
  
  Zero manual work, always fresh credentials.

COST: $0.40/secret/month + $0.05 per 10,000 API calls → negligible
```

### WAF — Web Application Firewall

**What it is**: Protects your web application from common attacks: SQL injection, XSS, DDoS, bot attacks.

```
SHOPEASE WAF RULES:
──────────────────────────────────────────────────────────────

AWS MANAGED RULES (pre-built, turn on with 1 click):
  AWSManagedRulesCommonRuleSet:
    → SQL injection protection
    → Cross-site scripting (XSS) protection
    → Known bad inputs filter
  
  AWSManagedRulesKnownBadInputsRuleSet:
    → Log4Shell vulnerability
    → SSRF (Server Side Request Forgery) attempts

CUSTOM RULES:
  Rate limiting: Block IP after 1,000 requests in 5 minutes
    → Prevents brute force attacks on /login
  
  Geographic blocking: Block all requests from specific countries
    → If not selling internationally, reduce attack surface
  
  Bot protection: Block known bot signatures
    → Except allow Googlebot (for SEO)
    
  IP reputation: Block IPs on known malicious lists
    → Updated automatically by AWS

EXAMPLE ATTACK BLOCKED:
  Attacker sends: GET /api/users?id=1 OR 1=1; DROP TABLE users;--
  
  Without WAF: Request reaches your code → potential SQL injection
  With WAF:    Request blocked at edge, attacker gets 403 Forbidden
               Your server never sees the request

ATTACH WAF TO: CloudFront, ALB, API Gateway

COST: $5/month per WebACL + $1/million requests + $1/month per rule
```

### KMS — Key Management Service

**What it is**: Create and manage encryption keys. Used to encrypt data at rest in S3, RDS, EBS, DynamoDB.

```
SHOPEASE uses KMS for:

DATABASE ENCRYPTION:
  RDS storage encrypted with KMS key
  If someone physically steals a hard drive from AWS → unreadable

S3 ENCRYPTION:
  All files in shopease-user-uploads encrypted at rest
  User's private invoices, addresses → encrypted with KMS

EBS ENCRYPTION:
  EC2 instance disk encrypted → OS, app code, temp files secured

CUSTOMER-MANAGED KEYS:
  ShopEase creates its own CMK (Customer Master Key)
  Controls who can use the key, audit key usage
  
  "Encrypt customer payment info with our own key
   — even AWS employees can't read this data"

COMPLIANCE: PCI-DSS, HIPAA, SOC2 all require encryption → KMS satisfies this
COST: $1/month per customer-managed key + $0.03 per 10,000 API calls
```

### Shield — DDoS Protection

```
AWS SHIELD STANDARD (free):
  Automatically included for all AWS customers
  Protects against common DDoS attacks (SYN floods, UDP reflection)

AWS SHIELD ADVANCED ($3,000/month):
  24/7 DDoS Response Team (DRT) available
  Financial protection (AWS credits if DDoS causes your bill to spike)
  Advanced attack mitigation
  
  USE WHEN: Financial services, government, high-profile targets
  ShopEase: Shield Standard is sufficient (Diwali traffic ≠ DDoS attack)
```

---

## 10. Layer 7 — Messaging & Events

**Decoupling services so they don't need to talk directly to each other.**

### SQS — Simple Queue Service

**What it is**: A message queue. Service A sends a message, Service B processes it — asynchronously, even if B is slow or down temporarily.

```
WITHOUT SQS (tight coupling):
  Customer places order → API directly calls:
    1. Payment service (100ms)
    2. Inventory service (50ms)
    3. Email service (200ms) ← email server is slow today
    4. SMS service (100ms)
    5. Warehouse service (150ms)
  
  Total API response time: 600ms
  
  If email server is down → entire order fails!
  User gets error even though payment succeeded.

WITH SQS (loose coupling):
  Customer places order → API does:
    1. Save order to DB (50ms)
    2. Put message in SQS queue (5ms)
    3. Return "Order placed!" to user (55ms total) ← fast!
  
  Separate services process the queue message:
    → Payment service reads message → processes payment
    → Inventory service reads message → updates stock
    → Email service reads message → sends email (even if delayed)
    → SMS service reads message → sends SMS
    → Warehouse service → prepares shipment
  
  If email server is down:
    Message stays in queue → email service retries when it comes back up
    Order is NOT affected!

SHOPEASE SQS QUEUES:
──────────────────────────────────────────────────────────────
shopease-orders-queue:       New orders to process
shopease-payment-queue:      Payment transactions
shopease-notification-queue: Emails and SMS to send
shopease-inventory-queue:    Stock updates from sellers
shopease-dlq:                Dead Letter Queue (failed messages)

DEAD LETTER QUEUE (DLQ):
  If a message fails processing 3 times → moved to DLQ
  Your team gets an alert → investigate why it failed
  Fix the bug → replay messages from DLQ

SQS TYPES:
  Standard Queue:  At least once delivery, best-effort ordering
                   (might deliver duplicate messages — handle idempotently)
  FIFO Queue:      Exactly once, strict order
                   Use for: payment processing (order matters!)

COST: $0.40 per million messages — essentially free
```

### SNS — Simple Notification Service

**What it is**: Publish-Subscribe (Pub/Sub) messaging. One publisher sends a message, multiple subscribers receive it simultaneously.

```
SNS vs SQS:
  SQS: One sender, one receiver (queue pattern)
  SNS: One sender, MANY receivers simultaneously (fan-out pattern)

SHOPEASE SNS use — Order Placed event:
  
  Topic: shopease-order-placed

  When order is placed → publish to SNS topic:
  sns.publish(
      TopicArn='arn:aws:sns:ap-south-1:123:shopease-order-placed',
      Message=json.dumps(order_data)
  )

  SNS fans out to ALL subscribers simultaneously:
  ┌─────────────────────────────────────────┐
  │           SNS Topic                     │
  │      shopease-order-placed              │
  └──┬──────────┬──────────┬───────────┬───┘
     │          │          │           │
     ▼          ▼          ▼           ▼
  SQS Queue  SQS Queue  Lambda     HTTP endpoint
  (payment)  (warehouse)(analytics)(seller webhook)

  Payment, Warehouse, Analytics, and Seller all get notified
  within milliseconds — simultaneously.

SNS DELIVERY TARGETS:
  SQS queues    → for reliable async processing
  Lambda        → for immediate serverless processing
  HTTP/HTTPS    → for webhooks to external services
  Email         → for human notifications (alerts)
  SMS           → for urgent alerts to on-call team
  Mobile push   → iOS/Android push notifications

SHOPEASE ALERTS via SNS:
  "Topic: shopease-alerts"
  Subscribers: arjun@shopease.com, +91-9876543210
  
  When: CloudWatch detects DB CPU > 90%
  → CloudWatch alarm → SNS → Email + SMS to on-call engineer
```

### EventBridge — Event Bus

**What it is**: A serverless event bus that connects AWS services and your applications. More powerful than SNS — supports filtering, scheduling, and integrating with 100+ AWS and SaaS services.

```
EVENTBRIDGE PATTERNS:

1. SERVICE-TO-SERVICE EVENTS:
   EC2 instance terminates → EventBridge → Lambda removes from monitoring
   RDS backup completes → EventBridge → SNS notifies team
   
2. SCHEDULED EVENTS (cron jobs):
   Every day at 2am → EventBridge → Lambda → generate daily reports
   Every Sunday 3am → EventBridge → Lambda → clean up expired sessions
   Every 1st of month → EventBridge → Lambda → send subscription invoices

3. SHOPEASE BUSINESS EVENTS:
   Order placed:
     Rule: event.source == "shopease.orders" AND status == "placed"
     Target 1: Lambda → payment processing
     Target 2: SQS → warehouse notification
     Target 3: Lambda → send confirmation email
   
   Product out of stock:
     Rule: inventory.quantity == 0
     Target: Lambda → notify seller + remove from search

4. SAAS INTEGRATIONS:
   Stripe payment confirmed → EventBridge → Lambda → update order status
   Razorpay webhook → EventBridge → order fulfillment service
```

### Kinesis — Real-Time Data Streaming

**What it is**: Real-time data streaming at massive scale. Process millions of events per second. Think of it as a high-speed highway for data.

```
WHEN TO USE KINESIS (vs SQS):
  SQS:    Queue for tasks (process each message once, at your pace)
  Kinesis: Stream for events (process in real-time, ordered, replay possible)

SHOPEASE Kinesis uses:

1. CLICKSTREAM ANALYTICS:
   Every click, page view, search → sent to Kinesis
   → Real-time: "What are users searching for right now?"
   → Personalization engine updates recommendations in real-time
   → Fraud detection (unusual clicking patterns)
   
   Scale: 1 million page views/day = ~12 events/second (easy)
   Kinesis handles millions/second with sharding

2. LIVE INVENTORY TRACKING:
   Each seller warehouse sends stock updates → Kinesis
   → Real-time inventory counts across all warehouses
   → Prevent overselling during Diwali flash sale

3. REAL-TIME FRAUD DETECTION:
   Every payment attempt → Kinesis → ML model checks in <100ms
   Suspicious pattern → block transaction before it processes

KINESIS COMPONENTS:
  Data Streams:     Real-time streaming (milliseconds latency)
  Data Firehose:    Load streaming data into S3, Redshift (buffer, batch)
  Data Analytics:   Run SQL queries on streaming data in real-time
```

---

## 11. Layer 8 — Monitoring

**You can't fix what you don't know is broken.**

### CloudWatch — The Central Monitoring Hub

**What it is**: AWS's monitoring service for logs, metrics, dashboards, and alerts.

```
THREE THINGS CloudWatch does:

1. METRICS — Numbers over time
   AWS Auto-collects for free:
     EC2: CPU%, Memory%, Network in/out, Disk I/O
     RDS: DB connections, Read/Write IOPS, Storage, Replica lag
     ALB: Request count, Latency, Error rates (4xx, 5xx)
     Lambda: Invocations, Duration, Errors, Throttles

   Custom metrics you send:
     shopease.orders.placed_per_minute    → business KPI
     shopease.cart.abandonment_rate       → product KPI
     shopease.payment.failure_rate        → critical alert

2. LOGS — Text output from your application
   Your app logs → CloudWatch Logs
   
   # In your Node.js/Python app:
   console.log(JSON.stringify({
     level: "ERROR",
     message: "Payment failed",
     order_id: "order-789",
     user_id: "user-12345",
     error: "Card declined",
     timestamp: new Date().toISOString()
   }))
   
   → CloudWatch Logs stores all logs
   → Search: "Find all ERROR logs in the last hour"
   → Log Insights: SQL-like queries across logs
   
   # Query: Why did orders drop at 2pm?
   fields @timestamp, message, order_id
   | filter level = "ERROR" and @timestamp > '2025-11-28 14:00:00'
   | stats count(*) by error

3. ALARMS — Automatic alerts when something goes wrong
   ShopEase Alarms:
   
   CRITICAL (page on-call now):
     ALB 5xx error rate > 5% for 5 minutes → SNS → SMS + call
     RDS CPU > 90% for 10 minutes         → SNS → SMS + call
     Payment failure rate > 10%           → SNS → SMS + call
   
   WARNING (Slack notification):
     EC2 CPU > 70% for 15 minutes → SNS → Slack
     ElastiCache cache hit rate < 70%     → SNS → Slack
     SQS queue depth > 10,000 messages    → SNS → Slack
   
   AUTO-HEALING:
     EC2 CPU > 80% → trigger Auto Scaling → add 3 more instances
     EC2 CPU < 20% → trigger Auto Scaling → remove 2 instances
```

### X-Ray — Distributed Tracing

**What it is**: Traces a single user request as it flows through all your microservices, showing exactly where time is spent and where errors occur.

```
WITHOUT X-Ray:
  User complains: "My order took 8 seconds to place"
  
  You look at:
    API server logs → request received at 14:23:00.000
    API server logs → response sent at 14:23:08.015
  
  WHERE in those 8 seconds? No idea.

WITH X-Ray:
  Complete trace of order placement:
  
  14:23:00.000  Request received by ALB
  14:23:00.010  EC2 API server: auth validation (10ms)
  14:23:00.020  EC2 API server: cart validation (10ms)
  14:23:00.030  ElastiCache: check session (5ms)
  14:23:00.035  RDS: BEGIN transaction
  14:23:00.040  RDS: INSERT INTO orders (5ms)
  14:23:00.045  RDS: UPDATE inventory SET qty = qty - 1 ←── 7,900ms here!
  14:23:07.945  RDS: COMMIT (slow because inventory table has no index)
  14:23:07.960  SQS: send order message (15ms)
  14:23:07.975  Response sent

  PROBLEM FOUND: inventory update takes 7.9 seconds
  FIX: Add index on inventory.product_id

X-Ray shows:
  → Flame graph of the full request
  → Which service is the bottleneck
  → Which downstream API calls are slow
  → Error rates per service
  → P50, P95, P99 latency percentiles
```

### CloudTrail — Audit Logs

**What it is**: Records every API call made in your AWS account — who did what, when, from where.

```
CLOUDTRAIL captures:
  Who:    IAM user "priya-dev" or role "EC2-Role"
  What:   DeleteBucket on S3
  When:   2025-11-28T14:23:00Z
  Where:  IP 106.51.42.89 (Priya's laptop)
  Result: Success/Failure

USE CASES:

SECURITY INCIDENT INVESTIGATION:
  "Someone deleted our production database at 3am."
  CloudTrail shows:
    User: compromised-iam-key
    Action: rds:DeleteDBInstance
    From IP: 45.123.67.89 (Russia)
    Time: 03:14:22 UTC
  
  Now you know what happened and can trace the breach.

COMPLIANCE:
  SOC2, ISO27001, PCI-DSS require audit logs
  "Prove that only authorized people accessed customer data"
  CloudTrail provides this proof.

ALERTS:
  "Alert me if anyone creates an IAM admin user"
  EventBridge + CloudTrail → if CreateUser with admin policy → SNS alert

COST: Free for 90 days of management events. 
      $2/100,000 data events for full logging.
```

---

## 12. Layer 9 — CI/CD

**Automatically build, test, and deploy your code.**

### ECR — Elastic Container Registry

**What it is**: AWS's private Docker image registry. Store your Docker images securely in AWS.

```
FLOW:
  Developer pushes code → ECR stores the Docker image
  ECS/EKS pulls image from ECR to deploy

WHY ECR vs Docker Hub:
  ✅ Private by default (your code stays in your AWS account)
  ✅ Integrated with IAM (role-based access)
  ✅ Vulnerability scanning (alerts you of known CVEs)
  ✅ Same region as ECS → faster pulls (no internet transfer cost)

SHOPEASE ECR REPOS:
  123456.ecr.amazonaws.com/shopease-api:v2.1
  123456.ecr.amazonaws.com/shopease-api:latest
  123456.ecr.amazonaws.com/shopease-search:v1.3
  123456.ecr.amazonaws.com/shopease-worker:v1.8

LIFECYCLE POLICY:
  Keep last 10 tagged images
  Delete untagged images after 7 days
  → Saves storage cost, keeps registry clean
```

### CodePipeline — CI/CD Pipeline

**What it is**: Automated pipeline that builds, tests, and deploys your code every time you push to Git.

```
SHOPEASE DEPLOYMENT PIPELINE:
──────────────────────────────────────────────────────────────
Developer pushes to GitHub main branch
          │
          ▼
STAGE 1: SOURCE
  CodePipeline detects push to main
  Downloads source code artifact
          │
          ▼
STAGE 2: BUILD (CodeBuild)
  Runs in a fresh Docker container:
  1. npm install (install dependencies)
  2. npm test (run all unit tests)
  3. npm run lint (check code style)
  4. docker build -t shopease-api:$COMMIT_HASH . (build image)
  5. docker push ECR/shopease-api:$COMMIT_HASH (push to ECR)
  
  If any step fails → pipeline stops, developer gets Slack notification
  
  BUILD TIME: ~4-6 minutes
          │
          ▼
STAGE 3: DEPLOY TO STAGING
  ECS updates staging service:
  → Pulls new Docker image from ECR
  → Rolls out 1 new task → health check passes → remove 1 old task
  → (Blue/green rolling deployment — zero downtime)
  
  Run integration tests against staging:
  → If tests fail → stop, alert team
          │
          ▼
STAGE 4: MANUAL APPROVAL
  Pipeline pauses → Slack notification to QA lead
  "New version on staging. Approve for production? [Approve] [Reject]"
  
  QA lead reviews staging → clicks Approve in console or Slack
          │
          ▼
STAGE 5: DEPLOY TO PRODUCTION
  ECS updates production service:
  → 1 new task → health check → remove 1 old → repeat
  → All 5 instances updated with zero downtime
  → Takes ~10 minutes
  
  CloudWatch monitors for errors during deployment:
  → If error rate spikes → automatic rollback to previous version

TOTAL: Code merged → Production in ~15 minutes (vs manual: hours)
```

---

## 13. Complete Architecture

**Every layer together — this is what ShopEase looks like in production:**

```
                          INTERNET USERS
                               │
                               ▼
                    ┌────────────────────┐
                    │     Route 53       │
                    │  (DNS + Failover)  │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │    CloudFront      │
                    │  (CDN + WAF + SSL) │
                    └─────────┬──────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
    Static files         API Requests        Admin portal
    (S3 directly)             │
                    ┌─────────▼──────────┐
                    │   Application LB   │
                    │  (ALB + health     │
                    │   checks)          │
                    └─────────┬──────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │  EC2 / ECS  │    │  EC2 / ECS  │    │  EC2 / ECS  │
  │  (AZ-a)     │    │  (AZ-b)     │    │  (AZ-c)     │
  │  API Server │    │  API Server │    │  API Server │
  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            │ (all connect to same data layer)
         ┌──────────────────┼──────────────────────────┐
         │                  │                          │
         ▼                  ▼                          ▼
  ┌─────────────┐    ┌─────────────┐          ┌─────────────┐
  │ ElastiCache │    │     RDS     │          │  DynamoDB   │
  │   (Redis)   │    │ PostgreSQL  │          │   (Cart,    │
  │   Cache     │    │ Multi-AZ    │          │  Sessions)  │
  └─────────────┘    └─────────────┘          └─────────────┘
         │
         ▼
  ┌─────────────┐
  │  SQS + SNS  │
  │  EventBridge│
  └──────┬──────┘
         │
    ┌────┴────┐
    │ Lambda  │
    │Functions│
    └────┬────┘
         │
    ┌────┴────┐
    │   S3    │
    │ (Images │
    │ Backups)│
    └─────────┘

SECURITY WRAPS EVERYTHING:
  IAM: controls access to all services
  Cognito: user authentication
  Secrets Manager: credentials
  KMS: encryption at rest
  VPC + Security Groups: network isolation

MONITORING:
  CloudWatch: metrics, logs, alarms
  X-Ray: request tracing
  CloudTrail: audit logs
```

---

## 14. Cost & Pricing Models

**Understanding AWS billing before your first ₹10,000 surprise bill.**

```
HOW AWS CHARGES:
──────────────────────────────────────────────────────────────
COMPUTE:     Per hour (EC2) or per 100ms (Lambda)
STORAGE:     Per GB per month (S3, EBS, EFS)
DATA OUT:    Per GB transferred OUT to internet (data IN is free)
REQUESTS:    Per million API calls (API Gateway, S3 requests)
DATABASE:    Per hour instance + per GB storage + I/O operations

SHOPEASE MONTHLY COSTS (production, 100K users):
  EC2 (3× t3.large, Reserved 1yr):   $130/month
  ALB:                                $20/month
  RDS PostgreSQL (db.r6g.large):      $120/month
  ElastiCache (cache.r6g.large):      $115/month
  DynamoDB (on-demand):               $40/month
  S3 (500 GB + requests):             $15/month
  CloudFront (1 TB out):              $85/month
  NAT Gateway (100 GB):               $35/month
  Route 53:                           $5/month
  Secrets Manager (10 secrets):       $4/month
  CloudWatch (custom metrics/logs):   $30/month
  WAF:                                $10/month
  ─────────────────────────────────────────────
  TOTAL:                              ~$609/month
  
  At 100,000 users = $0.007/user/month (less than ₹1/user/month)
```

**Cost saving strategies:**
```
1. RESERVED INSTANCES:
   On-demand EC2 t3.large: $0.095/hour = $69/month
   Reserved 1-year:        $0.057/hour = $41/month (40% savings)
   Reserved 3-year:        $0.036/hour = $26/month (62% savings)

2. SAVINGS PLANS:
   Commit to $0.10/hour of compute spend
   Works across EC2, Lambda, Fargate, any region
   Saves 20-40%

3. SPOT INSTANCES for batch jobs:
   Video encoding, report generation, ML training
   Use spot at $0.02/hour (vs $0.095 on-demand)
   Save 79% on fault-tolerant batch work

4. S3 LIFECYCLE POLICIES:
   Move old data to cheaper storage automatically
   Save 70-90% on archival data

5. RESERVED RDS:
   RDS is often the biggest cost
   1-year reserved: 40% savings on RDS

6. DATA TRANSFER OPTIMIZATION:
   CloudFront caches → less origin traffic → less NAT Gateway cost
   S3 Transfer Acceleration for large uploads
   VPC endpoints for DynamoDB/S3 → avoid NAT Gateway cost

7. RIGHT-SIZING:
   Use AWS Compute Optimizer to find over-provisioned instances
   "This t3.xlarge runs at 15% CPU → downsize to t3.medium, save $70/month"

FREE TIER (new accounts, 12 months):
  EC2 t2.micro:        750 hours/month free
  S3:                  5 GB free
  RDS db.t2.micro:     750 hours/month free
  Lambda:              1M requests/month free (forever)
  DynamoDB:            25 GB storage free (forever)
  CloudFront:          1 TB data transfer/month free
```

---

## 15. Quick Reference — All Services at a Glance

### Compute
| Service | What It Is | Use When |
|---------|-----------|---------|
| **EC2** | Virtual servers | Long-running apps, full OS control |
| **Lambda** | Serverless functions | Event-driven, short tasks (<15 min) |
| **ECS** | Managed Docker containers | Containerized apps, AWS-native |
| **EKS** | Managed Kubernetes | K8s workloads, multi-cloud |
| **Elastic Beanstalk** | PaaS (code → running app) | Small teams, quick deploy |
| **Fargate** | Serverless containers | Run containers without EC2 |
| **Batch** | Batch job processing | ML training, video encoding |

### Storage
| Service | What It Is | Use When |
|---------|-----------|---------|
| **S3** | Object storage (files) | Images, backups, static sites, any file |
| **EBS** | Block storage (disks) | EC2 hard drives |
| **EFS** | Shared network filesystem | Multiple EC2s need same files |
| **Glacier** | Archival storage | Long-term backups, compliance |
| **FSx** | Managed Windows/Lustre FS | Windows apps, HPC workloads |

### Database
| Service | What It Is | Use When |
|---------|-----------|---------|
| **RDS** | Managed SQL DB | PostgreSQL, MySQL, structured data |
| **Aurora** | AWS SQL DB (faster/better) | High performance, serverless option |
| **DynamoDB** | Managed NoSQL | Key-value, massive scale, sessions |
| **ElastiCache** | Managed Redis/Memcached | Caching, sessions, rate limiting |
| **Redshift** | Data warehouse | Analytics, BI, petabyte queries |
| **OpenSearch** | Managed Elasticsearch | Full-text search, log analytics |
| **DocumentDB** | Managed MongoDB | MongoDB workloads |
| **Timestream** | Time-series DB | IoT data, metrics over time |

### Networking
| Service | What It Is | Use When |
|---------|-----------|---------|
| **VPC** | Private network | Always — foundation of everything |
| **Route 53** | DNS + routing | Domain management, failover |
| **CloudFront** | CDN | Static assets, global users |
| **ALB** | HTTP Load Balancer | Web apps, path-based routing |
| **NLB** | TCP Load Balancer | Ultra-high performance, gaming |
| **API Gateway** | Managed API layer | Serverless APIs, webhooks |
| **NAT Gateway** | Private → internet | Private subnets need internet |
| **VPC Peering** | Connect two VPCs | Multi-account architecture |
| **Direct Connect** | Dedicated line to AWS | On-premise to AWS, compliance |

### Security
| Service | What It Is | Use When |
|---------|-----------|---------|
| **IAM** | Access control | Always — who can do what |
| **Cognito** | User authentication | User sign-up/sign-in |
| **Secrets Manager** | Store credentials | DB passwords, API keys |
| **KMS** | Encryption key management | Encrypt data at rest |
| **WAF** | Web application firewall | SQLi, XSS, DDoS protection |
| **Shield** | DDoS protection | All apps (Standard free) |
| **GuardDuty** | Threat detection | Unusual API patterns, intrusions |
| **Inspector** | Vulnerability scanning | EC2 and container security |
| **Certificate Manager** | Free SSL certificates | HTTPS for your domains |

### Messaging
| Service | What It Is | Use When |
|---------|-----------|---------|
| **SQS** | Message queue | Async task processing |
| **SNS** | Pub/sub notifications | One → many fan-out |
| **EventBridge** | Event bus | Service-to-service events, cron |
| **Kinesis** | Real-time streaming | Analytics, clickstream, IoT |
| **MSK** | Managed Kafka | High-throughput event streaming |
| **SES** | Email sending | Transactional emails, marketing |
| **Pinpoint** | Mobile/SMS marketing | Push notifications, campaigns |

### Monitoring & DevOps
| Service | What It Is | Use When |
|---------|-----------|---------|
| **CloudWatch** | Metrics, logs, alarms | Always — monitoring foundation |
| **X-Ray** | Distributed tracing | Debug performance issues |
| **CloudTrail** | API audit logs | Security, compliance |
| **CodePipeline** | CI/CD pipeline | Automated deployments |
| **CodeBuild** | Build service | Compile, test, Docker build |
| **CodeDeploy** | Deployment service | EC2, Lambda deployments |
| **ECR** | Container registry | Store Docker images |
| **Systems Manager** | EC2 management | Patch, run commands, no SSH |

### AI & ML
| Service | What It Is | Use When |
|---------|-----------|---------|
| **Bedrock** | Managed LLMs (Claude, Llama, etc.) | Build AI features |
| **SageMaker** | ML model training + deployment | Train custom ML models |
| **Rekognition** | Image/video analysis | Face detection, content moderation |
| **Comprehend** | NLP (sentiment, entities) | Text analysis |
| **Transcribe** | Speech to text | Audio → text conversion |
| **Polly** | Text to speech | Audio generation |
| **Translate** | Language translation | Multi-language apps |
| **Forecast** | Time-series forecasting | Demand forecasting, inventory |

---

## The ShopEase Journey Summary

```
WHEN STARTING:
  EC2 (1 instance) + RDS + S3 + Route53
  Cost: ~$100/month | Handles: 1,000 users

GROWING:
  + ALB + Auto Scaling + ElastiCache + CloudFront
  Cost: ~$250/month | Handles: 50,000 users

SCALING:
  + ECS/EKS + DynamoDB + SQS/SNS + Lambda
  Cost: ~$600/month | Handles: 500,000 users

AT SCALE:
  + Aurora + Kinesis + Redshift + Multi-region + WAF
  Cost: ~$3,000/month | Handles: 10,000,000 users

EACH LAYER adds availability, performance, and scale.
You don't start with everything — you grow into it.
```

---

*AWS has 200+ services. This guide covers the 35+ you'll use for 95% of real applications. Master these, and you can build anything.*

*Last updated: March 2026*
