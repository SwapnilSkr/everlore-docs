# Everlore Server - Data Model

## ID Format

All entities use custom string IDs with prefixes:

| Entity | Prefix | Example |
|--------|--------|---------|
| User | `usr_` | `usr_a1b2c3d4e5f6` |
| Template | `tpl_` | `tpl_g7h8i9j0k1l2` |
| Instance | `inst_` | `inst_m3n4o5p6q7r8` |
| Event | `evt_` | `evt_s9t0u1v2w3x4` |
| Memory | `mem_` | `mem_y5z6a7b8c9d0` |
| Summary | `sum_` | `sum_e1f2g3h4i5j6` |
| Vector | `vec_` | `vec_k7l8m9n0o1p2` |
| Dead Letter | `dlq_` | `dlq_q3r4s5t6u7v8` |

## MongoDB Collections

### users

Stores user accounts and preferences.

```typescript
{
  _id: string;                    // usr_*
  email: string;                  // Unique
  username: string;               // Unique, 3-30 chars, alphanumeric + underscore
  password_hash: string;          // Argon2 hash
  tier: 'free' | 'premium' | 'creator';
  preferences: {
    nsfw_enabled: boolean;        // Default: false
    preferred_model: string;      // Default: 'gpt-4o'
    theme: string;                // Default: 'dark'
    narration_length: 'concise' | 'detailed' | 'verbose';  // Default: 'detailed'
    auto_memory_curation: boolean; // Default: true
  };
  token_balance: number;          // Default: 15000
  created_at: Date;
  updated_at: Date;
}
```

**Indexes:**
- `{ email: 1 }` - Unique
- `{ username: 1 }` - Unique

---

### world_templates

Blueprint for world instances. Created by users with creator/premium tier.

```typescript
{
  _id: string;                    // tpl_*
  creator_id: string;             // Reference to users._id
  title: string;                  // 1-200 chars
  slug: string;                   // URL-friendly, unique
  description: string;            // 1-2000 chars
  is_published: boolean;          // Default: false
  is_sentient: boolean;           // Whether AI responds in first person
  is_nsfw_capable: boolean;       // Whether NSFW model routing is enabled
  version: number;                // Auto-incremented on publish
  
  // Core content
  seed_prompt: string;            // 10-10000 chars, world premise
  global_lore: string;            // Max 50000 chars, searchable lore
  
  // Template definitions
  base_stats_template: {
    [statName: string]: {
      default: number;
      min: number;
      max: number;
      description: string;
    }
  };
  
  flag_definitions: {
    [flagName: string]: {
      type: 'boolean' | 'integer' | 'string';
      default: any;
      description: string;
    }
  };
  
  scene_tags: string[];           // Suggested scene categories
  
  // Model routing preferences
  model_preferences: {
    logic?: string;               // Default: 'gpt-4o'
    narration_nsfw?: string;      // Default: 'pygmalionai/mythalion-13b'
    narration_sfw?: string;       // Default: 'gpt-4o'
    summary?: string;             // Default: 'gpt-4o-mini'
  };
  
  // Context limits
  max_context_memories: number;   // Default: 25, range: 5-50
  max_lore_results: number;       // Default: 10, range: 3-20
  
  created_at: Date;
  updated_at: Date;
}
```

**Indexes:**
- `{ is_published: 1, created_at: -1 }` - Browse published templates
- `{ creator_id: 1 }` - List creator's templates
- `{ slug: 1 }` - Unique, lookup by slug

---

### world_instances

A player's active game session based on a template.

```typescript
{
  _id: string;                    // inst_*
  template_id: string;            // Reference to world_templates._id
  template_version: number;       // Snapshot of template version at creation
  player_id: string;              // Reference to users._id
  
  // Runtime state (mutable)
  world_state: {
    [statName: string]: number;   // Current stat values (0-100)
  };
  
  active_flags: {
    [flagName: string]: any;      // Current flag values
  };
  
  current_scene: {
    tag: string;                  // Current scene type
    turn_count: number;           // Turns in current scene
    summary_pending: boolean;     // Whether summary generation needed
  };
  
  // Metadata
  meta: {
    total_events: number;         // Count of events in instance
    total_memories: number;       // Count of memories extracted
    total_tokens_consumed: number;
    last_active_at: Date;
    is_archived: boolean;         // Soft delete flag
  };
  
  created_at: Date;
  updated_at: Date;
}
```

**Indexes:**
- `{ player_id: 1, 'meta.is_archived': 1 }` - List player's instances
- `{ template_id: 1 }` - Find instances by template
- `{ 'meta.last_active_at': 1 }` - Sort by activity

**Tier Limits:**
| Tier | Max Instances | Max Memories |
|------|--------------|--------------|
| free | 3 | 100 |
| premium | 20 | 500 |
| creator | 50 | 1000 |

---

### events

Individual narrative exchanges between player and AI.

```typescript
{
  _id: string;                    // evt_*
  instance_id: string;            // Reference to world_instances._id
  player_id: string;              // Reference to users._id
  sequence: number;               // Auto-incrementing per instance
  
  type: 'narration' | 'intimate' | 'system' | 'memory';
  scene_tag: string;              // dialogue, combat, intimate, etc.
  
  data: {
    player_input: string;         // What player said
    ai_response: string;          // AI narrative response
    state_mutations: {
      [statName: string]: {
        op: 'add' | 'subtract' | 'set';
        value: number;
      }
    };
    flag_mutations: {
      [flagName: string]: {
        op: 'set' | 'increment' | 'decrement';
        value?: any;
      }
    };
    model_used: string;           // Which LLM generated response
    tokens_in: number;
    tokens_out: number;
  };
  
  is_user_edited: boolean;        // Whether player edited this event
  edit_history: Array<{
    previous_data: object;
    edited_at: Date;
  }>;
  
  created_at: Date;
  updated_at?: Date;
}
```

**Indexes:**
- `{ instance_id: 1, sequence: 1 }` - Unique, event ordering
- `{ instance_id: 1, type: 1 }` - Filter events by type
- `{ instance_id: 1, scene_tag: 1 }` - Filter by scene

---

### memories

Long-term facts extracted from narrative events.

```typescript
{
  _id: string;                    // mem_*
  instance_id: string;            // Reference to world_instances._id
  player_id: string;              // Reference to users._id
  
  text: string;                   // Memory content (1-1000 chars)
  type: 'relationship' | 'promise' | 'lore' | 'observation' | 'emotion' | 'secret';
  importance: number;             // 1-5 scale
  is_nsfw: boolean;
  
  source_event_ids: string[];     // Events that contributed to this memory
  pinecone_id: string;            // Vector ID in Pinecone (null if archived)
  
  // Usage tracking
  access_count: number;           // Times retrieved via RAG
  last_accessed_at: Date;
  
  is_archived: boolean;           // Soft delete (removed from Pinecone)
  created_at: Date;
  updated_at: Date;
}
```

**Indexes:**
- `{ instance_id: 1, is_archived: 1, importance: -1 }` - List active memories
- `{ instance_id: 1, type: 1 }` - Filter by memory type
- `{ last_accessed_at: 1, importance: 1 }` - Maintenance: find stale memories
- `{ pinecone_id: 1 }` - Unique (sparse), vector lookup

**Memory Types:**
| Type | Description |
|------|-------------|
| `relationship` | Dynamics between characters |
| `promise` | Commitments made |
| `lore` | World facts discovered |
| `observation` | Notable observations |
| `emotion` | Emotional states |
| `secret` | Hidden information |

---

### scene_summaries

Compressed summaries of long scene sequences.

```typescript
{
  _id: string;                    // sum_*
  instance_id: string;            // Reference to world_instances._id
  scene_tag: string;              // Type of scene summarized
  
  event_range: {
    start_sequence: number;
    end_sequence: number;
  };
  
  summary_text: string;           // 2-paragraph compressed narrative
  key_facts_extracted: string[];  // Important facts identified
  
  model_used: string;
  tokens_consumed: number;
  
  created_at: Date;
}
```

**Indexes:**
- `{ instance_id: 1, 'event_range.start_sequence': 1 }` - Find summaries for events

**Generation Trigger:**
- Generated after 12 turns in the same scene tag
- Provides context for RAG without loading full history

---

### dead_letter_jobs

Failed jobs that exceeded retry attempts.

```typescript
{
  _id: string;                    // dlq_*
  queue: string;                  // Which queue (e.g., 'generation')
  jobId: string;                  // Original BullMQ job ID
  data: object;                   // Original job data
  error: string;                  // Error message
  stack?: string;                 // Stack trace
  failedAt: Date;
}
```

**Purpose:**
- Debugging failed generations
- Manual retry capability
- Client notification of permanent failures

## Pinecone Vector Schema

### Lore Namespace (`lore_{templateId}`)

World lore embedded for semantic search.

```typescript
{
  id: string;                     // lore_{templateId}_{chunkIndex}
  values: number[];               // 1536-dimensional embedding
  metadata: {
    text: string;                 // Original text chunk (~500 chars)
    type: 'lore';
    importance: 5;                // Lore is always high importance
    is_nsfw: boolean;
    created_at: string;           // ISO 8601
  }
}
```

**Chunking Strategy:**
- Split by sentence boundaries
- Target ~500 characters per chunk
- Upserted when template is published

### Memory Namespace (`mem_{instanceId}`)

Extracted memories embedded for RAG.

```typescript
{
  id: string;                     // Vector ID (vec_*)
  values: number[];               // 1536-dimensional embedding
  metadata: {
    text: string;                 // Memory content
    type: 'relationship' | 'promise' | 'lore' | 'observation' | 'emotion' | 'secret';
    importance: number;           // 1-5
    is_nsfw: boolean;
    mongo_id: string;             // Reference to memories._id
    created_at: string;           // ISO 8601
  }
}
```

**Query Parameters:**
- `topK`: `max_context_memories` (default 25)
- Retrieved memories update `access_count` and `last_accessed_at`

## Redis Key Patterns

### Session Cache
```
Key: session:{instanceId}
Value: JSON string of session data
TTL: 3600 seconds (1 hour)
```

Session cache includes:
- `world_state`
- `active_flags`
- `current_scene`
- `seed_prompt`
- `global_lore`
- `is_sentient`
- `is_nsfw_capable`
- `model_preferences`
- `max_context_memories`
- `max_lore_results`
- `template_id`

### Generation Locks
```
Key: lock:gen:{playerId}:{instanceId}
Value: Job ID
TTL: 30 seconds
```

Prevents duplicate generation requests while processing.

### Rate Limiting
```
Key: rl:{action}:{userId}
Value: Request count
TTL: Window duration (varies by action)
```

**Actions & Limits:**
| Action | Max Requests | Window |
|--------|-------------|--------|
| chat | 10 | 60 seconds |
| memory_edit | 30 | 3600 seconds |
| template_create | 5 | 86400 seconds |
| auth_attempt | 10 | 300 seconds |

### Pub/Sub Channels
```
Channel: user:{userId}:events
Message: JSON event object
```

Events published:
- `generation_complete` - AI response ready
- `generation_failed` - Generation error
- `memories_curated` - New memories extracted
- `stream_progress` - Future: streaming support

## Data Flow Summary

### Template Creation → Publication
1. User creates template → `world_templates` collection
2. User publishes template
3. System chunks `global_lore`
4. System embeds each chunk
5. System upserts to Pinecone (`lore_{templateId}`)

### Instance Creation
1. User selects template
2. System validates tier limits
3. System initializes `world_state` from `base_stats_template`
4. System initializes `active_flags` from `flag_definitions`
5. System creates `world_instances` document

### Chat Turn
1. Player sends message via WebSocket
2. System loads session (Redis cache first)
3. System queries Pinecone (RAG)
4. System calls LLM with full context
5. System parses structured response
6. System persists event to `events` collection
7. System updates `world_instances` state
8. System enqueues memory curation job

### Memory Extraction (Async)
1. Memory Worker picks up curation job
2. Worker calls LLM to extract facts
3. Worker embeds each memory
4. Worker upserts to Pinecone (`mem_{instanceId}`)
5. Worker inserts to `memories` collection

### Scene Summary (Async)
1. After 12 turns in same scene, enqueue summary job
2. Summary Worker loads last 12 events
3. Worker calls LLM to compress narrative
4. Worker inserts to `scene_summaries` collection
5. Worker updates `current_scene.summary_pending` to false

### Maintenance (Scheduled)
1. Daily: Importance decay job archives stale low-importance memories
2. Weekly: Dedup scheduler finds instances with >20 memories
3. For each instance: Dedup job merges similar memories (cosine similarity > 0.95)
