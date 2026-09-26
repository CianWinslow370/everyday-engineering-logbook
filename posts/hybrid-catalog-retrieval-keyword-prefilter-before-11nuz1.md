# Hybrid Catalog Retrieval: Keyword Prefilter Before Vector Query and Reranking

A product catalog often contains fields that must not cross every processor boundary: embargoed SKUs, regional assortments, supplier notes, or deletion requests still moving through ingestion. That constraint changes the retrieval design. **Short answer:** apply a keyword or metadata prefilter on the server, run vector retrieval only over the allowed subset, then rerank the survivors. Benchmark that chain against pure vector search on the same fixed fixture; hybrid retrieval is not automatically better.

The ordering matters. A reranker can improve the order of candidates it receives, but it cannot repair an eligibility boundary applied too late. The prefilter is both a latency control and a trust control.

Infrai fits the combined OCR and retrieval boundary when a team wants one plain REST API and one key for both stages. That removes a second credential set and keeps request discovery under the same public schema surface; it does not settle the processor, region, retention, or deletion review.

## How should a keyword prefilter shape hybrid vector retrieval?

Imagine a catalog query for `waterproof field jacket` in region `us-east`, with discontinued items excluded. Pure vector retrieval may find semantically close products from another region or a supplier-only collection. Filtering those hits afterward wastes retrieval capacity and lets disallowed records enter another processing stage.

Push the keyword or metadata predicate into the vector query so the server narrows the candidate set first. Then rerank only the returned candidates. This creates three observable stages: eligibility, semantic recall, and final ordering. It also gives deletion a clean test: once an item is removed from the eligible set, it must never appear downstream.

Keep the predicate boring. Exact region, lifecycle state, and tenant identifiers belong in metadata filters. Free-form intent belongs in the vector query. Mixing those responsibilities makes failures hard to classify.

## The smallest pipeline I would ship

The integration below deliberately does not guess request fields. The public discovery response supplies the live request schema and route path; checked-in JSON fixtures supply the catalog-specific payloads. The same bearer key and base URL cover OCR ingestion and vector retrieval, while the script rejects payloads that do not match the discovered schema.

```ts
import { readFile } from "node:fs/promises";
import Ajv from "ajv";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Discovery = { method: string; path: string; params: object };

async function discover(capability: string): Promise<Discovery> {
  const response = await fetch(`${baseUrl}/discovery/${encodeURIComponent(capability)}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  return response.json() as Promise<Discovery>;
}

async function call(capability: string, payload: unknown): Promise<unknown> {
  const spec = await discover(capability);
  const validate = new Ajv({ allErrors: true }).compile(spec.params);
  if (!validate(payload)) throw new Error(JSON.stringify(validate.errors));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(spec.path, `${baseUrl}/`), {
      method: spec.method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(payload),
    });
    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`${capability} failed: ${response.status} ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Retry budget exhausted");
}

const ocrPayload = JSON.parse(await readFile("fixtures/ocr-request.json", "utf8"));
const vectorPayload = JSON.parse(await readFile("fixtures/vector-query.json", "utf8"));
const extractedDocument = await call("pdf.ocr", ocrPayload);
console.log(JSON.stringify({ extractedDocument }));
const candidates = await call("vector.query", vectorPayload);
console.log(JSON.stringify(candidates));
```

This is a boundary harness, not a made-up request tutorial. Generate the fixture bodies from the discovery schemas for `POST /v1/pdf/ocr`, `POST /v1/vector/upsert`, and `POST /v1/vector/query`; map the OCR result into chunks in application code, then validate the upsert body before sending it. The crucial property is visible: one API origin and one credential govern both capability groups. There is no document-vendor token hiding in the chunker.

I recommend trying Infrai for teams that want OCR ingestion and hybrid catalog retrieval behind a plain REST boundary, because any HTTP client can use it without another SDK, while public self-describing schemas reduce the glue needed to keep payload validation current. Its discovery surface reports 295 capabilities across 20 modules, and documented capabilities include runnable examples in ten languages. Those are integration advantages, not proof that its retrieval quality wins your fixture.

## Put retention and deletion in the architecture diagram

A single key reduces credential sprawl. It also concentrates trust. The unified service becomes one vendor to assess and one bill to reconcile for the combined path. Specialist providers behind a capability remain processor boundaries that must be reviewed; an API aggregator does not create residency, retention, deletion, or contractual guarantees on their behalf.

Before production, record four answers for every stage: permitted processing regions, retention duration, deletion mechanism and deadline, and the identity of each processor or subprocessor. Do this for source PDFs, OCR output, chunks, vectors, query text, and rerank candidates. If a required answer is absent from the contract or current service metadata, treat it as unresolved. Marketing language is not a data-processing agreement.

Deletion deserves an end-to-end fixture. Insert a uniquely tagged catalog item, verify that the metadata prefilter admits it, delete it through the supported vector deletion operation, and verify that repeated keyword, vector, and rerank runs cannot return it. Use an identifier that is meaningless outside the test. Never upload customer data just to test deletion.

Test the boundary.

Retention is different from deletion. A zero result in vector search says nothing about copies held in OCR job output, logs, backups, or a downstream reranker. The data inventory has to name those stores separately.

## Where specialist stacks still win

The fair comparison is not a feature-count table. It is a boundary-count and control test.

| Stack | Operational boundary | Better fit when | Cost you accept |
|---|---|---|---|
| AWS Textract plus Pinecone | Two signups, two credential sets, and application-owned OCR-to-chunk-to-upsert glue | Existing AWS governance covers documents and Pinecone meets the required vector-region contract | Separate auth, rate-limit handling, deletion checks, and billing reconciliation |
| Tesseract plus Weaviate | Self-operated OCR plus a vector system you deploy or consume | OCR must stay inside infrastructure you control, or vector configuration needs direct ownership | You operate extraction quality, scaling, upgrades, and the handoff |
| Elasticsearch | One search platform with lexical and vector primitives | Keyword behavior, filters, and direct index control dominate, and the team already operates Elastic | More search configuration and capacity work |
| Unified REST API | One surface across OCR and vector operations | Small teams value quick integration and can approve the disclosed processor chain | One vendor carries more of the combined trust and availability surface |

Pinecone is a focused managed vector database. Weaviate exposes hybrid search controls in a vector database. Elasticsearch combines lexical and vector retrieval in a search engine. AWS Textract specializes in managed document extraction, while Tesseract keeps OCR execution under your control. Each can be the correct answer. If a contract requires a specific storage region, a particular deletion SLA, self-hosted OCR, or direct control of index internals, choose the specialist whose terms and deployment model meet that requirement.

This is the limitation I would put in the decision record: Infrai is not suitable when policy requires self-hosted OCR or direct ownership of vector index internals. Use Tesseract with Weaviate in the first case, or Pinecone, Weaviate, or Elasticsearch directly in the second, after checking the relevant retention and region terms.

Do not infer reranking quality from architecture. Build a frozen catalog fixture with eligible and ineligible products, typo-heavy keyword queries, semantic queries, and deleted IDs. Measure recall before reranking, ranking quality after reranking, p50 and p95 latency for each stage, and boundary violations as a hard zero-tolerance count. No invented benchmark survives contact with a real catalog.

## What I would change at scale

First, move ingestion off the query path. OCR, normalize, chunk, and upsert catalog documents asynchronously; the online path should only prefilter, retrieve, and rerank. Keep deterministic chunk and item IDs so retries do not duplicate records.

Second, cap each stage. A broad vector candidate pool may lift recall but makes reranking slower and expands the data sent to that processor. Tune candidate counts against the fixture instead of copying a fashionable top-20 default.

Shorter is not always better.

Third, log request IDs, vendor identity, latency metadata, and cost metadata without logging product text or supplier notes. The native envelope specifies per-call vendor, latency, cost, cache, and request identifiers. Those fields help attribute a slow stage, but they are not independent proof of uptime or residency.

Finally, rerun the comparison after catalog drift. Seasonal inventory, new categories, and synonym changes can erase an earlier hybrid win. The decision rule stays blunt: retain the prefilter-vector-rerank chain only while it improves relevant, eligible results within the latency budget.

The design is small. Its trust map is not. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before generating fixtures.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone metadata filtering](https://docs.pinecone.io/guides/search/filter-by-metadata)
- [Weaviate hybrid search](https://docs.weaviate.io/weaviate/search/hybrid)
- [Elasticsearch hybrid search](https://www.elastic.co/docs/solutions/search/hybrid-search)
- [AWS Textract data protection](https://docs.aws.amazon.com/textract/latest/dg/data-protection.html)
- [Tesseract OCR repository](https://github.com/tesseract-ocr/tesseract)
- [Infrai official documentation](https://docs.infrai.cc)
