# AI Engine

Native Session, Context, Memory, RAG, Agent, Trace, Intent, Embedding Cache, Event Segmentation, and LLM Provider abstractions for LLM applications.

## Overview

The AI Engine is Talon's 9th engine — a first-class semantic abstraction layer purpose-built for LLM application development. It eliminates the need for external frameworks (LangChain, LlamaIndex) by providing native primitives for session management, conversation context, semantic memory (with hybrid search, EDU extraction, and knowledge graph), RAG document management, agent orchestration, execution tracing, intent recognition, embedding caching, event segmentation, and built-in LLM provider integration.

## Quick Start

```rust
use talon::{Talon, ContextMessage};
use talon_ai::{AiEngine, AiLlmConfig, LlmEndpoint, EmbedEndpoint};
use std::collections::BTreeMap;

let db = Talon::open("./data")?;
let ai = db.ai()?;

// Configure LLM (Chat + Embed)
ai.configure_llm(AiLlmConfig {
    chat: Some(LlmEndpoint {
        base_url: "https://api.openai.com/v1".into(),
        api_key: Some("sk-...".into()),
        model: "gpt-4o-mini".into(),
        max_retries: 2,
        timeout_secs: 60,
    }),
    embed: Some(EmbedEndpoint {
        base_url: "https://api.openai.com/v1".into(),
        api_key: Some("sk-...".into()),
        model: "text-embedding-3-small".into(),
        dimensions: 1536,
        timeout_secs: 30,
    }),
})?;

// Create a session
ai.create_session("chat-001", BTreeMap::new(), None)?;

// Append messages
ai.append_message("chat-001", &ContextMessage {
    role: "user".into(),
    content: "What is Talon?".into(),
    timestamp: 0, // auto-set if 0
    token_count: Some(5),
})?;

// Store memory (auto-embed, no manual vector needed)
let memory_id = ai.add_memory("User prefers Rust", BTreeMap::new(), None, false)?;

// Smart recall (Hybrid Search: BM25 + Vector + Graph + Rerank)
let results = ai.recall("Rust preference", 5, 0.4, 0.6, 0.0, false, None, 0)?;
```

## API Reference

### Session Management

#### `create_session`
```rust
pub fn create_session(
    &self,
    id: &str,
    metadata: BTreeMap<String, String>,
    ttl_secs: Option<u64>,
) -> Result<Session, Error>
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `&str` | Unique session identifier |
| `metadata` | `BTreeMap<String, String>` | Custom key-value metadata |
| `ttl_secs` | `Option<u64>` | Auto-expire after N seconds (None = never) |

Returns the created `Session`.

#### `create_session_if_not_exists`
```rust
pub fn create_session_if_not_exists(
    &self,
    id: &str,
    metadata: BTreeMap<String, String>,
    ttl_secs: Option<u64>,
) -> Result<(Session, bool), Error>
```
Idempotent session creation. Returns `(session, is_new)` — `is_new=true` means newly created. Safe for concurrent calls (e.g., group chat scenarios).

#### `get_session`
```rust
pub fn get_session(&self, id: &str) -> Result<Option<Session>, Error>
```
Returns `None` for expired sessions (lazy expiration).

#### `list_sessions`
```rust
pub fn list_sessions(&self) -> Result<Vec<Session>, Error>
```
Lists all active sessions (excludes archived and expired).

#### `delete_session`
```rust
pub fn delete_session(&self, id: &str) -> Result<(), Error>
```
Cascade delete: removes session + all context messages + trace records.

#### `update_session`
```rust
pub fn update_session(
    &self,
    id: &str,
    metadata: BTreeMap<String, String>,
) -> Result<Session, Error>
```
Merge new metadata into existing session. Existing keys are overwritten, new keys are added.

#### `cleanup_expired_sessions`
```rust
pub fn cleanup_expired_sessions(&self) -> Result<usize, Error>
```
Batch purge all expired sessions (cascade deletes context + traces). Returns count deleted.

#### Tags
```rust
pub fn add_session_tags(&self, id: &str, tags: &[String]) -> Result<(), Error>
pub fn remove_session_tags(&self, id: &str, tags: &[String]) -> Result<(), Error>
pub fn get_session_tags(&self, session_id: &str) -> Result<Vec<String>, Error>
pub fn search_sessions_by_tag(&self, tag: &str) -> Result<Vec<Session>, Error>
```

#### Archive
```rust
pub fn archive_session(&self, id: &str) -> Result<(), Error>
pub fn unarchive_session(&self, id: &str) -> Result<(), Error>
pub fn list_archived_sessions(&self) -> Result<Vec<Session>, Error>
pub fn search_sessions_by_metadata(&self, key: &str, value: &str) -> Result<Vec<Session>, Error>
```

#### Export & Stats
```rust
pub fn export_session(&self, session_id: &str) -> Result<ExportedSession, Error>
pub fn export_sessions(&self, session_ids: &[&str]) -> Result<Vec<ExportedSession>, Error>
pub fn session_stats(&self, session_id: &str) -> Result<SessionStats, Error>
pub fn sessions_stats(&self, session_ids: &[&str]) -> Result<Vec<SessionStats>, Error>
```

#### Data Types
```rust
pub struct Session {
    pub id: String,
    pub created_at: i64,              // timestamp (ms)
    pub metadata: BTreeMap<String, String>,
    pub archived: bool,
    pub expires_at: Option<i64>,      // timestamp (ms), None = never
}
```

### Conversation Context

#### `append_message`
```rust
pub fn append_message(&self, session_id: &str, msg: &ContextMessage) -> Result<(), Error>
```

```rust
pub struct ContextMessage {
    pub role: String,            // "user" | "assistant" | "system" | "tool"
    pub content: String,         // Message text
    pub timestamp: i64,          // Timestamp (ms)
    pub token_count: Option<u32>, // Token count for window management
}
```

#### `get_history`
```rust
pub fn get_history(&self, session_id: &str, limit: Option<usize>) -> Result<Vec<ContextMessage>, Error>
```
Get conversation history (ascending by time). `limit: None` returns all messages.

#### `get_recent_messages`
```rust
pub fn get_recent_messages(&self, session_id: &str, n: usize) -> Result<Vec<ContextMessage>, Error>
```
Get the most recent N messages (returned in chronological order).

#### `get_context_window`
```rust
pub fn get_context_window(&self, session_id: &str, max_tokens: u32) -> Result<Vec<ContextMessage>, Error>
```
Get messages fitting within a token budget. Takes from latest messages backwards, returns in chronological order.

#### `get_context_window_with_prompt`
```rust
pub fn get_context_window_with_prompt(&self, session_id: &str, max_tokens: u32) -> Result<Vec<ContextMessage>, Error>
```
Builds complete LLM input context: `system_prompt + context_summary + recent messages`.

Token budget allocation strategy:
1. Deduct system_prompt tokens (if set)
2. Deduct context_summary tokens (if set)
3. Remaining budget for recent messages

#### `get_context_window_smart`
```rust
pub fn get_context_window_smart(&self, session_id: &str, max_tokens: u32) -> Result<Vec<ContextMessage>, Error>
```
Smart context window with automatic compression:
- If existing summary or conversation not too long: returns `get_context_window_with_prompt()`
- If total tokens > `max_tokens × 2` and Chat Provider configured: auto-generates summary via LLM
- If no Chat Provider: falls back to simple truncation

#### `set_system_prompt` / `get_system_prompt`
```rust
pub fn set_system_prompt(&self, session_id: &str, prompt: &str, token_count: u32) -> Result<(), Error>
pub fn get_system_prompt(&self, session_id: &str) -> Result<Option<String>, Error>
```
Set/get a persistent system prompt for the session. `token_count` is used for context window budget calculation.

#### `set_context_summary` / `get_context_summary`
```rust
pub fn set_context_summary(&self, session_id: &str, summary: &str, token_count: u32) -> Result<(), Error>
pub fn get_context_summary(&self, session_id: &str) -> Result<Option<String>, Error>
```
Store/retrieve a conversation summary. Automatically inserted between system_prompt and recent messages in context window.

#### `auto_summarize`
```rust
pub fn auto_summarize(&self, session_id: &str, opts: SummarizeOptions) -> Result<String, Error>
```
Automatically generates context summary via LLM. Requires configured Chat Provider.

```rust
pub struct SummarizeOptions {
    pub max_summary_tokens: u32,    // Max summary length (default 200)
    pub purge_old: bool,            // Whether to delete summarized messages (default false)
    pub custom_prompt: Option<String>, // Custom summarize prompt (None = built-in template)
}
```

#### `compact_context`
```rust
pub fn compact_context(&self, session_id: &str, keep_recent_n: usize) -> Result<u64, Error>
```
Keep the most recent N messages, delete the rest. Returns count deleted. Typically used after `set_context_summary()` for safe context compression.

#### `clear_context`
```rust
pub fn clear_context(&self, session_id: &str) -> Result<u64, Error>
```
Clear all messages while preserving the session. Returns count deleted.

### Semantic Memory

#### Core Operations

##### `store_memory`
```rust
pub fn store_memory(&self, entry: &MemoryEntry, embedding: &[f32]) -> Result<(), Error>
```
Store a vectorized long-term memory (manual embedding).

##### `auto_store_memory`
```rust
pub fn auto_store_memory(
    &self,
    content: &str,
    metadata: BTreeMap<String, String>,
    ttl_secs: Option<u64>,
) -> Result<u64, Error>
```
Auto-embed and store memory. Requires configured Embed Provider. Returns memory ID.

##### `add_memory` ⭐
```rust
pub fn add_memory(
    &self,
    content: &str,
    metadata: BTreeMap<String, String>,
    ttl_secs: Option<u64>,
    extract_facts: bool,
) -> Result<u64, Error>
```
**Enhanced memory storage** with automatic:
1. Embedding (with cache)
2. Vector index write
3. FTS index write (for hybrid search)
4. [Optional] **EDU structured fact extraction** via LLM when `extract_facts=true`
   - Extracts participants, entities, time, event type
   - Each EDU independently embedded and stored
   - EDU entities written to **Knowledge Graph** (Graph Engine)
5. Returns memory ID

##### `recall` ⭐
```rust
pub fn recall(
    &self,
    query: &str,
    k: usize,
    fts_weight: f64,       // BM25 weight (recommended 0.4)
    vec_weight: f64,       // Vector weight (recommended 0.6)
    temporal_boost: f64,   // Time-aware weight (0.0 = off, recommended 0.3)
    rerank: bool,          // Enable LLM Rerank (requires chat config)
    rerank_top_k: Option<usize>,  // Final count after rerank
    graph_depth: usize,    // Graph expansion hops (0 = off, recommended 1-2)
) -> Result<Vec<HybridRecallResult>, Error>
```
**Smart recall pipeline**:
```text
Query → [Graph Expand] → [Hybrid Search: BM25 + Vector] → RRF
      → [Temporal Rerank] → [LLM Rerank] → Top-K
```

```rust
pub struct HybridRecallResult {
    pub entry: MemoryEntry,
    pub rrf_score: f64,           // RRF fusion score
    pub bm25_score: Option<f64>,  // BM25 score (if FTS matched)
    pub vector_dist: Option<f32>, // Vector distance (if vector matched)
}
```

#### Basic Search

##### `search_memory`
```rust
pub fn search_memory(&self, query_embedding: &[f32], k: usize) -> Result<Vec<MemorySearchResult>, Error>
```
Pure vector similarity search. Auto-skips expired memories.

##### `auto_search_memory`
```rust
pub fn auto_search_memory(&self, query: &str, k: usize) -> Result<Vec<MemorySearchResult>, Error>
```
Auto-embed query and search. Requires configured Embed Provider.

##### `search_memory_with_filter`
```rust
pub fn search_memory_with_filter(
    &self,
    embedding: &[f32],
    k: usize,
    filters: &BTreeMap<String, String>,
) -> Result<Vec<MemorySearchResult>, Error>
```
Vector search + metadata key-value filter (AND semantics).

#### Memory Management

##### `update_memory`
```rust
pub fn update_memory(
    &self,
    id: u64,
    content: Option<&str>,
    metadata: Option<BTreeMap<String, String>>,
) -> Result<(), Error>
```
Update text and/or metadata (does NOT update vector). To update vector, delete + store.

##### `delete_memory`
```rust
pub fn delete_memory(&self, id: u64) -> Result<(), Error>
```

##### `memory_count`
```rust
pub fn memory_count(&self) -> Result<u64, Error>
```

##### `list_memories`
```rust
pub fn list_memories(&self, offset: usize, limit: usize) -> Result<Vec<MemoryEntry>, Error>
```
Paginated listing, skips expired, sorted by ID ascending.

##### `store_memory_with_ttl`
```rust
pub fn store_memory_with_ttl(&self, entry: &MemoryEntry, embedding: &[f32], ttl_secs: u64) -> Result<(), Error>
```

##### `store_memories_batch`
```rust
pub fn store_memories_batch(&self, entries: &[MemoryEntry], embeddings: &[Vec<f32>]) -> Result<(), Error>
```
Batch store multiple memories. `entries` and `embeddings` must be equal length.

#### Memory Maintenance

##### `find_duplicate_memories`
```rust
pub fn find_duplicate_memories(&self, threshold: f32) -> Result<Vec<DuplicatePair>, Error>
```
Find memory pairs with cosine distance below threshold (typical: 0.05–0.1).

##### `deduplicate_memories`
```rust
pub fn deduplicate_memories(&self, threshold: f32) -> Result<usize, Error>
```
Auto-remove duplicates (keeps newer memory). Returns count removed.

##### `cleanup_expired_memories`
```rust
pub fn cleanup_expired_memories(&self) -> Result<usize, Error>
```

##### `memory_stats`
```rust
pub fn memory_stats(&self) -> Result<MemoryStats, Error>
```
```rust
pub struct MemoryStats {
    pub total: usize,
    pub expired: usize,
    pub permanent: usize,
}
```

#### Memory Knowledge Graph

Built on Talon's Graph Engine. EDU entities and relationships are automatically written to the graph when `add_memory(content, meta, ttl, extract_facts=true)` is called.

##### `graph_expand_entities`
```rust
pub fn graph_expand_entities(&self, seed_entities: &[String], max_hops: usize) -> Result<Vec<String>, Error>
```
N-hop BFS expansion from seed entities. Max hops capped at 5. Returns discovered entity names.

Use case: "Alice's friend likes what?" → Alice → friend → Bob → likes → Pizza

##### `graph_memory_stats`
```rust
pub fn graph_memory_stats(&self) -> Result<(u64, u64), Error>
```
Returns `(vertex_count, edge_count)`.

#### Data Types
```rust
pub struct MemoryEntry {
    pub id: u64,
    pub content: String,
    pub metadata: BTreeMap<String, String>,
    pub created_at: i64,           // timestamp (ms)
    pub expires_at: Option<i64>,   // timestamp (ms), None = never
}

pub struct MemorySearchResult {
    pub entry: MemoryEntry,
    pub distance: f32,
}
```

### RAG Document Management

#### Document CRUD
```rust
pub fn store_document(&self, doc: &RagDocumentWithChunks) -> Result<u64, Error>
pub fn store_document_with_ttl(&self, doc: &RagDocumentWithChunks, ttl_secs: u64) -> Result<u64, Error>
pub fn get_document(&self, doc_id: u64) -> Result<Option<RagDocumentWithChunks>, Error>
pub fn list_documents(&self) -> Result<Vec<RagDocument>, Error>
pub fn delete_document(&self, doc_id: u64) -> Result<(), Error>
pub fn document_count(&self) -> Result<usize, Error>
pub fn replace_document(&self, doc_id: u64, doc: &RagDocumentWithChunks) -> Result<(), Error>
pub fn replace_document_with_ttl(&self, doc_id: u64, doc: &RagDocumentWithChunks, ttl_secs: u64) -> Result<(), Error>
pub fn store_documents_batch(&self, docs: &[RagDocumentWithChunks]) -> Result<Vec<u64>, Error>
pub fn delete_documents_batch(&self, doc_ids: &[u64]) -> Result<usize, Error>
pub fn cleanup_expired_documents(&self) -> Result<usize, Error>
```

#### Chunk-Level Operations
```rust
pub fn get_chunk(&self, chunk_id: u64) -> Result<Option<RagChunk>, Error>
pub fn update_chunk(&self, chunk_id: u64, text: &str, embedding: &[f32]) -> Result<(), Error>
pub fn delete_chunk(&self, chunk_id: u64) -> Result<(), Error>
```

#### Search
```rust
pub fn search_chunks(&self, query_embedding: &[f32], k: usize, metadata_filter: Option<&BTreeMap<String, String>>) -> Result<Vec<RagSearchResult>, Error>
pub fn search_chunks_hybrid(&self, query_text: &str, query_embedding: &[f32], k: usize) -> Result<Vec<RagSearchResult>, Error>
pub fn search_chunks_by_keyword(&self, keyword: &str, limit: usize) -> Result<Vec<RagSearchResult>, Error>
```

#### Versioning & Metadata
```rust
pub fn get_document_version(&self, doc_id: u64) -> Result<Option<u64>, Error>
pub fn list_document_versions(&self, doc_id: u64) -> Result<Vec<u64>, Error>
pub fn get_document_at_version(&self, doc_id: u64, version: u64) -> Result<Option<RagDocumentWithChunks>, Error>
pub fn document_stats(&self) -> Result<(usize, usize, usize), Error>   // (doc_count, chunk_count, total_bytes)
pub fn search_documents_by_metadata(&self, key: &str, value: &str) -> Result<Vec<RagDocument>, Error>
```

#### Data Types
```rust
pub struct RagDocumentWithChunks {
    pub document: RagDocument,
    pub chunks: Vec<RagChunkInput>,
}
pub struct RagDocument {
    pub id: u64,
    pub title: String,
    pub source: Option<String>,
    pub metadata: Option<serde_json::Value>,
}
pub struct RagChunkInput {
    pub text: String,
    pub embedding: Vec<f32>,
    pub metadata: Option<serde_json::Value>,
}
```

### Agent Primitives

#### Tool Call Caching
```rust
pub fn cache_tool_result(&self, tool_name: &str, input_hash: &str, result: &str, ttl_secs: Option<u64>) -> Result<(), Error>
pub fn get_cached_tool_result(&self, tool_name: &str, input_hash: &str) -> Result<Option<ToolCacheEntry>, Error>
pub fn invalidate_tool_cache(&self, tool_name: &str) -> Result<u64, Error>
```
Cache expensive tool call results with TTL. `invalidate_tool_cache` removes all cached results for a tool.

```rust
pub struct ToolCacheEntry {
    pub tool_name: String,
    pub input_hash: String,
    pub result: String,        // JSON string
    pub cached_at: i64,         // timestamp (ms)
}
```

#### Agent State Persistence
```rust
pub fn save_agent_state(
    &self,
    agent_id: &str,
    step_id: &str,
    state: &str,
    metadata: BTreeMap<String, String>,
) -> Result<AgentStep, Error>

pub fn get_agent_state(&self, agent_id: &str) -> Result<Option<AgentStep>, Error>
pub fn list_agent_steps(&self, agent_id: &str) -> Result<Vec<AgentStep>, Error>
pub fn get_agent_step_count(&self, agent_id: &str) -> Result<usize, Error>
pub fn rollback_agent_to_step(&self, agent_id: &str, step_id: &str) -> Result<usize, Error>
pub fn clear_agent_steps(&self, agent_id: &str) -> Result<usize, Error>
pub fn delete_agent_state(&self, agent_id: &str) -> Result<(), Error>
```
Persist agent execution steps for checkpointing, pause/resume, and rollback.
- `rollback_agent_to_step` — removes all steps after the specified checkpoint, returns count removed
- `clear_agent_steps` — removes all step history, returns count removed
- `delete_agent_state` — deletes all agent data including step index

```rust
pub struct AgentStep {
    pub agent_id: String,
    pub step_id: String,          // Caller-defined (recommended: incremental)
    pub state: String,             // JSON string (structure defined by caller)
    pub metadata: BTreeMap<String, String>,
    pub created_at: i64,           // timestamp (ms)
}
```

### Execution Trace

#### Logging
```rust
pub fn log_trace(&self, record: &TraceRecord) -> Result<(), Error>
```

```rust
pub struct TraceRecord {
    pub session_id: String,
    pub run_id: String,
    pub operation: String,         // "llm_call", "tool_call", "embedding", etc.
    pub input_summary: String,     // Summary of input
    pub output_summary: String,    // Summary of output
    pub duration_ms: i64,
    pub token_count: i64,
    pub timestamp: i64,            // timestamp (ms)
}
```

#### Querying
```rust
pub fn query_traces_by_session(&self, session_id: &str) -> Result<Vec<TraceRecord>, Error>
pub fn query_traces_by_run(&self, run_id: &str) -> Result<Vec<TraceRecord>, Error>
pub fn query_traces_by_operation(&self, operation: &str) -> Result<Vec<TraceRecord>, Error>
pub fn query_traces_by_session_and_operation(&self, session_id: &str, operation: &str) -> Result<Vec<TraceRecord>, Error>
pub fn query_traces_in_range(&self, start_ms: i64, end_ms: i64) -> Result<Vec<TraceRecord>, Error>
pub fn export_traces(&self, session_id: Option<&str>) -> Result<Vec<TraceRecord>, Error>
```

#### Analytics
```rust
pub fn get_token_usage(&self, session_id: &str) -> Result<i64, Error>
pub fn get_token_usage_by_run(&self, run_id: &str) -> Result<i64, Error>
pub fn trace_stats(&self, session_id: Option<&str>) -> Result<TraceStats, Error>
pub fn trace_performance_report(&self, session_id: Option<&str>, slow_threshold_ms: i64) -> Result<TracePerformanceReport, Error>
```

```rust
pub struct TraceStats {
    pub total_traces: usize,
    pub total_tokens: i64,
    pub total_duration_ms: i64,
    pub tokens_by_operation: BTreeMap<String, i64>,
}

pub struct TracePerformanceReport {
    pub total_traces: usize,
    pub total_tokens: i64,
    pub avg_duration_ms: f64,
    pub max_duration_ms: i64,
    pub p95_duration_ms: i64,
    pub slow_operations: Vec<TraceRecord>,
    pub avg_duration_by_operation: BTreeMap<String, f64>,
}
```

### Embedding Cache

#### `cache_embedding`
```rust
pub fn cache_embedding(&self, content_hash: &str, embedding: &[f32]) -> Result<(), Error>
```
Cache an embedding vector. `content_hash` is caller-computed (e.g., FNV-1a, SHA256 prefix).

#### `get_cached_embedding`
```rust
pub fn get_cached_embedding(&self, content_hash: &str) -> Result<Option<Vec<f32>>, Error>
```

#### `embed_with_cache` (internal, used by `add_memory`/`recall`)
```rust
pub(crate) fn embed_with_cache(&self, text: &str) -> Result<Vec<f32>, Error>
```
Automatic embed + cache pipeline:
1. Compute FNV-1a hash of text
2. Check KV cache (hit → return immediately)
3. Call Embedding API
4. Write cache
5. Return vector

Used internally by `add_memory()` and `recall()`.

#### `invalidate_embedding_cache`
```rust
pub fn invalidate_embedding_cache(&self) -> Result<u64, Error>
```
Clear all cached embeddings. Returns count deleted.

#### `embedding_cache_count`
```rust
pub fn embedding_cache_count(&self) -> Result<usize, Error>
```

### Event Segmentation

Based on ES-Mem (Event Segmentation-Based Memory) theory. Pure Rust implementation, zero LLM calls.

#### `EventSegmenter`
```rust
pub struct EventSegmenter { ... }

pub fn new(config: &EventSegmentConfig) -> Self
pub fn with_defaults() -> Self
pub fn process_message(&mut self, embedding: &[f32], timestamp_ms: i64) -> (u64, bool)
pub fn current_event_id(&self) -> u64
pub fn restore(config, event_id, msg_count, prev_embedding, prev_timestamp) -> Self
```

`process_message` detects event boundaries using:
1. Time gap threshold (default: 30 minutes)
2. Semantic similarity threshold (default: 0.4, cosine)
3. Max event size hard limit (default: 20 messages)

Returns `(event_id, is_new_event)`.

```rust
pub struct EventSegmentConfig {
    pub similarity_threshold: f32,  // 0.0-1.0, below triggers segmentation
    pub time_gap_threshold: i64,    // ms, above triggers segmentation
    pub max_event_size: usize,      // Hard limit per event
}
```

#### Temporal Boost Utility
```rust
pub fn apply_temporal_boost(
    rrf_scores: &mut [(usize, f64)],
    created_at_times: &[i64],
    reference_time_ms: i64,
    temporal_weight: f64,
    decay_scale_ms: f64,
)
```
Apply time-proximity boost to search results. Closer to reference time = higher score.

### LLM Provider (Built-in)

Built-in OpenAI-compatible HTTP client. Supports all OpenAI-compatible APIs (OpenAI, DeepSeek, Ollama, Qwen, Volcengine, etc.).

#### `configure_llm`
```rust
pub fn configure_llm(&self, config: AiLlmConfig) -> Result<(), Error>
pub fn get_llm_config(&self) -> Result<Option<AiLlmConfig>, Error>
pub fn clear_llm_config(&self) -> Result<(), Error>
```

```rust
pub struct AiLlmConfig {
    pub chat: Option<LlmEndpoint>,    // Chat/summarize provider
    pub embed: Option<EmbedEndpoint>,  // Embedding provider
}

pub struct LlmEndpoint {
    pub base_url: String,      // e.g., "https://api.openai.com/v1"
    pub api_key: Option<String>,
    pub model: String,          // e.g., "gpt-4o-mini", "deepseek-chat"
    pub max_retries: u8,        // default 2 (exponential backoff: 100ms, 200ms, 400ms...)
    pub timeout_secs: u32,      // default 60
}

pub struct EmbedEndpoint {
    pub base_url: String,      // Standard: auto-appends /embeddings; Volcengine multimodal: use full path
    pub api_key: Option<String>,
    pub model: String,          // e.g., "text-embedding-3-small", "bge-m3"
    pub dimensions: u32,        // Required: embedding dimensions
    pub timeout_secs: u32,      // default 30
}
```

#### Internal LLM Functions
```rust
// Used internally by auto_summarize, add_memory (EDU extraction), recall (LLM rerank)
pub fn chat_completion(endpoint, messages, temperature, max_tokens) -> Result<String, Error>
pub fn embed_texts(endpoint, texts) -> Result<Vec<Vec<f32>>, Error>
```
- `chat_completion`: Retries on 5xx (exponential backoff), stops on 4xx
- `embed_texts`: Supports standard OpenAI batch mode and Volcengine multimodal (per-item)

### Token Count

Precise BPE token counting based on tiktoken-rs. No network required — vocabulary data embedded at compile time.

```rust
pub fn count_tokens(text: &str, encoding: TokenEncoding) -> Result<u32, Error>
pub fn count_tokens_default(text: &str) -> Result<u32, Error>   // Uses o200k_base
pub fn count_tokens_batch(texts: &[&str], encoding: TokenEncoding) -> Result<Vec<u32>, Error>

pub enum TokenEncoding {
    Cl100kBase,  // GPT-4 / GPT-3.5-turbo / text-embedding-3-*
    O200kBase,   // GPT-4o / o1 / o3 (default)
}
```

### Intent Recognition

#### `query_by_intent`
```rust
pub fn query_by_intent(&self, query: &IntentQuery) -> Result<IntentResult, Error>
```

Supported intents:
- `FindSimilar`: Semantic search across RAG + Memory, merged and deduplicated
- `GetContext`: Auto-assemble complete LLM input context for a session
- `Recall`: Search long-term memories
- `CheckCache`: Check tool call cache
- `GetProgress`: Get agent execution progress

```rust
pub struct IntentQuery {
    pub kind: IntentKind,
    pub session_id: Option<String>,
    pub query_embedding: Option<Vec<f32>>,
    pub top_k: Option<usize>,
    pub max_tokens: Option<u32>,
    pub tool_name: Option<String>,
    pub input_hash: Option<String>,
    pub agent_id: Option<String>,
    pub metadata_filter: Option<BTreeMap<String, String>>,
}

pub enum IntentKind {
    FindSimilar,
    GetContext,
    Recall,
    CheckCache,
    GetProgress,
}

pub struct IntentResult {
    pub kind: IntentKind,
    pub found: bool,
    pub data: Option<serde_json::Value>,
}
```

## Accessing the AI Engine

```rust
// Write mode (Replica nodes return Error::ReadOnly)
let ai = db.ai()?;

// Read-only mode (available on Replica nodes)
let ai = db.ai_read()?;
```

## Best Practices

1. **Session lifecycle**: Use TTL for automatic cleanup; use `create_session_if_not_exists` for concurrent scenarios
2. **Token management**: Use `get_context_window_smart()` for automatic compression, or `get_context_window_with_prompt()` for manual control
3. **Memory storage**: Use `add_memory(content, meta, ttl, true)` for full power (Vector + FTS + EDU + Graph); use `false` for lightweight storage
4. **Memory recall**: Use `recall()` for production-grade retrieval with hybrid search, temporal awareness, and optional LLM rerank
5. **Memory dedup**: Periodically call `deduplicate_memories()` to avoid redundancy
6. **Tool caching**: Cache expensive API calls with appropriate TTL
7. **Tracing**: Record all LLM calls via `log_trace()` for debugging and cost tracking; use `trace_performance_report()` for analysis
8. **Embedding cache**: `add_memory()` and `recall()` automatically use embedding cache; for manual usage, use `cache_embedding()` / `get_cached_embedding()`
9. **LLM config**: Configure once via `configure_llm()` — all auto-* features (auto_store, auto_search, auto_summarize, add_memory, recall with rerank) use it automatically
10. **Event segmentation**: Use `EventSegmenter` to detect topic shifts in continuous conversations for coherent memory grouping
