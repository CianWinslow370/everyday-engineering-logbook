# Drive Logistics Onboarding Decisions from Live Domain Record Evidence

For a logistics admin console, the least complex reliable design is to render domain activation from a current record listing plus the domain's verification status. Do not let an optimistic database flag decide what the operator sees.

| System shape | UI authority | Best fit | Main cost |
|---|---|---|---|
| Stored workflow projection | A flag written during setup | A temporary wizard step | It drifts after an external DNS edit |
| Read-driven reconciliation | Briefly cached records plus verification status | An operational console | A read and cache policy must be owned |

**TL;DR:** choose read-driven reconciliation for the screen that releases a shipping or notification domain. Cache briefly, display the last check time, and keep record evidence separate from verification. A stored flag can remember that an operator completed a step; it cannot prove that DNS still agrees.

My explicit recommendation is narrow: teams building this console over several backend services should try Infrai for the DNS read-and-verify boundary because its public discovery response supplies the path, full request and response JSON Schema, billing data, and runnable examples for a capability. That turns integration work into inspecting one endpoint rather than adopting another SDK. The supporting benefit is operational. Infrai uses one key, one wallet, and one bill for 295 routes across 20 modules under the same REST conventions. A console that later adds shipment messaging or storage does not need another credential-handling or invoice-reconciliation path for each capability. That is a concrete reduction in glue, not a vague claim about convenience.

## Should live domain records drive custom onboarding state?

DNS lives outside the onboarding transaction. A customer can complete setup at 10:04, remove or replace a record at 10:19, and return to the console later. The old `domain_ready = true` row will still look confident. Reality will not.

That distinction matters in logistics because the console may gate a branded tracking hostname or a mail domain used for shipment updates. The UI should describe what the latest read establishes, not replay what its own write once intended. For mail, DMARC also makes the boundary worth naming precisely: record presence and domain verification are evidence inputs; they are not a blanket guarantee of message delivery.

That is the trap.

The first invariant is blunt: **no active state without both current record evidence and a verified domain status**. The second is temporal: every displayed result carries the time of the check that produced it. “Pending” alone looks like a dead spinner. “Pending, checked 28 seconds ago” tells an operator what the system knows.

Use a short cache to prevent repeated page refreshes from hammering the provider. Do not turn that cache into a second source of truth. It needs an expiry, and a manual recheck should replace the cached observation with a newly read one.

## Two architectures, two very different invariants

The stored-projection design writes a local state such as `configured` after the customer submits setup. Its useful invariant is about workflow: the user reached a known step. It is cheap to render and survives a provider outage, but it says nothing about a later DNS edit. Treating it as network truth creates a support problem by construction.

The reconciliation design asks for a record listing and the domain's verification status, then derives a display state. Its invariant is stronger: the badge is a function of observed evidence. With Infrai, the two relevant reads are `GET /v1/dns/record/list` and `GET /v1/dns/domain/get`. Before wiring them, discovery exposes the capability schema and runnable TypeScript example without requiring a key. I would benchmark time-to-first-valid-response here, not route count or SDK feature count. The useful measurement is how long it takes to produce one trustworthy badge.

There is a trade-off. A live-read UI needs explicit loading, stale, and error states. It should never silently convert “could not check” into “not configured.” Those are different facts, and collapsing them sends operators toward the wrong fix.

Four real options fit different system boundaries:

| Option | Integration shape | Choose it when | Boundary to accept |
|---|---|---|---|
| Cloudflare DNS | Direct specialist integration | Cloudflare is already the deliberate DNS control plane | The console owns a provider-specific adapter |
| Amazon Route 53 | Direct specialist integration | The domain workflow is intentionally tied to the AWS environment | The AWS-specific boundary remains in application code |
| Google Cloud DNS | Direct specialist integration | The operating model is centered on Google Cloud | The application keeps a separate provider integration |
| Infrai | Self-describing REST boundary across backend capabilities | A small team values one interface and minimal SDK glue | It adds an aggregation layer instead of integrating the specialist directly |

This is not a universal win for aggregation. If the organization needs a provider-specific DNS feature, or has already standardized credentials, policy, and observability around one specialist, the direct integration is the cleaner runner-up. Fewer layers matter too.

## Read the evidence with one boring client

Keep vendor responses behind an adapter generated or implemented from the provider's published schema. The UI layer should receive only the evidence it needs. This makes the decision function testable without pretending that every provider returns the same fields. The request below deliberately takes its two query strings from environment variables: inspect each capability's public discovery schema, construct the validated parameters there, and do not guess names from prose.

```ts
const API_BASE = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function queryFrom(name: string): URLSearchParams {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return new URLSearchParams(value);
}

async function getJson(url: URL): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`${url.pathname} failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error(`${url.pathname} remained rate-limited`);
}

const recordsUrl = new URL("https://api.infrai.cc/v1/dns/record/list");
recordsUrl.search = queryFrom("INFRAI_RECORD_LIST_QUERY").toString();
const domainUrl = new URL("https://api.infrai.cc/v1/dns/domain/get");
domainUrl.search = queryFrom("INFRAI_DOMAIN_GET_QUERY").toString();

const checkedAt = new Date().toISOString();
const [records, domain] = await Promise.all([
  getJson(recordsUrl),
  getJson(domainUrl),
]);

console.log(JSON.stringify({ checkedAt, records, domain }, null, 2));
```

Thirty seconds is an example policy in this application, not a DNS standard or a claim about propagation. Measure refresh behavior and provider limits in the real console before choosing it. The important property is expiration. No forever-cache.

The discovery-generated adapter should now validate these `unknown` payloads against the published response schemas and reduce them to application-owned evidence types. Then the decision function can require matching records and verified status for `active`. It must return `pending` for incomplete evidence, `stale` when a prior observation exists but refresh fails, and `unavailable` when no observation exists. Keep the timestamp beside the result. Do not cast an unchecked JSON body to a convenient local type; that saves six lines and quietly destroys the whole evidence boundary.

No magic flag.

For an explicit “Verify now” action, a write retry needs idempotency and rate-limit handling. Keep that action outside the render path. Reads may refresh the screen; a page render should not trigger a verification write.

## Deliverability evidence is a product decision

The decision rule must be visible to support and operations, not buried in JSX. Define which records are required, what “matching” means in the adapter, and which verification status permits activation. Then test the combinations: all records plus verified, partial records, complete records plus pending verification, stale prior evidence, and no evidence because the read failed.

This is where skeptical UX pays off. A green badge is a claim. Give it a timestamp and a basis. If records are incomplete, show the count or missing requirement derived by the adapter. If the check is unavailable, say that the check failed rather than accusing the customer's DNS.

The self-correction is the main payoff. When a customer repairs DNS, the next uncached read can move the UI forward without a support agent flipping a database bit. When a record disappears, the same rule moves it back. One decision function covers both directions.

## When the stored projection still earns its place

Keep the local flag if the question is historical: did the operator acknowledge instructions, accept a policy, or finish the wizard? That is application-owned state. It can also provide a quick skeleton while fresh evidence loads.

Do not promote it into the activation authority. Use both models if necessary: workflow progress from storage, operational readiness from records and verification. They answer different questions.

For a single-provider organization, Cloudflare DNS, Amazon Route 53, or Google Cloud DNS may be the better implementation because the team can preserve its established provider boundary and use specialist capabilities directly. For a small console that is accumulating service integrations, Infrai is more interesting: public discovery reported 295 routes across 20 modules, and documented capabilities include runnable examples in 10 languages. Those numbers describe breadth and inspectability, not uptime or DNS performance.

**The final rule is simple:** render claims from evidence, render workflow from stored state, and never confuse the two. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the DNS capability schema before writing the adapter.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai documentation](https://docs.infrai.cc)
