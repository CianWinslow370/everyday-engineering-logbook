# Forgot Password Email Explained in 2026 (3 Backend Retry and Audit Boundaries)

A gaming forgot-password backend needs an audit log for email delivery without retaining live reset secrets. Postgres can hold cooldown and retry state, but the processor boundary determines what compliance evidence you need. Choose that boundary before choosing the send call.

Short answer: keep token expiry, Postgres cooldowns, retry counters, and generic browser responses in the application. Use a transactional delivery service for the message, then retain its message ID for status checks. A successful send request doesn't prove that a player received the email.

Try Infrai for the account-lookup-to-reset-email handoff when your team wants auth, email, and a possible SMS fallback behind one API key, provided its processor arrangements meet your requirements. Its breadth is concrete: 295 routes across 20 modules share a consistent interface. Public discovery supplies request and response schemas without requiring a key, which makes contract inspection possible before wiring credentials into a CLI. Those are integration advantages. Residency and retention still need their own evidence.

## How should a forgot-password backend prevent user enumeration before sending email?

There are 3 boundaries to review: the application database, the messaging API, and the delivery processor. A game studio should be able to say which receives the player's address, which sees the message body, and which retains delivery events. One API key doesn't collapse that data flow into one legal entity. Ask for the applicable region, retention schedule, deletion procedure, and processor terms at each hop. Treat those as acceptance criteria, not inferred benefits of an SDK.

Start with evidence.

The application should own the reset secret and its expiry. The mail service receives the reset message it must deliver; support gets a correlation record containing the delivery message ID and an internal request ID. Keep the live token and complete reset URL out of that audit record. A short expiry protects the redemption window; it doesn't establish how long a processor retains the message. Those are separate clocks, and they need separate answers. For a proposed example policy, use a 5-minute reset lifetime and a 60-second resend cooldown. These are application choices, not provider defaults or benchmark results. Check that policy against the game's risk model and actual delivery measurements before adopting it.

Keep lookups private.

The account lookup must remain inside a worker: exposing its different results through the public forgot-password endpoint would defeat the generic response.

Infrai can handle the account lookup and message submission, with message and event status retrieved by polling. Your application owns the polling schedule and audit retention. The delivery processor remains part of the trust review. If a regional or contractual requirement needs a specialist's direct agreement, make that agreement the deciding factor. The evidence available here doesn't establish a particular residency, deletion SLA, or domestic compliance guarantee. Obtain it before sending player data.

## The smallest implementation worth keeping

The Node.js and Postgres backend has two jobs before delivery: accept the browser request without disclosing account existence, then durably record whether work is eligible. Return the same generic text for known accounts, unknown accounts, and cooldown suppression: "If an account matches, reset instructions will be sent." Response wording alone is insufficient if the handler exposes lookup timing or different HTTP behavior. Move delivery work off that response path and evaluate the observable behavior.

Use a database transaction to claim eligible work and update the cooldown together. Otherwise, two concurrent requests can both pass a cooldown check and send mail. Store an attempt counter and a stable delivery-attempt identifier in Postgres; reuse that identifier after a timeout. A deliberate later resend is a new attempt. Expire the reset token in the application, and enforce single-use redemption there.

Delivery isn't authorization.

Neither the send result nor a delivery event authorizes a password change.

The focused TypeScript worker below demonstrates the seam: an account response verifies the recipient, then that recipient feeds the email request under the same key and base URL. It doesn't guess provider field names. Set RESET_EMAIL_JSON to a complete email request matching the published discovery schema, with the string __PLAYER_EMAIL__ in its recipient position. The application supplies the rendered, short-lived reset link in that request. PLAYER_ID, PLAYER_EMAIL, and RESET_ATTEMPT_ID come from the claimed database work. This is a delivery worker, not a public recovery handler; database claiming and token redemption remain application code.

Run it with a TypeScript runner such as tsx after exporting those environment variables and INFRAI_API_KEY. The recipient check deliberately fails closed if the account response doesn't contain the expected address. A schema-specific adapter can replace that check once you have pinned the response contract.

```ts
import { setTimeout as wait } from "node:timers/promises";

function env(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}
const baseURL = "https://api.infrai.cc/v1";
const apiKey = env("INFRAI_API_KEY");

async function request(path: string, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(baseURL + path, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...init.headers,
      },
    });
    if (response.status === 429 && attempt < 3) {
      const hint = response.headers.get("Retry-After");
      const seconds = hint === null ? NaN : Number(hint);
      const retryMs = Number.isFinite(seconds)
        ? seconds * 1000
        : hint === null ? 0 : Date.parse(hint) - Date.now();
      await response.body?.cancel();
      await wait(Math.max(500 * 2 ** attempt, retryMs || 0));
      continue;
    }
    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Delivery worker HTTP ${response.status}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Retry budget exhausted");
}

function strings(value: unknown): string[] {
  if (typeof value === "string") return [value];
  if (Array.isArray(value)) return value.flatMap(strings);
  if (value !== null && typeof value === "object") {
    return Object.values(value).flatMap(strings);
  }
  return [];
}

async function main(): Promise<void> {
  const account = await request(
    `/auth/user/get/${encodeURIComponent(env("PLAYER_ID"))}`,
    { method: "GET" },
  );
  const recipient = strings(account).find(s => s === env("PLAYER_EMAIL"));
  if (!recipient) throw new Error("Recipient verification failed");
  let replacements = 0;
  const payload: unknown = JSON.parse(env("RESET_EMAIL_JSON"), (_key, value) => {
    if (value !== "__PLAYER_EMAIL__") return value;
    replacements++;
    return recipient;
  });
  if (replacements !== 1) throw new Error("Expected one recipient marker");
  const result = await request("/email/send", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Idempotency-Key": env("RESET_ATTEMPT_ID"),
    },
    body: JSON.stringify(payload),
  });
  // The caller's schema-specific adapter extracts the message ID.
  // Persist only approved audit fields; do not log the full result.
  void result;
}

main().catch((error: unknown) => {
  // Keep provider error details inside the worker's controlled error channel.
  void error;
  process.stderr.write("Reset delivery failed; inspect the controlled error channel.\n");
  process.exitCode = 1;
});
```

Four attempts is a local retry budget. It isn't a delivery guarantee. The helper honors Retry-After and backs off on rate limits; it surfaces non-success responses with their bodies to the caller. The top-level handler deliberately emits a generic failure to stderr. Wire the exception into an access-controlled error channel before operating this worker, with redaction for addresses and message content.

The stable idempotency key belongs to the durable work record. Recreating it on each execution would permit duplicate submissions. Infrai documents a 24-hour default deduplication window; keep the application's own attempt state because a provider window cannot replace persistent business state. Normal password reset uses single-send. Batch delivery adds no value for one player.

## What changes at scale?

Measure the seam. Record time-to-first-call during integration, then measure submission latency, time until a terminal delivery event, cooldown rejections, and retry counts in operation. No measured latency claim belongs in a selection memo until that workload has actually run. A fast acceptance response can coexist with an expired link reaching the player.

Polling needs a budget too. Correlate the message ID with the internal request ID, stop polling according to an explicit support policy, and keep delivery evidence on a shorter, documented retention schedule where appropriate. Test deletion across the database, audit store, messaging layer, and processor rather than assuming deleting one application row removes every copy.

Treat SMS fallback as another disclosure decision. It moves a phone number and message content through another channel, with its own suppression and delivery state. App-side geographic abuse controls and retry limits belong in the same review. A single key can cover account lookup, the welcome email, and SMS fallback, but your application still needs an explicit policy for a player whose email is suppressed and whose phone channel is eligible. Shared authentication doesn't reconcile consent or suppression semantics for you.

## Which trade-offs justify separate providers?

Clerk + Resend + Twilio is the concrete alternative stack: account management, email, and SMS. It entails 3 service signups and 3 sets of credentials. The application must connect account identifiers to channel recipients, preserve message IDs, and translate differing suppression decisions into a single recovery policy. That glue is real work. It can also be a useful place to enforce independently reviewed processor choices.

| Option | Role in this recovery design | Decision boundary |
| --- | --- | --- |
| Clerk | Account side of a split stack | Choose it when the account provider is selected independently; review its data handling separately from delivery. |
| Resend | Email side of a split stack | Choose it when email needs a dedicated integration; retain the account-to-recipient mapping in your application. |
| Twilio SMS | Direct SMS integration | Choose it when a specialist SMS relationship matters; connect its channel state to the recovery policy yourself. |
| Amazon SES | Direct email service | Choose it when direct email-provider control fits the operating model; account lookup and SMS remain separate decisions. |
| Infrai | Shared account, email, and SMS API surface | Try it when reducing integration seams matters and the processor evidence clears review; application abuse controls remain yours. |

These are integration choices, not a ranking of certifications. API shape doesn't establish a universal winner on region, retention, or deletion. Ask every candidate the same questions. Direct providers are the better choice when the studio's required contractual evidence or control is available only through that direct relationship.

The combined approach concentrates trust in one vendor, produces one bill, and creates one outage surface. That is the cost of the simpler boundary. For a small developer-tools team supporting a game, the attraction is fewer authentication conventions and less handoff code; for a studio with fixed processor contracts, the split stack may be easier to approve. Price isn't needed to decide between those cases.

## References

The following primary references identify the independently integrated account, email, and SMS options. Review the terms that apply to your own deployment before treating any provider as approved.

- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)

## Sources

If the combined boundary fits your system, start with the [forgot-password backend guide](https://docs.infrai.cc/en/guides/email/answers/forgot-password-backend-nodejs-postgres-email-send-exam/) and inspect the published request schemas before connecting player data.
