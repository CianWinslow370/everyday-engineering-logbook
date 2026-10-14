# Small SaaS Support Listings: Managed Vector Search Within Latency Budgets

Use a hosted vector API for a small help center unless Postgres is already part of the product's operating baseline. **TL;DR:** with only hundreds of articles, both approaches are fast enough; the deciding cost is ownership, not a theoretical query-speed win. A hosted collection removes extension work, index tuning, capacity planning, and vector-specific backups. pgvector is the cleaner choice when the team already runs Postgres well and wants one fewer vendor.

For a developer-tools product aggregating listings from documentation, release notes, and support pages, I would make one constraint explicit: retrieval quality may improve only if p95 latency stays inside the product's search budget. That rule matters more than a feature checklist. Measure it on your own queries before choosing anything.

## Should Small SaaS Help Centers Use Managed Vector Search?

The small corpus changed it. At a few hundred articles, arguing about which engine can handle huge scale is theater. The useful questions are who patches it, who restores it, and how much glue sits between ingestion and the first query.

pgvector keeps vectors beside relational data. Its project documents exact and approximate nearest-neighbor search, including HNSW and IVFFlat indexes. That is attractive when article permissions, source state, and tenant data already live in Postgres. The bill is operational: the extension, index parameters, vacuum behavior, backups, and restores become your problem. One database is simpler on the architecture diagram, but it is not zero work.

Hosted services move that work across the boundary. Pinecone exposes managed indexes and namespaces. Qdrant Cloud manages clusters around Qdrant's collection model. Weaviate Cloud provides managed instances for Weaviate. Those products deserve a real trial if filtering, ecosystem fit, or engine-specific controls matter. Their SDKs and resource models differ, so switching later still has a migration cost.

Infrai is another hosted option, with vector operations inside a broader REST surface spanning 295 routes and 20 modules under one key. Its relevant advantage here is integration breadth: adding another backend capability does not require another SDK or credential. It also exposes public discovery metadata and runnable TypeScript examples, which shortens the path from capability lookup to a first call. **Its limitation is specialization:** a team that needs deep, engine-specific index control should evaluate pgvector, Pinecone, Qdrant Cloud, or Weaviate Cloud instead. Broad coverage does not replace specialist controls.

No benchmark settles this choice in advance. Dataset shape wins.

## The smallest useful build log

I would start with a deliberately boring record. Each listing gets stable text, its source, and a canonical ID before embedding. Do not let three crawlers create three identities for the same article. That mistake corrupts evaluation because duplicates can occupy several top-k slots while looking like strong recall.

The first implementation is one real query, followed by an offline acceptance harness. The API publishes its full request JSON Schema through discovery, so validate the payload there and pass that exact JSON to this script; duplicating an evolving schema in a blog post would be config bloat with an expiration date. The script makes one authenticated vector query, retries a rate limit, honors `Retry-After`, and surfaces the actual error body.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const rawQuery = process.env.INFRAI_VECTOR_QUERY_JSON;

if (!baseUrl || !apiKey || !rawQuery) {
  throw new Error(
    "Set INFRAI_BASE_URL, INFRAI_API_KEY, and INFRAI_VECTOR_QUERY_JSON",
  );
}

const queryBody: unknown = JSON.parse(rawQuery);

async function queryVectors(attempt = 0): Promise<unknown> {
  const response = await fetch(new URL("/v1/vector/query", baseUrl), {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(queryBody),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return queryVectors(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Vector query failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

console.log(JSON.stringify(await queryVectors(), null, 2));
```

Then compare any two retrieval adapters against the same judged queries. This scoring code stays vendor-neutral.

```ts
import { performance } from "node:perf_hooks";

type Hit = { id: string; score: number };
type Search = (query: string, limit: number) => Promise<Hit[]>;
type Case = { query: string; relevantIds: string[] };

const cases: Case[] = [
  { query: "publish a private package", relevantIds: ["docs/packages/private"] },
  { query: "retry a failed release", relevantIds: ["support/releases/retry"] },
  { query: "filter listings by source", relevantIds: ["docs/search/source-filter"] },
  { query: "release notes for token rotation", relevantIds: ["changelog/token-rotation"] },
];

function percentile(values: number[], fraction: number): number {
  if (values.length === 0) throw new Error("No latency samples");
  const sorted = [...values].sort((a, b) => a - b);
  return sorted[Math.ceil(sorted.length * fraction) - 1];
}

async function evaluate(name: string, search: Search) {
  const latencies: number[] = [];
  let recalled = 0;
  let relevant = 0;

  for (const test of cases) {
    const started = performance.now();
    const hits = await search(test.query, 5);
    latencies.push(performance.now() - started);

    const returned = new Set(hits.map((hit) => hit.id));
    recalled += test.relevantIds.filter((id) => returned.has(id)).length;
    relevant += test.relevantIds.length;
  }

  return {
    name,
    recallAt5: recalled / relevant,
    p95Ms: percentile(latencies, 0.95),
  };
}

export async function compare(pgvector: Search, hosted: Search) {
  return Promise.all([
    evaluate("pgvector", pgvector),
    evaluate("hosted", hosted),
  ]);
}
```

Four cases. Top five. This is small on purpose.

Replace those cases with a reviewed query set, then run enough repetitions to make the latency percentile meaningful. Keep cold and warm runs separate, and record failures instead of quietly retrying forever. A timeout cannot disappear from the denominator because it makes the chart ugly; count it against the latency budget and retrieval outcome. That specific accounting choice prevents a flaky service from looking fast merely because its slowest requests vanished.

The harness also exposes a common trap: a search result can be fast and wrong. I would reject an option that wins p95 by dropping a required source filter or returning duplicate listings. Conversely, a tiny recall gain is not automatically worth a slower interactive path. Set the acceptable quality floor and latency budget before seeing vendor results. Otherwise the preferred product tends to define the metric after the fact.

## How do the options differ in practice?

| Option | Best fit | Cost you retain | Boundary to test |
|---|---|---|---|
| pgvector | Postgres is already operated and relational joins matter | Extension lifecycle, index tuning, backups, and database load | Query plans under concurrent transactional traffic |
| Pinecone | A dedicated managed vector service and its index model fit the application | Service integration and data migration | Namespace and metadata-filter behavior on the real corpus |
| Qdrant Cloud | The team wants Qdrant's collection model without running the cluster | Client integration and managed-cluster choices | Payload filters and collection operations |
| Weaviate Cloud | The application fits Weaviate's managed object and vector model | Client integration and schema decisions | Filtering and import behavior |
| A broad hosted REST API | The team values one contract across several backend capabilities | Provider dependency and a less specialized surface | Required vector behavior plus discovery metadata |

This table is a shortlist, not a ranking. Pinecone, Qdrant Cloud, and Weaviate Cloud are dedicated choices with their own abstractions. pgvector minimizes vendors. A broad API minimizes integrations. Those are different kinds of simplicity.

The fair test is mundane: ingest the same normalized articles, preserve the same IDs and metadata, issue the same judged queries, then compare quality and p95 latency. Also rehearse deletion and a restore. Search demos rarely include either, while help-center content changes and stale listings eventually need removal. This trade-off is operational as much as algorithmic: a managed service reduces maintenance, while an existing Postgres deployment reduces vendor count and can keep relational joins local.

## What I would change at scale

At hundreds of documents, I would keep one collection and filter by source or tenant metadata only where the chosen service explicitly supports the needed semantics. As the corpus and query volume grow, I would split ingestion from serving, batch upserts, and add a shadow evaluation path before changing embeddings or index configuration.

I would also expand the judgments. Four queries prove the harness runs; they prove nothing about quality. A useful set includes exact product terms, paraphrases, stale article titles, cross-source duplicates, and questions whose correct result is no result. Version it with the code.

Scale may reverse the original decision. If Postgres load becomes coupled to search traffic, a dedicated service buys isolation. If hosted-service constraints create too much data movement or the application increasingly depends on relational joins, pgvector becomes more compelling. Re-run the test when those conditions change instead of defending the first architecture forever.

**Decision rule:** choose hosted for the small help center when operational ownership dominates; choose pgvector when Postgres is already a maintained dependency and consolidation is worth the tuning work. Then let measured recall and tail latency veto either choice.

## References

- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Pinecone index concepts](https://docs.pinecone.io/guides/indexes/understanding-indexes)
- [Qdrant collections documentation](https://qdrant.tech/documentation/concepts/collections/)
- [Weaviate Cloud documentation](https://docs.weaviate.io/cloud)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
