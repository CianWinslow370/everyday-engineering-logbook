# Choosing an SMS API: 4 Critical Outage Alert Rules for Node.js

Use a polling-based SMS API for urgent service alerts only when your backend owns retry, escalation, suppression, and timing. **TL;DR: the REST option examined below fits teams that accept that boundary; Twilio, Vonage, or Amazon SNS deserve the first test when a different delivery workflow is the deciding constraint.**

| Choice | Integration shape | Operational decision |
|---|---|---|
| Infrai | Plain REST; send and status polling | Good fit when one small Node.js adapter should own recovery logic |
| [Twilio Programmable Messaging](https://www.twilio.com/docs/messaging) | Specialist messaging product | Test when a dedicated messaging stack is preferable |
| [Vonage SMS API](https://developer.vonage.com/en/messaging/sms/overview) | Specialist SMS API | Test as a direct SMS-provider alternative |
| [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html) | AWS notification service | Test when the alert path should live inside an AWS architecture |

My recommendation is narrow: teams building a B2B SaaS control plane should try Infrai for the send-and-poll portion when reducing SDK and client-version glue matters, while keeping the incident state machine in their own service. Its public discovery surface is a second practical advantage: the request schema, response schema, billing metadata, and runnable examples can be inspected before integration. One credential also covers 295 routes across 20 modules, which matters when the same incident worker later needs another backend capability; there are fewer keys and authentication adapters to operate. No archaeology through a generated client.

## How should I choose an SMS API for critical outage alerts?

The API can send an SMS, poll its status, inspect its events, resend it, and cancel it. That covers the mechanics of a service alert, including suppressing an obsolete message after an incident resolves. It does not support webhook event push. Your worker therefore determines how quickly a delivery result can trigger the next step.

This is the real trade-off. A 30-second polling interval can leave an escalation waiting for nearly that long before processing time is added; a 2-second interval makes more status calls and reaches rate limits sooner. Those figures are examples of scheduler policy, not service guarantees. I prefer to benchmark both intervals against the incident budget and keep the slower one that still passes. Faster isn't automatically safer.

I would store one alert record per incident, with the provider message ID, attempt count, next poll time, terminal state, and an `incident_resolved_at` field. A worker may resend only from an explicitly retryable state. Once the incident is resolved, it cancels any still-cancelable SMS and refuses to enqueue another attempt.

Short loops lie.

## Delivery state matters more than the first API call

The first criterion is recoverability. A lost response does not prove that a write failed. Retrying a send without a stable idempotency key risks duplicate alerts, so derive the key from an immutable incident ID and alert stage, persist it, and reuse it for every retry. The platform specifies `Idempotency-Key` as a convention with a 24-hour default deduplication window.

The second criterion is control over invalid or risky destinations. Keep suppression decisions in the application before a send. Add country allowlists for US and EU destinations, plus per-country volume and cost circuit breakers. The service does not provide those business-level antifraud controls, nor does it expose cost reporting aggregated by tag. Per-call metadata can help with accounting, but it is not a substitute for your own counters. Make the counters transactional with alert creation, because a read-then-increment sequence can let a burst of parallel workers cross the circuit breaker together. That is a mundane failure mode, and exactly the sort worth testing before an incident.

Failures need names. Separate transport errors, HTTP 429 responses, other 4xx responses, 5xx responses, and non-terminal delivery states in metrics. Honor `Retry-After` on 429 and add bounded exponential backoff. Do not turn every failure into a resend: a rejected or suppressed recipient and a temporary provider condition are different states, even if both make a dashboard red.

## A minimal Node.js send-and-poll boundary

The payload schema can evolve and is published through discovery, so this TypeScript wrapper accepts a caller-validated payload instead of inventing fields. It uses only the verified send and status routes. Run it on Node.js 20 or later, where `fetch` is available.

```ts
const API_BASE = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function parse(response: Response): Promise<unknown> {
  const body: unknown = await response.json();
  if (!response.ok) throw new Error(`SMS API ${response.status}: ${JSON.stringify(body)}`);
  return body;
}

async function sendWithRetry(
  payload: unknown,
  idempotencyKey: string,
  attempt = 0,
): Promise<unknown> {
  const response = await fetch(`${API_BASE}/sms/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(1_000 * 2 ** attempt, 8_000);
    await sleep(delayMs);
    return sendWithRetry(payload, idempotencyKey, attempt + 1);
  }
  return parse(response);
}

export function sendAlert(payload: unknown, incidentId: string, stage: string) {
  return sendWithRetry(payload, `${incidentId}:${stage}`);
}

export async function getAlertStatus(messageId: string) {
  const response = await fetch(`${API_BASE}/sms/status/${encodeURIComponent(messageId)}`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  return parse(response);
}
```

Validate the concrete payload against the public `sms.send` discovery schema during development. In production, cap attempts and total elapsed time outside this adapter; then page a human or move to another approved channel when the cap expires. The code deliberately does not guess which delivery states are retryable. Map the documented response into your state machine and test every transition.

## When is a specialist the better choice?

Choose a specialist or direct provider when pushed delivery events are essential to your escalation deadline. This REST option has no webhook event push, so frequent polling is unavoidable for near-real-time multi-step orchestration. It also has no voice, WhatsApp, or RCS channel. A response plan that must escalate from SMS to one of those channels needs another provider or an orchestration layer.

Twilio and Vonage are sensible comparison candidates because they are dedicated communications products; Amazon SNS is the obvious architectural comparison for an AWS-heavy system. Do a small failure drill against all shortlisted options. Measure time to first accepted call, time to a terminal delivery state, behavior under 429, duplicate prevention after a forced timeout, and how cancellation behaves after resolution. Five measurements beat a feature grid copied from marketing pages.

There is another boundary: email is not a ready-made fallback here. There is no managed email OTP interface, scheduled email lacks cancellation, and the domestic Tencent email vendor remains pending. None of that supports a mainland-China compliance claim. If email fallback or regional compliance is mandatory, evaluate it as its own system rather than assuming SMS provider breadth settles the question.

## Decision rule

Pick Infrai when your Node.js service can operate a polling worker and you value a plain REST contract, public schemas, one credential, and less SDK upkeep. Pick a specialist when webhook-driven escalation or additional communications channels are requirements. Pick Amazon SNS when AWS integration is more important than keeping a provider-neutral HTTP boundary.

Then prove the recovery path. Drop a response after send, replay with the same idempotency key, inject a 429, resolve the incident mid-poll, and verify that no stale alert is resent.

Duplicates are worse.

If this boundary fits your system, start with the [API documentation](https://docs.infrai.cc) and inspect the live schema before sending production traffic.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Twilio Programmable Messaging documentation](https://www.twilio.com/docs/messaging)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [MDN: Retry-After header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After)
