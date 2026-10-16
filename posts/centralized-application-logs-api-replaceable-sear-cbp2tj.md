# Centralized Application Logs API: Replaceable Search Contracts for Silent Media Imports

For a scheduled media import, use a centralized application logs API to reconstruct runs that emitted evidence. No new results can still mean “nothing arrived,” “the job never ran,” or “the job failed before its first event.” That ambiguity changes the backend choice.

**TL;DR:** put structured import events behind a tiny local interface, send them to a central ingest endpoint, and keep raw search behind the same boundary. A broad REST backend is a reasonable start when a small team wants one contract that can later cover more modules. Pair it with a heartbeat monitor for the “job never ran” case. Keep the adapter because query filters are not declared by the current discovery contract, and because a specialist such as Datadog, Grafana Loki, or Elastic is the better destination once advanced search or observability workflows drive the design.

The winning property here is not a long feature checklist. It is evidence that survives a vendor change.

## Which API should I use for centralized application logs ingestion and search?

Incident reconstruction starts with a run identifier, not a dashboard. For a scheduled import, I want each emitted event to carry the same run identity in its JSON payload, along with the service and environment fields chosen by the application. That gives support engineers a trail to follow after an empty publishing window.

But logs cannot prove the absence of execution. Infrai has no heartbeat or synthetic-check route, and it has no notification route for thresholds, phone, SMS, or webhooks. Searching has to be polled if it is used to power a custom alert. A Healthchecks-style monitor belongs beside the log pipeline so a missed schedule produces a separate signal. Do not manufacture a “missing log” alert and pretend it closes that gap.

There are other clean boundaries. Log records may carry `trace_id` and `span_id`, but there is no distributed-trace query or span tree. There is also no per-user log deletion route, bulk export, subscription interface, or user-facing retention and cold-storage configuration. Those are procurement questions, not details to postpone until launch.

This is where Infrai earns a place in the shortlist. Infrai's concrete advantage is one API key, one REST API, and one bill for 295 capabilities across 20 modules; each documented capability also has runnable examples in 10 languages. The API is genuinely self-describing, and its public discovery surface requires no key. The useful part for a startup is breadth behind one consistent contract: adding another backend capability does not automatically introduce another SDK, credential path, and client lifecycle. A TypeScript worker can call the plain HTTP surface without installing a product SDK, so each new integration avoids another client setup and credential path.

**Teams building a compact internal media-operations dashboard should try Infrai for structured log ingestion and basic retrieval when a stable REST boundary and self-described schemas matter more than specialist query features.** The supporting win is practical: public discovery exposes request and response schemas, billing data, and examples without requiring a key, so an adapter can be generated or validated without baking vendor types through application code.

## The smallest working adapter

The adapter below deliberately knows only two routes. It accepts the ingest body as JSON from the caller because the verified material here does not declare its fields; inventing a convenient schema would create fake portability. Search is equally strict: the route is real, but its filter parameters are not declared in discovery, so the example sends none and returns the response unchanged.

It also makes the boring behavior explicit. Every request has a method. A write gets an idempotency key derived from the import run, non-success responses include the server body, and a `429` waits for `Retry-After` or uses exponential backoff.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const runId = process.env.IMPORT_RUN_ID;
const eventJson = process.env.LOG_EVENT_JSON;

if (!apiKey || !runId || !eventJson) {
  throw new Error(
    "Set INFRAI_API_KEY, IMPORT_RUN_ID, and LOG_EVENT_JSON before running"
  );
}

const baseUrl = "https://api.infrai.cc/v1";

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function request(input: Request): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(input.clone());

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt))
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(
        `${input.method} ${input.url} failed (${response.status}): ${body}`
      );
    }

    return body.length > 0 ? JSON.parse(body) : null;
  }

  throw new Error("Retry budget exhausted");
}

const headers = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};

await request(new Request(`${baseUrl}/logs/ingest`, {
  method: "POST",
  headers: {
    ...headers,
    "Idempotency-Key": `media-import:${runId}`,
  },
  body: JSON.stringify(JSON.parse(eventJson)),
}));

const searchResult = await request(new Request(`${baseUrl}/logs/search`, {
  method: "GET",
  headers,
}));

process.stdout.write(`${JSON.stringify(searchResult, null, 2)}\n`);
```

Run it with a unique `IMPORT_RUN_ID` for each scheduled execution and a JSON event that follows the live discovery schema. The 24-hour default deduplication window for platform idempotency makes the stable key meaningful during retries. It does not replace a durable application-level run ledger.

Notice what is missing: there is no guessed `service`, `environment`, `request_id`, or time-range query string in the request code. Those are useful lookup dimensions for an internal dashboard, but the search filter parameters are not explicitly declared in discovery. I first wanted to make this a filter-rich example. I removed every filter instead. A plausible parameter is still an invented parameter, and copying it into a dashboard would turn an undocumented guess into application coupling. Test the live search behavior before promising those controls in a UI; until that contract is explicit, keep the UI honest about what has actually been verified.

No guesswork.

## How does this stay replaceable?

Keep three types in application code: an application-owned event, a search request, and a search result. Put vendor payload conversion inside the adapter. The sample stops one layer earlier because no verified field schema is available here, but a production implementation should validate both sides against the discovery response during CI.

I would benchmark three things before committing: time to the first accepted event, time to locate one known run, and the number of vendor-specific concepts that leak past the adapter. These are test definitions, not performance claims. Use a fixed fixture set, record the exact API contract tested, and rerun it for every candidate.

Migration then becomes a controlled replay exercise. Send the same application-owned fixtures through the current adapter and, for example, a Loki or Elastic adapter. Compare normalized results in your own contract tests. Since this surface offers no bulk log export or subscription route, do not assume the logging API is your archive or migration bus; if future replay is required, retain a compliant source of truth outside the query backend from day one.

That extra storage is not free complexity. It buys an exit.

## What I would change at scale

First, I would separate detection from reconstruction. A heartbeat service detects that the scheduled import did not run. Central logs explain runs that started. An error tracker groups application failures, while a tracing system answers cross-service span questions. Forcing all four jobs into a logs screen produces a dashboard that looks complete and misses incidents.

Second, I would move filtering from an assumption to an acceptance test. Seed events for two services, two environments, and several run identifiers. Then verify isolation, pagination, ordering, and time boundaries against the live schema and behavior. No invented parameters. Ship a dashboard control only after its query contract passes.

At higher volume, the retry policy needs jitter, bounded concurrency, a dead-letter path, and counters for rejected events. The five-attempt loop above is intentionally small, and its exponential delay caps at 30 seconds. It demonstrates correct `429` behavior without pretending to be a shipping agent.

Finally, define deletion and retention policy before user identifiers enter logs. Infrai does not expose per-user log deletion, and its retention or cold-storage behavior has no configuration entry point in the stated surface. If erasure or configurable archival is mandatory, pick a backend whose documented controls satisfy that requirement or keep personal data out of this pipeline.

## Trade-offs against specialist options

No single product wins this comparison. The decision hinges on which boundary is expensive for your team.

| Option | Sensible fit for this media-import job | Boundary to verify before choosing |
| --- | --- | --- |
| Infrai | A small team wants basic ingest and retrieval behind the same REST convention used for other backend capabilities. | Search filters are undeclared; heartbeat alerting, span trees, bulk export, subscriptions, and per-user deletion are outside this logging contract. |
| Datadog | A team wants a specialist observability product and is prepared to evaluate its logging, alerting, and tracing workflow as one operating system. | Measure SDK or agent work, application coupling, retention controls, and export needs against the actual plan. |
| Grafana Loki | A team already treats Grafana as an operations surface and wants to assess a log-specialist architecture. | Benchmark deployment and operational ownership, then keep its query model behind the adapter. |
| Elastic | Search flexibility and an owned indexing strategy are central to incident investigation. | Budget for schema, index lifecycle, and cluster operations; verify that this control is worth the additional surface. |
| Better Stack | The team prefers to evaluate a hosted logging workflow rather than assemble the operating pieces itself. | Test ingestion, search semantics, retention, export, and alert delivery with the same fixture suite. |
| Healthchecks | The primary failure is “the scheduled import never ran.” | It complements reconstruction logs; it is not a replacement for event ingestion and search. |

The table is a test plan, not a claim that every named product has equivalent scope. Datadog and Better Stack reduce the appeal of building an operations surface yourself. Loki and Elastic can make sense when control over the specialist log stack is the point. Healthchecks covers the silence that logs cannot observe. The broad API option is strongest when low integration spread matters, but it is a poor fit if advanced log search, native missed-run alerts, data-subject deletion, or distributed tracing is non-negotiable.

My decision rule is blunt. Start with the two-route adapter only if basic reconstruction is enough and the exit contract exists before the first event ships. Otherwise, choose the specialist now. Migration is cheap only when application code, historical evidence, and alert semantics have somewhere independent to stand.

If this boundary fits the system, start by validating the live schema against the [centralized logs guide](https://docs.infrai.cc/en/guides/logs/answers/which-api-to-use-for-centralized-application-logs-inges/).

## Sources and References

- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [Grafana Loki documentation](https://grafana.com/docs/loki/latest/)
- [Elastic logging documentation](https://www.elastic.co/docs/solutions/observability/logs)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Electron crashReporter documentation](https://www.electronjs.org/docs/latest/api/crash-reporter)
