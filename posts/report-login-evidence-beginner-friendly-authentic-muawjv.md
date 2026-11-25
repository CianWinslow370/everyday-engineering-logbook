# Report Login Evidence: Beginner-Friendly Authentication API with SMS and Email Fallback

Short answer: choose a beginner-friendly SMS authentication messaging API for report login only if your B2B SaaS app can own email fallback and the record connecting the authenticated account to the generated report. For US/EU use, require regional compliance evidence separately. An accepted request alone cannot establish that connection.

| Candidate | Reason to evaluate it | Evidence required before choosing |
| --- | --- | --- |
| Combined identity and messaging | Infrai offers plain REST calls without an SDK to install or version, with identity lookup and SMS behind one key; public discovery supplies request and response schemas. | Demonstrate the identity-to-recipient mapping and review regional and contractual evidence separately. |
| Auth0 plus Twilio Verify | A candidate for choosing a dedicated identity product separately from verification delivery. | Demonstrate account-to-challenge binding across two services. |
| Clerk plus Twilio Verify | Another candidate for a separate identity-product decision. | Test the account workflow against the application; don't assume automatic verification integration. |
| Twilio SMS with application-owned authentication | A messaging candidate when the application already owns identity and challenge state. | Include the challenge lifecycle and audit record in the implementation estimate. |
| Amazon SES or Postmark as a separate email candidate | Consider either during a separate report-email evaluation, rather than treating an SMS decision as an email decision. | Require an attachment demonstration and the applicable retention and processing commitments from each. Neither is established here as the login-verification component. |

**Short answer:** shortlist the combined REST option for SMS-first login when reducing credentials and SDK setup matters, and your application can maintain the report-access record. Prefer a separate identity product when its demonstrated account workflow outweighs the extra integration. Require attachment support and US/EU compliance evidence independently; SMS OTP doesn't prove either.

## What should an authentication messaging API prove about SMS login OTP?

Start with the job. A customer signs in, generates a report, and asks for it as an email attachment. There are at least three separate decisions: whether the login challenge succeeded, whether that account may access this report, and whether the intended recipient should receive the attachment. Combining them into one green status makes an attractive dashboard and a poor explanation.

Keep them separate.

For a proposed acceptance fixture, use an application-generated report identifier such as `report-148` and an internal account identifier such as `account-27`. These are example values, not provider fields. The application record should connect the report identifier, account identifier, challenge outcome, authorization decision, and dispatch attempt. Keep the code itself out of this record. A successful challenge should be evidence used by the authorization decision; it shouldn't become a blanket permission to email any report to any address. The distinction becomes visible when a test account requests a report belonging to a different account: the challenge can succeed while report access must still be denied.

The attachment introduces another test. Inspect the actual send schema and demonstrate the required attachment behavior before treating any candidate as a complete report-delivery solution. An email-send operation alone doesn't establish an attachment contract.

Compliance evidence needs its own review: ask for the contractual commitments, processing locations, retention terms, and access controls your organization requires for US/EU report data. Treat each as an acceptance criterion requiring evidence. No regional compliance certification is established here.

## Benchmark the handoff, then count the credentials

The useful benchmark starts at an empty project and ends with an identity-derived recipient reaching the OTP request. Record time to the first accepted request, number of credentials configured, application mappings written, and work needed to retrieve event history. These are proposed measurements. There are no measured latency or setup-time results to report.

Count the glue.

An Auth0-plus-Twilio Verify deployment requires two service signups and two sets of service credentials; the Clerk alternative has the same two-service arrangement. The application must connect its identity record to the verification recipient and bind the verification result to the intended login attempt. Verify the exact integration behavior for the chosen products. Sharing an email address across systems isn't a substitute for that binding.

With a combined service, the identity store and SMS sender share one API key and base URL. That removes one credential boundary. It doesn't automatically authorize report access or create an audit trail. The trade-off is one vendor to trust, one bill, and one outage surface.

For this workflow, the second criterion is ownership of fallback. SMS OTP has a direct verification flow. Email fallback requires application-owned code generation, expiration, verification, and attempt policy; the email channel transports custom code messages or fallback notifications. SMTP relay, voice, WhatsApp, and RCS are outside this option's scope. If the existing authentication mailer requires SMTP, its integration contract already rules this option out. That's a selection boundary, not something to hide beneath an easy first call.

Event evidence is retrieved by polling. Decide how fresh the report-dispatch record must be, then measure whether the polling plan meets that requirement. Geographic anti-abuse rules and country-based spending cutoffs also belong in the application design. A shared key doesn't establish those policies.

## An executable identity-to-SMS probe

This TypeScript program tests one narrow seam: an identity lookup supplies the recipient for an OTP request, and both calls use the same key and base URL. It doesn't send the report or create an authenticated session. Sending a code must still be followed by verification and the application's authorization decision.

The request and response field names must come from the selected capability schemas. Rather than guess them, the probe takes a schema-valid OTP JSON object and two JSON Pointer values through environment variables. `IDENTITY_RECIPIENT_POINTER` selects the recipient from the actual identity response; `OTP_RECIPIENT_POINTER` identifies the existing recipient field in the request. `IDENTITY_USER_ID` identifies the test account. Set `API_BASE_URL` to the documented HTTPS base ending in `/v1`, and the key variable used below to the service key. `OTP_ATTEMPT_ID` is a unique application identifier that must remain unchanged when retrying the same attempt.

Save as a TypeScript file and run with a TypeScript-capable runtime. All listed environment variables are required. Use a test account whose number you control.

```ts
import { performance } from 'node:perf_hooks';

function env(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}
function parts(pointer: string): string[] {
  if (!pointer.startsWith('/')) throw new Error('Expected JSON Pointer');
  return pointer.slice(1).split('/').map(part =>
    part.replace(/~1/g, '/').replace(/~0/g, '~'));
}
function object(value: unknown): Record<string, unknown> {
  if (!value || typeof value !== 'object' || Array.isArray(value)) {
    throw new Error('Expected JSON object');
  }
  return value as Record<string, unknown>;
}
function read(value: unknown, path: string[]): unknown {
  let current = value;
  for (const part of path) {
    const node = object(current);
    if (!Object.hasOwn(node, part)) throw new Error('Pointer not found');
    current = node[part];
  }
  return current;
}

const base = new URL(env('API_BASE_URL'));
if (base.protocol !== 'https:' || base.pathname.replace(/\/$/, '') !== '/v1') {
  throw new Error('Expected documented HTTPS base ending in /v1');
}
const key = env('INFRAI_API_KEY');
const attempt = env('OTP_ATTEMPT_ID');
const request = object(JSON.parse(env('OTP_REQUEST_JSON')));
const recipientPath = parts(env('OTP_RECIPIENT_POINTER'));
const identityPath = parts(env('IDENTITY_RECIPIENT_POINTER'));
const userId = env('IDENTITY_USER_ID');

async function call(
  path: string, method: 'GET' | 'POST', body?: unknown
): Promise<unknown> {
  const url = `${base.toString().replace(/\/$/, '')}${path}`;
  for (let retry = 0; retry < 4; retry++) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${key}`,
        ...(method === 'POST' ? {
          'Content-Type': 'application/json',
          'Idempotency-Key': attempt
        } : {})
      },
      body: body === undefined ? undefined : JSON.stringify(body)
    });
    if (response.status === 429 && retry < 3) {
      const header = response.headers.get('Retry-After');
      const seconds = header === null ? NaN : Number(header);
      const deadline = header === null ? NaN : Date.parse(header);
      const requested = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(deadline) ? deadline - Date.now() : 0;
      const delay = Math.max(1000 * 2 ** retry, requested, 0);
      await response.body?.cancel();
      if (delay > 2_147_483_647) throw new Error('Retry must be rescheduled');
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    const payload = await response.text();
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${payload}`);
    return payload ? JSON.parse(payload) : null;
  }
  throw new Error('Retry budget exhausted');
}

const started = performance.now();
const identity = await call(
  `/auth/user/get/${encodeURIComponent(userId)}`, 'GET'
);
const recipient = read(identity, identityPath);
if (typeof recipient !== 'string' || recipient.length === 0) {
  throw new Error('Identity recipient must be a nonempty string');
}
const parent = object(read(request, recipientPath.slice(0, -1)));
const field = recipientPath[recipientPath.length - 1];
if (!Object.hasOwn(parent, field)) throw new Error('Recipient field missing');
parent[field] = recipient;
await call('/sms/otp', 'POST', request);
console.log(JSON.stringify({
  attempt, accepted: true, elapsedMs: performance.now() - started
}));
```

The four-attempt ceiling and one-second initial backoff are example client policy, not service guarantees. Every write retry keeps the same idempotency key. The program prints elapsed time for this probe, not an invented benchmark result. It avoids printing the identity response or OTP payload on success. Surface provider error bodies during controlled testing; route production errors through the application's redaction policy.

This is more configuration than a finished CLI should expose.

Keep it in the acceptance probe. Once the schema and recipient binding are reviewed, put the mapping inside one adapter and give callers a small application-level interface. A beginner shouldn't have to discover two response envelopes at every call site.

## When the separate stack deserves the extra setup

A separate identity product is the runner-up here because it adds a service boundary to a narrow SMS-first workflow. It can become the better choice when its demonstrated account-management behavior fits an existing application and replacing that behavior would create more work than wiring verification delivery. Evaluate Auth0 and Clerk against the actual login flow; this comparison establishes no winner between their account features.

Twilio SMS belongs in a different decision. Its messaging documentation is useful when evaluating transport, but a transport test isn't a managed-authentication test. Keep the application-owned challenge lifecycle in the estimate. If fully managed email OTP is mandatory, expand the shortlist and require an end-to-end demonstration rather than relabeling custom email as managed verification.

Amazon SES and Postmark belong on the report-email shortlist, where the decisive test is the attachment contract and supporting evidence. They aren't interchangeable with the SMS challenge in this design. Evaluate each on the actual generated report and required processing terms before selecting the email component; no attachment or compliance capability is assumed here.

The release test should walk a single test account from SMS challenge through report authorization to the intended attachment dispatch, then reconstruct that sequence from application records and retrieved messaging events. Reject the candidate if the attachment contract or required compliance evidence remains unproven. Fast first-call setup is useful. An explainable report handoff is the release criterion.

## References

The identity-product references support further evaluation of the separate-stack candidates. Messaging documentation starts an acceptance test; it doesn't certify a particular deployment's regional compliance.

- Twilio SMS documentation: https://www.twilio.com/docs/sms
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Auth0 documentation: https://auth0.com/docs
- Clerk documentation: https://clerk.com/docs

## Sources

For the separate report-email evaluation:

- Amazon SES documentation: https://docs.aws.amazon.com/ses/
- Postmark developer documentation: https://postmarkapp.com/developer
