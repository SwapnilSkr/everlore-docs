# Everlore Server - Worker Architecture

## Overview

Workers handle CPU-intensive and background tasks asynchronously, keeping the API server responsive. The worker system uses BullMQ for reliable job processing with features like retries, priority, and rate limiting.

## Worker Architecture

```
                    ┌─────────────────────────────────────┐
                    │         Worker Process              │
                    │         (worker/index.ts)           │
                    └─────────────────────────────────────┘
                                      │
           ┌──────────────────────────┼──────────────────────────┐
           │                          │                          │
           ▼                          ▼                          ▼
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   Generation Worker │  │   Memory Worker     │  │   Summary Worker    │
│   (Priority: High)  │  │   (Priority: Med)   │  │   (Priority: Low)   │
│   Concurrency: 3    │  │   Concurrency: 5    │  │   Concurrency: 2    │
│   Rate: 10/min      │  │   Rate: 20/min      │  │   No limit          │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
           │                          │                          │
           └──────────────────────────┼──────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │      Maintenance Worker             │
                    │      (Scheduled Tasks)              │
                    │      Concurrency: 1                 │
                    └─────────────────────────────────────┘
```

## Queue Configuration

### Worker Setup (`worker/index.ts`)

```typescript
const generationWorker = new Worker('generation', generationProcessor, {
  connection,
  concurrency: 3,
  limiter: { max: 10, duration: 60000 },  // 10 per minute
})

const memoryWorker = new Worker('memory-curation', memoryProcessor, {
  connection,
  concurrency: 5,
  limiter: { max: 20, duration: 60000 },  // 20 per minute
})

const summaryWorker = new Worker('scene-summary', summaryProcessor, {
  connection,
  concurrency: 2,
})

const maintenanceWorker = new Worker('maintenance', maintenanceProcessor, {
  connection,
  concurrency: 1,
})
```

### Scheduled Maintenance Jobs

```typescript
// Daily at 3 AM - Importance decay
await maintenanceQueue.add('decay', 
  { task: 'importance_decay' },
  { repeat: { pattern: '0 3 * * *' }, priority: 20 }
)

// Weekly on Sunday at 4 AM - Schedule deduplications
await maintenanceQueue.add('dedup-scheduler',
  { task: 'schedule_dedups' },
  { repeat: { pattern: '0 4 * * 0' }, priority: 20 }
)
```

---

## Generation Processor (`worker/processors/generation.processor.ts`)

Handles AI narrative generation for player messages.

### Job Data

```typescript
{
  instanceId: string
  playerId: string
  userMessage: string
  session: {
    world_state: Record<string, number>
    active_flags: Record<string, any>
    current_scene: { tag: string, turn_count: number }
    seed_prompt: string
    global_lore: string
    is_sentient: boolean
    is_nsfw_capable: boolean
    model_preferences: ModelPreferences
    max_context_memories: number
    max_lore_results: number
    template_id: string
  }
  recentEvents: Event[]  // Last 6 events
  activeSummary: string | null  // Scene summary
}
```

### Processing Flow

```
1. RAG Query
   ├── Embed user message
   ├── Query lore_{templateId} namespace (topK: max_lore_results)
   └── Query mem_{instanceId} namespace (topK: max_context_memories)

2. Prompt Building
   ├── Build system prompt with world context
   ├── Add scene summary (if exists)
   ├── Add recent events (token-budgeted)
   └── Add current user message

3. Model Selection
   ├── Default: model_preferences.narration_sfw
   └── If nsfw_capable && classifyScene() === 'nsfw':
       Use model_preferences.narration_nsfw

4. LLM Call
   ├── Model: Selected model
   ├── Temperature: 0.85
   ├── Max tokens: 800
   └── Response format: JSON Schema

5. Response Processing
   ├── Parse JSON (with repair fallback)
   ├── Validate state_mutations
   └── Validate flag_mutations

6. State Application
   ├── Apply mutations to world_state
   ├── Apply mutations to active_flags
   └── Clamp values to defined limits

7. Persistence
   ├── Create event document
   ├── Insert to events collection
   └── Update world_instances state

8. Session Update
   ├── Update Redis cache
   ├── Release generation lock
   └── Publish completion to user's Redis channel

9. Follow-up Jobs
   ├── Enqueue memory-curation (delay: 1s, priority: 5)
   └── If turn_count >= 12: Enqueue scene-summary (delay: 5s, priority: 10)
```

### JSON Schema for LLM Output

```typescript
{
  narrative: string           // AI response text
  state_mutations: {
    [statName]: {
      op: 'add' | 'subtract' | 'set'
      value: number
    }
  }
  flag_mutations: {
    [flagName]: {
      op: 'set' | 'increment' | 'decrement'
      value?: any
    }
  }
  scene_tag: 'dialogue' | 'combat' | 'intimate' | 
             'exploration' | 'existential' | 'cosmic' | 'mundane'
  emotional_tone: string
}
```

### Error Handling

- **Retry**: 2 attempts with exponential backoff (3s delay)
- **Dead Letter**: After max retries, stored in `dead_letter_jobs` collection
- **Client Notification**: Failure published to user's Redis channel
- **Lock Release**: Always released, even on failure

---

## Memory Processor (`worker/processors/memory.processor.ts`)

Extracts important facts from player-AI exchanges for long-term memory.

### Job Data

```typescript
{
  instanceId: string
  playerId: string
  eventId: string
  playerInput: string
  aiResponse: string
  sceneTag: string
}
```

### Extraction Prompt

```
You are a memory curator for a narrative engine. Given the following exchange 
between a player and a world, extract 0-3 important facts that should be 
remembered long-term.

Rules:
- Only extract facts that would matter 10+ turns from now
- Each fact must be a self-contained sentence
- Rate importance 1-5 (5 = critical plot point, 1 = minor detail)
- Classify type: relationship, promise, lore, observation, emotion, secret
- Flag if NSFW content is referenced
- If nothing is worth remembering, return an empty array
```

### Memory Types

| Type | Description | Example |
|------|-------------|---------|
| `relationship` | Character dynamics | "The guard respects the player for their honesty" |
| `promise` | Commitments made | "The player promised to return the amulet" |
| `lore` | World facts discovered | "The temple was built in the Age of Stars" |
| `observation` | Notable observations | "The player noticed the hidden door" |
| `emotion` | Emotional states | "The player felt guilty about the lie" |
| `secret` | Hidden information | "The player knows the mayor is corrupt" |

### Processing Flow

```
1. Call LLM (gpt-4o) with extraction prompt
2. Parse JSON response
3. For each extracted memory:
   a. Generate IDs (mem_*, vec_*)
   b. Embed memory text
   c. Upsert vector to Pinecone (mem_{instanceId} namespace)
   d. Insert document to memories collection
4. Update instance meta.total_memories
5. Publish memories_curated event to client
```

### Vector Metadata

```typescript
{
  text: string          // Memory content
  type: string          // Memory type
  importance: number    // 1-5
  is_nsfw: boolean
  mongo_id: string      // Link to MongoDB document
  created_at: string    // ISO timestamp
}
```

---

## Summary Processor (`worker/processors/summary.processor.ts`)

Compresses long scene sequences into compact summaries for context efficiency.

### Job Data

```typescript
{
  instanceId: string
  sceneTag: string
  startSequence: number
  endSequence: number
}
```

### Trigger Condition

- After 12 turns in the same scene tag
- Provides context for RAG without loading full history

### Processing Flow

```
1. Load events from startSequence to endSequence
2. Skip if < 6 events (not worth summarizing)
3. Format conversation text
4. Call LLM (gpt-4o-mini) with compression prompt
5. Create summary document
6. Insert to scene_summaries collection
7. Update instance current_scene.summary_pending = false
```

### Compression Prompt

```
Compress the following scene into exactly 2 paragraphs. Preserve: 
- Key decisions
- Emotional shifts  
- Promises made/broken
- State changes
- NSFW content descriptors (note "intimate scene occurred", no explicit content)

The summary must be self-contained and understandable without the original text.
```

### Summary Document

```typescript
{
  _id: string                    // sum_*
  instance_id: string
  scene_tag: string
  event_range: {
    start_sequence: number
    end_sequence: number
  }
  summary_text: string           // 2-paragraph compressed narrative
  key_facts_extracted: string[]  // Important facts identified
  model_used: string             // 'gpt-4o-mini'
  tokens_consumed: number
  created_at: Date
}
```

---

## Maintenance Processor (`worker/processors/maintenance.processor.ts`)

Handles scheduled cleanup and optimization tasks.

### Task: `importance_decay`

Archives stale, low-importance memories to manage vector storage.

**Schedule**: Daily at 3:00 AM

**Criteria for Archival:**
```typescript
{
  importance: { $lt: 3 },           // Low importance (1-2)
  last_accessed_at: { $lt: 30d },   // Not accessed in 30 days
  is_archived: false                 // Not already archived
}
```

**Process:**
```
1. Find stale memories matching criteria
2. For each memory:
   a. Delete vector from Pinecone (if pinecone_id exists)
   b. Update MongoDB: is_archived = true, pinecone_id = null
3. Return count archived
```

---

### Task: `dedup_memories`

Merges semantically similar memories to reduce redundancy.

**Schedule**: Triggered by `schedule_dedups` scheduler

**Similarity Threshold:** cosine similarity > 0.95

**Process:**
```
1. Load all non-archived memories for instance
2. If < 2 memories, skip
3. Embed all memory texts
4. Compare all pairs with cosine similarity
5. For similar pairs:
   a. Keep higher importance memory
   b. Merge source_event_ids
   c. Delete lower importance vector from Pinecone
   d. Archive lower importance memory in MongoDB
6. Return count merged
```

---

### Task: `schedule_dedups`

Schedules deduplication for active instances with many memories.

**Schedule**: Weekly on Sunday at 4:00 AM

**Process:**
```
1. Find instances with meta.total_memories > 20
2. For each instance:
   a. Enqueue dedup_memories task
   b. Priority: 20 (low)
3. Return count scheduled
```

---

## Shared Worker Libraries

### LLM Client (`worker/lib/llm-client.ts`)

Unified interface for OpenAI and OpenRouter APIs.

```typescript
export async function callLLM(request: {
  model: string
  messages: Array<{ role: string, content: string }>
  temperature?: number      // Default: 0.8
  maxTokens?: number        // Default: 600
  responseSchema?: object   // JSON Schema (OpenAI only)
  responseFormat?: { type: string }
}): Promise<string>
```

**Provider Selection:**
```typescript
const client = OPENAI_MODELS.has(model) 
  ? getOpenAI()      // api.openai.com
  : getOpenRouter()  // openrouter.ai/api/v1
```

---

### Structured Output (`worker/lib/structured-output.ts`)

Validates and repairs LLM JSON responses.

```typescript
export interface GenerationOutput {
  narrative: string
  state_mutations: Record<string, Mutation>
  flag_mutations: Record<string, FlagMutation>
  scene_tag: string
  emotional_tone: string
}

export function enforceSchema(rawResponse: string): GenerationOutput
```

**Validation:**
- Checks required fields exist
- Validates mutation operations
- Validates scene_tag against allowed values
- Provides defaults for missing fields

**Repair Strategies:**
1. Extract JSON from markdown code blocks
2. Find JSON object in text
3. Fallback: use raw text as narrative

---

### NSFW Classifier (`worker/lib/nsfw-classifier.ts`)

Rule-based content classification for model routing.

```typescript
export function classifyScene(
  userMessage: string,
  recentEvents: Event[],
): 'sfw' | 'nsfw'
```

**Signal Detection:**
- Keyword matching (NSFW_SIGNALS set)
- Pattern matching (NSFW_PATTERNS regex)
- Scene momentum (recent intimate scenes)

**Threshold:** 3+ signals → 'nsfw'

**Signals:**
```typescript
const NSFW_SIGNALS = new Set([
  'kiss', 'kissing', 'touch', 'caress', 'stroke',
  'undress', 'naked', 'bare', 'skin', 'moan', 'gasp',
  'bed', 'embrace', 'desire', 'hunger', 'passion',
  // ... more
])
```

**Patterns:**
```typescript
const NSFW_PATTERNS = [
  /take\s+(off|your)/i,
  /come\s+closer/i,
  /i\s+want\s+(you|this)/i,
  /don['']t\s+stop/i,
  // ... more
]
```

---

## Error Handling

### Worker-Level Error Handling

```typescript
worker.on('failed', (job, err) => {
  console.error(`[${worker.name}] Job ${job?.id} failed:`, err.message)
})

worker.on('completed', (job) => {
  console.log(`[${worker.name}] Job ${job.id} completed`)
})
```

### Dead Letter Queue (Generation Only)

```typescript
generationWorker.on('failed', async (job, err) => {
  if (job && job.attemptsMade >= (job.opts.attempts || 1)) {
    // Store in dead_letter_jobs collection
    await coll('dead_letter_jobs').insertOne({
      _id: generateId('dlq'),
      queue: 'generation',
      jobId: job.id,
      data: job.data,
      error: err.message,
      stack: err.stack,
      failedAt: new Date(),
    })
    
    // Notify client
    await redis.publish(`user:${job.data.playerId}:events`, JSON.stringify({
      type: 'generation_failed',
      instanceId: job.data.instanceId,
      message: 'The world could not respond. Please try again.',
    }))
    
    // Release lock
    await redis.del(`lock:gen:${job.data.playerId}:${job.data.instanceId}`)
  }
})
```

### Graceful Shutdown

```typescript
const shutdown = async () => {
  console.log('Shutting down workers...')
  await generationWorker.close()
  await memoryWorker.close()
  await summaryWorker.close()
  await maintenanceWorker.close()
  process.exit(0)
}
process.on('SIGTERM', shutdown)
process.on('SIGINT', shutdown)
```

---

## Monitoring & Observability

### Console Logging

```typescript
// Job completion
console.log(`[${w.name}] Job ${job.id} completed`)

// Job failure
console.error(`[${w.name}] Job ${job?.id} failed:`, err.message)

// Dead letter handling failure
console.error('Failed to handle dead letter:', (dlqErr as Error).message)
```

### Recommended Additions

For production monitoring, consider:

1. **Metrics Collection**
   - Job processing time histograms
   - Queue depth gauges
   - Error rate counters

2. **Structured Logging**
   - JSON format with correlation IDs
   - Log aggregation (ELK, Datadog, etc.)

3. **Alerting**
   - High queue depth
   - Elevated error rates
   - Stuck jobs

4. **Tracing**
   - Distributed tracing for job chains
   - OpenTelemetry integration
