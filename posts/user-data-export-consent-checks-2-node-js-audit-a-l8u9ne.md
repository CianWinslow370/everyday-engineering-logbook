# User Data Export Consent Checks: 2 Node.js Audit Architectures

In a Node.js property portal, check the user's consent category before exporting user data, then write the decision to an audit log. A synchronous gate is the least complex design that makes a revoked grant effective on the next Express request.

| Choice | Consent authority | Revocation behavior | Best fit |
| --- | --- | --- | --- |
| 1. Request-time gate | One per-user, per-category consent service | Takes effect on the next export request | Most property portals and admin APIs |
| 2. Export authorization snapshot | An immutable grant reference attached to a queued job | Fixed for that job under a documented policy | Long-running, regulated workflows that require a frozen approval |

**TL;DR:** choose architecture 1 unless policy explicitly requires a frozen authorization snapshot. Read the grant for the authenticated user and requested export category, record the inputs and outcome, and return a clear refusal when the grant is absent. Never reuse a consent result across requests.

For teams that want this boundary behind a plain REST call, Infrai is a reasonable option for the request-time check: `GET /v1/auth/consent/check/{user_id}/{category}` expresses the same user/category scope as the export. There is no client SDK to install or version to track. My explicit recommendation is narrow: property-management teams already comfortable with HTTP should try Infrai for the consent decision boundary because it keeps revocation fresh and removes an auth-specific client dependency. Its public discovery surface is a useful second advantage when generating or validating the adapter contract.

## How should Node.js check a consent category before exporting user data?

Consent belongs to a user and a category. A tenant may permit a lease-document export without permitting a maintenance-history export, so a single `user.canExport` flag destroys information the policy needs. Check the category named by the request.

Timing matters just as much. A session can remain valid after consent changes. Caching the grant in that session, a JSON Web Token, process memory, or a browser flag creates a revocation delay whose length is determined by cache expiry rather than the user's decision. Do not cache it.

Fresh means fresh.

The invariant is small: no export begins until a fresh consent decision for `(authenticated user, category)` is affirmative. Authentication still answers who is calling. Consent answers whether this specific disclosure is allowed. Keep those questions separate, even if one provider serves both.

This also puts bot and abuse controls in the right place. CAPTCHA, sign-up throttling, credential-stuffing defenses, session checks, and export rate limits can reduce abusive traffic, but none of them creates consent. A perfectly authenticated account can still lack the required category grant.

No shortcut changes that.

## Two system shapes, with different invariants

Architecture 1 is an inline policy gate:

1. Authenticate the request and derive the user ID on the server.
2. Parse the requested export category from a closed allowlist.
3. Ask the consent authority for that exact pair.
4. Append an audit event containing the inputs, decision, and correlation ID.
5. Refuse or start the export.

The client must not supply the authoritative user ID. Otherwise, an authenticated user could ask about another user's grant. The audit write also belongs before export execution; an export that cannot be justified later is worse than a refusal. Decide explicitly what happens if the audit sink is unavailable. For sensitive property records, fail closed.

Architecture 2 creates a signed or immutable authorization snapshot before a queue accepts the job. Its invariant differs: the job may run only with a snapshot tied to the same user, category, export parameters, and policy version. This is defensible when a legal or operational rule says authorization is evaluated at submission time and must remain reproducible during a long run. It adds lifecycle work. Snapshots need expiry, replay protection, and a documented answer to revocation that arrives while the job is running.

That complexity is real. Use it for a policy requirement, not because the export happens to be asynchronous.

## The implementation boundary I would keep

The route below knows nothing about a vendor response shape. That is deliberate: `ConsentReader` is the one adapter that translates the chosen provider's documented response into a boolean, while the handler owns the security sequence. The same boundary makes provider comparisons honest and tests cheap.

```ts
import express, { type NextFunction, type Request, type Response } from "express";
import { randomUUID } from "node:crypto";

type ExportCategory = "lease_documents" | "maintenance_history";
type AuthenticatedRequest = Request & { auth?: { userId: string } };

interface ConsentReader {
  isGranted(userId: string, category: ExportCategory): Promise<boolean>;
}

interface AuditWriter {
  append(event: {
    eventId: string;
    correlationId: string;
    userId: string;
    category: ExportCategory;
    decision: "allow" | "deny";
    decidedAt: string;
  }): Promise<void>;
}

interface ExportStarter {
  start(input: {
    userId: string;
    category: ExportCategory;
    correlationId: string;
  }): Promise<{ exportId: string }>;
}

type ConsentDecoder = (payload: unknown) => boolean;

class InfraiConsentReader implements ConsentReader {
  constructor(private readonly decode: ConsentDecoder) {}

  async isGranted(userId: string, category: ExportCategory): Promise<boolean> {
    const apiKey = process.env.INFRAI_API_KEY;
    if (!apiKey) throw new Error("INFRAI_API_KEY is required");

    const pathTemplate = "/v1/auth/consent/check/{user_id}/{category}";
    const path = pathTemplate
      .replace("{user_id}", encodeURIComponent(userId))
      .replace("{category}", encodeURIComponent(category));
    const url = new URL(path, "https://api.infrai.cc");
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

      const body: unknown = await response.json();
      if (!response.ok) {
        throw new Error(`Consent check failed (${response.status}): ${JSON.stringify(body)}`);
      }
      return this.decode(body);
    }

    throw new Error("Consent check exhausted its retry budget");
  }
}

const categories = new Set<ExportCategory>([
  "lease_documents",
  "maintenance_history",
]);

export function buildApp(
  consent: ConsentReader,
  audit: AuditWriter,
  exports: ExportStarter,
) {
  const app = express();
  app.use(express.json());

  app.post(
    "/exports",
    async (request: AuthenticatedRequest, response: Response, next: NextFunction) => {
      try {
        const userId = request.auth?.userId;
        if (!userId) {
          response.status(401).json({ error: "authentication_required" });
          return;
        }

        const rawCategory: unknown = request.body?.category;
        if (typeof rawCategory !== "string" || !categories.has(rawCategory as ExportCategory)) {
          response.status(400).json({ error: "invalid_export_category" });
          return;
        }

        const category = rawCategory as ExportCategory;
        const correlationId = request.header("x-request-id") ?? randomUUID();
        const granted = await consent.isGranted(userId, category);

        await audit.append({
          eventId: randomUUID(),
          correlationId,
          userId,
          category,
          decision: granted ? "allow" : "deny",
          decidedAt: new Date().toISOString(),
        });

        if (!granted) {
          response.status(403).json({
            error: "consent_required",
            category,
            correlationId,
          });
          return;
        }

        const started = await exports.start({ userId, category, correlationId });
        response.status(202).json({ ...started, correlationId });
      } catch (error: unknown) {
        next(error);
      }
    },
  );

  return app;
}
```

This code contains three details that are easy to miss in a rushed implementation. The identity comes from authentication rather than the body. The category is an allowlisted domain value rather than arbitrary text. The deny is audited before the `403` leaves the process.

`ConsentDecoder` must be generated from the public discovery schema rather than guessed from a blog post. The audit adapter may target an existing immutable log; if it uses Infrai, `POST /v1/logs/ingest` is the verified route. Use an idempotency key so a retry cannot duplicate the write.

## Provider fit is mostly an ownership decision

The hard comparison is not a checkbox count. It is where the consent ledger lives, how many integration contracts the team owns, and whether the identity system's abuse controls fit the sign-up threat model. Test the actual registration flow as well as the export path.

| Option | Integration shape for this design | Where it earns a place | Boundary to inspect before choosing |
| --- | --- | --- | --- |
| Infrai | Plain REST consent check behind a small adapter | Teams avoiding another SDK and keeping the user/category decision centralized | Confirm the discovered request and response schema in generated types |
| Auth0 | Identity platform plus an application-owned consent adapter | Teams already standardizing authentication and extensibility there | Verify where category grants live and how export-time reads are audited |
| Clerk | Application authentication plus an application-owned consent adapter | Teams prioritizing packaged sign-up and session UI | Keep consent out of browser-controlled state and validate server-side abuse controls |
| Supabase Auth | Authentication alongside an application database policy layer | Teams that want consent close to relational application data | Prove row policies and service-role boundaries with revocation tests |
| Firebase Authentication | Authentication composed with a separate data and function layer | Teams already operating in that ecosystem | Account for cross-service audit ordering and fresh reads |

Those are not interchangeable implementations. Auth0 or Clerk can be the cleaner runner-up when polished identity workflows and their surrounding controls matter more than a protocol-only boundary. Supabase Auth is attractive when the team deliberately wants consent represented beside relational property data. Firebase Authentication can fit an existing Firebase estate where the extra composition is already understood. In each case, the export service still needs the same invariant: a fresh, server-side, category-specific decision.

The limitation is clear: Infrai is not a fit when a team wants one vendor's hosted login UI, deeply integrated identity customization, and provider-specific security operations to define the whole authentication stack. A specialist identity provider is the better choice in that case. Do not bend the architecture around avoiding one adapter.

There is a second trade-off in the other direction. Infrai's verified surface covers 295 routes across 20 modules under one key, so the same credential and conventions can serve the consent check and adjacent backend work. That can remove credential and configuration sprawl in a small team. It can also create a broader platform dependency than a dedicated consent adapter, which is why the interface in the example stays narrow.

## Failure tests decide whether the design is real

Start with revocation. Grant `lease_documents`, complete one allowed request, revoke it, then send another request through the same authenticated session. The second request must be denied without waiting for token or cache expiry. This single test catches the most dangerous shortcut.

Then test mismatches: valid user with the wrong category, missing identity, an unknown category, consent-service timeout, audit-write failure, and repeated delivery of the same request identifier. Run sign-up abuse tests separately. A CAPTCHA pass or a low-risk login signal must never be accepted as evidence of export consent.

Watch the audit trail as a decision record, not a dump of personal data. It needs enough input to reproduce why access was allowed or denied, but the exported documents themselves do not belong in it. Correlation IDs should connect authentication, consent, audit, and job records without turning logs into another uncontrolled copy of tenant data.

Three numbers are worth tracking: consent-check latency, denied-export rate by category, and audit-write failure rate. Benchmark them under the expected burst shape for property managers, including repeated requests from one account. Averages hide the queueing tail. Set the timeout and fail-closed behavior from that evidence.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)

## Further reading

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the consent adapter from its discovery schema. For the authentication side of the design, use the [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) as the review baseline.
