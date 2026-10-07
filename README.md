# Neo4j Community vs Enterprise for Knowledge Graphs and GraphRAG

> **Goal:** Pick the right Neo4j edition for building a Knowledge Graph (KG) and a GraphRAG pipeline.
> **Research date:** 2026-10-07. **Latest Neo4j release at that date:** 2026.09.
> Neo4j now uses calendar versions (`YYYY.MM`). Its Cypher dialect is **Cypher 25**.

---

## TL;DR

| Stage | Recommended edition | Why |
|---|---|---|
| Learning, PoC, prototype, single-team internal tool | **Community Edition (CE)** | Free (GPLv3). It has everything GraphRAG needs: vector indexes, full-text indexes, hybrid search, the `SEARCH` clause, APOC Core, GDS Community and the `neo4j-graphrag` library. |
| Local development of something going to production on Enterprise | **Enterprise Edition (EE) via Neo4j Desktop** (free developer license) | Same features as production, so you don't find out about schema or performance differences late. |
| Production, multi-tenant, regulated data, HA required | **Enterprise Edition**, or managed **AuraDB** | RBAC and document-level security, clustering, online backup, multiple databases, CDC, native `VECTOR` type, existence and type constraints, faster Cypher runtimes. |
| Very large KG (billions of nodes or rich properties) | **EE + Infinigraph** subscription | Sharded property databases. |

**Bottom line:** Start on **Community** to build and test the KG and GraphRAG pipeline. Move to **Enterprise** (or AuraDB) when you need any of these: high availability, user or tenant access control on retrieved context, more than one database, online backups, or vector and analytics performance at scale. The data model and Cypher queries carry over between editions, so this upgrade path costs little.

---

## 1. The two editions in brief

### Neo4j Community Edition (CE)
- Open source under **GPLv3**, free to use.
- Single server, **one user database** (plus the `system` database).
- Full property graph, ACID transactions, Cypher, drivers and the APIs.
- No clustering, no fine-grained security, offline backup only.

### Neo4j Enterprise Edition (EE)
- **Commercial license** (subscription). Free licenses exist for:
  - **Developers:** EE through **Neo4j Desktop** for local development.
  - **Startups:** the Startup Program, currently for companies with up to 20 employees.
  - **Evaluation:** a time-limited trial.
  - **Education and non-profits:** academic, Graphs4Good and data journalism programs.
- Adds security, scale, availability and performance features on top of CE.
- **Infinigraph** is an EE add-on subscription that shards property data across a cluster, and it includes a Graph Data Science (GDS) Enterprise license.

### Managed alternative: AuraDB (Neo4j cloud)
AuraDB runs Enterprise-grade Neo4j for you. It has Free, Professional, Business Critical and Virtual Dedicated Cloud tiers. Use it if you don't want to run clusters yourself. All Aura tiers use the pipelined runtime. CDC needs Business Critical or Virtual Dedicated Cloud.

---

## 2. Feature comparison

### 2.1 Core database

| Feature | CE | EE |
|---|:-:|:-:|
| Property graph model, native graph storage | ✅ | ✅ |
| ACID transactions | ✅ | ✅ |
| Cypher (Cypher 25) and cost-based optimizer | ✅ | ✅ |
| Drivers (Python, Java, JS, .NET, Go), HTTP API, Bolt | ✅ | ✅ |
| Store format | `aligned` only | `block` (default since 5.22), `aligned` |
| Cypher runtimes | **Slotted only** | Slotted, **Pipelined** (default), **Parallel** |
| Multiple user databases | ❌ (1) | ✅ |
| Composite databases (query across databases) | ❌ | ✅ |
| Change Data Capture (CDC) | ❌ | ✅ |

### 2.2 Indexes and constraints (matter for KG quality)

| Feature | CE | EE |
|---|:-:|:-:|
| Range, text, point, lookup indexes | ✅ | ✅ |
| **Full-text index** (Lucene, used for keyword and hybrid search) | ✅ | ✅ |
| **Vector index** (HNSW, used for semantic search) | ✅ | ✅ |
| Embeddings stored as `LIST<FLOAT>` | ✅ | ✅ |
| Embeddings stored as native **`VECTOR`** type (Cypher 25, 2025.10+) | ❌ (needs block format) | ✅ |
| `SEARCH` clause with in-index filtering (GA) | ✅ | ✅ |
| Vector binary quantization and rescoring (2026.06+) | ✅ | ✅ |
| Property **uniqueness** constraints | ✅ | ✅ |
| Property **existence** constraints | ❌ | ✅ |
| Property **type** constraints | ❌ | ✅ |
| **Key** constraints (unique + exists) | ❌ | ✅ |

> **What this means for a KG:** In CE you can enforce that an entity `id` is unique, but you can't enforce that `name` or `embedding` exists, or that a property has the right type. You have to check those in your ingestion code. That is fine for a PoC, but it is a real gap for an LLM-extracted KG in production, where extraction output is noisy.

### 2.3 Security

| Feature | CE | EE |
|---|:-:|:-:|
| Native authentication (username and password) | ✅ | ✅ |
| Role-based access control (RBAC) | ❌ | ✅ |
| Sub-graph and label-level access control | ❌ | ✅ |
| Property-level access control | ❌ | ✅ |
| LDAP, Active Directory, SSO (OIDC) | ❌ | ✅ |

> **What this means for GraphRAG:** In CE every connected user can read the whole graph. To keep users from retrieving documents they shouldn't see, you have to filter in the application layer, for example a `WHERE c.tenant IN $allowed` filter on every retriever query. EE can enforce this in the database itself.

### 2.4 Scale, availability and operations

| Feature | CE | EE |
|---|:-:|:-:|
| Autonomous clustering (HA, read scaling) | ❌ | ✅ |
| Online (hot) backup and restore, incremental backup | ❌ (offline `neo4j-admin database dump` only) | ✅ |
| Multiple databases (one per tenant or environment) | ❌ | ✅ |
| Sharded property databases (Infinigraph) | ❌ | ✅ (add-on) |
| Kubernetes Helm charts | ✅ (standalone) | ✅ (clustered) |

### 2.5 Ecosystem and plugins

| Component | CE | EE |
|---|---|---|
| **APOC Core** | ✅ | ✅ |
| **GenAI functions** (`ai.text.embed()`, `ai.text.embedBatch()`) | ✅ | ✅ |
| **Graph Data Science (GDS)** | Community: all algorithms, **max 4 CPU cores**, model catalog limited to 3 models, no model persistence | Enterprise license: unlimited cores, model persistence, Arrow import and export, graph backup, runs in a cluster |
| **neo4j-graphrag** (Python) | ✅ | ✅ |
| LangChain, LlamaIndex, Haystack integrations | ✅ | ✅ |
| LLM Knowledge Graph Builder (Neo4j Labs app) | ✅ | ✅ |
| Bloom (graph exploration UI) | Limited free version | Full version |

---

## 3. What GraphRAG actually needs

A typical GraphRAG pipeline has these parts:

```
Documents ─► Chunk ─► Embed ─► LLM entity/relation extraction ─► Entity resolution
                                                                      │
                                                                      ▼
User question ─► Embed ─► Vector / hybrid search on chunks ─► Graph traversal (k-hop)
                                                                      │
                          Community summaries (GDS Leiden/Louvain) ◄──┤
                                                                      ▼
                                              Context assembly ─► LLM answer
```

| GraphRAG capability | Neo4j feature | Available in CE? |
|---|---|:-:|
| Store chunks, entities, relations | Property graph | ✅ |
| Semantic retrieval | Vector index + `SEARCH` | ✅ |
| Keyword retrieval | Full-text index | ✅ |
| Hybrid retrieval (vector + keyword) | Vector + full-text (`HybridRetriever` in neo4j-graphrag) | ✅ |
| Graph-enhanced retrieval | Cypher traversal (`VectorCypherRetriever`) | ✅ |
| Text2Cypher | Cypher + schema introspection | ✅ |
| Entity resolution / dedup | APOC (`apoc.refactor.mergeNodes`), GDS Node Similarity / WCC | ✅ (GDS ≤ 4 cores) |
| Community detection for global summaries (Microsoft-style GraphRAG) | GDS Leiden / Louvain | ✅ (≤ 4 cores) |
| Generating embeddings inside the database | `ai.text.embed()` | ✅ |
| Schema enforcement on extracted data | Existence, type and key constraints | ❌ (EE) |
| Per-user / per-tenant retrieval security | RBAC, sub-graph access control | ❌ (EE), use application-level filtering |
| One KG per tenant or domain | Multiple databases | ❌ (EE) |
| Compact embedding storage | Native `VECTOR` type (block format) | ❌ (EE) |
| Fast analytical traversals over large KGs | Pipelined / Parallel runtime | ❌ (EE) |
| Sync KG to downstream systems (Kafka, search) | CDC | ❌ (EE) |
| 24/7 uptime for a chatbot / agent | Clustering + online backup | ❌ (EE) |

**Conclusion:** All the *functional* building blocks of GraphRAG work in **Community Edition**. Enterprise adds what you need for **production**: governance, security, availability, multi-tenancy and performance.

---

## 4. Decision guide

Answer these in order. The first **Yes** points you to Enterprise (or AuraDB).

1. Must users only retrieve documents they're allowed to see, and must the **database** enforce it (audit or compliance)? → **EE**
2. Do you need **HA / zero-downtime** (customer-facing chatbot, agent, SLA)? → **EE** (cluster) or AuraDB
3. Do you need **separate databases** per tenant, domain or environment on one server? → **EE**
4. Is the KG large (roughly 50M+ nodes, or 10M+ embedded chunks), or do you need fast analytical queries and GDS on more than 4 cores? → **EE** (+ GDS Enterprise)
5. Do you need **online backups** or point-in-time restore? → **EE**
6. Do you need to **stream changes** to Kafka, a search index or a cache? → **EE** (CDC)
7. Does your app embed or distribute Neo4j in **closed-source** software where GPLv3 is a problem? → **EE** commercial license

If every answer is **No**, use **Community Edition**.

### Cost and risk

| | Community | Enterprise (self-hosted) | AuraDB |
|---|---|---|---|
| License cost | Free | Commercial subscription (free for dev, startups, eval) | Pay-as-you-go, Free tier available |
| Ops effort | Low (single node) | High (cluster, backups, upgrades) | Lowest (managed) |
| Lock-in risk | None | License renewal | Cloud vendor |
| Migration CE → EE | — | Simple: dump and load, same Cypher | Import via `neo4j-admin` dump upload |

---

## 5. Recommendation for this project

1. **Phase 1, PoC (now):** **Neo4j Community Edition 2026.x** in Docker with APOC Core and GDS Community, plus `neo4j-graphrag` (Python).
   - Store embeddings as `LIST<FLOAT>` and query with `SEARCH ... IN (VECTOR INDEX ...)`.
   - Enforce entity uniqueness with uniqueness constraints. Check that required properties exist in the ingestion code.
   - If you need per-user access, add `tenant` / `acl` properties to chunks and filter in every retriever query.
2. **Phase 2, pre-production:** Develop against **EE through Neo4j Desktop** (free developer license), or ask about the **Startup Program** if you qualify. Add existence, type and key constraints and test the native `VECTOR` type.
3. **Phase 3, production:** **Enterprise Edition** (3-node cluster, online backup, RBAC) or **AuraDB Professional / Business Critical** if you'd rather not run it yourself. Add a GDS Enterprise license if graph analytics runs on more than 4 cores.

---

## 6. Quick start (Community Edition + GraphRAG)

```bash
docker run -d --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/your-strong-password \
  -e NEO4J_PLUGINS='["apoc", "graph-data-science"]' \
  -v $PWD/data:/data \
  neo4j:2026-community
```
> For Enterprise, use the `neo4j:2026-enterprise` image and add `-e NEO4J_ACCEPT_LICENSE_AGREEMENT=yes`. Check Docker Hub for the exact current tag.

```cypher
// Uniqueness constraint (CE + EE)
CREATE CONSTRAINT entity_id IF NOT EXISTS
FOR (e:Entity) REQUIRE e.id IS UNIQUE;

// Vector index with a filterable property (CE + EE)
CREATE VECTOR INDEX chunk_embedding IF NOT EXISTS
FOR (c:Chunk) ON c.embedding
WITH [c.tenant]
OPTIONS { indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}};

// Full-text index for hybrid search
CREATE FULLTEXT INDEX chunk_text IF NOT EXISTS
FOR (c:Chunk) ON EACH [c.text];

// EE only: existence constraint
// CREATE CONSTRAINT entity_name_exists FOR (e:Entity) REQUIRE e.name IS NOT NULL;
```

```bash
pip install "neo4j-graphrag[openai]"   # or [anthropic], [ollama], ...
```

Retrievers to use from `neo4j-graphrag`: `VectorRetriever`, `HybridRetriever`, `VectorCypherRetriever` (vector + graph traversal), `HybridCypherRetriever` and `Text2CypherRetriever`. Build the graph with `SimpleKGPipeline`.

---

## 7. Sources

- Neo4j Operations Manual: edition comparison: https://neo4j.com/docs/operations-manual/current/introduction/
- Store formats (block = EE, aligned = CE default): https://neo4j.com/docs/operations-manual/current/database-internals/store-formats/
- Cypher runtimes (CE = slotted only): https://neo4j.com/docs/cypher-manual/current/planning-and-tuning/runtimes/concepts/
- Constraints by edition: https://neo4j.com/docs/cypher-manual/current/constraints/managing-constraints/
- Vector indexes (CE: `LIST<FLOAT>`; `VECTOR` type needs block format): https://neo4j.com/docs/cypher-manual/25/indexes/semantic-indexes/vector-indexes/
- `SEARCH` clause in CE (community thread): https://community.neo4j.com/t/search-clause-with-where-in-community-edition-ga-status-and-supported-predicates-for-multi-tenant-filtering/78969
- Change Data Capture (EE / Aura BC / VDC): https://neo4j.com/docs/cdc/current/
- GDS editions (CE limited to 4 cores, 3 models): https://neo4j.com/docs/graph-data-science/current/introduction/
- Licensing (developer, startup, evaluation): https://neo4j.com/licensing/
- Infinigraph / property sharding: https://neo4j.com/blog/graph-database/property-sharding-infinigraph/
- neo4j-graphrag Python package: https://neo4j.com/developer/genai-ecosystem/graphrag-python/
- Neo4j 2026 changelog: https://github.com/neo4j/neo4j/wiki/Neo4j-2026-changelog

> ⚠️ Neo4j changes edition boundaries between releases. Check the edition table in the Operations Manual for your target version before you commit.
