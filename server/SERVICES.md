# Everlore Server - Service Layer Architecture

## Overview

The service layer implements business logic, separating route handlers from data operations. Services are organized by domain and provide a clean API for the route layer.

## Service Structure

```
src/services/
├── template.service.ts      # World template management
├── instance.service.ts      # Game instance lifecycle
├── generation.service.ts    # AI generation orchestration
├── memory.service.ts        # Chronicle (events/memories) management
├── rag.service.ts          # Retrieval Augmented Generation
└── model-router.service.ts # LLM model selection
```

---

## Template Service (`template.service.ts`)

Manages world templates - blueprints for game instances.

### Interface

```typescript
export const templateService = {
  create(creatorId: string, data: CreateTemplateData): Promise<Template>
  update(templateId: string, creatorId: string, data: PartialTemplate): Promise<Template>
  publish(templateId: string, creatorId: string): Promise<{ success: boolean }>
  getById(templateId: string): Promise<Template | null>
  listPublished(page: number, limit: number, search?: string): Promise<PaginatedTemplates>
  listByCreator(creatorId: string): Promise<Template[]>
}
```

### Key Operations

#### Create Template
- Generates unique slug from title
- Appends timestamp if slug collision
- Sets default model preferences
- Validates creator tier (creator/premium required)

#### Publish Template
- Sets `is_published: true`
- Increments version counter
- Embeds and upserts `global_lore` to Pinecone
- Chunks lore into ~500 character segments
- Creates vectors in `lore_{templateId}` namespace

### Slug Generation
```typescript
function slugify(text: string): string {
  return text
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '')
}
```

### Lore Chunking Strategy
- Split by sentence boundaries (`/[.!?]\s+/`)
- Target: ~500 characters per chunk
- Batch upsert: 100 vectors at a time

---

## Instance Service (`instance.service.ts`)

Manages player game instances and session state.

### Interface

```typescript
export const instanceService = {
  create(playerId: string, templateId: string, tier: string): Promise<{ instance: Instance, template: Template }>
  getById(instanceId: string, playerId: string): Promise<Instance | null>
  list(playerId: string, includeArchived: boolean): Promise<EnrichedInstance[]>
  archive(instanceId: string, playerId: string): Promise<{ success: boolean }>
  loadSession(instanceId: string, playerId: string): Promise<SessionData>
}
```

### Tier Limits

```typescript
const TIER_LIMITS = {
  free: { max_instances: 3, max_memories: 100 },
  premium: { max_instances: 20, max_memories: 500 },
  creator: { max_instances: 50, max_memories: 1000 },
}
```

### Instance Creation Flow

1. **Validate Template**
   - Must exist and be published
   - Uses `template_id` from request

2. **Check Tier Limits**
   - Count non-archived instances for player
   - Throw if limit exceeded

3. **Initialize State**
   ```typescript
   // World state from template defaults
   worldState[stat] = template.base_stats_template[stat].default
   
   // Flags from template defaults
   activeFlags[flag] = template.flag_definitions[flag].default
   ```

4. **Set Initial Scene**
   ```typescript
   current_scene = {
     tag: 'dialogue',
     turn_count: 0,
     summary_pending: false,
   }
   ```

### Session Management

#### Load Session

Checks Redis cache first, falls back to MongoDB:

```typescript
// Try Redis
const cached = await redis.get(`session:${instanceId}`)
if (cached) return JSON.parse(cached)

// Load from MongoDB
const instance = await coll('world_instances').findOne({ ... })
const template = await coll('world_templates').findOne({ ... })

// Build session object
const session = {
  world_state: instance.world_state,
  active_flags: instance.active_flags,
  current_scene: instance.current_scene,
  seed_prompt: template.seed_prompt,
  global_lore: template.global_lore,
  // ... more fields
}

// Cache in Redis (1-hour TTL)
await redis.set(`session:${instanceId}`, JSON.stringify(session), 'EX', 3600)
```

#### Archive Instance

- Sets `meta.is_archived: true`
- Clears Redis session cache
- Instance excluded from default list queries

---

## Generation Service (`generation.service.ts`)

Orchestrates AI narrative generation through the job queue.

### Interface

```typescript
export const generationService = {
  dispatch(params: {
    instanceId: string
    playerId: string
    userMessage: string
  }): Promise<string>  // Returns job ID
  
  loadInstance(instanceId: string, playerId: string): Promise<LoadedInstance>
}
```

### Dispatch Flow

1. **Load Session**
   - Uses `instanceService.loadSession()` (with caching)
   
2. **Fetch Recent Events**
   - Last 6 events for immediate context
   - Reverse order for chronological prompt building

3. **Fetch Active Summary**
   - Most recent scene summary covering earlier events
   - Provides long-term context without loading full history

4. **Enqueue Job**
   ```typescript
   const job = await queue.add('generate', {
     instanceId,
     playerId,
     userMessage,
     session,        // Cached session data
     recentEvents,   // Last 6 events
     activeSummary,  // Scene summary or null
   }, {
     priority: 1,
     attempts: 2,
     backoff: { type: 'exponential', delay: 3000 }
   })
   ```

### Job Priority Levels

| Queue | Priority | Description |
|-------|----------|-------------|
| generation | 1 | User-facing, highest priority |
| memory-curation | 5 | Background memory extraction |
| scene-summary | 10 | Compress scene history |
| maintenance | 20 | Scheduled cleanup tasks |

---

## Memory Service (`memory.service.ts`)

Manages chronicle features - events and memories.

### Interface

```typescript
export const memoryService = {
  // Events
  getEvents(instanceId: string, playerId: string, opts: QueryOptions): Promise<PaginatedEvents>
  editEvent(eventId: string, playerId: string, updates: EventUpdates): Promise<{ success: boolean }>
  
  // Memories
  getMemories(instanceId: string, playerId: string, opts: QueryOptions): Promise<Memory[]>
  editMemory(memoryId: string, playerId: string, updates: MemoryUpdates): Promise<{ success: boolean }>
  deleteMemory(memoryId: string, playerId: string): Promise<{ success: boolean }>
}
```

### Event Operations

#### Get Events
- Pagination with `page` and `limit`
- Optional filtering by `type`
- Returns in chronological order (oldest first)

#### Edit Event
- Validates ownership via `player_id`
- Pushes previous data to `edit_history`
- Sets `is_user_edited: true`

### Memory Operations

#### Get Memories
- Sorted by importance (descending)
- Optional inclusion of archived memories

#### Edit Memory
1. Update MongoDB document
2. If text changed:
   - Re-embed new text
   - Update existing Pinecone vector, OR
   - Create new vector if previously archived

#### Delete Memory
1. Delete from MongoDB
2. Delete vector from Pinecone (if exists)
3. Decrement instance memory count

---

## RAG Service (`rag.service.ts`)

Implements Retrieval Augmented Generation for context-aware AI responses.

### Interface

```typescript
export async function queryRag(
  templateId: string,
  instanceId: string,
  queryText: string,
  maxLoreResults: number,
  maxMemoryResults: number,
): Promise<{
  loreTexts: string[]
  memoryTexts: string[]
  retrievedMemoryMongoIds: string[]
}>
```

### Query Flow

1. **Embed Query**
   ```typescript
   const queryEmbedding = await embed(queryText)
   ```

2. **Parallel Vector Queries**
   ```typescript
   const [loreResults, memoryResults] = await Promise.all([
     pinecone.namespace(`lore_${templateId}`).query({ ... }),
     pinecone.namespace(`mem_${instanceId}`).query({ ... })
   ])
   ```

3. **Extract Text & IDs**
   ```typescript
   const loreTexts = loreResults.matches.map(m => m.metadata.text)
   const memoryTexts = memoryResults.matches.map(m => m.metadata.text)
   const mongoIds = memoryResults.matches.map(m => m.metadata.mongo_id)
   ```

4. **Update Access Metrics**
   ```typescript
   await coll('memories').updateMany(
     { _id: { $in: mongoIds } },
     { $inc: { access_count: 1 }, $set: { last_accessed_at: new Date() } }
   )
   ```

### Use Cases

- **Lore Retrieval**: Find relevant world background for current context
- **Memory Recall**: Retrieve past interactions and established facts
- **Importance Scoring**: Track which memories are frequently accessed

---

## Model Router Service (`model-router.service.ts`)

Routes requests to appropriate LLM based on content type and preferences.

### Interface

```typescript
export function isOpenAIModel(model: string): boolean
export function selectNarrationModel(preferences: ModelPreferences, isNsfw: boolean): string
export function selectLogicModel(preferences: ModelPreferences): string
export function selectSummaryModel(preferences: ModelPreferences): string
```

### Model Sets

```typescript
const OPENAI_MODELS = new Set([
  'gpt-4o',
  'gpt-4o-mini',
  'gpt-3.5-turbo'
])
```

### Selection Logic

#### Narration Model
```typescript
if (isNsfw && preferences.narration_nsfw) {
  return preferences.narration_nsfw  // e.g., 'pygmalionai/mythalion-13b'
}
return preferences.narration_sfw || 'gpt-4o'
```

#### Logic Model
```typescript
return preferences.logic || 'gpt-4o'
```

#### Summary Model
```typescript
return preferences.summary || 'gpt-4o-mini'
```

### Provider Routing

The LLM client determines provider based on model name:

```typescript
const client = OPENAI_MODELS.has(model) 
  ? getOpenAI()      // OpenAI API
  : getOpenRouter()  // OpenRouter API (alternate models)
```

---

## Service Dependencies

```
template.service
  ├── mongo (world_templates collection)
  ├── pinecone (lore namespace)
  └── embedding utility

instance.service
  ├── mongo (world_instances collection)
  └── redis (session cache)

generation.service
  ├── instance.service (loadSession)
  ├── mongo (events collection)
  └── bullmq (generation queue)

memory.service
  ├── mongo (events, memories collections)
  └── pinecone (memory namespace)

rag.service
  ├── pinecone (lore + memory namespaces)
  ├── embedding utility
  └── mongo (update access counts)

model-router.service
  └── (pure logic, no external deps)
```

---

## Error Handling Strategy

All services follow consistent error patterns:

### Not Found Errors
```typescript
const existing = await coll('xxx').findOne({ _id, user_id: userId })
if (!existing) throw new Error('Xxx not found')
```

Results in HTTP 404 response.

### Validation Errors
```typescript
if (instanceCount >= limits.max_instances) {
  throw new Error(`Instance limit reached (${limits.max_instances})`)
}
```

Results in HTTP 400 response.

### Authorization Errors
```typescript
if (user.tier !== 'creator' && user.tier !== 'premium') {
  throw new Error('Creator or premium tier required')
}
```

Results in HTTP 403 response.

---

## Best Practices

1. **Service Purity**: Services don't access request/response objects directly
2. **Ownership Checks**: Always verify `player_id` matches authenticated user
3. **Atomic Updates**: Use MongoDB atomic operations where possible
4. **Caching Strategy**: Cache session data in Redis with appropriate TTL
5. **Error Messages**: User-friendly messages for client display
6. **Type Safety**: Use TypeScript interfaces for all data structures
