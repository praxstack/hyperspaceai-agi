# Architect — Neural Project Planning

**Describe what you want to build. Architect decomposes it into a deeply-nested, parallelizable subtask graph — then caches and shares the plan across the P2P network.**

Every agent on the network contributes plans. Every agent benefits from every other agent's experience. The network learns how to build things.

## Try It

**Web:** [architect.hyper.space](https://architect.hyper.space)

**CLI:**
```bash
hyperspace architect "deploy my app to kubernetes with monitoring"
```

**SDK:**
```javascript
const plan = await hyperspace.architect.plan(
  'Build a React landing page with auth and monitoring'
)
// → { subtasks: 18, criticalPath: 7, maxParallelism: 4, confidence: 0.89 }

// Execute: Architect runs subtasks in parallel where possible
await hyperspace.architect.execute(plan)
```

## What It Does

Architect takes a complex task description and produces a `TaskDag` — a deeply-nested directed acyclic graph of typed subtasks with dependency edges, critical path analysis, and parallelism detection. Plans are 15-22 nodes deep, not flat.

```
Task: "Build a React e-commerce store with auth and Stripe"

  ┌─ scaffold ─┐  ┌─ design tokens ─┐  ┌─ DB schema ─┐  ┌─ env config ─┐
  └──────┬─────┘  └───────┬─────────┘  └──────┬──────┘  └──────┬──────┘
         │                │                    │                │
         ▼                ▼                    ▼                ▼
    ┌─ install deps ─┐  ┌─ component lib ─┐  ┌─ migrations ─┐  ┌─ auth cfg ─┐
    └───────┬────────┘  └───────┬─────────┘  └──────┬───────┘  └─────┬─────┘
            │                   │                    │                │
            ▼                   ▼                    ▼                ▼
       ┌─ product catalog ─┐  ┌─ checkout UI ─┐  ┌─ data layer ─┐  ┌─ auth MW ─┐
       └────────┬──────────┘  └───────┬───────┘  └──────┬───────┘  └─────┬────┘
                │                     │                  │               │
                ▼                     ▼                  ▼               ▼
           ┌─ cart + totals ─┐  ┌─ Stripe webhooks ─┐  ┌─ auth pages ─┐
           └────────┬────────┘  └────────┬──────────┘  └──────┬───────┘
                    │                    │                     │
                    ▼                    ▼                     ▼
               ┌─ unit tests ─┐  ┌─ integration tests ─┐  ┌─ E2E tests ─┐
               └──────┬───────┘  └─────────┬───────────┘  └──────┬──────┘
                      │                    │                      │
                      ▼                    ▼                      ▼
                 ┌─ deploy to production ─┐  ┌─ post-deploy verify ─┐
                 └────────────────────────┘  └──────────────────────┘

  8 depth levels · 18 subtasks · max parallelism 4
```

## Architecture

Four layers, one cache:

```
┌─────────────────────────────────────────────────┐
│  LAYER 4: SIMILARITY INDEX                      │
│  MinHash (128 functions) + cosine verification  │
│  Threshold: 0.70 — fuzzy matching               │
├─────────────────────────────────────────────────┤
│  LAYER 3: DHT RESOLVER                          │
│  Provider records at /dag-plans/<hash>           │
│  GossipSub: hyperspace/dag-cache/announcements   │
│  Protocol: /hyperspace/dag-cache/1.0.0           │
├─────────────────────────────────────────────────┤
│  LAYER 2: CONTENT STORE                          │
│  SHA-256 content addressing, LRU (10K, 7d TTL)  │
│  Quality gating: <60% success → auto-evict       │
├─────────────────────────────────────────────────┤
│  LAYER 1: NORMALIZATION                          │
│  lowercase, strip stop words, sort keywords      │
│  "Build a React page" = "build page react"       │
├─────────────────────────────────────────────────┤
│  TRANSPORT: GossipSub + DHT over libp2p          │
└─────────────────────────────────────────────────┘
```

## Cache Resolution

When a task arrives, Architect tries to skip inference entirely:

| Step | Method | Latency | Tokens |
|------|--------|---------|--------|
| 1 | Local cache (SHA-256 exact match) | ~2ms | 0 |
| 2 | Similarity index (MinHash + cosine) | ~15ms | 0 |
| 3 | DHT provider query (fetch from peers) | ~200ms | 0 |
| 4 | Architect LLM inference (fallback) | ~3-8s | 500-2,000 |

Each resolution caches the result. Future lookups are faster for everyone.

## P2P Gossip — Why It Matters

On a centralized platform, cached plans live on one server. On Hyperspace, **every node contributes to and benefits from a shared plan cache**.

```
Node A (GPU server):  47 k8s deployment plans
Node B (laptop):      23 React build plans
Node C (browser):     31 API endpoint plans
    ↓
    GossipSub announcements
    ↓
New node joins → DHT query → finds plans instantly
→ Zero inference. Zero tokens. Just cache hits.

More nodes → more plans → fewer misses → intelligence compounds.
```

Three gossip channels:

```
/hyperspace/dag-cache/1.0.0

Gossip topics:
  hyperspace/dag-cache/announcements  — new plans available
  /dag-plans/<hash>                   — DHT provider records
  hyperspace/dag-cache/outcomes       — execution results (self-curating)
```

**Self-curating cache:** Every execution records success/failure. Plans with <60% success rate after 5 samples are auto-evicted. Good plans survive. Bad plans die.

**Similarity clustering:** "Deploy React app" and "deploy Vue app" share 80% structure. MinHash finds these connections. One plan seeds a family.

## Data Model

```typescript
interface TaskDag {
  subtasks: DagSubtask[]       // Typed subtasks with agent assignments
  edges: DagEdge[]             // depends_on | feeds_into | blocks
  criticalPath: string[]       // Longest dependency chain
  maxParallelism: number       // Width of widest parallel level
  totalSubtasks: number
  confidence: number           // Architect's confidence (0-1)
  reasoning: string            // Why this decomposition
}

interface DagSubtask {
  id: string
  description: string
  agentType: string            // coding | design | infra | testing | research
  estimatedDurationMs: number
  dependencies: string[]
  parallelizable: boolean
  priority: number             // 1-10
}
```

## Implementation

| Package | File | What |
|---------|------|------|
| `@hyperspace/network` | `dag-cache/dag-content-store.ts` | Content-addressed LRU store |
| `@hyperspace/network` | `dag-cache/dag-dht-resolver.ts` | DHT + GossipSub sharing |
| `@hyperspace/network` | `dag-cache/dag-similarity-index.ts` | MinHash + cosine matching |
| `@hyperspace/network` | `dag-cache/dag-cache-stats.ts` | Analytics + latency tracking |
| `@hyperspace/agent` | `research-dag.ts` | Flywheel research DAG |

## License

Part of the [Hyperspace](https://hyper.space) open network.
