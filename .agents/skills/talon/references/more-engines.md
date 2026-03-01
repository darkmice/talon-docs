# TimeSeries Engine

High-performance time-series storage with downsampling, retention policies, and 540K pts/s ingestion.

## Quick Start

```rust
use talon::{Talon, TsSchema, TsQuery};

let db = Talon::open("./data")?;
let schema = TsSchema::new(vec!["cpu".into(), "mem".into()]);
let ts = db.create_timeseries("metrics", schema)?;

ts.insert(1700000000_000, &[0.85, 0.72])?;
ts.insert_batch(&[(1700000001_000, vec![0.90, 0.68])])?;

let points = ts.query(&TsQuery {
    start: Some(1700000000_000), end: None, order_asc: true, limit: Some(100),
})?;
```

## API Reference

### Create / Open
```rust
pub fn create_timeseries(&self, name: &str, schema: TsSchema) -> Result<TsEngine, Error>
pub fn open_timeseries(&self, name: &str) -> Result<TsEngine, Error>
```

### Write
```rust
pub fn insert(&self, timestamp_ms: i64, values: &[f64]) -> Result<(), Error>
pub fn insert_batch(&self, points: &[(i64, Vec<f64>)]) -> Result<(), Error>
```

### Query
```rust
pub fn query(&self, q: &TsQuery) -> Result<Vec<DataPoint>, Error>
```
`TsQuery`: `start`, `end` (Option<i64>), `order_asc` (bool), `limit` (Option<usize>)

### Aggregation (Downsampling)
```rust
pub fn aggregate(&self, q: &TsAggQuery) -> Result<Vec<AggBucket>, Error>
```
`TsAggQuery`: `start`, `end`, `interval_ms` (i64), `func` (Avg|Sum|Min|Max|Count|First|Last), `field_index` (usize), `fill` (None|Null|Previous|Linear|Value(f64))

### Retention
```rust
pub fn set_retention(&self, duration_ms: u64) -> Result<(), Error>
pub fn get_retention(&self) -> Result<Option<u64>, Error>
pub fn purge_expired(&self) -> Result<u64, Error>
pub fn purge_before(&self, cutoff_ms: i64) -> Result<u64, Error>
pub fn purge_by_tag(&self, tag_filters: &[(String, String)]) -> Result<u64, Error>
```

### Tags
```rust
pub fn tag_values(&self, tag_name: &str) -> Result<Vec<String>, Error>
pub fn all_tag_values(&self) -> Result<BTreeMap<String, Vec<String>>, Error>
```

### InfluxDB Line Protocol
```rust
use talon::parse_line_protocol;
let line = "cpu,host=server01 usage=0.85 1700000000000000000";
let points = parse_line_protocol(line)?;
```

---

# MessageQueue Engine

Built-in message queue with consumer groups, dead letter queues, priority, and 1.6M msg/s throughput.

## Quick Start

```rust
let db = Talon::open("./data")?;
db.mq()?.create_topic("events", 0)?;
let msg_id = db.mq()?.publish("events", b"user_login")?;
db.mq()?.subscribe("events", "analytics")?;
let msgs = db.mq()?.poll("events", "analytics", "worker1", 10)?;
for msg in &msgs { db.mq()?.ack("events", "analytics", "worker1", msg.id)?; }
```

## API Reference

### Topic
```rust
pub fn create_topic(&self, topic: &str, max_len: u64) -> Result<(), Error>
pub fn delete_topic(&self, topic: &str) -> Result<(), Error>
pub fn list_topics(&self) -> Result<Vec<String>, Error>
pub fn describe_topic(&self, topic: &str) -> Result<TopicInfo, Error>
pub fn set_topic_ttl(&self, topic: &str, ttl_ms: u64) -> Result<(), Error>
```

### Publish
```rust
pub fn publish(&self, topic: &str, payload: &[u8]) -> Result<u64, Error>
pub fn publish_batch(&self, topic: &str, payloads: &[&[u8]]) -> Result<Vec<u64>, Error>
pub fn publish_with_key(&self, topic: &str, payload: &[u8], key: &str) -> Result<u64, Error>
pub fn publish_delayed(&self, topic: &str, payload: &[u8], delay_ms: u64) -> Result<u64, Error>
pub fn publish_with_priority(&self, topic: &str, payload: &[u8], priority: u8) -> Result<u64, Error>
pub fn publish_with_ttl(&self, topic: &str, payload: &[u8], ttl_ms: u64) -> Result<u64, Error>
pub fn publish_advanced(&self, topic: &str, payload: &[u8], key: Option<&str>, delay_ms: Option<u64>, ttl_ms: Option<u64>, priority: Option<u8>) -> Result<u64, Error>
```

### Consume
```rust
pub fn subscribe(&self, topic: &str, group: &str) -> Result<(), Error>
pub fn poll(&self, topic: &str, group: &str, consumer: &str, count: usize) -> Result<Vec<Message>, Error>
pub fn poll_block(&self, topic: &str, group: &str, consumer: &str, count: usize, block_ms: u64) -> Result<Vec<Message>, Error>
pub fn poll_with_filter(&self, topic: &str, group: &str, consumer: &str, count: usize, key_filter: &str) -> Result<Vec<Message>, Error>
pub fn ack(&self, topic: &str, group: &str, consumer: &str, message_id: u64) -> Result<(), Error>
pub fn nack(&self, topic: &str, group: &str, consumer: &str, message_id: u64) -> Result<(), Error>
```

### Dead Letter Queue
```rust
pub fn set_max_retries(&self, topic: &str, max_retries: u32) -> Result<(), Error>
pub fn poll_dlq(&self, topic: &str, group: &str, consumer: &str, count: usize) -> Result<Vec<Message>, Error>
```

### Message Structure
```rust
pub struct Message {
    pub id: u64, pub payload: Vec<u8>, pub timestamp: i64,
    pub retry_count: u32, pub deliver_at: i64, pub expire_at: i64,
    pub key: Option<String>, pub priority: u8,
}
```

---

# Full-Text Search Engine

Inverted index + BM25 scoring with Elasticsearch-compatible queries, Chinese tokenizer (Jieba), and hybrid search.

## Quick Start

```rust
let db = Talon::open("./data")?;
db.fts()?.index("articles", "doc1", "Talon is an AI-native database")?;
let hits = db.fts()?.search("articles", "database", 10)?;
```

## API Reference

### Index Management
```rust
pub fn create_index(&self, name: &str, config: &FtsConfig) -> Result<(), Error>
pub fn drop_index(&self, name: &str) -> Result<(), Error>
pub fn list_indexes(&self) -> Result<Vec<FtsIndexInfo>, Error>
pub fn reindex(&self, name: &str) -> Result<u64, Error>
pub fn add_alias(&self, alias: &str, index: &str) -> Result<(), Error>
pub fn remove_alias(&self, alias: &str) -> Result<(), Error>
```

### Document Operations
```rust
pub fn index_doc(&self, name: &str, doc: &FtsDoc) -> Result<(), Error>
pub fn index_doc_batch(&self, name: &str, docs: &[FtsDoc]) -> Result<(), Error>
pub fn get_doc(&self, name: &str, doc_id: &str) -> Result<Option<FtsDoc>, Error>
pub fn update_doc(&self, name: &str, doc_id: &str, doc: &FtsDoc) -> Result<(), Error>
pub fn delete_doc(&self, name: &str, doc_id: &str) -> Result<bool, Error>
pub fn delete_by_query(&self, name: &str, query: &str) -> Result<u64, Error>
```

### Search
```rust
pub fn search(&self, name: &str, query: &str, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_bool(&self, name: &str, query: &BoolQuery, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_phrase(&self, name: &str, phrase: &str, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_term(&self, name: &str, field: &str, term: &str, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_range(&self, name: &str, query: &RangeQuery, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_regexp(&self, name: &str, pattern: &str, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_wildcard(&self, name: &str, pattern: &str, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_fuzzy(&self, name: &str, query: &str, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn search_multi_field(&self, name: &str, query: &MultiFieldQuery, limit: usize) -> Result<Vec<SearchHit>, Error>
pub fn suggest(&self, name: &str, prefix: &str, limit: usize) -> Result<Vec<String>, Error>
```

### Boolean Query
```rust
let query = BoolQuery { must: vec!["database".into()], should: vec!["AI".into()], must_not: vec!["legacy".into()] };
let hits = db.fts()?.search_bool("articles", &query, 10)?;
```

### Hybrid Search (BM25 + Vector RRF)
```rust
let hits = hybrid_search(&store, &HybridQuery {
    fts_index: "articles", vec_index: "emb_idx",
    query_text: "AI database", query_vec: &embedding,
    limit: 10, pre_filter: Some(vec![("namespace", "tenant_a")]),
    ..Default::default()
})?;
```

### Analyzers
`Standard` (Unicode, default), `Jieba` (Chinese), `Whitespace`, `Keyword` (exact match)

### Elasticsearch Bulk Import
```rust
use talon::parse_es_bulk;
let ndjson = r#"{"index":{"_index":"articles","_id":"1"}}
{"title":"Talon","body":"AI database"}"#;
let items = parse_es_bulk(ndjson)?;
```

---

# GEO Engine

Geohash-based spatial indexing with Redis GEO compatibility.

## Quick Start

```rust
let db = Talon::open("./data")?;
db.geo()?.create("places")?;
db.geo()?.geo_add("places", "office", 116.4074, 39.9042)?;
let nearby = db.geo()?.geo_search("places", 116.4074, 39.9042, 500.0, GeoUnit::Meters, 10)?;
let dist = db.geo()?.geo_dist("places", "office", "cafe", GeoUnit::Meters)?;
```

## API Reference

### Write
```rust
pub fn create(&self, name: &str) -> Result<(), Error>
pub fn geo_add(&self, name: &str, member_key: &str, lng: f64, lat: f64) -> Result<(), Error>
pub fn geo_add_batch(&self, name: &str, members: &[(&str, f64, f64)]) -> Result<(), Error>
pub fn geo_add_nx(&self, ...) -> Result<bool, Error>   // NX: only if not exists
pub fn geo_add_xx(&self, ...) -> Result<bool, Error>   // XX: only if exists
pub fn geo_del(&self, name: &str, member_key: &str) -> Result<bool, Error>
```

### Read
```rust
pub fn geo_pos(&self, name: &str, member_key: &str) -> Result<Option<GeoPoint>, Error>
pub fn geo_dist(&self, name: &str, key1: &str, key2: &str, unit: GeoUnit) -> Result<Option<f64>, Error>
pub fn geo_hash(&self, name: &str, member_key: &str) -> Result<Option<String>, Error>
pub fn geo_members(&self, name: &str) -> Result<Vec<String>, Error>
pub fn geo_count(&self, name: &str) -> Result<u64, Error>
```

### Search
```rust
pub fn geo_search(&self, name: &str, lng: f64, lat: f64, radius: f64, unit: GeoUnit, limit: usize) -> Result<Vec<GeoMember>, Error>
pub fn geo_search_box(&self, name: &str, lng: f64, lat: f64, width: f64, height: f64, unit: GeoUnit, limit: usize) -> Result<Vec<GeoMember>, Error>
pub fn geo_search_store(&self, name: &str, dest: &str, lng: f64, lat: f64, radius: f64, unit: GeoUnit, limit: usize) -> Result<u64, Error>
pub fn geo_fence(&self, name: &str, lng: f64, lat: f64, radius: f64, unit: GeoUnit) -> Result<Vec<GeoMember>, Error>
```

### SQL Integration
```sql
SELECT id, ST_DISTANCE(location, GEOPOINT(39.9, 116.4)) AS dist_m FROM places ORDER BY dist_m LIMIT 10;
SELECT * FROM places WHERE ST_WITHIN(location, 39.9, 116.4, 1000);
```

### GEO + Vector Fusion
```rust
let hits = geo_vector_search(&store, &GeoVectorQuery {
    geo_name: "places", vec_name: "embeddings",
    lng: 116.4074, lat: 39.9042, radius_m: 1000.0,
    query_vec: &embedding, k: 10,
})?;
```

---

# Graph Engine

Property graph with BFS/DFS traversal, shortest path, PageRank, and 935K reads/s.

## Quick Start

```rust
let db = Talon::open("./data")?;
db.graph()?.create("social")?;
let alice = db.graph()?.add_vertex("social", None, Some("person"), Some(&props))?;
let bob = db.graph()?.add_vertex("social", None, Some("person"), Some(&props))?;
db.graph()?.add_edge("social", alice, bob, Some("follows"), None)?;
let friends = db.graph()?.bfs("social", alice, 2, Direction::Out)?;
```

## API Reference

### Vertex
```rust
pub fn add_vertex(&self, graph: &str, id: Option<u64>, label: Option<&str>, properties: Option<&serde_json::Value>) -> Result<u64, Error>
pub fn get_vertex(&self, graph: &str, id: u64) -> Result<Option<Vertex>, Error>
pub fn update_vertex(&self, graph: &str, id: u64, properties: &serde_json::Value) -> Result<(), Error>
pub fn delete_vertex(&self, graph: &str, id: u64) -> Result<(), Error>
pub fn vertices_by_label(&self, graph: &str, label: &str) -> Result<Vec<Vertex>, Error>
pub fn vertex_count(&self, graph: &str) -> Result<u64, Error>
```

### Edge
```rust
pub fn add_edge(&self, graph: &str, from: u64, to: u64, label: Option<&str>, properties: Option<&serde_json::Value>) -> Result<u64, Error>
pub fn get_edge(&self, graph: &str, id: u64) -> Result<Option<Edge>, Error>
pub fn delete_edge(&self, graph: &str, edge_id: u64) -> Result<(), Error>
pub fn out_edges(&self, graph: &str, vertex_id: u64) -> Result<Vec<Edge>, Error>
pub fn in_edges(&self, graph: &str, vertex_id: u64) -> Result<Vec<Edge>, Error>
pub fn edges_by_label(&self, graph: &str, label: &str) -> Result<Vec<Edge>, Error>
pub fn edge_count(&self, graph: &str) -> Result<u64, Error>
```

### Traversal
```rust
pub fn bfs(&self, graph: &str, start: u64, max_depth: u32, direction: Direction) -> Result<Vec<u64>, Error>
pub fn bfs_filter<F>(&self, graph: &str, start: u64, max_depth: u32, direction: Direction, filter: F) -> Result<Vec<u64>, Error>
pub fn k_hop_neighbors(&self, graph: &str, start: u64, k: u32, direction: Direction) -> Result<Vec<u64>, Error>
pub fn shortest_path(&self, graph: &str, from: u64, to: u64, direction: Direction) -> Result<Option<Vec<u64>>, Error>
pub fn neighbors(&self, graph: &str, vertex_id: u64, direction: Direction) -> Result<Vec<u64>, Error>
```
Direction: `Out`, `In`, `Both`

### Analytics
```rust
pub fn pagerank(&self, graph: &str, iterations: u32, damping: f64) -> Result<Vec<(u64, f64)>, Error>
pub fn degree_centrality(&self, graph: &str, direction: Direction) -> Result<Vec<(u64, f64)>, Error>
pub fn weighted_shortest_path(&self, graph: &str, from: u64, to: u64, weight_key: &str) -> Result<Option<(Vec<u64>, f64)>, Error>
```

### GraphRAG Fusion
```rust
let hits = graph_vector_search(&store, &GraphVectorQuery {
    graph: "knowledge", vec_name: "embeddings",
    start: root_id, max_depth: 3, direction: Direction::Out,
    query_vec: &embedding, k: 10,
})?;

let hits = graph_fts_search(&store, &GraphFtsQuery {
    graph: "knowledge", fts_name: "articles",
    start: root_id, max_depth: 2, direction: Direction::Out,
    query: "AI database", k: 10,
})?;
```

---

# Cross-Engine Fusion Queries

| Combination | Function | Use Case |
|-------------|----------|----------|
| GEO + Vector | `geo_vector_search` | Nearby semantic search |
| GEO + Vector (Box) | `geo_box_vector_search` | Bounding box + vector |
| Graph + Vector | `graph_vector_search` | GraphRAG |
| Graph + FTS | `graph_fts_search` | Knowledge graph retrieval |
| FTS + Vector | `hybrid_search` | Hybrid retrieval (RRF) |
| Graph + FTS + Vector | `triple_search` | Triple fusion search |

### Triple Search (Graph + FTS + Vector)
```rust
let hits = triple_search(&store, &TripleQuery {
    graph: "knowledge", fts_name: "articles", vec_name: "embeddings",
    start: root_id, max_depth: 2, direction: Direction::Out,
    text_query: "AI database", vec_query: &embedding, k: 10,
})?;
```

### AI Application Patterns
```
RAG:          User Query → Embedding → hybrid_search(FTS + Vector) → LLM Context
GraphRAG:     User Query → Root Node → graph_vector_search(Graph + Vector) → LLM Context
Location AI:  User Location → geo_vector_search(GEO + Vector) → Nearby Semantic Results → LLM
```
