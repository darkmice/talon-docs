# AI Engine (Commercial — requires `talon-ai`)

Native Session, Context, Memory, RAG, Agent, Trace, Intent, and Embedding Cache abstractions for LLM applications.

> ⚠️ **Commercial License**: The AI Engine is distributed as a pre-compiled library (`talon-ai`) and requires a valid subscription.
> No source code is provided. Obtain from https://releases.talon.dev/ or via private Cargo registry.

## Overview

The AI Engine is Talon’s commercial extension — a first-class semantic abstraction layer purpose-built for LLM application development. It is distributed as a separate `talon-ai` crate (pre-compiled, no source code) that injects AI capabilities into Talon via the `TalonAiExt` extension trait. It eliminates the need for external frameworks (LangChain, LlamaIndex) by providing native primitives for session management, conversation context, semantic memory, RAG document management, agent orchestration, execution tracing, intent recognition, and embedding caching.

## Quick Start

```rust
use talon::Talon;
use talon_ai::TalonAiExt;  // Commercial extension trait
use std::collections::BTreeMap;

let db = Talon::open("./data")?;
let ai = db.ai()?;  // Injected by TalonAiExt

// Create a session
ai.create_session("chat-001", BTreeMap::new(), None)?;

// Append messages
ai.append_message("chat-001", &ContextMessage {
    role: "user".into(),
    content: "What is Talon?".into(),
    token_count: Some(5),
})?;

// Store semantic memory
ai.store_memory("chat-001", "User prefers Rust", &embedding, None)?;

// Search memories
let memories = ai.search_memories("chat-001", &query_embedding, 5)?;
```

## API Reference

### Session Management

#### `create_session`
```rust
pub fn create_session(&self, id: &str, metadata: BTreeMap<String, String>, ttl_secs: Option<u64>) -> Result<(), Error>
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | `&str` | Unique session identifier |
| `metadata` | `BTreeMap<String, String>` | Custom key-value metadata |
| `ttl_secs` | `Option<u64>` | Auto-expire after N seconds |

#### `get_session`
```rust
pub fn get_session(&self, id: &str) -> Result<Option<Session>, Error>
```

#### `list_sessions`
```rust
pub fn list_sessions(&self) -> Result<Vec<Session>, Error>
```

#### `delete_session`
```rust
pub fn delete_session(&self, id: &str) -> Result<(), Error>
```
Cascade delete: removes session + all context messages + traces.

#### `update_session`
```rust
pub fn update_session(&self, id: &str, metadata: BTreeMap<String, String>) -> Result<(), Error>
```

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

#### `cleanup_expired_sessions`
```rust
pub fn cleanup_expired_sessions(&self) -> Result<usize, Error>
```
Batch purge all expired sessions. Returns count deleted.

### Conversation Context

#### `append_message`
```rust
pub fn append_message(&self, session_id: &str, msg: &ContextMessage) -> Result<(), Error>
```

```rust
pub struct ContextMessage {
    pub role: String,        // "user" | "assistant" | "system"
    pub content: String,     // Message text
    pub token_count: Option<u32>, // Token count for window management
}
```

#### `get_history`
```rust
pub fn get_history(&self, session_id: &str, last_n: usize) -> Result<Vec<ContextMessage>, Error>
```
Get the most recent N messages for a session.

#### `get_recent_messages`
```rust
pub fn get_recent_messages(&self, session_id: &str, limit: usize) -> Result<Vec<ContextMessage>, Error>
```
Get recent messages with a limit.

#### `get_context_window`
```rust
pub fn get_context_window(&self, session_id: &str, max_tokens: u32) -> Result<Vec<ContextMessage>, Error>
```
Get messages fitting within a token budget (auto-truncation for LLM context length).

#### `get_context_window_with_prompt`
```rust
pub fn get_context_window_with_prompt(&self, session_id: &str, max_tokens: u32) -> Result<(Option<String>, Vec<ContextMessage>), Error>
```
Like `get_context_window` but also returns the system prompt if set.

#### `set_system_prompt` / `get_system_prompt`
```rust
pub fn set_system_prompt(&self, session_id: &str, prompt: &str) -> Result<(), Error>
pub fn get_system_prompt(&self, session_id: &str) -> Result<Option<String>, Error>
```
Set/get a persistent system prompt for the session. Automatically included in context window.

#### `set_context_summary` / `get_context_summary`
```rust
pub fn set_context_summary(&self, session_id: &str, summary: &str) -> Result<(), Error>
pub fn get_context_summary(&self, session_id: &str) -> Result<Option<String>, Error>
```
Store/retrieve a conversation summary. Useful for long conversations that exceed the context window.

#### `clear_context`
```rust
pub fn clear_context(&self, session_id: &str) -> Result<u64, Error>
```
Clear all messages while preserving the session. Returns count deleted.

### Semantic Memory

#### `store_memory`
```rust
pub fn store_memory(
    &self, session_id: &str,
    text: &str,
    embedding: &[f32],
    ttl_secs: Option<u64>,
) -> Result<String, Error>
```
Store a vectorized long-term memory. Returns memory ID.

#### `search_memories`
```rust
pub fn search_memories(&self, session_id: &str, query_embedding: &[f32], k: usize) -> Result<Vec<MemorySearchResult>, Error>
```
Semantic similarity search over memories.

#### `update_memory`
```rust
pub fn update_memory(&self, memory_id: &str, text: &str, embedding: &[f32]) -> Result<(), Error>
```

#### `delete_memory`
```rust
pub fn delete_memory(&self, memory_id: &str) -> Result<(), Error>
```

#### `memory_count`
```rust
pub fn memory_count(&self) -> Result<u64, Error>
```
Total number of stored memories.

#### `search_memory_with_filter`
```rust
pub fn search_memory_with_filter(&self, query_embedding: &[f32], k: usize, filter: impl Fn(&MemoryEntry) -> bool) -> Result<Vec<MemoryEntry>, Error>
```
Search memories with a custom predicate filter.

#### `list_memories`
```rust
pub fn list_memories(&self, offset: usize, limit: usize) -> Result<Vec<MemoryEntry>, Error>
```
Paginated listing of all memories.

#### `store_memory_with_ttl`
```rust
pub fn store_memory_with_ttl(&self, entry: &MemoryEntry, embedding: &[f32], ttl_secs: u64) -> Result<(), Error>
```
Store memory with explicit TTL.

#### `store_memories_batch`
```rust
pub fn store_memories_batch(&self, entries: &[(&MemoryEntry, &[f32])]) -> Result<(), Error>
```
Batch store multiple memories with embeddings.

#### `find_duplicate_memories`
```rust
pub fn find_duplicate_memories(&self, threshold: f32) -> Result<Vec<DuplicatePair>, Error>
```
Find memory pairs with cosine similarity above threshold.

#### `deduplicate_memories`
```rust
pub fn deduplicate_memories(&self, threshold: f32) -> Result<usize, Error>
```
Automatically remove duplicate memories. Returns count removed.

#### `cleanup_expired_memories`
```rust
pub fn cleanup_expired_memories(&self) -> Result<usize, Error>
```

#### `memory_stats`
```rust
pub fn memory_stats(&self) -> Result<MemoryStats, Error>
```
Returns total count, expired count, storage usage, etc.

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
pub fn search_chunks(&self, query_embedding: &[f32], k: usize) -> Result<Vec<RagSearchResult>, Error>
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
pub fn cache_tool_result(&self, tool_name: &str, args_hash: &str, result: &[u8], ttl_secs: Option<u64>) -> Result<(), Error>
pub fn get_cached_tool_result(&self, tool_name: &str, args_hash: &str) -> Result<Option<Vec<u8>>, Error>
pub fn invalidate_tool_cache(&self, tool_name: &str) -> Result<u64, Error>
```
Cache expensive tool call results with TTL. `invalidate_tool_cache` removes all cached results for a tool.

#### Agent State Persistence
```rust
pub fn save_agent_state(&self, agent_id: &str, step: &AgentStep) -> Result<(), Error>
pub fn get_agent_state(&self, agent_id: &str) -> Result<Option<AgentStep>, Error>
pub fn list_agent_steps(&self, agent_id: &str) -> Result<Vec<AgentStep>, Error>
pub fn get_agent_step_count(&self, agent_id: &str) -> Result<usize, Error>
pub fn rollback_agent_to_step(&self, agent_id: &str, step_id: &str) -> Result<usize, Error>
pub fn clear_agent_steps(&self, agent_id: &str) -> Result<usize, Error>
pub fn delete_agent_state(&self, agent_id: &str) -> Result<(), Error>
```
Persist agent execution steps for checkpointing.
- `rollback_agent_to_step` — removes all steps after the specified checkpoint, returns count removed
- `clear_agent_steps` — removes all steps for an agent, returns count removed
- `delete_agent_state` — deletes all agent data including steps

```rust
pub struct AgentStep {
    pub step_id: String,
    pub action: String,
    pub input: serde_json::Value,
    pub output: Option<serde_json::Value>,
    pub timestamp_ms: i64,
}
```

### Execution Trace

#### Logging
```rust
pub fn log_trace(&self, record: &TraceRecord) -> Result<(), Error>
```

```rust
pub struct TraceRecord {
    pub run_id: String,
    pub session_id: Option<String>,
    pub operation: String,   // "llm_call", "tool_call", "embedding", etc.
    pub input: serde_json::Value,
    pub output: Option<serde_json::Value>,
    pub latency_ms: u64,
    pub token_usage: Option<u32>,
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
pub fn trace_performance_report(&self, session_id: Option<&str>) -> Result<serde_json::Value, Error>
```
- `trace_stats` — aggregate count, token usage, grouped by operation type
- `trace_performance_report` — latency statistics, slow operation detection

### Embedding Cache

#### `cache_embedding`
```rust
pub fn cache_embedding(&self, text_hash: &str, embedding: &[f32]) -> Result<(), Error>
```
Cache an embedding vector to avoid redundant API calls to external embedding services.

#### `get_cached_embedding`
```rust
pub fn get_cached_embedding(&self, content_hash: &str) -> Result<Option<Vec<f32>>, Error>
```

#### `invalidate_embedding_cache`
```rust
pub fn invalidate_embedding_cache(&self) -> Result<u64, Error>
```
Clear all cached embeddings. Returns count deleted.

#### `embedding_cache_count`
```rust
pub fn embedding_cache_count(&self) -> Result<usize, Error>
```
Get the number of cached embeddings.

### Intent Recognition

#### `query_by_intent`
```rust
pub fn query_by_intent(&self, query: &IntentQuery) -> Result<IntentResult, Error>
```

```rust
pub struct IntentQuery {
    pub text: String,
    pub context: Option<Vec<ContextMessage>>,
}

pub struct IntentResult {
    pub kind: IntentKind,        // Sql, Kv, Vector, Fts, Geo, Graph, Ts, Mq, Ai, Unknown
    pub confidence: f32,
    pub suggested_action: Option<String>,
}

pub enum IntentKind {
    Sql, Kv, Vector, Fts, Geo, Graph, Ts, Mq, Ai, Unknown,
}
```
Routes natural language queries to the appropriate engine.

## Accessing the AI Engine

```toml
# Cargo.toml— add talon-ai from private registry (pre-compiled)
[dependencies]
talon = "0.1"
talon-ai = { version = "0.1", registry = "talon-ai" }
```

```rust
use talon_ai::TalonAiExt;

// Write mode (Replica nodes return Error::ReadOnly)
let ai = db.ai()?;

// Read-only mode (available on Replica nodes)
let ai = db.ai_read()?;
```

### Hybrid Memory (v2.1) — 智能记忆 Pipeline

> **v2.1 新增**: 内置 LLM/Embedding 配置 + 全自动 Hybrid Search 管线。
> 无需外部 embedding 调用，引擎自动完成 embed → 双写 → 混合检索 → LLM 精排 → Graph 推理。

#### LLM / Embedding 配置

```rust
pub fn configure_llm(&self, config: AiLlmConfig) -> Result<(), Error>
pub fn get_llm_config(&self) -> Result<Option<AiLlmConfig>, Error>
pub fn clear_llm_config(&self) -> Result<(), Error>
```

```rust
pub struct AiLlmConfig {
    pub chat: Option<LlmEndpoint>,   // Chat/Completion LLM
    pub embed: Option<EmbedEndpoint>, // Embedding model
}
pub struct LlmEndpoint {
    pub base_url: String,         // e.g. "https://api.openai.com/v1"
    pub api_key: Option<String>,
    pub model: String,            // e.g. "gpt-4o-mini"
    pub max_retries: u8,          // default 2
    pub timeout_secs: u32,        // default 60
}
pub struct EmbedEndpoint {
    pub base_url: String,
    pub api_key: Option<String>,
    pub model: String,
    pub dimensions: u32,          // e.g. 1536, 2048
    pub timeout_secs: u32,
}
```

**SDK Examples:**
```python
# Python
db.ai_set_llm_config({
    "chat": {"base_url": "https://api.openai.com/v1", "api_key": "sk-...", "model": "gpt-4o-mini"},
    "embed": {"base_url": "https://api.openai.com/v1", "api_key": "sk-...", "model": "text-embedding-3-small", "dimensions": 1536},
})
```

#### `add_memory` — 自动 Embedding + 双写

```rust
pub fn add_memory(
    &self, content: &str,
    metadata: BTreeMap<String, String>,
    ttl_secs: Option<u64>,
    extract_facts: bool,
) -> Result<u64, Error>
```

自动完成：
1. 调用 Embedding API 生成向量（FNV 哈希缓存）
2. 写入向量索引（语义搜索）
3. 写入 FTS 索引（关键词搜索）
4. 存储元数据到 KV
5. [可选] `extract_facts=true` → LLM 提取 EDU（结构化事件单元），每个 EDU 独立 embed + 存储 + Graph 写入

| Parameter | Type | Description |
|-----------|------|-------------|
| `content` | `&str` | 记忆文本 |
| `metadata` | `BTreeMap<String, String>` | 自定义 key-value 元数据 |
| `ttl_secs` | `Option<u64>` | 过期时间（秒），None = 永不过期 |
| `extract_facts` | `bool` | 是否用 LLM 提取结构化事件（需 chat config） |

#### `recall` — 智能混合召回 Pipeline

```rust
pub fn recall(
    &self, query: &str, k: usize,
    fts_weight: f64, vec_weight: f64,
    temporal_boost: f64, rerank: bool,
    rerank_top_k: Option<usize>,
    graph_depth: usize,
) -> Result<Vec<serde_json::Value>, Error>
```

**完整 Pipeline (Phase 1-4):**

```
Query → Graph Expand → Hybrid(BM25 + Vector) → RRF Fusion → Temporal Decay → LLM Rerank → Top-K
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` | `&str` | — | 搜索查询文本 |
| `k` | `usize` | 10 | 返回结果数量 |
| `fts_weight` | `f64` | 0.4 | FTS (BM25) 路权重 |
| `vec_weight` | `f64` | 0.6 | 向量 (Cosine) 路权重 |
| `temporal_boost` | `f64` | 0.0 | 时间衰减权重（推荐 0.3，0=关闭） |
| `rerank` | `bool` | false | 启用 LLM Listwise Rerank（需 chat config） |
| `rerank_top_k` | `Option<usize>` | = k | Rerank 后保留数量 |
| `graph_depth` | `usize` | 0 | Graph 实体扩展跳数（推荐 1-2，0=关闭） |

**Pipeline 各阶段说明:**

- **Graph Expand (Phase 4):** 从 query 中提取实体（支持中英文），在知识图谱中 BFS N-hop 扩展关联实体，增强查询
- **Hybrid Search:** 并行 BM25 全文检索 + 向量语义搜索，RRF (Reciprocal Rank Fusion) 融合排序
- **Temporal Decay:** 时间衰减函数，近期记忆权重更高
- **LLM Rerank (Phase 3):** Listwise Rerank — 用 LLM 对候选集重排序，显著提升精度

**返回格式:**
```json
[
  {
    "entry": { "content": "...", "metadata": {...} },
    "rrf_score": 0.85,
    "bm25_score": 12.3,
    "vector_dist": 0.15
  }
]
```

#### `deduplicate_memories` — 记忆去重

```rust
pub fn deduplicate_memories(&self, threshold: f32) -> Result<usize, Error>
```
基于 Embedding 向量余弦相似度去重。`threshold` 推荐 0.05（越小越严格）。返回删除数量。

#### Graph Memory Stats

```rust
pub fn graph_memory_stats(&self) -> Result<(usize, usize), Error>  // (vertex_count, edge_count)
```

#### Auto-Summarize

```rust
pub fn auto_summarize(&self, session_id: &str, opts: SummarizeOptions) -> Result<String, Error>
pub fn get_context_window_smart(&self, session_id: &str, max_tokens: u32) -> Result<Vec<ContextMessage>, Error>
```

使用 LLM 自动为对话生成摘要，并用摘要 + 最新消息构建 token-aware 上下文窗口。

## Best Practices

1. **Session lifecycle**: Use TTL for automatic cleanup of stale sessions
2. **Token management**: Use `get_context_window()` to fit within LLM limits
3. **Memory dedup**: Periodically call `find_duplicate_memories()` to avoid redundancy
4. **Tool caching**: Cache expensive API calls with appropriate TTL
5. **Tracing**: Record all LLM calls for debugging and cost tracking
6. **Embedding cache**: Hash text content and cache embeddings to reduce API costs
7. **Hybrid recall**: Use `recall()` with `fts_weight=0.4, vec_weight=0.6, temporal_boost=0.3` as starting point
8. **Graph depth**: Set `graph_depth=1` for entity-aware recall; max 5 to prevent deep traversal
9. **LLM Rerank**: Enable `rerank=true` for precision-critical scenarios; costs 1 LLM call per search

