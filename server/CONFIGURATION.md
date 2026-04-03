# Everlore Server - Configuration

## Environment Variables

All configuration is managed through environment variables. The server loads `.env` automatically in development.

### Required Variables

These must be set for production deployment:

| Variable | Description | Example |
|----------|-------------|---------|
| `JWT_SECRET` | Symmetric key for JWT signing | Long random string (32+ chars) |
| `MONGODB_URI` | MongoDB connection string | `mongodb://user:pass@host:27017/everlore` |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` |
| `OPENAI_API_KEY` | OpenAI API key | `sk-...` |
| `PINECONE_API_KEY` | Pinecone API key | `pcsk_...` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | HTTP server port | `3000` |
| `CLIENT_ORIGINS` | Comma-separated CORS origins | `http://localhost:3000,http://localhost:8080` |
| `OPENROUTER_API_KEY` | OpenRouter API key (optional) | `` |
| `PINECONE_INDEX` | Pinecone index name | `nexus-memories` |

## Configuration File

### `.env.example` (Reference)

```bash
# =============================================================================
# REQUIRED FOR PRODUCTION
# =============================================================================

# JWT Secret - Generate with: openssl rand -base64 32
JWT_SECRET=your-super-secret-jwt-key-change-in-production

# MongoDB Connection
# Format: mongodb://[user:pass@]host:port/database
MONGODB_URI=mongodb://localhost:27017/everlore

# Redis Connection
# Format: redis://[user:pass@]host:port
REDIS_URL=redis://localhost:6379

# OpenAI API Key
# Get from: https://platform.openai.com/api-keys
OPENAI_API_KEY=sk-...

# Pinecone API Key
# Get from: https://app.pinecone.io
PINECONE_API_KEY=pcsk_...

# =============================================================================
# OPTIONAL
# =============================================================================

# HTTP Server Port
PORT=3000

# CORS Origins (comma-separated)
CLIENT_ORIGINS=http://localhost:3000,http://localhost:8080,https://app.everlore.com

# OpenRouter API Key (for alternate models)
# Get from: https://openrouter.ai/keys
OPENROUTER_API_KEY=

# Pinecone Index Name
PINECONE_INDEX=nexus-memories
```

### Environment Loading (`src/config/env.ts`)

```typescript
const requiredEnvVars = [
  'JWT_SECRET',
  'MONGODB_URI',
  'REDIS_URL',
  'OPENAI_API_KEY',
  'PINECONE_API_KEY',
] as const

const optionalEnvVars = {
  PORT: '3000',
  CLIENT_ORIGINS: 'http://localhost:3000,http://localhost:8080',
  OPENROUTER_API_KEY: '',
  PINECONE_INDEX: 'nexus-memories',
} as const
```

**Development Fallbacks**:
- Missing required vars log warnings
- Development defaults provided for convenience
- **NEVER use defaults in production**

## TypeScript Configuration

### `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "types": ["bun-types"]
  },
  "include": ["src/**/*.ts", "worker/**/*.ts"]
}
```

**Key Settings**:
- `target: ESNext` - Use latest JavaScript features
- `module: ESNext` - ES modules for tree-shaking
- `moduleResolution: bundler` - Bun-compatible resolution
- `strict: true` - Maximum type safety
- `noEmit: true` - Bun handles compilation

## Package Configuration

### `package.json`

```json
{
  "name": "everlore-server",
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "bun run --watch src/index.ts",
    "dev:worker": "bun run --watch worker/index.ts",
    "start": "bun run src/index.ts",
    "start:worker": "bun run worker/index.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@elysiajs/cors": "^1.4.1",
    "@elysiajs/jwt": "^1.4.1",
    "@pinecone-database/pinecone": "^7.1.0",
    "argon2": "^0.44.0",
    "bullmq": "^5.73.0",
    "elysia": "^1.4.18",
    "ioredis": "^5.10.1",
    "js-tiktoken": "^1.0.21",
    "mongodb": "^7.1.1",
    "openai": "^6.33.0",
    "uuid": "^13.0.0"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "typescript": "^6.0.2"
  }
}
```

### Scripts

| Script | Purpose | Usage |
|--------|---------|-------|
| `dev` | Start API with hot reload | Development |
| `dev:worker` | Start worker with hot reload | Development |
| `start` | Start API (production) | Production |
| `start:worker` | Start worker (production) | Production |
| `typecheck` | TypeScript type checking | CI/CD |

## Runtime Configuration

### Bun-Specific Settings

**bunfig.toml** (optional):
```toml
[install]
# Use exact versions
exact = true

[run]
# Environment loader
env = ".env"
```

### Memory Limits

```bash
# Increase heap size for large contexts
bun --max-old-space-size=4096 run src/index.ts
```

## Service Configuration

### MongoDB Connection (`src/config/mongo.ts`)

```typescript
export async function connectMongo(): Promise<Db> {
  client = new MongoClient(env.MONGODB_URI)
  await client.connect()
  database = client.db()
  await createIndexes(database)
  return database
}
```

**Connection Options** (automatic):
- `maxPoolSize`: 10 (default)
- `serverSelectionTimeoutMS`: 30000

### Redis Connection (`src/config/redis.ts`)

```typescript
export async function connectRedis() {
  client = new Redis(env.REDIS_URL, { maxRetriesPerRequest: null })
  subscriber = new Redis(env.REDIS_URL, { maxRetriesPerRequest: null })
  queueClient = new Redis(env.REDIS_URL, { maxRetriesPerRequest: null })
}
```

**Connection Types**:
- `client`: General operations
- `subscriber`: Pub/sub only
- `queueClient`: BullMQ dedicated

**Important**: `maxRetriesPerRequest: null` required for BullMQ

### Pinecone Configuration (`src/config/pinecone.ts`)

```typescript
export function getPinecone(): Pinecone {
  return new Pinecone({ apiKey: env.PINECONE_API_KEY })
}

export function getPineconeIndex() {
  return getPinecone().index(env.PINECONE_INDEX)
}
```

## Feature Configuration

### Rate Limits (`src/middleware/rate-limit.ts`)

```typescript
const LIMITS: Record<string, { max: number; windowSeconds: number }> = {
  chat: { max: 10, windowSeconds: 60 },           // 10/min
  memory_edit: { max: 30, windowSeconds: 3600 },  // 30/hour
  template_create: { max: 5, windowSeconds: 86400 }, // 5/day
  auth_attempt: { max: 10, windowSeconds: 300 },  // 10/5min
}
```

### Tier Limits (`src/services/instance.service.ts`)

```typescript
const TIER_LIMITS: Record<string, { max_instances: number; max_memories: number }> = {
  free: { max_instances: 3, max_memories: 100 },
  premium: { max_instances: 20, max_memories: 500 },
  creator: { max_instances: 50, max_memories: 1000 },
}
```

### Worker Concurrency (`worker/index.ts`)

```typescript
const generationWorker = new Worker('generation', generationProcessor, {
  concurrency: 3,
  limiter: { max: 10, duration: 60000 },  // 10/min
})

const memoryWorker = new Worker('memory-curation', memoryProcessor, {
  concurrency: 5,
  limiter: { max: 20, duration: 60000 },  // 20/min
})

const summaryWorker = new Worker('scene-summary', summaryProcessor, {
  concurrency: 2,
})
```

### Context Limits (`src/schemas/template.schema.ts`)

```typescript
{
  max_context_memories: 25,  // 5-50 range
  max_lore_results: 10,      // 3-20 range
}
```

### LLM Parameters (`worker/processors/generation.processor.ts`)

```typescript
const MAX_CONTEXT_TOKENS = 6000

// Generation
const rawResponse = await callLLM({
  model: modelId,
  temperature: 0.85,
  maxTokens: 800,
})

// Memory Extraction
temperature: 0.3
maxTokens: 500

// Scene Summary
temperature: 0.3
maxTokens: 400
```

## CORS Configuration

```typescript
// src/index.ts
.use(cors({ origin: env.CLIENT_ORIGINS }))
```

**Example Origins**:
```
Development: http://localhost:3000,http://localhost:8080
Production:  https://app.everlore.com,https://everlore.com
```

## Development Configuration

### Local Development Stack

```yaml
# docker-compose.dev.yml
version: '3.8'
services:
  mongo:
    image: mongo:7.0
    ports:
      - "27017:27017"
    volumes:
      - mongo_dev:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mongo_dev:
```

### Start Development Environment

```bash
# 1. Start infrastructure
docker-compose -f docker-compose.dev.yml up -d

# 2. Copy environment
cp .env.example .env
# Edit .env with your API keys

# 3. Install dependencies
bun install

# 4. Start API (terminal 1)
bun run dev

# 5. Start Worker (terminal 2)
bun run dev:worker
```

### Debug Configuration

```bash
# Debug API
bun --inspect run src/index.ts

# Debug Worker
bun --inspect run worker/index.ts
```

## Production Configuration

### Environment Checklist

- [ ] `JWT_SECRET`: 32+ character random string
- [ ] `MONGODB_URI`: Production MongoDB with auth
- [ ] `REDIS_URL`: Production Redis with auth
- [ ] `OPENAI_API_KEY`: Production API key
- [ ] `PINECONE_API_KEY`: Production API key
- [ ] `CLIENT_ORIGINS`: Production domain only
- [ ] `PORT`: 80 or 443 (or use reverse proxy)

### Security Hardening

```bash
# Remove dev defaults
unset JWT_SECRET  # Force explicit setting

# Use strong secrets
JWT_SECRET=$(openssl rand -base64 48)

# Restrict CORS
CLIENT_ORIGINS=https://app.everlore.com
```

### Health Check

```bash
# API health
curl https://api.everlore.com/health

# Expected response
{"ok":true,"timestamp":"2025-01-15T10:30:00.000Z"}
```

## Configuration Validation

The server validates configuration on startup:

```typescript
// src/config/env.ts
function loadEnv(): Env {
  const missing: string[] = []
  for (const key of requiredEnvVars) {
    if (!process.env[key]) missing.push(key)
  }
  if (missing.length > 0) {
    console.warn(`Warning: Missing env vars: ${missing.join(', ')}`)
  }
  // ...
}
```

**Warnings in Development**: Missing vars use defaults
**Errors in Production**: Should fail fast (add explicit checks)

## Secrets Management

### Local Development

`.env` file (gitignored):
```bash
# Never commit this file!
OPENAI_API_KEY=sk-...
```

### Production

**Option 1: Cloud Provider Secret Manager**
- AWS Secrets Manager
- GCP Secret Manager
- Azure Key Vault

**Option 2: Kubernetes Secrets**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: everlore-secrets
type: Opaque
stringData:
  JWT_SECRET: "..."
  OPENAI_API_KEY: "..."
```

**Option 3: Docker Secrets**
```bash
echo "my-secret" | docker secret create jwt_secret -
```
