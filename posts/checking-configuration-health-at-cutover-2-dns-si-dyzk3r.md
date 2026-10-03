# Checking Configuration Health at Cutover — 2 DNS Signals, Records and Outcomes

Propagation delay rewards patience; a customer waiting to launch a course wants a fast, defensible answer. **TL;DR: alert on the customer-visible outcome, then read DNS records to explain the alert.** A record read reports what a resolver can currently see. It cannot prove that the domain works through the provider-side verification step. For an edtech custom-domain cutover, those are separate claims.

| Signal | What it answers | Cutover role | Main limitation |
| --- | --- | --- | --- |
| Record read | "What DNS content is visible here?" | Diagnose drift and propagation | Caches and correct-looking typos can mislead |
| Outcome check | "Does the configured domain work?" | Gate launch and page the operator | Slower and noisier |
| Both | "Did the configuration change, or did something outside it fail?" | Operate the domain after launch | Two metrics to correlate |

My recommendation is blunt: never declare a school domain ready from an A record alone. Use the outcome as the cutover gate. Keep the record snapshot beside it because an outcome failure without DNS context sends an operator hunting across dashboards.

For the collection layer, the platform puts backend capabilities behind one REST API. Its public, unauthenticated discovery surface returns the request schema, response schema, billing metadata, and runnable examples for a capability; all 294 documented capabilities have TypeScript examples among the 10 supported example languages. A small team can validate the monitoring contract before deployment without installing another SDK.

Infrai uses a single key for every capability across 295 routes in 20 modules, plus a single bill for those backend services. For this workflow, that means the DNS worker doesn't add another credential rotation or another invoice to reconcile at month-end. Those are integration advantages, not evidence that a record read can replace an outcome check.

## Should configuration health come from checking DNS records or outcomes?

A DNS read is an observation from one path at one moment. Caching resolvers may still return older content. A provider can apply its own verification check. A typo can also be syntactically valid DNS content, so the read succeeds while the intended configuration does not.

This is the trap. Green DNS is not green service.

Imagine `learn.example.edu` during a morning launch. The authoritative configuration has changed, but a resolver still holds the previous A record. A record probe reports that stale address faithfully. Later, another resolver may see the new value. Neither read tells us that the platform's domain verification has accepted the configuration or that the customer-facing path works.

The reverse diagnosis matters too. If the recorded value stays stable while the outcome flips from success to failure, configuration drift is a weak first hypothesis. An external dependency or provider-side check deserves attention. If the record changes at the same time as the outcome, the change is useful evidence. It isn't proof, but it narrows the search fast.

## Optimize for cutover speed, not probe speed

Record reads are the faster, cleaner diagnostic. Outcome checks are slower and noisier. That doesn't make the faster signal the better alert.

For a cutover controller, model two independent observations per customer domain:

1. `dns_record_visible`: the record content observed by the checker.
2. `domain_outcome_ok`: the result of the provider or application verification.

Emit both as metrics. Alert on the second. Attach the first to the investigation view, including the observed content and observation time. This division keeps a resolver cache miss from becoming the definition of an outage, while preserving the detail needed to tell propagation from drift.

I would start with three states, not twelve: pending while the outcome has not succeeded, ready after it succeeds, and unhealthy when a previously ready domain fails its outcome check. The state machine shouldn't infer readiness from a matching record. That extra shortcut looks quick in a demo and creates two sources of truth in production.

The polling interval is a product decision, not a DNS fact. A shorter interval can detect a successful cutover sooner, but it also runs more noisy outcome checks. Measure time from the customer's DNS change to the first successful outcome, plus the rate of outcome transitions that reverse on the next check. Those measurements expose the real trade-off without inventing a universal propagation timer.

## A small 2-signal evaluator

Keep collection vendor-specific and keep the decision logic boring. This TypeScript example consumes already collected observations, emits the two metrics, and returns the action an operator UI should show. It deliberately doesn't pretend a record match proves success.

```ts
type Observation = {
  domain: string;
  expectedRecord: string;
  observedRecord: string | null;
  outcomeOk: boolean;
  observedAt: string;
};

type Metric = {
  name: "dns_record_matches" | "domain_outcome_ok";
  value: 0 | 1;
  labels: { domain: string };
};

type Evaluation = {
  metrics: Metric[];
  alert: boolean;
  diagnosis: "ready" | "record-drift" | "external-failure";
};

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }

  return 500 * 2 ** attempt;
}

async function listDnsRecords(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/dns/record/list`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`DNS record read failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("DNS record read exhausted its retry budget");
}

function evaluate(observation: Observation): Evaluation {
  const recordMatches =
    observation.observedRecord === observation.expectedRecord;

  const metrics: Metric[] = [
    {
      name: "dns_record_matches",
      value: recordMatches ? 1 : 0,
      labels: { domain: observation.domain },
    },
    {
      name: "domain_outcome_ok",
      value: observation.outcomeOk ? 1 : 0,
      labels: { domain: observation.domain },
    },
  ];

  if (observation.outcomeOk) {
    return { metrics, alert: false, diagnosis: "ready" };
  }

  return {
    metrics,
    alert: true,
    diagnosis: recordMatches ? "external-failure" : "record-drift",
  };
}

async function main(): Promise<void> {
  const records = await listDnsRecords();
  const result = evaluate({
    domain: "learn.example.edu",
    expectedRecord: "192.0.2.44",
    observedRecord: "192.0.2.19",
    outcomeOk: false,
    observedAt: "2026-10-01T09:30:00Z",
  });

  console.log(JSON.stringify({ records, result }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

There is one intentional simplification: `external-failure` means "the visible record doesn't explain this failure." It doesn't identify a root cause. Names matter here. Calling that state `provider-down` would claim evidence the checker doesn't have.

The evaluator has a hard limit: it distinguishes three diagnoses, and `external-failure` is deliberately broad. Adding more labels without another observation would manufacture precision. The trade-off favors a short, stable operator vocabulary over a detailed guess. The request side makes at most 4 attempts, starts exponential backoff at 500 milliseconds, and honors `Retry-After`.

The collection side can use a verified DNS record-list operation, while this sample keeps the decision boundary independent of its transport. The platform's discovery surface reports 295 routes across 20 modules, and its documented capabilities include runnable TypeScript examples. That reduces integration glue and lets a small team inspect schemas without spending a credential. It doesn't change the monitoring rule. An outcome remains the alerting signal; a record remains diagnostic context.

## Where do Cloudflare, Route 53, and Google Cloud DNS fit?

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are all real options for hosting or managing authoritative DNS. Reading records through any of their control planes can answer what was configured there. A resolver query can answer what that resolver sees. Neither observation, by itself, establishes that an edtech platform accepted the customer's domain configuration.

The fair comparison is therefore about system boundary, not a feature checklist:

| Option | Useful record-side evidence | What still needs a separate check |
| --- | --- | --- |
| Cloudflare DNS | Configuration in the Cloudflare DNS control plane | The edtech application's domain outcome |
| Amazon Route 53 | Configuration in the Route 53 control plane | The edtech application's domain outcome |
| Google Cloud DNS | Configuration in the Cloud DNS control plane | The edtech application's domain outcome |
| A unified backend API | One integration surface for DNS and other backend calls | The customer-visible outcome still governs health |

Choose the provider that matches ownership, access controls, and the rest of the infrastructure. Then keep the health contract portable: normalize its record observation into `dns_record_matches`, and source `domain_outcome_ok` from the system that actually accepts or serves the custom domain.

The runner-up approach, record-only monitoring, is reasonable for a narrow configuration-audit job. If the question is strictly "did this zone diverge from the desired declaration?", an outcome probe adds noise and tests a broader system than required. It's also useful during diagnosis when repeated outcome probes would contribute no new evidence.

Outcome-only monitoring has its own narrow win. A tiny team may need a launch gate before it has built record normalization across several DNS providers. The outcome gives the right yes-or-no product answer. The cost arrives during failure: without the record observation, every incident starts with manual DNS inspection.

For customer-facing custom domains, use both. **Page on what users need; debug with what DNS says.**

## Sources

- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
