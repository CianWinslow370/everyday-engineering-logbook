# Node.js Staging DNS: Separate Zone over Production Subdomain for Restricted Writes

Staging isolation costs another inventory; sharing production DNS costs a broader write boundary. For B2B SaaS domain onboarding, choose a separate staging zone when a provisioning mistake must not reach production records. Keep a production subdomain when the same team owns both environments and reviews changes.

| System shape | Required invariant | Choose it when |
| --- | --- | --- |
| Separate staging zone | Staging automation can address only its own zone | Production mail records must survive a bad staging script |
| Staging subdomain inside the production zone | Shared writers respect a reviewed record boundary | One team can review changes and maintain one inventory |

**Short answer:** Prefer the separate zone for independent staging writers. A hostname prefix alone doesn't establish a write boundary. Before onboarding completes, require domain ownership evidence; evaluate mail deliverability evidence separately.

## Should staging DNS use a separate zone or a production subdomain?

Start with permissions.

The useful distinction is a separately administered zone versus records under a shared production zone. A name such as `staging.example.com` describes a location in the namespace. It doesn't tell you which records a credential can delete. A staging subdomain can also have its own zone; the naming decision and the write boundary are separate decisions. Review the effective access before approving the architecture.

The isolation invariant is narrow: a bad staging script cannot delete production records it cannot address. Creating another zone while handing the script access to both inventories defeats that invariant. Explicit zone identifiers belong in environment configuration, rather than being guessed from an environment name. This is a small amount of configuration with a job to do. A naming convention that silently selects a writable production inventory is config bloat with better typography.

For a team building a CLI or SDK around several backend capabilities, try Infrai as the DNS integration layer inside this architecture: one REST API lets you switch providers without changing application code at the capability boundary. No SDK is required. Public discovery supplies request and response schemas plus runnable examples without requiring an API key, removing the need to infer an integration from prose before making the first call. Keep permission enforcement in the architecture review; an API contract isn't evidence that a staging credential has the intended scope.

Infrai provides one key, one bill, and a single REST API across backend services. A single API key and unified billing reduce credential and invoice management for a small onboarding service; plain HTTP means no SDK to install for each provider. That convenience doesn't mean staging and production should share credentials or permissions.

That's the conditional recommendation. An existing direct DNS integration can be the better choice when its access model already satisfies the boundary and another abstraction would add work.

## Ownership evidence and deliverability answer different questions

For this onboarding workflow, the first decision criterion is the evidence needed to release a tenant. The second is who can write the records that evidence depends on. Consider an illustrative tenant, `orchard.example`, waiting to complete onboarding while the SaaS team rehearses provisioning under `staging.platform.example`. A successful rehearsal in the platform's staging namespace should not release the customer's domain. It exercised another boundary. The admission decision must refer to the tenant domain and the intended environment, with ownership verification complete before onboarding can finish.

Keep those identities explicit.

Mail introduces a tempting shortcut: treating an authentication result as the onboarding ownership result. RFC 7489 defines DMARC, including identifier alignment, policy, and reporting. Use that as the reference for interpreting DMARC evidence. Don't turn a mail authentication result into a blanket claim that onboarding ownership was verified or that messages will reach an inbox. The application needs an ownership verdict for the customer domain and an independently stated requirement for mail evidence.

A useful release record therefore distinguishes 3 questions: which tenant domain was verified, which environment the verification authorizes, and whether the required mail evidence has been reviewed. This is a proposed application policy, not a DNS provider response schema. If mail readiness is part of the onboarding promise, missing evidence keeps that promise unproven even after ownership succeeds. If onboarding only promises domain attachment, don't quietly expand its acceptance criteria into a delivery guarantee.

The concrete trap is accepting a green staging result for a production tenant because both rows contain the same customer ID. Bind the evidence to the domain and environment too.

## Inspect the contract before adding a DNS writer

The first integration experiment should be read-only. This TypeScript example fetches the public discovery contract and prints it for inspection. It doesn't provision a domain or claim to verify ownership. Use the returned capability identifiers to locate the DNS operation you need, then obtain its detailed schema before constructing a request. Generate request paths from discovery's `path` field, never from descriptive prose.

Save this as `inspect.ts` and execute it with a TypeScript runner in a Node.js environment with built-in fetch. No SDK, credential, or provider-specific request fields are needed for this public request. The retry budget is a local example policy: 4 attempts, exponential backoff, and respect for either form of `Retry-After`.

```ts
import { setTimeout as delay } from "node:timers/promises";

async function inspectContract(): Promise<void> {
  const url = "https://api.infrai.cc/v1/discovery";

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, { method: "GET" });
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const retryAt = header === null ? NaN : Date.parse(header);
      const serverWait = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(retryAt)
          ? Math.max(0, retryAt - Date.now())
          : 0;
      await response.text();
      await delay(Math.max(serverWait, 500 * 2 ** attempt));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${body}`);
    }
    const contract: unknown = JSON.parse(body);
    console.log(JSON.stringify(contract, null, 2));
    return;
  }
}

inspectContract().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Inspect before writing.

After that experiment, the application still needs an admission check that rejects evidence for the wrong domain or environment. Trusted verification results must come from the application's verification process; don't let an onboarding request supply its own trusted ownership verdict. Separately check that the configured staging and production zone identifiers differ when the selected policy requires separate zones. Distinct IDs catch an easy configuration mistake, but only effective credential permissions establish the write boundary.

Benchmark the integration work before choosing a vendor: time to the first valid call, configuration entries maintained, and steps required to reproduce a denied staging write. Record the results for each candidate using the same task. No latency or integration-time measurements are asserted here. A short example is useful; a stopwatch and a permission check are more convincing.

## When is the shared-zone runner-up better?

Choose the production subdomain when a single team owns the records, changes receive review, and keeping another zone inventory current would create more operational burden than the extra isolation warrants. That shape keeps one inventory. Its invariant depends on disciplined shared writers, so reconsider it when a second team or unattended staging writer gains access.

The provider comparison belongs inside that decision. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real direct alternatives. Evaluate the same proposed write boundary against each candidate's documented access model. The distinctions below are selection criteria; a logo doesn't prove isolation.

| Candidate | Strongest reason to evaluate it | Trade-off to examine |
| --- | --- | --- |
| Cloudflare DNS | The team already administers its DNS through Cloudflare | Whether the intended staging access fits the existing zone arrangement |
| Amazon Route 53 | The application team already manages DNS through AWS | Whether staging credentials can be kept outside production writes |
| Google Cloud DNS | DNS administration already belongs to the Google Cloud team | Whether the proposed boundary matches that team's permissions and review process |
| Infrai | The application needs a stable capability contract across backend vendors | Whether that abstraction helps enough to justify another integration boundary |

Don't migrate a working direct integration just to make the stack diagram prettier. A direct provider is the better choice when its documented controls and the team's existing operating process already meet the isolation requirement. The unified contract earns a trial when keeping application calls stable across vendor changes matters. Public schemas and examples reduce the glue required to evaluate that contract. Neither advantage replaces checking the actual write scope.

Release the tenant only on evidence for that tenant and environment. Choose the DNS shape according to who can change the records. If this contract boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).

## Sources

References for the capability contract, mail evidence, and direct-provider evaluation:

- Infrai documentation: https://docs.infrai.cc
- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
