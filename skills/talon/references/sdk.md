# Talon 多语言 SDK

所有 SDK 通过 FFI 绑定 `libtalon`，接口模式一致。仓库：https://github.com/darkmice/talon-sdk

## 支持平台

| OS | Arch | 静态库 | 动态库 |
|----|------|--------|--------|
| macOS | arm64 / amd64 | `libtalon.a` | `libtalon.dylib` |
| Linux | amd64 / arm64 / loongarch64 / riscv64 | `libtalon.a` | `libtalon.so` |
| Windows | amd64 | `talon.lib` | `talon.dll` |

---

## Go SDK

静态链接 `libtalon.a`，编译后单二进制零依赖。

```bash
go get github.com/darkmice/talon-sdk/go
```

```go
import talon "github.com/darkmice/talon-sdk/go"

db, err := talon.Open("./data")
defer db.Close()

// SQL
rows, _ := db.SQL("SELECT * FROM users")

// KV
db.KvSet("key", "value", nil)          // nil = 无 TTL
ttl := uint64(3600)
db.KvSet("session:x", "tok", &ttl)     // 1h TTL
val, _ := db.KvGet("key")
db.KvDel("key")
newVal, _ := db.KvIncr("counter")
db.KvMset([]string{"k1", "k2"}, []string{"v1", "v2"})
keys, _ := db.KvKeysLimit("user:", 0, 100)

// Vector
db.VectorCreate("embeddings", 384, "cosine")
db.VectorInsert("embeddings", 1, vec)
results, _ := db.VectorSearch("embeddings", queryVec, 10, "cosine")

// TimeSeries
db.TsCreate("metrics", []string{"host"}, []string{"cpu", "mem"})
db.TsInsert("metrics", map[string]interface{}{"host": "srv1", "cpu": 85.5})

// MQ
db.MqCreate("events")
db.MqPublish("events", map[string]interface{}{"type": "login"})
msgs, _ := db.MqPoll("events", 10)
db.MqAck("events", msgID)

// FTS
db.FtsCreateIndex("articles")
db.FtsIndex("articles", "doc1", map[string]string{"title": "AI", "body": "..."})
hits, _ := db.FtsSearch("articles", "AI", 10)
results, _ := db.FtsHybridSearch("articles", "vectors", "AI", queryVec,
    &talon.HybridSearchOpts{Metric: "cosine", Limit: 10, FtsWeight: 0.7, VecWeight: 0.3})

// GEO
db.GeoCreate("shops")
db.GeoAdd("shops", "starbucks", 121.47, 31.23)
nearby, _ := db.GeoSearch("shops", 121.47, 31.23, 1000, "m", nil)

// Graph
db.GraphCreate("social")
v1, _ := db.GraphAddVertex("social", "person", map[string]string{"name": "Alice"})
v2, _ := db.GraphAddVertex("social", "person", map[string]string{"name": "Bob"})
db.GraphAddEdge("social", v1, v2, "knows", nil)
bfs, _ := db.GraphBFS("social", v1, 3, "out")

// AI
db.AiCreateSession("s1", nil, nil)
db.AiAppendMessage("s1", map[string]interface{}{"role": "user", "content": "Hi"})
history, _ := db.AiGetHistory("s1", nil)

// AI v2.1 — Hybrid Memory
db.AiSetLlmConfig(map[string]interface{}{
    "chat":  map[string]interface{}{"base_url": "https://api.openai.com/v1", "api_key": "sk-...", "model": "gpt-4o-mini"},
    "embed": map[string]interface{}{"base_url": "https://api.openai.com/v1", "api_key": "sk-...", "model": "text-embedding-3-small", "dimensions": 1536},
})
db.AiAddMemory("Alice prefers dark mode", nil, nil, false)
results, _ := db.AiRecall("What does Alice prefer?", 5, 0.4, 0.6, 0.3, false, 0, 1)

// Ops
stats, _ := db.DatabaseStats()
db.Persist()
```

---

## Python SDK

使用 `ctypes` 加载动态库，无 pip 依赖。

```bash
git clone https://github.com/darkmice/talon-sdk.git
```

```python
from python.talon.client import Talon

db = Talon("./data")

# SQL
rows = db.sql("SELECT * FROM users")

# KV
db.kv_set("key", "value")
db.kv_set("session:x", "tok", ttl=3600)
val = db.kv_get("key")
db.kv_del("key")
new_val = db.kv_incr("counter")
db.kv_mset(["k1", "k2"], ["v1", "v2"])
keys = db.kv_keys_limit("user:", offset=0, limit=100)

# Vector
db.vector_create("embeddings", 384, "cosine")
db.vector_insert("embeddings", 1, [0.1, 0.2])
results = db.vector_search("embeddings", query_vec, k=10)

# TimeSeries
db.ts_create("metrics", tags=["host"], fields=["cpu", "mem"])
db.ts_insert("metrics", {"host": "srv1", "cpu": 85.5})

# MQ
db.mq_create("events")
db.mq_publish("events", {"type": "login"})
msgs = db.mq_poll("events", count=10)

# FTS
db.fts_create_index("articles")
db.fts_index("articles", "doc1", {"title": "AI", "body": "..."})
hits = db.fts_search("articles", "AI", limit=10)
results = db.fts_hybrid_search("articles", "vectors", "AI", query_vec,
    metric="cosine", limit=10, fts_weight=0.7, vec_weight=0.3)

# GEO
db.geo_create("shops")
db.geo_add("shops", "starbucks", lng=121.47, lat=31.23)
nearby = db.geo_search("shops", lng=121.47, lat=31.23, radius=1000)

# Graph
db.graph_create("social")
v1 = db.graph_add_vertex("social", "person", {"name": "Alice"})
v2 = db.graph_add_vertex("social", "person", {"name": "Bob"})
db.graph_add_edge("social", v1, v2, "knows")
bfs = db.graph_bfs("social", v1, max_depth=3)

# AI
db.ai_create_session("s1")
db.ai_append_message("s1", {"role": "user", "content": "Hi"})
history = db.ai_get_history("s1")

# AI v2.1 — Hybrid Memory
db.ai_set_llm_config({
    "chat":  {"base_url": "https://api.openai.com/v1", "api_key": "sk-...", "model": "gpt-4o-mini"},
    "embed": {"base_url": "https://api.openai.com/v1", "api_key": "sk-...", "model": "text-embedding-3-small", "dimensions": 1536},
})
db.ai_add_memory("Alice prefers dark mode")
results = db.ai_recall("What does Alice prefer?", k=5, temporal_boost=0.3, graph_depth=1)

db.close()
```

---

## Node.js SDK

使用 `ffi-napi` 加载动态库。

```bash
git clone https://github.com/darkmice/talon-sdk.git
npm install ffi-napi ref-napi
```

```javascript
const { Talon } = require('./nodejs');
const db = new Talon('./data');

// SQL
const rows = db.sql('SELECT * FROM users');

// KV
db.kvSet('key', 'value');
db.kvSet('session:x', 'tok', 3600);
const val = db.kvGet('key');
db.kvIncr('counter');

// Vector
db.vectorCreate('embeddings', 384, 'cosine');
db.vectorInsert('embeddings', 1, [0.1, 0.2]);
const results = db.vectorSearch('embeddings', queryVec, 10, 'cosine');

// FTS + Hybrid
db.ftsCreateIndex('articles');
db.ftsIndex('articles', 'doc1', { title: 'AI', body: '...' });
const hybrid = db.ftsHybridSearch('articles', 'vectors', 'AI', queryVec, {
  metric: 'cosine', limit: 10, ftsWeight: 0.7, vecWeight: 0.3,
});

// AI
db.aiCreateSession('s1');
db.aiAppendMessage('s1', { role: 'user', content: 'Hi' });
const history = db.aiGetHistory('s1');

// AI v2.1 — Hybrid Memory
db.aiSetLlmConfig({
  chat:  { base_url: 'https://api.openai.com/v1', api_key: 'sk-...', model: 'gpt-4o-mini' },
  embed: { base_url: 'https://api.openai.com/v1', api_key: 'sk-...', model: 'text-embedding-3-small', dimensions: 1536 },
});
db.aiAddMemory('Alice prefers dark mode');
const results = db.aiRecall('What does Alice prefer?', 5, 0.4, 0.6, 0.3, false, undefined, 1);

db.close();
```

---

## Java SDK

使用 JNA 加载动态库。

```xml
<dependency>
  <groupId>net.java.dev.jna</groupId>
  <artifactId>jna</artifactId>
  <version>5.14.0</version>
</dependency>
```

```java
import io.talon.Talon;

try (Talon db = new Talon("./data")) {
    // SQL
    var rows = db.sql("SELECT * FROM users");

    // KV
    db.kvSet("key", "value", null);
    db.kvSet("session:x", "tok", 3600L);
    String val = db.kvGet("key");
    long newVal = db.kvIncr("counter");

    // Vector
    db.vectorCreate("embeddings", 384, "cosine");
    db.vectorInsert("embeddings", 1, new float[]{0.1f, 0.2f});
    var results = db.vectorSearch("embeddings", queryVec, 10, "cosine");

    // AI
    db.aiCreateSession("s1", null, null);
    db.aiAppendMessage("s1", Map.of("role", "user", "content", "Hi"));
    var history = db.aiGetHistory("s1", null);

    // AI v2.1 — Hybrid Memory
    db.aiSetLlmConfig(Map.of(
        "chat", Map.of("base_url", "https://api.openai.com/v1", "api_key", "sk-...", "model", "gpt-4o-mini"),
        "embed", Map.of("base_url", "https://api.openai.com/v1", "api_key", "sk-...", "model", "text-embedding-3-small", "dimensions", 1536)
    ));
    db.aiAddMemory("Alice prefers dark mode", null, null, false);
    var results = db.aiRecall("What does Alice prefer?", 5, 0.4, 0.6, 0.3, false, 0, 1);
}
```

---

## .NET SDK

使用 P/Invoke 加载动态库。

```csharp
using TalonDb;

using var db = new TalonClient("./data");

// SQL
var rows = db.Sql("SELECT * FROM users");

// KV
db.KvSet("key", "value");
db.KvSet("session:x", "tok", 3600);
var val = db.KvGet("key");
var newVal = db.KvIncr("counter");

// Vector
db.VectorCreate("embeddings", 384, "cosine");
db.VectorInsert("embeddings", 1, new float[] { 0.1f, 0.2f });
var results = db.VectorSearch("embeddings", queryVec, 10, "cosine");

// AI
db.AiCreateSession("s1");
db.AiAppendMessage("s1", new() { ["role"] = "user", ["content"] = "Hi" });
var history = db.AiGetHistory("s1");

// AI v2.1 — Hybrid Memory
db.AiSetLlmConfig(new() {
    ["chat"] = new() { ["base_url"] = "https://api.openai.com/v1", ["api_key"] = "sk-...", ["model"] = "gpt-4o-mini" },
    ["embed"] = new() { ["base_url"] = "https://api.openai.com/v1", ["api_key"] = "sk-...", ["model"] = "text-embedding-3-small", ["dimensions"] = 1536 }
});
db.AiAddMemory("Alice prefers dark mode");
var results = db.AiRecall("What does Alice prefer?", 5, 0.4, 0.6, 0.3, false, 0, 1);
```

---

## 接口命名规则

| 引擎 | Go | Python | Node.js | Java | .NET |
|------|-----|--------|---------|------|------|
| SQL | `SQL()` | `sql()` | `sql()` | `sql()` | `Sql()` |
| KV | `KvSet/KvGet` | `kv_set/kv_get` | `kvSet/kvGet` | `kvSet/kvGet` | `KvSet/KvGet` |
| Vector | `VectorCreate` | `vector_create` | `vectorCreate` | `vectorCreate` | `VectorCreate` |
| TS | `TsCreate` | `ts_create` | `tsCreate` | `tsCreate` | `TsCreate` |
| MQ | `MqPublish` | `mq_publish` | `mqPublish` | `mqPublish` | `MqPublish` |
| FTS | `FtsSearch` | `fts_search` | `ftsSearch` | `ftsSearch` | `FtsSearch` |
| GEO | `GeoAdd` | `geo_add` | `geoAdd` | `geoAdd` | `GeoAdd` |
| Graph | `GraphBFS` | `graph_bfs` | `graphBfs` | `graphBfs` | `GraphBfs` |
| AI | `AiCreateSession` | `ai_create_session` | `aiCreateSession` | `aiCreateSession` | `AiCreateSession` |
| AI v2.1 | `AiAddMemory/AiRecall` | `ai_add_memory/ai_recall` | `aiAddMemory/aiRecall` | `aiAddMemory/aiRecall` | `AiAddMemory/AiRecall` |

各语言遵循自身命名惯例：Go PascalCase、Python snake_case、JS/Java camelCase、.NET PascalCase。
