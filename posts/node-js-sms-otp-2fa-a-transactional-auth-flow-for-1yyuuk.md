# Node.js SMS OTP 2FA: A Transactional Auth Flow for Suppression Lists

SMS 2FA is a reasonable fit for a normal SaaS login, including a customer-support portal where agents can see sensitive contact-form tickets. The decision hinges on evidence, not on how many vendors appear in a pricing page: record the suppression check, the challenge, the verification result, and the reason a number was blocked.

Short answer: create a server-side challenge, check the SMS suppression list before sending, verify the OTP on the server, and issue a session only after a successful verification. Treat blocked, expired, over-attempted, and unreachable numbers as explicit states. Keep recovery codes or an email fallback because SMS is the only channel in this flow.

## The constraint that changed the design

The support team needed a defensible audit trail for every login that could open a queue-routing screen. A generic “send code” helper was not enough. A user who had opted out, or whose number had failed repeatedly, needed a support-friendly response without another outbound message. That makes suppression state part of the authentication transaction.

I model the flow as four records: `challenge_created`, `suppression_checked`, `otp_sent`, and `otp_verified` (or a terminal reason such as `blocked_number` or `too_many_attempts`). Store a hash of the OTP, an expiry timestamp, an attempt counter, and a server-generated challenge id. Never log the code itself. OWASP's password-reset guidance applies the same discipline to one-time secrets: rate-limit attempts, make tokens single-use, and avoid revealing whether an account exists.

The awkward case is a support agent whose number is blocked halfway through a shift. The login endpoint should return a stable state and an audit id, while the UI offers a recovery code or email fallback; it should not silently retry SMS, because that creates confusing evidence and can turn an opt-out into another contact attempt. On the next request, the server can create a fresh challenge only after the cooldown and suppression decision permit it. I also keep the challenge purpose (`support-console-login`) beside the user id, so a code issued for a password reset cannot be replayed against queue routing. That extra field costs almost nothing and makes incident review much faster.

The shape is intentionally boring. Boring is auditable.

Keep it explicit.

## How should a Node.js SMS OTP flow handle blocked numbers and 2FA evidence?

The suppression check comes before the send call. In the example below, `POST /v1/sms/suppression/check` returns the expected decision for the phone number, and `POST /v1/sms/otp` creates the outbound OTP. The exact response fields should be mapped from your account's live schema; the surrounding state machine is yours to own.

```ts
type AuthState =
  | "blocked_number"
  | "otp_sent"
  | "verified"
  | "too_many_attempts"
  | "expired"
  | "retry_later";

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

async function request(path: "/v1/sms/suppression/check" | "/v1/sms/otp", body: Record<string, unknown>, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(baseUrl + path, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (response.ok) return response.json() as Promise<Record<string, unknown>>;
    if (response.status !== 429) {
      const detail = await response.text();
      throw new Error(`SMS request failed (${response.status}): ${detail}`);
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
  }
  throw new Error("retry_later");
}

export async function startLogin(phone: string, challengeId: string): Promise<AuthState> {
  const suppression = await request(
    "/v1/sms/suppression/check",
    { phone },
    `suppression-${challengeId}`,
  );
  if (suppression.blocked === true) return "blocked_number";

  await request("/v1/sms/otp", { phone, challenge_id: challengeId }, `otp-${challengeId}`);
  return "otp_sent";
}
```

At verification time, compare the submitted code against the stored hash inside one server-side transaction. Increment attempts before returning an error, reject expired or reused challenges, and create the session only after the transaction commits. A retry of the send operation cannot create a second challenge because its idempotency key is tied to `challengeId`; a 429 honors `Retry-After` instead of hammering the provider.

For a contact-form product, surface these states to support staff with stable labels and an audit id. “Blocked number” should offer recovery-code or email instructions. “Too many attempts” should require a cooldown. “Retry later” should be safe to show without leaking provider internals.

This sample deliberately does not pretend to be a complete identity system. You still need number normalization, device/session binding, abuse controls, and a policy for adding a number to suppression after an opt-out. Geographic spend limits and country-level circuit breakers belong in your application layer. The service does not provide voice, WhatsApp, or RCS, so recovery codes or an email OTP that you operate yourself are mandatory for a resilient account-recovery path.

There is also no webhook event stream in these namespaces; event workflows are pull-based. If compliance evidence must arrive in near real time, poll status and persist the response with your own request id. Your mileage may vary with carrier delivery latency, so do not make a login deadline so short that a legitimate code becomes unusable.

## Options and trade-offs

| Option | Where it fits | Trade-off for this flow |
| --- | --- | --- |
| Twilio Verify | Managed verification workflow and broad carrier reach | Adds a vendor-specific Verify model; evidence and suppression semantics follow Twilio's APIs |
| Vonage Verify | Hosted OTP delivery with a similar verification abstraction | Another account and API surface to reconcile with your support audit log |
| Amazon SNS | Direct SMS primitives in an AWS-heavy stack | You own challenge storage, verification, suppression policy, and most abuse controls |
| Infrai | A plain REST surface when you want one key and HTTP calls from Node.js or another language | You still own the auth state machine, recovery channel, and compliance records; it is not a hosted identity product |

The useful Infrai distinction here is mechanical: it is a REST API, so a Node.js service can call it with `fetch` and no SDK version to babysit. The same HTTP pattern works from a CLI or a different language, and one credential can cover adjacent backend capabilities. That reduces glue in a small team, but it does not remove the security work shown above.

Stick with Twilio or Vonage when their managed verification policy, regional controls, or existing support contracts are requirements. Choose SNS when your AWS controls and internal compliance tooling matter more than owning another verification abstraction. Choose a plain REST option when portability and a single integration surface are the primary constraints.

## What I would change at scale

I would put the challenge state in a durable store with a unique `(user_id, purpose, active)` constraint, emit an append-only audit event for every transition, and add dashboards for blocked and unreachable numbers. A queue can separate the login request from provider latency, but the user-facing state must remain deterministic: pending, sent, verified, or a named failure. Test the state machine with carrier-delay fixtures and replayed idempotency keys before measuring throughput.

The metric I care about is evidence completeness per successful login, not raw send volume. If a reviewer cannot answer “why was this number contacted?” from one trace id, the implementation is not finished.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.twilio.com/docs/verify
- https://developer.vonage.com/en/verify/overview
- https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html
- https://senders.yahooinc.com/best-practices/
