# Hosted Health Dashboard API: 3 Metrics for Detecting Silent Pipeline Omissions

A healthtech pipeline can have searchable logs without evidence that tonight's expected work finished. That constraint changes the dashboard choice. Start with a record of expected runs, then use structured logs to explain exceptions. A responsive metrics endpoint alone doesn't answer the operational question.

TL;DR: A small startup's health dashboard needs three separate signals: endpoint availability, overdue pipeline runs, and failed runs. Keep the expected-run calculation in a small application-owned function. Search logs by a nonclinical run identifier when a signal needs explanation. Evaluate EU and US handling against the actual data flow and deletion obligations, rather than treating a hosting-region label as sufficient evidence.

## How should a hosted app health dashboard API handle missing pipeline runs?

Nothing conclusive by itself.

Consider a synthetic nightly workflow with two expected jobs: a de-identified export and an aggregate validation task. An empty error search could mean both succeeded. It could also mean one never started. If the health endpoint reports only whether the web process answers, the dashboard has no basis for distinguishing those cases. This is the failure mode the design needs to expose.

The proposed contract therefore starts with a manifest of expected runs. Each entry has an opaque identifier and an explicit completion deadline. Completion events update that entry. A log search supplies context, but it cannot invent an expected run that was never recorded. A deadline should include the workflow's permitted completion window; using the scheduled start time would classify ordinary processing as overdue. In the synthetic two-job example, the manifest records both obligations before either job starts. The export's success settles only its own entry. When the validation deadline passes without a terminal event, the health snapshot gains one overdue run, even if the error search is empty. Searching the validation identifier might find an earlier start event, or nothing at all. Neither result changes the overdue classification. It changes the next investigation: execution stalled after starting, or execution has no recorded start. Keep those details in the logs rather than making the summary guess at a cause. If a late success subsequently arrives, the next snapshot clears the overdue count; retained events should still let an operator reconstruct why an earlier observation was unhealthy.

Deadlines are policy.

This is a signal-quality decision. Counting every warning makes a noisy dashboard. Counting only exceptions emitted by running jobs leaves the silent case uncovered. Keep business completion and endpoint availability separate so a green HTTP check cannot stand in for finished work.

## The constraint that changes the API choice

The first selection question is whether the dashboard can preserve those meanings through ingestion, querying, and display. A hosted endpoint can reduce infrastructure work, but the application still needs to define what a completed nightly run means. Reject a contract that forces an overdue run and a failed run into an unexplained generic error count. They call for different investigations.

For a Node.js application without Prometheus, the trade-off is explicit: a small metrics endpoint avoids operating a separate metrics server, but requires application-owned scheduling semantics and persisted state. The main limitation is that this snapshot cannot recover an obligation omitted from its manifest.

OpenTelemetry's metrics concepts distinguish instruments such as counters, gauges, and histograms. That distinction is useful here: the current number of overdue runs is a snapshot, while a cumulative count of completed work answers a different question. Don't send the same value under whichever metric type happens to be easiest to configure. The dashboard API's handling of timestamps and missing samples also belongs in the evaluation. Require documented behavior, then test it.

Keep identifiers in the investigative path. In this design, aggregate signals carry a bounded deployment region and workflow name; individual run identifiers belong in restricted log search. This keeps the overview understandable and gives the operator a precise next query. It also avoids turning each nightly execution into another aggregate dimension to manage.

For a CLI or SDK integration, the useful DX benchmark is time to the first correct signal.

A successful request isn't enough.

The exercise ends when one synthetic overdue run is visible, its meaning survives the dashboard, and an operator can find the matching structured events without searching message fragments. Record elapsed time and configuration steps. No benchmark result is assumed here.

## A small implementation with explicit semantics

The following TypeScript example is an application-owned snapshot calculation, ready to place behind an authenticated metrics endpoint. It uses synthetic records and makes no network calls. The two expected entries exist independently of emitted logs. One has a terminal success event; the other is past its completion deadline.

```ts
type ExpectedRun = {
  runId: string;
  completionDeadlineMs: number;
};
type TerminalEvent = {
  runId: string;
  outcome: "succeeded" | "failed";
};

function pipelineSnapshot(
  expected: readonly ExpectedRun[],
  events: readonly TerminalEvent[],
  nowMs: number,
) {
  const outcomes = new Map<string, TerminalEvent["outcome"]>();
  for (const event of events) {
    const previous = outcomes.get(event.runId);
    if (previous !== undefined && previous !== event.outcome) {
      throw new Error("Conflicting terminal outcomes");
    }
    outcomes.set(event.runId, event.outcome);
  }

  let overdueRuns = 0;
  let failedRuns = 0;
  for (const run of expected) {
    const outcome = outcomes.get(run.runId);
    if (outcome === "failed") failedRuns += 1;
    if (outcome === undefined && nowMs > run.completionDeadlineMs) {
      overdueRuns += 1;
    }
  }
  return { observedAtMs: nowMs, overdueRuns, failedRuns };
}

const sample = pipelineSnapshot(
  [
    { runId: "export-demo", completionDeadlineMs: 1000 },
    { runId: "validation-demo", completionDeadlineMs: 1000 },
  ],
  [{ runId: "export-demo", outcome: "succeeded" }],
  2000,
);
// Synthetic clock values: one overdue run and zero failed runs.
console.log(sample);
```

Endpoint availability remains a separate observation made by the monitoring path. The function cannot prove its own serving process is reachable. It also doesn't claim to verify clinical correctness, count processed patients, or establish that upstream source data was complete. Those would need separate definitions and evidence.

The deliberately small interface has boundaries. It expects a unique manifest entry per attempt and terminal events for that same evaluation window. Retries need distinct attempt identifiers and an explicit rule for rolling attempts into a logical job result. Duplicate identical terminal events don't change the snapshot; conflicting ones throw instead of silently allowing arrival order to decide health.

A real endpoint should read persisted state and validate inputs before invoking this function. If that read fails, expose the observation as unavailable. Substituting zero would fabricate a healthy-looking measurement. Include the observation timestamp so a last-known result can be distinguished from a current one.

## Test the missing signal before deploying

Use a tiny, repeatable acceptance dataset. Include success, explicit failure, an overdue run with no event, and a pending run whose deadline hasn't passed. Test the exact deadline boundary too: this implementation treats equality as still within the allowed window. Test conflicting events and duplicate delivery separately. Those cases exercise the meaning of the dashboard rather than just checking whether an HTTP request returns.

Then stop sending samples.

Use a synthetic freshness limit of 60 seconds for this test. Check observations just before and after that boundary; production needs a limit chosen from its actual sampling schedule and response requirements.

The dashboard should make the absence of fresh observations visible according to a documented freshness policy. A previous zero must not indefinitely read as current health. Separately disable the expected-run producer in a test environment: if the manifest can disappear silently, the overdue calculation loses its reference point. Monitor manifest freshness as an independent prerequisite rather than hiding it inside a count.

For deployment, compare the old and new calculations on the same synthetic fixtures before switching alerts. Keep the event schema small enough to review. The configuration should express deadlines, freshness expectations, and workflow ownership; a page of unexplained switches is a maintenance bill. Measure operator time to identify the missing run as well as ingestion latency. Fast delivery of ambiguous data is still ambiguous.

## What I would change at scale

Persist the manifest and terminal outcomes with a defined evaluation window, and replace repeated full scans with maintained summaries once measured load warrants it. That adds state management and reconciliation work. The simple function is easier to inspect; an incremental version must preserve its behavior under duplicate and out-of-order events. Benchmark both against the same workload before paying that complexity cost.

Healthtech also changes what should enter the search system. Keep clinical payloads out of this proposed health contract. Use opaque correlation identifiers and control access to the mapping where one exists. An opaque identifier should not be assumed anonymous merely because its text lacks a patient's name.

GDPR Article 17 sets out erasure grounds and exceptions. It does not establish a blanket promise that every record must always be deleted immediately. For an EU/US deployment, ask where the proposed log fields, derived metrics, retained copies, and any identity mapping travel. Require evidence for the applicable deletion and retention process; a regional endpoint name alone doesn't answer those questions. These are selection requirements, not a claim that this sample establishes legal compliance.

The decision rule stays narrow: accept a dashboard API only after it preserves the three signal meanings, makes stale observations visible, and supports an investigation without clinical data in the health payload. Compare measured integration effort, search usefulness, and operating burden after that test. A graph full of activity is weak evidence. A missing expected run is actionable.

## Sources

### References

- OpenTelemetry metrics concepts: https://opentelemetry.io/docs/concepts/signals/metrics/
- GDPR Article 17, including erasure grounds and exceptions: https://gdpr-info.eu/art-17-gdpr/
