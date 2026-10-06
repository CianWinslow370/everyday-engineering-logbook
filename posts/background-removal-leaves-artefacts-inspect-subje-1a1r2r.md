# Background Removal Leaves Artefacts: Inspect Subject Evidence Before Transfer

TL;DR: When background removal leaves a halo, a jagged outline, or a missing pale edge, stop tuning the final composite first. Keep the original upload, inspect it at native resolution, and separate three stages: source decode, mask generation, and alpha compositing. For customer-support product photos, run a cheap local preflight, upload the original when the edge is uncertain, and moderate the unmodified source alongside the cutout. That spends bandwidth where quality is actually at risk.

A white shoe against a white wall is not a rendering bug waiting for a CSS fix. The boundary may carry very little color difference. Lossy compression, resizing, or a baked-in matte can remove more evidence before segmentation begins. Once those pixels are gone, a sharper export cannot recover them.

The constraint changes the design: support uploads need a fast preview, but the moderation decision cannot depend on a bandwidth-saving derivative alone. Preview speed and evidence retention are different jobs. For a queue dominated by small, high-contrast catalog shots, sending every original first wastes transfer time without improving the operator's first view. For a queue containing pale fabrics, translucent packaging, and reflective objects, throwing away the original is a much worse trade-off. I would benchmark those two queues separately instead of hiding both behind one average.

Pixels vanish.

## How should you debug background removal artefacts on a low-contrast subject?

Start with the untouched file. Decode it once, respect its intrinsic dimensions, and record the format, width, height, and byte length. Do not compare a cutout with a thumbnail produced by an earlier step. That test confounds resampling with segmentation.

JPEG is lossy and has no alpha channel. PNG supports lossless compression and alpha. WebP can be lossy or lossless and supports alpha. Those capabilities do not make one format universally better, but they identify transformations that may have happened before the remover saw the pixels. Converting a damaged JPEG to PNG preserves the decoded damage; it does not reverse it.

Check the path in order:

1. Decode the original and apply its intended orientation.
2. View it at 100% without a mask. Look for blur, ringing, clipping, and weak separation between subject and background.
3. Render the predicted mask as grayscale. Missing detail here points to segmentation or insufficient source evidence.
4. Composite the same foreground and mask over black, white, and a saturated checkerboard. A fringe that changes with the backing color points to alpha or matte handling.
5. Compare at identical pixel dimensions. Browser scaling can make a sound mask look soft or a bad one look acceptable.

One trap is inspecting transparency only over white. A pale fringe disappears there, then becomes obvious on a dark storefront card. Use at least 3 deliberately different backdrops during diagnosis, not because 3 is a quality standard, but because a single backdrop cannot expose background-dependent fringing.

## The smallest useful preflight

The first implementation needs one pure function, one policy object, and a result the upload UI can explain. The browser can decode and sample the image before network transfer. That makes the fast path fast.

This sample estimates local color variation around the outer fifth of the frame. It is a triage signal, not a background-removal score. Product photos can touch the frame, use textured backdrops, or place the subject off-center, so calibrate the threshold against representative uploads and review false accepts and false rejects.

```ts
export type PreflightPolicy = {
  sampleStep: number;
  lowVariationThreshold: number;
};

export type PreflightResult = {
  width: number;
  height: number;
  bytes: number;
  mime: string;
  edgeVariation: number;
  preserveOriginal: boolean;
};

const pixelDistance = (data: Uint8ClampedArray, a: number, b: number): number =>
  (Math.abs(data[a] - data[b]) +
    Math.abs(data[a + 1] - data[b + 1]) +
    Math.abs(data[a + 2] - data[b + 2])) / 3;

export async function preflightImage(
  file: File,
  policy: PreflightPolicy,
): Promise<PreflightResult> {
  const bitmap = await createImageBitmap(file, { imageOrientation: "from-image" });
  const canvas = new OffscreenCanvas(bitmap.width, bitmap.height);
  const context = canvas.getContext("2d", { willReadFrequently: true });
  if (!context) throw new Error("2D canvas is unavailable");

  context.drawImage(bitmap, 0, 0);
  bitmap.close();
  const { data } = context.getImageData(0, 0, canvas.width, canvas.height);
  const bandX = Math.max(1, Math.floor(canvas.width / 5));
  const bandY = Math.max(1, Math.floor(canvas.height / 5));
  let total = 0;
  let comparisons = 0;

  for (let y = 0; y < canvas.height; y += policy.sampleStep) {
    for (let x = 0; x < canvas.width - policy.sampleStep; x += policy.sampleStep) {
      const inOuterBand = x < bandX || x >= canvas.width - bandX ||
        y < bandY || y >= canvas.height - bandY;
      if (!inOuterBand) continue;

      const current = (y * canvas.width + x) * 4;
      const next = (y * canvas.width + x + policy.sampleStep) * 4;
      total += pixelDistance(data, current, next);
      comparisons += 1;
    }
  }

  const edgeVariation = comparisons === 0 ? 0 : total / comparisons;
  return {
    width: canvas.width,
    height: canvas.height,
    bytes: file.size,
    mime: file.type,
    edgeVariation,
    preserveOriginal: edgeVariation < policy.lowVariationThreshold,
  };
}
```

No universal value belongs in `lowVariationThreshold`. Label a set of real support uploads, plot the score distributions, and choose the operating point that matches the cost of a damaged listing versus an extra original upload. Version that policy, or a threshold change will quietly rewrite moderation behavior.

The result drives two paths. A confident image can send a compact preview first. An uncertain image keeps and uploads the original, with the preview as a separate derivative. Never overwrite the source object with the cutout.

This heuristic has hard limitations. It is not appropriate as an accept-or-reject classifier, and it performs poorly as a proxy when the subject touches the frame or the backdrop is textured. In those cases, skip the shortcut: preserve the original and let the downstream mask plus human-review path make the decision. That costs bandwidth. It also keeps the evidence needed to diagnose a bad boundary.

## Why does the halo move when the background changes?

A cutout is usually represented by color channels plus alpha. In straight alpha, stored color is independent of opacity. In premultiplied alpha, color components are already multiplied by alpha. Mixing those interpretations can create dark or light borders at partially transparent pixels. The W3C compositing specification defines the math; use it instead of guessing from one preview.

If the source was previously composited over white, edge pixels may contain a blend of subject color and white even when the new mask is correct. Removing white cannot reconstruct the original foreground color from that flattened image without uncertainty. Retaining the first uploaded file matters.

Export the mask itself, then render the cutout on three known backgrounds. If the mask has a bite missing, investigate input evidence and segmentation. If the mask looks right but the composite has a fringe, inspect color decontamination, premultiplication, and resizing. If only the small preview fails, trace the derivative encoder and resampler.

Do not sharpen the alpha edge as a default repair. It can hide a soft halo on one item while cutting translucent plastic, fine straps, or motion-softened boundaries on another. Fix the failed stage.

## What would change at scale?

Move the same staged contract behind a queue, but keep the client preflight. Give the original an immutable identifier and checksum. Derived previews should record the source identifier, transform version, decoded dimensions, output dimensions, and mask version. Moderation receives both source and derivative references, not one mutable URL.

Observability should answer a narrow set of questions: which input format and dimensions entered the pipeline, which transform version ran, how long each stage took, and why an image was routed for review. Do not log image bytes or public object URLs as convenient debug context. Uploads can contain customer data, so retention and access control belong in the design.

Build a fixed evaluation set that overrepresents hard boundaries: white-on-white products, reflective packaging, translucent edges, fine straps, shadows, and subjects touching the frame. Keep the originals. Review masks and composites separately because one aggregate score cannot tell an operator whether segmentation or rendering failed.

Bandwidth still matters. Use dimensions and risk signals to choose upload order, cap concurrent transfers, and resume interrupted originals where the transport supports it. But a local heuristic is not proof that an original is disposable. The bandwidth win is a scheduling decision; the quality loss may be permanent.

At higher volume, add a manual-review state instead of forcing every upload into pass or reject. Use machine-readable reasons such as `LOW_EDGE_EVIDENCE` and `ALPHA_COMPOSITE_MISMATCH`, then show plain language to the support operator. A queue also has a limitation: it adds delay and operational state, so a synchronous path remains the better fit for small images that decode quickly and pass a calibrated preflight. The decision rule remains short: preserve evidence, diagnose the earliest failing stage, and spend extra bytes on ambiguous edges.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://developer.mozilla.org/en-US/docs/Web/API/Window/createImageBitmap
- https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas
- https://www.w3.org/TR/compositing-1/
- https://www.w3.org/TR/png-3/
