# Everlore Server - Infrastructure

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              LOAD BALANCER                                   │
│                        (Nginx / AWS ALB / Cloudflare)                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
           ▼                           ▼                           ▼
┌───────────────────┐      ┌───────────────────┐      ┌───────────────────┐
│   API Server 1    │      │   API Server 2    │      │   API Server N    │
│   Bun + Elysia    │      │   Bun + Elysia    │      │   Bun + Elysia    │
│   Port: 3000      │      │   Port: 3000      │      │   Port: 3000      │
└───────────────────┘      └───────────────────┘      └───────────────────┘
           │                           │                           │
           └───────────────────────────┼───────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
           ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
           │   MongoDB   │    │    Redis    │    │  Pinecone   │
           │   Cluster   │    │   Cluster   │    │   Index     │
           └─────────────┘    └─────────────┘    └─────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────┐
                    │    BullMQ Job Queue     │
                    │    (Redis-backed)       │
                    └─────────────────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
                    ▼                                     ▼
        ┌───────────────────────┐          ┌───────────────────────┐
        │    Worker Pool 1      │          │    Worker Pool 2      │
        │  (Generation Focus)   │          │  (Background Tasks)   │
        │  - Generation (3)     │          │  - Memory (5)         │
        │                       │          │  - Summary (2)        │
        └───────────────────────┘          └───────────────────────┘
                    │                                     │
                    └──────────────────┬──────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────┐
                    │     LLM Providers       │
                    │  ┌─────────────────┐    │
                    │  │   OpenAI API    │    │
                    │  └─────────────────┘    │
                    │  ┌─────────────────┐    │
                    │  │  OpenRouter API │    │
                    │  └─────────────────┘    │
                    └─────────────────────────┘
```

## Component Specifications

### API Server

**Runtime**: Bun 1.x
- High-performance JavaScript runtime
- Built-in TypeScript support
- Native WebSocket support

**Framework**: Elysia
- Bun-native HTTP framework
- TypeBox validation
- Built-in WebSocket handling

**Resource Requirements**:
| Metric | Development | Production |
|--------|-------------|------------|
| CPU | 1 core | 2+ cores |
| RAM | 512 MB | 1 GB |
| Instances | 1 | 3+ |

**Horizontal Scaling**:
- Stateless design
- Session data in Redis
- Multiple instances behind load balancer

### Database: MongoDB

**Deployment Options**:

#### Option A: MongoDB Atlas (Recommended)
- Managed service
- Auto-scaling
- Built-in backups
- Global clusters

**Configuration**:
```
Cluster Tier: M10 (Production) or M0 (Development)
Storage: 10GB+ (scales automatically)
Region: Same as application servers
Backup: Daily snapshots
```

#### Option B: Self-Hosted
```yaml
# docker-compose.yml example
version: '3.8'
services:
  mongo:
    image: mongo:7.0
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    environment:
      MONGO_INITDB_DATABASE: everlore
```

**Replica Set** (Production):
- Primary: 1 node
- Secondaries: 2 nodes
- Arbiter: 1 node (optional)

**Indexes**:
See [DATA_MODEL.md](DATA_MODEL.md) for complete index list.

### Cache & Queue: Redis

**Deployment Options**:

#### Option A: Redis Cloud / AWS ElastiCache
- Managed service
- Automatic failover
- Multi-AZ deployment

#### Option B: Self-Hosted
```yaml
# docker-compose.yml example
services:
  redis:
    image: redis:7-alpine
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
```

**Redis Cluster** (High Availability):
```
Minimum: 3 master nodes + 3 replica nodes
Sharding: Automatic across masters
Failover: Automatic replica promotion
```

**Keyspaces**:
- Session cache: `session:*`
- Rate limiting: `rl:*`
- Generation locks: `lock:gen:*`
- Pub/Sub: `user:*:events`
- BullMQ: `bull:*` (managed by library)

### Vector Database: Pinecone

**Service**: Pinecone Serverless (Recommended)

**Configuration**:
```
Index Name: nexus-memories
Dimension: 1536 (OpenAI text-embedding-3-small)
Metric: cosine
Cloud Provider: aws
Region: us-east-1 (match app servers)
```

**Namespaces**:
- `lore_{templateId}` - Template lore chunks
- `mem_{instanceId}` - Instance memories

**Alternative**: Self-hosted (Chroma, Qdrant)
```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage
```

## Deployment Strategies

### Docker Deployment

#### API Server Dockerfile
```dockerfile
FROM oven/bun:1 AS base
WORKDIR /app

# Copy dependencies
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

# Copy source
COPY . .

# Expose port
EXPOSE 3000

# Run
CMD ["bun", "run", "src/index.ts"]
```

#### Worker Dockerfile
```dockerfile
FROM oven/bun:1 AS base
WORKDIR /app

COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

COPY . .

# Workers don't expose ports
CMD ["bun", "run", "worker/index.ts"]
```

#### Docker Compose (Full Stack)
```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    ports:
      - "3000:3000"
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - MONGODB_URI=mongodb://mongo:27017/everlore
      - REDIS_URL=redis://redis:6379
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - PINECONE_API_KEY=${PINECONE_API_KEY}
    depends_on:
      - mongo
      - redis
    deploy:
      replicas: 2

  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - MONGODB_URI=mongodb://mongo:27017/everlore
      - REDIS_URL=redis://redis:6379
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - PINECONE_API_KEY=${PINECONE_API_KEY}
    depends_on:
      - mongo
      - redis
    deploy:
      replicas: 2

  mongo:
    image: mongo:7.0
    volumes:
      - mongo_data:/data/db
    ports:
      - "27017:27017"

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

volumes:
  mongo_data:
  redis_data:
```

### Kubernetes Deployment

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: everlore-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: everlore-api
  template:
    metadata:
      labels:
        app: everlore-api
    spec:
      containers:
      - name: api
        image: everlore/api:latest
        ports:
        - containerPort: 3000
        envFrom:
        - secretRef:
            name: everlore-secrets
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
```

```yaml
# worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: everlore-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: everlore-worker
  template:
    metadata:
      labels:
        app: everlore-worker
    spec:
      containers:
      - name: worker
        image: everlore/worker:latest
        envFrom:
        - secretRef:
            name: everlore-secrets
        resources:
          requests:
            memory: "1Gi"
            cpu: "1000m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
```

### Cloud Platform Deployment

#### AWS

**Services**:
- Compute: ECS Fargate or EKS
- Database: DocumentDB (MongoDB-compatible) or MongoDB Atlas
- Cache: ElastiCache for Redis
- Load Balancer: Application Load Balancer
- DNS: Route 53
- CDN: CloudFront (optional)

**Reference Architecture**:
```
Route 53 → CloudFront → ALB → ECS (API)
                          ↓
                    ElastiCache (Redis)
                          ↓
                    DocumentDB (MongoDB)
                          ↓
                    Pinecone (External)
```

#### Google Cloud Platform

**Services**:
- Compute: Cloud Run or GKE
- Database: MongoDB Atlas or self-hosted on Compute Engine
- Cache: Memorystore for Redis
- Load Balancer: Cloud Load Balancing

#### Azure

**Services**:
- Compute: Container Apps or AKS
- Database: Cosmos DB (MongoDB API) or MongoDB Atlas
- Cache: Azure Cache for Redis
- Load Balancer: Application Gateway

## Monitoring & Logging

### Health Checks

**Endpoint**: `GET /health`

**Response**:
```json
{
  "ok": true,
  "timestamp": "2025-01-15T10:30:00.000Z"
}
```

**Kubernetes Probes**:
- Liveness: `/health` (restart if failing)
- Readiness: `/health` (remove from LB if failing)

### Recommended Metrics

**Application Metrics**:
- Request rate (RPS)
- Response latency (p50, p95, p99)
- Error rate (4xx, 5xx)
- WebSocket connection count

**Queue Metrics**:
- Queue depth per queue
- Job processing time
- Job failure rate
- Worker utilization

**Database Metrics**:
- Query latency
- Connection pool usage
- Index hit ratio
- Replication lag

### Logging

**Structured Log Format** (JSON):
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "info",
  "service": "everlore-api",
  "traceId": "abc-123",
  "message": "Request processed",
  "method": "POST",
  "path": "/auth/login",
  "status": 200,
  "duration": 45
}
```

**Log Aggregation**:
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Datadog
- Splunk
- CloudWatch (AWS)

## Backup & Recovery

### MongoDB Backups

**Atlas**: Automatic daily snapshots, point-in-time recovery

**Self-Hosted**:
```bash
# Daily backup cron
0 2 * * * mongodump --host localhost --db everlore --out /backups/$(date +\%Y\%m\%d)

# Restore
mongorestore --host localhost --db everlore /backups/20250115/everlore
```

### Redis Backups

**Configuration**:
```
appendonly yes
appendfsync everysec
```

**Backup**:
```bash
# Trigger BGSAVE
redis-cli BGSAVE

# Copy RDB file
cp /data/dump.rdb /backups/redis-$(date +%Y%m%d).rdb
```

### Disaster Recovery

**RPO (Recovery Point Objective)**: 1 hour
**RTO (Recovery Time Objective)**: 30 minutes

**Procedure**:
1. Restore MongoDB from latest snapshot
2. Restore Redis (sessions will be lost, users re-authenticate)
3. Pinecone vectors rebuild from MongoDB (template republish, memory re-embed)
4. Scale workers to clear job backlog

## Scaling Guidelines

### Vertical Scaling

**API Server**: 
- CPU-bound: Add cores for concurrent requests
- Memory-bound: Increase for large payloads

**Workers**:
- Generation: Memory-intensive (context building)
- Memory/Summary: CPU-intensive (LLM calls)

### Horizontal Scaling

**API Servers**: 
- Scale based on request rate
- Target: <70% CPU utilization

**Workers**:
- Scale based on queue depth
- Target: <30s queue latency

**Auto-scaling Rules**:
```yaml
api:
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 70

worker:
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: bull_queue_depth
      targetAverageValue: 100
```

## Security Considerations

See [SECURITY.md](SECURITY.md) for detailed security documentation.

**Infrastructure Security**:
- Private subnets for databases
- Security groups / firewall rules
- VPC peering for managed services
- TLS for all connections

## Cost Optimization

### Development
```
MongoDB: Atlas M0 (free tier)
Redis: Self-hosted (Docker)
Pinecone: Serverless (pay per query)
Compute: Local development or small VM
```

### Production (Small)
```
MongoDB: Atlas M10 ($60/mo)
Redis: ElastiCache cache.t3.micro ($15/mo)
Pinecone: Serverless (~$20-50/mo)
Compute: 2x API + 2x Worker ($100-200/mo)
```

### Production (Large)
```
MongoDB: Atlas M50+ ($500+/mo)
Redis: ElastiCache cache.r6g.large ($200+/mo)
Pinecone: Serverless ($200+/mo)
Compute: 10+ API + 10+ Worker ($1000+/mo)
```
