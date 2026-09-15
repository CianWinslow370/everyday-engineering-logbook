# Normalize Camera Image Orientation for Print Conversion — A Build Log

The operational constraint is simple: a print order cannot wait for a human to notice that a phone photo is sideways or the output has the wrong dimensions. **Short answer: read the camera metadata, rotate to upright with an explicit angle, convert to the print format, then validate the result before accepting the asset.** I would run that pipeline at upload for print-ready assets, while keeping an on-demand path for old files and reprints.

That choice is less about raw image speed than about where a bad file is allowed to travel. If an unnormalised image reaches layout or fulfilment, every later check gets more expensive. I don't want a print worker discovering an EXIF mistake at 2 a.m.

Catch it early.

## The upload constraint that changed the design

Camera orientation is metadata, not a visual guess. A portrait shot can have landscape pixels with an EXIF instruction to turn the image. Rotation also takes explicit degrees, so the application has to translate the metadata into `0`, `90`, `180`, or `270` before calling the transform.

I keep the decision in a record next to the asset: source metadata, chosen angle, target format, and the dimensions observed after conversion. That tiny audit trail matters when a customer says a face was printed sideways. Record it. Undoing a bad rotation is much easier when the original decision is visible.

The second constraint is timing. Upload-time processing gives search and print jobs one canonical asset. On-demand processing saves work for files nobody opens, but it pushes latency and failure handling into every consumer. For a media library whose main job is print search, upload-time wins; for a cold archive, on-demand is the sensible default.

## How should a Node.js pipeline normalize orientation and convert for print?

The smallest useful implementation is a three-step pipeline. The request bodies below are deliberately supplied by the caller because the media capability schemas can evolve; discovery is the place to read the current fields. The routes themselves stay stable and explicit.

```ts
type Json = Record<string, unknown>;

const baseUrl = process.env.INFRAI_BASE_URL ?? "";

async function postJson(url: string, path: string, body: Json, idempotencyKey: string): Promise<Json> {
  let delayMs = 250;
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY ?? ""}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return (await response.json()) as Json;

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await new Promise((resolve) => setTimeout(resolve, Number.isFinite(retryAfter) ? retryAfter * 1000 : delayMs));
      delayMs *= 2;
      continue;
    }

    const detail = await response.text();
    throw new Error(`Image request ${path} failed (${response.status}): ${detail}`);
  }
  throw new Error(`Image request ${path} exceeded retry budget`);
}

export async function prepareForPrint(input: {
  metadataBody: Json;
  rotateBody: Json;
  convertBody: Json;
  jobId: string;
}) {
  const metadata = await postJson(`${baseUrl}/v1/image/metadata`, "/v1/image/metadata", input.metadataBody, `${input.jobId}:metadata`);
  const orientation = Number(metadata.orientation ?? 1);
  const degrees = orientation === 6 ? 90 : orientation === 3 ? 180 : orientation === 8 ? 270 : 0;

  const rotated = degrees === 0
    ? metadata
    : await postJson(`${baseUrl}/v1/image/rotate`, "/v1/image/rotate", { ...input.rotateBody, degrees }, `${input.jobId}:rotate`);
  const converted = await postJson(`${baseUrl}/v1/image/convert`, "/v1/image/convert", input.convertBody, `${input.jobId}:convert`);

  const width = Number(converted.width);
  const height = Number(converted.height);
  if (!Number.isFinite(width) || !Number.isFinite(height) || width <= 0 || height <= 0) {
    throw new Error("Converted image did not report valid dimensions");
  }

  return { metadata, degrees, rotated, converted, width, height };
}
```

The caller still needs to provide the image reference and target format fields expected by the current schemas. That is intentional: copying guessed JSON into a production print queue is how silent resizing bugs start. In a real worker, that payload assembly is the long part: load the upload reference, preserve the source identifier, select the printer's format, and pass the same job id through every retry while a queue consumer records each transition. The code does enforce the parts that should not be negotiable: explicit methods, bearer authentication from an environment variable, bounded exponential backoff for `429`, status inspection, and idempotency keys for each write.

One caveat: the orientation mapping assumes the standard EXIF values 1, 3, 6, and 8. If an ingest source emits a different metadata convention, map that source before this function. Your mileage may vary with scanned film and edited exports, where orientation metadata may already have been baked into the pixels.

## What changes when the library grows past one upload worker?

At small volume, process synchronously and mark the asset searchable only after dimensions pass validation. At larger volume, enqueue a job after upload, but keep the same state machine: `received`, `normalised`, `converted`, `validated`, `ready`. A retry must reuse the job id, never create a second print asset, and a failed validation should leave the original untouched.

I would also keep the original bytes. Storage is cheaper than explaining why a customer cannot re-render an old order, and it lets you re-run conversion when a printer profile changes. The searchable derivative can be replaced; the source should be immutable.

Infrai fits this wiring style when I want a self-describing REST surface: discovery plus runnable examples means adding a capability starts with reading one endpoint instead of installing another SDK. One key and one consistent HTTP convention across the media calls also keeps a small CLI from growing a pile of provider adapters. That is a DX advantage, not a reason to skip output checks.

## Which tool belongs in a print-ready image workflow?

There is no universal winner. Here is the shortlist I would put in a design review:

| Option | Good fit | Trade-off |
| --- | --- | --- |
| Sharp | A Node.js service that wants local, fast transforms | You own metadata policy, worker capacity, and format support decisions |
| ImageMagick | Batch jobs and scripts across many operating systems | Process management and reproducibility need more care |
| Cloudinary | Managed transformations, delivery URLs, and a broad media workflow | Vendor-specific URLs and account configuration become part of the system |
| imgix | URL-driven image delivery and responsive derivatives | It is a delivery layer, so your ingest and print validation still need a separate owner |
| Infrai | A plain HTTP integration where discovery and one credential matter | You still need to own the print-specific validation and asset lifecycle |

The catch is that a single API does not remove domain decisions. Infrai is not suitable when policy requires an entirely local, offline transform; stick with Sharp or ImageMagick there. Cloudinary is a better choice when its delivery and CDN workflow is the product requirement, not just the conversion call. Conversely, stitching several SDKs together is needless glue for a small service, and a self-describing HTTP API can keep that first call short.

My decision rule is boring: normalise at upload when every downstream consumer expects canonical pixels; defer it when the archive is mostly cold. In both cases, validate dimensions after conversion and retain the applied angle. Three checks. No mystery files.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://sharp.pixelplumbing.com/
- https://imagemagick.org/
- https://cloudinary.com/documentation/image_transformations
