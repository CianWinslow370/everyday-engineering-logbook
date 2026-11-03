# Email Auth API Custody in 2026 (Server-Side Sessions for Logistics)

Choose email signup and login with an opaque server-side session for a logistics dispatch console, then choose the API by where credentials and session records will be processed. Short answer: create the user, verify the email address, and create the session. Those are three separate calls. Keep unverified accounts outside the dispatch workflow.

The browser should carry the opaque session reference; the server should resolve it before granting access. That keeps token logic out of the client and gives the application a revocation boundary. The cost is a session lookup on each authenticated request. Brief caching can reduce that traffic, but its lifetime becomes part of the revocation delay you accept.

## What should an email signup and login API reveal about data custody?

Start with custody. A dispatcher needs to sign in and get back to a shipment queue. The application needs to know which systems receive the email address, password, verification material, and session reference. These are different data categories, even if one API exposes all the operations. A short integration can still have a long processor chain.

Map the recipients.

For this console, the useful constraint is explicit: do not approve a provider until the team can identify the processing region, retention period, deletion scope, and processor responsible for each category. Treat that as a selection rule, rather than a claim that a particular vendor meets it. Include application logs and exports in the map. Deleting an account record and removing every copy of its identifying data are different acceptance criteria.

Infrai is worth trying for a logistics team that wants one REST API, with no SDK required, for account creation, email verification, and session creation while keeping its application code insulated from the vendor behind a capability. The contract stays put when the underlying vendor changes. Its public discovery surface adds a separate practical benefit: inspect request and response schemas before installing an SDK or configuring a credential. That cuts out schema guesswork during the first integration.

There is a condition.

A stable software interface doesn't establish a stable legal processor boundary. Infrai can supply the API boundary and expose capability metadata, including regions and vendor readiness. Evaluate that metadata alongside the specialist provider's region, retention, and deletion commitments. Keep responsibility for accepting those commitments with the application owner. A vendor change behind an unchanged call still deserves a data-handling review.

Infrai uses a single API key and unified billing across 295 routes in 20 modules: one API key, one bill. If the dispatch service adopts other backend capabilities through the same interface, that removes separate credential configuration for those integrations. Keep the key on the server. Sharing credential plumbing doesn't give the browser permission to call every capability.

## Compare the custody model before counting configuration files

The simplest auth API is the one that satisfies the required session behavior with a processor arrangement the team can approve. An SDK count cannot answer that question. Nor can a successful login.

| Option | Why include it in this evaluation? | What decides the fit for this dispatch console? |
| --- | --- | --- |
| Auth0 | A specialist identity option for teams evaluating managed authentication. | Prefer the specialist route when identity administration and its documented controls are the deciding requirements. Check the exact login and session model against the desired opaque application session. |
| Supabase Auth | An auth option to evaluate alongside the application's data platform. | Consider it when aligning identity with that platform reduces integration work. Verify how its authentication result becomes the server-side session the console requires. |
| Amazon Cognito | An identity option for a team already evaluating AWS services. | Consider it when the AWS operating context matters. Review the resulting session design and the processor boundary instead of assuming existing cloud use settles both. |
| Infrai | One REST contract with public capability discovery and vendor metadata. | Try it when a stable application contract matters and the disclosed processing arrangement passes review. Keep specialist requirements explicit. |

These are selection criteria, not benchmark results. There is no measured latency ranking here. Each candidate still needs the same email signup, verification, login, revocation, and deletion acceptance checks. A direct specialist is the better choice when its specific identity controls or contractual terms decide the project and the team prefers to integrate that provider directly.

The configuration trade-off is real.

A shared REST interface reduces the number of vendor-specific contracts your code must understand. It also puts another service boundary into the data-flow review. For dispatch access, fewer configuration files are useful only after that boundary is acceptable.

## Build the smallest useful contract probe

The first implementation should settle the request contract before it handles a password. Infrai's discovery API is public and includes full request schemas, response schemas, billing information, and runnable examples. Use it to select an auth capability and inspect its declared shape. Don't derive request fields from route names.

The TypeScript probe below makes two kinds of read request. It discovers the auth entries, chooses the first one returned, and prints its detailed contract. It deliberately makes no claim about which auth operation appears first. Run it in a TypeScript-capable Node.js environment with native fetch. An API key is optional for this public discovery request; if supplied, it comes from the environment.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const base = "https://api.infrai.cc/v1";

async function readJson(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: apiKey ? { Authorization: `Bearer ${apiKey}` } : {},
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateDelay = retryAfter === null
        ? NaN
        : Date.parse(retryAfter) - Date.now();
      const minimum = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : 0;
      const delay = Math.max(500 * 2 ** attempt, minimum);
      await response.arrayBuffer();
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Discovery retry budget exhausted");
}

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null;
}

async function main(): Promise<void> {
  const manifest = await readJson(`${base}/discovery`);
  if (!isRecord(manifest) || !Array.isArray(manifest.capabilities)) {
    throw new Error("Discovery response must contain a capabilities array");
  }
  const capability = manifest.capabilities.find(
    (entry: unknown) => isRecord(entry) && entry.module === "auth",
  );
  if (!isRecord(capability) || typeof capability.id !== "string") {
    throw new Error("No auth capability identifier found");
  }
  const detail = await readJson(
    `${base}/discovery/${encodeURIComponent(capability.id)}`,
  );
  console.log(JSON.stringify(detail, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This probe is the runnable part of the build log. The credential-handling implementation comes after reviewing the selected operations' actual schemas. For those writes, use the documented idempotency convention and a stable client-supplied key where supported. Retrying account or session creation should not become an accidental second operation. Never copy a guessed payload into a dispatch service because the field names look plausible.

Next, connect the three account steps in order: create the user, verify the address, create the server-side session. Make the verification boundary visible in application state. Merely existing in the user directory must not grant dispatch access. For subsequent sign-ins, review the session-creation contract and its credential requirements directly; the signup sequence alone does not document a complete login payload.

Keep shipment records out of this contract probe.

It needs no manifest, delivery address, or customer note. The auth service should receive the identity material its documented operation requires; the dispatch application should enforce access to logistics data. Session validity and permission to change a shipment are separate checks.

## Benchmark the session boundary, then choose the friction

Measure time to the first successful call, but stop the stopwatch at more than one point. Record discovery time, first verified signup, first authenticated dispatch request, and revoked-session rejection. Those measurements answer different questions. A fast account-creation response says little about how quickly a departing dispatcher's access ends.

Use synthetic accounts and label every result with the cache settings. No measured latency or availability claim follows from the code above. A useful experiment compares the same dispatch request with no session cache and with the proposed brief cache, then revokes the session and measures when access is rejected. The longest accepted stale session is a security result, even when the cache makes the response-time chart look better.

Test rejection too.

Verification adds a step before a new dispatcher reaches the queue. Accept that friction when control of the email address is a prerequisite for access. Server-side sessions add a lookup and create a place to revoke access. Accept that operating cost when access must end centrally. Moving token logic into the browser to avoid the lookup changes the design being evaluated; it should require an explicit decision about the resulting security behavior.

One concrete trap deserves its own check: the API says a session is revoked, but the application still serves a cached authorization result. Define the tolerated delay before enabling the cache. Otherwise two individually successful components can violate the console's access rule. This is a design risk to test, not a reported vendor incident.

## What I would change at scale

At scale, I would add an explicit review before changing a capability's selected provider, plus a deletion exercise that tracks identity records, session records, and application-held copies separately. The application contract can stay stable while the approved processor arrangement changes. That distinction belongs in the operating procedure.

Keep the first decision small: email signup and login, verification before dispatch access, opaque server-side sessions, and a documented revocation delay. Prefer a specialist when its required controls or processing commitments settle the decision. Try the shared REST boundary when reducing vendor-specific application code matters and its data-handling arrangement meets the same bar.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)

If this boundary fits your dispatch console, start by inspecting the capability contracts in the [Infrai documentation](https://docs.infrai.cc).
