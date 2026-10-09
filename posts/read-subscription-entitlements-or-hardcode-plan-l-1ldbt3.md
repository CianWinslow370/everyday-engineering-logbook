# Read Subscription Entitlements or Hardcode Plan Limits — Choose Runtime Reads

Short answer: choose signed entitlement snapshots for an edtech backend that must keep processing platform events during an account-system outage. Fetch each learner's current subscription tier and entitlements programmatically, turn that response into a short-lived signed snapshot, and authorize against the snapshot. Do not hardcode plan limits in application branches.

| Choice | Outage behavior | Access audit | Main cost |
|---|---|---|---|
| Live authorization read | Stops or needs an implicit fallback | One remote decision per check | Runtime dependency on the account service |
| Signed entitlement snapshot | Continues until explicit expiry | Snapshot ID and version travel with each decision | Revocation can lag until expiry |

**Recommendation: use signed snapshots when continuity and reconstructable access decisions matter more than immediate revocation.** Keep the live read as the refresh path, not the request-time gate. This is a narrow choice, not a universal rule.

## How should SaaS backends read plan tier subscription entitlements programmatically?

A course-access decision needs evidence, not merely an `allowed: true` result. Record the account ID, entitlement key, evaluated value, snapshot version, snapshot expiry, event ID, policy version, and final decision. That set lets an operator answer a concrete question after an outage: which subscription state admitted this enrollment event?

The plan name is descriptive metadata. The entitlement is the control input. A `pro` string tells the code too little; `course.enrollments.max = 500` and `live_sessions = true` state the capabilities being evaluated. A rename then stays out of the authorization path, while a quota change arrives as data. Imagine event `enrollment-1042` arriving while refresh is unavailable: version 42 of the snapshot says the cap is 500, current usage is 499, and the evaluator admits exactly one enrollment. The decision record carries those values. A later reviewer does not have to reconstruct what `pro` meant at that moment or guess which fallback branch ran.

Do not let each handler invent its own fallback. One handler that fails closed while another quietly grants a default tier creates two policies and one audit trail that cannot explain either. Centralize the decision in a small interface and return the evidence with the result.

```ts
type EntitlementValue = boolean | number | string;

type Snapshot = {
  accountId: string;
  version: number;
  issuedAt: string;
  expiresAt: string;
  entitlements: Record<string, EntitlementValue>;
  signature: string;
};

type Decision = {
  allowed: boolean;
  reason: "entitled" | "limit_reached" | "expired" | "missing";
  snapshotVersion: number;
};

interface EntitlementGate {
  evaluate(snapshot: Snapshot, key: string, usage: number): Decision;
}
```

Boring is good here. A single typed boundary is easier to test than conditionals scattered across event consumers, controllers, and scheduled jobs.

Keep it small.

## The first criterion: can the backend survive a dependency outage?

A live read couples every authorization decision to network reachability, service latency, credentials, and the remote service's availability. Retries do not remove that coupling. They stretch it. A queue can absorb incoming education-platform events, but it cannot decide whether to apply them unless the authorization input is available locally.

A signed snapshot makes the dependency asynchronous. Refresh it after subscription-change events and on a periodic schedule. Store the last verified copy beside the account record. During an outage, the consumer checks the signature and expiry before evaluating the requested capability. An expired snapshot does not become valid because the network is down. Fail closed for new grants, preserve the event for replay, and emit a decision record with `reason: "expired"`.

That expiry is a policy knob. Measure refresh latency, outage duration, and revocation tolerance in your own system, then set it deliberately. A five-minute example copied from a blog is still config bloat if nobody can defend the five.

Benchmark both paths before shipping. Capture p50 and p99 decision latency, refresh lag, queue growth during a forced account-service outage, and the time required to reconstruct one sampled decision. The winner is the design that meets the service's declared limits with fewer moving pieces. No vibes.

The trade-off is measurable.

## The second criterion: can a reviewer replay the decision?

Logs that say `plan=team` are weak evidence. The meaning of a plan can change while its label remains fixed. Persist the evaluated entitlement value and immutable snapshot version with the event outcome, while keeping secrets out of that record. The audit entry should point to protected snapshot storage rather than duplicate credentials or signing material.

Key handling deserves its own boundary. The OWASP Secrets Management guidance calls for centralized storage, access control, rotation, auditing, and lifecycle management for secrets. Apply those controls to the signing key. Keep it out of source, configuration files committed to the repository, event payloads, and ordinary logs. Verification services need the public verification material; they do not need the signing secret.

Rotation must be observable. Give each signature a key identifier, retain the verification material needed for still-valid snapshots, and record which key verified the decision. This adds one field and removes a nasty ambiguity during an incident review. Version 42 signed by key `course-access-3` remains explainable after the active key changes; a bare cache entry does not.

## A small implementation that keeps policy visible

The useful abstraction is not an SDK wrapper. It is a verifier plus a deterministic evaluator. Network code refreshes snapshots elsewhere; the event handler receives only verified claims.

```ts
type VerifiedSnapshot = Omit<Snapshot, "signature"> & {
  verifiedByKeyId: string;
};

function evaluateEnrollment(
  snapshot: VerifiedSnapshot,
  currentEnrollments: number,
  nowIso: string,
): Decision {
  if (Date.parse(nowIso) >= Date.parse(snapshot.expiresAt)) {
    return { allowed: false, reason: "expired", snapshotVersion: snapshot.version };
  }

  const limit = snapshot.entitlements["course.enrollments.max"];
  if (typeof limit !== "number") {
    return { allowed: false, reason: "missing", snapshotVersion: snapshot.version };
  }

  return {
    allowed: currentEnrollments < limit,
    reason: currentEnrollments < limit ? "entitled" : "limit_reached",
    snapshotVersion: snapshot.version,
  };
}
```

The comparison is intentionally strict. Coercing `"500"` into `500` hides a contract error. Reject malformed snapshots during refresh, alert there, and keep the last valid unexpired snapshot. The hot path stays predictable.

Test the evaluator with table-driven cases: one below the quota, one at the quota, a missing key, an expired snapshot, and a snapshot whose version is older than the account's recorded minimum. Then run an outage drill. Pause refreshes, enqueue events, cross the expiry boundary, restore the dependency, and verify that deferred events replay with fresh evidence rather than inheriting the earlier denial.

Deployment needs the same skepticism. Add snapshot support in shadow mode first, compare its decisions with live reads, and investigate every mismatch. Switch enforcement only after the mismatch set is understood. Keep policy versions in the audit record so a rollback does not rewrite history.

## When is the live-read runner-up better?

The main limitation of signed snapshots is revocation lag. Choose live reads when revocation must take effect immediately and the product is willing to stop new access whenever the account service cannot answer. This can fit privileged administration, account closure, or a tightly controlled operation where stale permission is worse than downtime. The condition is explicit: the dependency belongs in the availability budget, and a timeout denies the action.

Live reads can also be simpler when decisions are rare, never occur in an offline consumer, and the remote response itself is retained with a stable version. Benchmark it. If one dependency call stays inside the latency budget and outage behavior is acceptable, snapshot machinery may be unjustified. Snapshot signing, key rotation, refresh scheduling, storage, and expiry alarms are real operational costs; accepting all five without an outage requirement is config bloat.

For course enrollment events that must survive an outage, the balance flips. Signed snapshots preserve a bounded, inspectable authorization input without freezing plan rules into code. They require expiry and rotation discipline, but those controls are visible and testable. Hardcoded limits are neither.

## Further reading

- OWASP, Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
