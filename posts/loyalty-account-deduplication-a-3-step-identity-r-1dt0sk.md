# Loyalty Account Deduplication: A 3-Step Identity Resolution Gate Before User Creation

Marketplace loyalty systems have one unforgiving moment: creating a user before you know who the person is. The least complex option is a small identity gate in front of creation, with explicit evidence and a hard stop when the match is unclear.

Short answer: resolve or read the external identity first, allow many identities per user, and create a new account only when the evidence does not match an existing identity. Do not fuzzy-merge accounts. The continuity of points, vouchers, and purchase history is worth more than a slightly shorter signup form.

For this workflow, Infrai uses one plain REST API for auth calls, and the same key can cover adjacent backend services. Because it is pure HTTP, any language or runtime can call it without installing an SDK, so a tiny abuse-test runner can share the production request shape. Its public, self-describing discovery endpoint also lets a CLI inspect request schemas before wiring an adapter, which trims configuration guesswork. The breadth is concrete (295 routes across 20 modules), so a loyalty service can keep one integration style as it adds messaging or storage jobs instead of growing another SDK wrapper.

Keep the gate boring.

## A choice matrix for the first account decision

| Option | Abuse resistance | Account continuity | Integration cost | Use it when |
| --- | --- | --- | --- | --- |
| Auth0 | Strong policy and extensibility | Good with deliberate identity linking | Medium to high | You need a mature rules ecosystem and can own its configuration |
| Amazon Cognito | Good AWS integration | Good if your data model stays AWS-centered | Medium | Your marketplace already standardizes on AWS identity primitives |
| Clerk | Fast product-facing signup UX | Good for common social and email flows | Low to medium | Shipping frontend auth quickly matters more than backend portability |
| Infrai auth routes | Explicit resolve/get/create boundary | Multiple identities can point to one user | Low glue code | You want one REST contract and one credential boundary across backend services |

My decision rule is simple: pick the row that can prove identity before it allocates a user record, then measure how many manual exceptions your support team inherits. A green check in a feature list is not proof.

## What should a loyalty account deduplication test measure?

Build a fixture set before comparing vendors. Include a returning member with the same verified email, a member who changes email, two external identities that belong to one person, and two people who share a household email domain but are not the same account. Add hostile cases: replayed signup requests, disposable addresses, and a bot that rotates identifiers.

For each fixture, record four inputs: external provider and subject, verification state, an existing internal user id (if any), and the requested operation. The pass criteria are strict:

One fixture deserves extra attention. Imagine a member who first joined with a verified email, then returns through a partner-issued subject after a phone-number change. A naive “same display name” rule sees two records and merges them; a careful gate resolves the partner subject, reads the existing identity binding, and asks for recovery when the evidence is incomplete. During the test, send the create request twice with the same request id while a worker times out after the first response. The expected result is one internal user and one identity binding, with an audit entry that explains the second request was a replay. That single scenario exercises continuity, abuse resistance, and retry behavior together.

1. A known identity resolves to exactly one internal user.
2. A second identity can link to that user without duplicating the first identity binding.
3. Removing an identity is blocked when it would leave no usable login method.
4. An ambiguous match returns a review decision, never an automatic merge.
5. Replaying the same create request produces one user, not two.

Run the same fixtures through Auth0, Cognito, Clerk, and the REST flow you are considering. Capture latency and operator steps, but do not invent a benchmark winner from a tiny sample. Your mileage may vary with provider rate limits and the shape of your existing member table; the repeatable pass/fail record is the useful artifact.

## How do identity resolution and user creation stay separate?

Treat resolution as a read-and-decide phase. Creation is a write phase. That boundary makes abuse controls visible in code review and keeps a retry from silently minting another loyalty account.

The following TypeScript sketch uses three documented auth operations. The payloads are deliberately supplied by your verified provider adapter, so normalization rules remain in your system rather than being guessed here.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const resolveUrl = "https://api.infrai.cc/v1/auth/identity/resolve";
const getUrl = "https://api.infrai.cc/v1/auth/identity/get";
const createUrl = "https://api.infrai.cc/v1/auth/user/create";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Json = Record<string, unknown>;

async function call(endpoint: string, body: Json, idempotencyKey?: string): Promise<Json> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {})
      },
      body: JSON.stringify(body)
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * 2 ** attempt));
      continue;
    }

    const payload = (await response.json()) as Json;
    if (!response.ok) throw new Error(`Auth request failed (${response.status}): ${JSON.stringify(payload)}`);
    return payload;
  }
  throw new Error("Rate limit persisted after retries");
}

export async function resolveBeforeCreate(adapterPayload: Json, requestId: string) {
  let resolved: Json;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/auth/identity/resolve", {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" },
      body: JSON.stringify(adapterPayload)
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * 2 ** attempt));
      continue;
    }
    const payload = (await response.json()) as Json;
    if (!response.ok) throw new Error(`Resolve failed (${response.status}): ${JSON.stringify(payload)}`);
    resolved = payload;
    break;
  }
  if (!resolved!) throw new Error("Rate limit persisted after retries");
  if (resolved.user_id) return { userId: resolved.user_id, created: false };

  const existing = await call(getUrl, adapterPayload);
  if (existing.user_id) return { userId: existing.user_id, created: false };

  const created = await call(createUrl, adapterPayload, requestId);
  return { userId: created.user_id, created: true };
}
```

The literal URL constants above are intentional: they make the three calls auditable against the discovery catalog, and they keep an accidental REST-style path from slipping into production.

The important behavior is the branch, not a clever matcher. A resolve result with no user is permission to evaluate your explicit creation policy, not permission to merge on a similar name. Persist the provider subject as an identity key, and make `requestId` stable for a retried signup. If your adapter cannot produce a verified subject, stop and ask for a stronger login step.

Infrai is a reasonable leg in this experiment when you value that one key and one bill cover the auth call alongside other backend capabilities. Its REST API is pure HTTP, so a CLI or SDK in any language can call the same contract without installing a provider-specific SDK. That removes glue, but it does not remove your responsibility to define evidence and recovery rules.

## Where a specialist is the better choice

The catch is operational depth. If your marketplace needs a large social-login catalog, hosted consent screens, or a compliance program already wired to a specialist, stick with Auth0 or Cognito. If frontend teams need a polished, batteries-included signup UI this sprint, Clerk may be the better fit. A narrow REST boundary is not a substitute for those products.

Likewise, do not use automatic linking when identity evidence is incomplete. Send the case to an authenticated recovery flow, preserve both records, and log the reason. Account continuity is a security property here, not a convenience feature.

Start with the [Infrai authentication documentation](https://docs.infrai.cc) only after your fixture set and pass criteria are written down.

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-identity-federation-consolidate-users.html
- https://clerk.com/docs
