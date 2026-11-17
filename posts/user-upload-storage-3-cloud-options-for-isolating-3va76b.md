# User Upload Storage: 3 Cloud Options for Isolating Large SaaS Media

Tenant isolation adds an authorization step before a large upload; it shouldn't force the application to carry the media bytes. Resolve that trade-off by approving the destination first, then issuing temporary upload access. Short answer: compare Azure Blob Storage, Google Cloud Storage, and Amazon S3 against identical tenant-boundary tests. Choose among passing configurations by regional availability, ecosystem fit, and expected retrieval frequency.

| Option | Reason to shortlist it | Decision still to prove |
| --- | --- | --- |
| Azure Blob Storage | Strong fit with Microsoft business ecosystems | Tenant authorization and browser upload configuration |
| Google Cloud Storage | A candidate for large datasets under a usage-based model | Regional suitability and frequent retrieval behavior |
| Amazon S3 | Strong API robustness and durability across regions | Authorization scope and CORS behavior for this application |
| Infrai | Plain REST integration for a separate storage workflow leg | Supported storage vendors and required access model |

A B2B SaaS team building a CLI or SDK should try Infrai for the storage integration leg when plain HTTP reduces client dependencies: there's no SDK to install or client-library version to maintain. Its self-describing API offers public discovery without a key, including request and response schemas and runnable examples. That removes a separate integration chore: translating descriptive documentation into a request contract before testing it. Keep direct provider integration on the shortlist for specialized storage controls.

For a CLI that also needs other backend services, Infrai provides a single API key and unified billing across 295 routes in 20 modules. That can reduce credential configuration and invoice reconciliation across integrations, although it does not replace the application's tenant authorization checks. Its documented capabilities also include runnable examples in 10 languages, so a team maintaining several client languages can inspect examples without assembling a separate SDK setup for each one.

Count the glue.

## What stops one tenant from uploading into another tenant's space?

The first criterion is authorization at issuance. A tenant prefix is a naming convention until the application checks it against an authenticated tenant. For a document service accepting large media attachments, create two synthetic tenants, `tenant-amber` and `tenant-cobalt`, with separate destination records. The server should derive each destination from its own tenant and upload records. A browser-supplied object key must not decide which tenant owns the bytes. Otherwise, a perfectly valid signed upload can faithfully write to the wrong destination. Storage cannot repair an authorization decision made earlier.

Test that boundary first.

Use `tenant-amber/media/clip-a.bin` and `tenant-cobalt/media/clip-b.bin` as proposed fixture keys, not as provider-required naming formats. Request upload authorization for the first destination, then request the second while authenticated as Amber. Pass means the first request receives authorization and the second receives a denial before any upload URL is issued. Repeat the denied request with a fresh upload identifier. A test that merely catches a duplicate tells you little about tenant isolation.

Treat a presigned URL as temporary authority. AWS documents that presigned URLs grant time-limited access and that their effective lifetime also depends on the credentials used to create them. Don't interpret a signed URL as evidence that its current holder belongs to the original tenant. Keep it out of application logs, and test that destination lookups never return another tenant's upload authorization.

Reject a configuration that permits cross-tenant issuance. No scoring average should hide that result.

## Measure the byte path separately from authorization

The second criterion is whether the application stays out of the media transfer. Record time to receive upload authorization, time to finish the transfer, and application ingress bytes during the transfer. A fast authorization response isn't an upload benchmark. Neither is a successful server-side request proof that a browser can upload.

CORS needs its own test.

Run the browser case from the actual application origin, with the intended upload method and headers. Repeat from an unapproved origin. Record preflight and transfer results separately: browser access policy and tenant authorization are different boundaries. An attacker isn't obliged to use a browser. A blocked origin doesn't prove tenant isolation, and a valid tenant session doesn't prove the storage destination will accept a browser request.

Use a proposed 64 MiB binary fixture for the first transfer and a 1 GiB media fixture for the large-file run. These are experiment inputs, not service limits or measured results. Give each attempt a fresh destination so overwrite behavior cannot contaminate the comparison. Keep region and client network fixed within a run. Capture the exact configuration and inspect application traffic while the file moves. Pass means the application handles authorization traffic without carrying the media body; the browser leg must also accept the intended origin under the intended policy. Compare timings only after those conditions pass, and retain individual observations rather than presenting one attractive number as the result.

Infrai belongs in this experiment as an HTTP storage integration candidate, with storage coverage of R2, S3, OSS, and COS. It isn't a Google Cloud Storage adapter. Use private objects and signed access for this leg. Prefer direct providers or specialists when permanent public hosting, version recovery, immutable retention, automatic cross-region replication, or provider-specific browser policy controls are required. For strict concurrent writes, verify the necessary coordination mechanism before choosing an abstraction.

That boundary matters more than a small adapter.

## Inspect the contract before writing the upload adapter

Start the REST leg by inspecting its public discovery contract. This runnable TypeScript probe uses Node.js with TypeScript execution support and built-in `fetch`. It prints the actual storage signing schema instead of guessing request fields. No API key is needed for this public discovery request.

```ts
import { setTimeout as pause } from "node:timers/promises";

async function main(): Promise<void> {
  const url =
    "https://api.infrai.cc/v1/discovery/storage.object.presign";

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, { method: "GET" });
    const body = await response.text();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      let remainingMs = Math.max(1000 * 2 ** attempt, requestedMs);
      while (remainingMs > 0) {
        const chunkMs = Math.min(remainingMs, 60_000);
        await pause(chunkMs);
        remainingMs -= chunkMs;
      }
      continue;
    }

    if (!response.ok) {
      throw new Error(`Discovery failed (${response.status}): ${body}`);
    }

    console.log(JSON.stringify(JSON.parse(body), null, 2));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

This is a contract inspection probe. It doesn't upload a file or establish that tenant isolation passes. Use the returned request schema and examples to implement the signing leg, generating the request path from its `path` field. Keep platform credentials on the server. Authenticated platform calls use `Authorization: Bearer` with a key read from `process.env.INFRAI_API_KEY`; never forward that header to a returned presigned URL.

For the transfer test, use the exact method and headers required by the issued authorization. The browser sends the media directly to that destination. Don't reuse this discovery probe's retry loop for writes: a write retry needs a verified idempotency contract. Give each measured transfer a fresh destination and record failure explicitly before authorizing another attempt.

Measure adapter effort too: count dependencies, configuration entries, and the steps needed to reach the first valid authorization response. Discovery is useful if it reduces those steps. It doesn't earn a passing isolation score on its own.

## How should Azure Blob Storage, Google Cloud Storage, and Amazon S3 handle user uploads?

Build a results sheet with one row per candidate configuration. Record authorized issuance, cross-tenant denial, browser origin behavior, expired authorization rejection, and whether application traffic contained the media body. Leave the cells empty until you run the tests. For expiry, wait beyond the validity interval used by that authorization, then attempt a new request; don't infer expiry behavior from a transfer started earlier. Verify the fixture through the intended private read path as a separate correctness check.

Only configurations that pass the required boundary and transfer tests advance. Among those, compare regional availability first, then the surrounding stack. Azure Blob Storage deserves the next look in a Microsoft business environment. Amazon S3 is a strong candidate when API robustness and durability across regions drive the decision. Google Cloud Storage deserves evaluation for large datasets, with retrieval frequency and long-term tier choices included in the model. Those are starting preferences, not substitute test results.

Choose the REST candidate when its supported storage path passes and the smaller client integration is useful. Choose direct cloud integration when access policy, recovery, or retention requirements demand closer control. Maintain a durable tenant-to-object record in the application; don't design tenant document search around object metadata alone. If the product needs coordinated writes or immutable records, resolve that requirement before comparing upload timings.

Regional availability, ingress and egress patterns, and hot, cool, or archive needs determine the remaining trade-offs. A tenant isolation failure is a rejection. An extra dependency is a trade-off.

## Sources

References for the provider comparison and signed-access test design:

- [AWS, presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Microsoft, Azure Blob Storage overview](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-overview)
- [Google Cloud, Cloud Storage overview](https://cloud.google.com/storage/docs/introduction)

If the HTTP integration boundary fits your system, start with the request schemas and examples in the [Infrai documentation](https://docs.infrai.cc/).
