---
layout: post
title: "Background Removal in the Browser: U2-Net on ONNX Runtime Web, a Zoomed Second Pass and a Guided-Filter Matte"
date: 2026-10-27 12:00:00 +0530
author: Alok Singh
description: "How my background remover turns a 320\u00d7320 saliency mask into a full-resolution cut-out with soft edges, entirely in a Web Worker, with WebGPU when it's there and WebAssembly when it isn't"
canonical_url: "https://bestlatestapps.com/blog/background-removal-in-the-browser-u2net-onnx/"
tags: [tools, howto]
---
Originally published at [bestlatestapps.com](https://bestlatestapps.com/blog/background-removal-in-the-browser-u2net-onnx/).

{% raw %}
### How my background remover turns a 320×320 saliency mask into a full-resolution cut-out with soft edges, entirely in a Web Worker, with WebGPU when it's there and WebAssembly when it isn't

Background removal is one of those features that feels like it needs a server: a big model, a GPU, an upload. For a lot of photos, it doesn't. A small segmentation network can run in a browser tab, and most of the quality people notice, clean edges and hair that doesn't look cut out with scissors, comes from what you do *after* the network has run.

This article walks through the [background remover](https://bestlatestapps.com/tools/background-remover/) on my site using its real source: `p2-aiimage.js` (the page), `p2-aiimage-worker.js` (inference) and `p2-aiimage-core.js` (the image maths). Snippets are taken or trimmed from those files. I'll be clear where something is a heuristic, and about the kinds of photo it handles badly.

------------------------------------------------------------------------

## 1. The constraints that shaped everything

Running a model in a page means paying for it in download size, memory and the main thread's responsiveness. The tool's choices follow from that:

- **The model is U2-Net's small variant, `u2netp`**, by Xuebin Qin and colleagues, released under Apache-2.0. The ONNX file the page downloads is about 4.4 MB. The full-size U2-Net is far larger; for a free web tool, the small one is the right trade.
- **The runtime is ONNX Runtime Web** (MIT licence), served from the site as gzipped WebAssembly: about 3.6 MB for the CPU build, or about 6.8 MB for the WebGPU build. Both numbers come from the files the page serves, and the page states them.
- **Both are downloaded once and kept** in the browser's Cache Storage, so the second photo starts immediately.
- **All inference runs in a Web Worker**, so the page stays scrollable and the Stop button stays responsive while the model works.
- **The photo stays in the tab.** Its pixels are copied to the worker with `postMessage`; the tool's code doesn't upload them. The only network requests the tool makes are for the model and runtime files themselves.

------------------------------------------------------------------------

## 2. Choosing WebGPU or WebAssembly, and falling back

Before starting the worker, the page asks whether the browser has a usable GPU adapter:

``` javascript
function chooseEp() {
  if (!navigator.gpu || typeof navigator.gpu.requestAdapter !== "function") return Promise.resolve("wasm");
  return navigator.gpu.requestAdapter().then(function (adapter) {
    return adapter ? "webgpu" : "wasm";
  }, function () { return "wasm"; });
}
```

`navigator.gpu` existing isn't enough; `requestAdapter()` can still resolve to `null` on unsupported hardware or blocklisted drivers. Only a real adapter selects WebGPU.

Even then, WebGPU can fail later: the runtime may not boot, or a run may error. The page handles all three failure points (boot failure, a worker error event, a job error) the same way: if it was on WebGPU and hasn't fallen back yet, it shows "Continuing on the processor…", spawns a fresh worker with the WebAssembly build, and re-queues the job. The user sees a slower result, not an error.

Inside the worker, the WebAssembly runtime is configured conservatively: `numThreads = 1` and no proxy worker. Multi-threaded WebAssembly would need cross-origin isolation headers, which affect everything else a page loads; a single thread avoids that and behaves the same in every browser.

------------------------------------------------------------------------

## 3. Shipping a runtime as a gzipped file, carefully

The WebAssembly binary is served gzipped and decompressed in the worker with the browser's own `DecompressionStream`. The code then checks the result is really WebAssembly before handing it to the runtime:

``` javascript
function gunzip(bytes) {
  if (typeof DecompressionStream !== "function") return Promise.reject(new Error("runtime"));
  var stream = new Blob([bytes]).stream().pipeThrough(new DecompressionStream("gzip"));
  return new Response(stream).arrayBuffer().then(function (buf) {
    var out = new Uint8Array(buf);
    if (out.length < 8 || out[0] !== 0 || out[1] !== 97 || out[2] !== 115 || out[3] !== 109) throw new Error("bad-wasm");
    return out;
  });
}
```

Bytes 0, 97, 115, 109 are `\0asm`, the WebAssembly magic number. A truncated download or a proxy that mangled the file fails here with a clear "the runtime file was incomplete, reload the page" message, instead of an opaque compile error deep inside the runtime.

Downloads stream through `response.body.getReader()` so the page can show "Downloading the cut-out model, 2.1 MB of 4.4 MB". Once complete, the bytes are written to Cache Storage under a versioned cache name. Next time, the same function returns the cached bytes and the label becomes "ready on this device". If Cache Storage isn't available, the tool still works; it just downloads again next visit.

------------------------------------------------------------------------

## 4. Decoding the photo the browser's way

Before any AI, the photo has to become pixels the right way up:

``` javascript
return createImageBitmap(file, { imageOrientation: "from-image", premultiplyAlpha: "none" }).catch(function () {
  return createImageBitmap(file, { imageOrientation: "from-image" });
}).catch(function () { return createImageBitmap(file); }).catch(function () { return viaImg(); });
```

The cascade matters. `imageOrientation: "from-image"` applies the EXIF rotation, so a portrait shot on a phone isn't processed sideways. Not every browser accepts every option, so each failure retries with fewer options, ending with a plain `<img>` element. Files over 40 MB are refused up front, and HEIC files that the browser can't decode get a specific message asking for a JPG or PNG instead of a generic failure.

The decoded image is then fitted to at most 2,560 pixels on the long side and about 16.7 megapixels, using a high-quality canvas resize. That's the "fitted from 4032 × 3024" note you may see. It keeps memory predictable on phones, where a full-resolution float buffer of a large photo can exhaust a tab.

------------------------------------------------------------------------

## 5. Feeding U2-Net: 320 × 320, normalised

U2-Net takes a 320 × 320 RGB tensor in planar (channel-first) order. The core prepares it in one function:

``` javascript
var MEAN = [0.485, 0.456, 0.406], STD = [0.229, 0.224, 0.225];
// U2-Net input: rect of an RGBA image squeezed to size×size, scaled by its maximum, ImageNet mean and std.
function u2Input(rgba, w, h, rect, size) {
  var px = resize(rgba, w, h, 4, 3, rect, size, size), n = size * size, max = 1e-6, i;
  for (i = 0; i < px.length; i++) if (px[i] > max) max = px[i];
  var out = new Float32Array(3 * n);
  for (i = 0; i < n; i++) {
    for (var c = 0; c < 3; c++) out[c * n + i] = (px[i * 3 + c] / max - MEAN[c]) / STD[c];
  }
  return out;
}
```

Three details:

- **The resize is area-averaging when shrinking**, not nearest-neighbour or plain bilinear. The core's `resize()` builds per-axis weight tables and averages every source pixel that falls under a destination pixel. Downscaling a 2,560-pixel photo to 320 with bilinear sampling would alias badly, and the model would see a noisier image than the photo really is.
- **Pixels are divided by the image's maximum, then normalised** with the ImageNet mean and standard deviation. That matches the preprocessing U2-Net's reference code uses; feed a model inputs scaled differently from its training and quality drops quietly.
- **The squeeze isn't aspect-preserving.** A 4:3 photo is stretched into a square. That's how the model is commonly fed, and the mask is stretched back afterwards.

The model's output is checked too. If it's already in the 0 to 1 range, it's used as-is; otherwise it's min-max normalised. That small guard means a model export that skips the final sigmoid still works.

------------------------------------------------------------------------

## 6. A second, zoomed pass

A 320-pixel mask of a full photo is coarse. If the subject fills a third of the frame, it gets maybe a hundred pixels across in the mask. So the worker runs the model a second time on a crop around the subject:

``` javascript
return inferMask(sess, rgba, w, h, null).then(function (full) {
  var box = A.maskBox(full, 320, 320, 0.2);
  var crop = box ? A.cropAround(box, 320, 320, w, h) : null;
  var zoomP = crop && A.worthZoom(crop, w, h)
    ? (say("Refining the outline…", 0, 0, "run"), inferMask(sess, rgba, w, h, crop))
    : Promise.resolve(null);
  // ...
```

- `maskBox` finds the bounding box of everything above 0.2 in the first mask.
- `cropAround` maps that box back to image coordinates and widens it by 12% of its size plus 8 pixels on every side, clamped to the image, so the edges of the subject aren't at the edge of the crop.
- `worthZoom` only runs the second pass if the crop is less than 70% of the image area. If the subject already fills the frame, a second pass would see nearly the same picture.

The two masks are then combined on a working grid of at most 1,024 pixels on the long side: the zoomed mask, which has more detail, replaces the full-frame mask inside the crop, and the full-frame mask is used outside it. The cost is a second inference on photos with smaller subjects, which is the case where it helps most.

------------------------------------------------------------------------

## 7. From a soft mask to a matte: the guided filter

A segmentation mask, even a good one, is blurry at the boundary. It knows roughly where the person is; it doesn't know which pixels in a strand of hair are hair. The fix the tool uses is the **guided filter** from Kaiming He, Jian Sun and Xiaoou Tang. It models the output, locally, as a linear function of the photo's colours:

*alpha ≈ a<sub>R</sub>·R + a<sub>G</sub>·G + a<sub>B</sub>·B + b*, with *a* and *b* fitted in every small window.

Where the photo has a strong edge, the fitted alpha follows that edge, even if the coarse mask was blurry there. The core implements the colour version with box filters, solving a 3 × 3 system per pixel with explicit cofactors. The part I find most interesting is that it only applies the filter where the mask is uncertain:

``` javascript
// Guided-filter coefficients that only act in the uncertain band around the coarse edge.
function refineCoeffs(I, p, w, h, opts) {
  var n = w * h, box = boxer(w, h);
  var r = opts.r || Math.max(2, Math.round(Math.max(w, h) / 100));
  var g = guided(I, p, w, h, r, opts.eps || 1e-3, box), lo = new Float32Array(n), hi = new Float32Array(n), i;
  for (i = 0; i < n; i++) { lo[i] = p[i] > 0.02 ? 1 : 0; hi[i] = p[i] > 0.98 ? 1 : 0; }
  box(lo, r, lo); box(hi, r, hi);
  for (i = 0; i < n; i++) {
    var wt = Math.min(1, lo[i] * 8) * Math.min(1, (1 - hi[i]) * 8);
    g.a[i] *= wt; g.a[i + n] *= wt; g.a[i + 2 * n] *= wt;
    g.b[i] = wt * g.b[i] + (1 - wt) * p[i];
  }
  return g;
}
```

`lo` marks "probably some subject" and `hi` marks "certainly subject". Box-blurring both and multiplying gives a weight that's 1 in the band around the boundary and fades to 0 deep inside the subject and far out in the background. In those confident regions the coarse mask is kept; the guided filter can't, for instance, punch holes in a black shirt in front of a black wall. The window radius scales with the image (1% of the long side), and the regulariser is 1e-3.

Then `finishAlpha` stretches the result so values under 0.02 become fully transparent and over 0.98 fully opaque, which removes faint haze in the background and pinholes in the subject.

------------------------------------------------------------------------

## 8. Applying the matte at full resolution

The guided filter runs on the working grid of at most 1,024 pixels. The photo can be 2,560. Upscaling the finished alpha would bring back the blur we just removed. Instead, the code upsamples the filter's *coefficients* and evaluates them against the full-resolution pixels:

``` javascript
upRows(planes, w, h, W, H, function (y, rows) {
  var ar = rows[0], ag = rows[1], ab = rows[2], bb = rows[3];
  for (var x = 0, p = y * W * 4; x < W; x++, p += 4) {
    var r = rgba[p] / 255, g = rgba[p + 1] / 255, b = rgba[p + 2] / 255;
    var a = finishAlpha(ar[x] * r + ag[x] * g + ab[x] * b + bb[x]) * (rgba[p + 3] / 255), a8 = Math.round(a * 255);
    // ...
  }
});
```

Because *a* and *b* vary smoothly, they survive bilinear upsampling well, and multiplying them by the full-resolution colours puts the edge exactly where the full-resolution photo has it. This is the same idea behind fast guided upsampling in the literature. `upRows` does it one row at a time with two cached source rows, so the full-resolution coefficient planes never exist in memory at once.

The original alpha is multiplied in too, so a PNG that already had transparency keeps it.

------------------------------------------------------------------------

## 9. Fixing edge colours: foreground estimation

There's a second, subtler problem at soft edges. A hair pixel that's 40% hair and 60% green wall has a greenish colour. Give it 40% alpha and put it on white, and you get a green fringe. The tool estimates the true foreground colour at partially transparent pixels using the blur-fusion method of Marco Forte and François Pitié (2021): blurred estimates of foreground and background colours, refined with the compositing equation, in two passes, one with a large radius (4.5% of the image size) and one with a small radius (0.6%).

At full resolution, only pixels with alpha strictly between 0 and 255 get their colour replaced:

``` javascript
if (a8 <= 0) { rgba[p] = rgba[p + 1] = rgba[p + 2] = rgba[p + 3] = 0; continue; }
if (fg && a8 < 255) {
  var f0 = rows[4][x], f1 = rows[6][x], f2 = rows[8][x], u = 1 - a;
  rgba[p] = clamp(f0 + a * (r - a * f0 - u * rows[5][x]), 0, 1) * 255;
  // same for green and blue
}
rgba[p + 3] = a8;
```

Fully transparent pixels also have their colour zeroed. That isn't cosmetic: it means the PNG doesn't secretly contain the old background in invisible pixels, which some editors reveal when you change blend modes.

These steps are pure functions in the core file and have Node tests, including one that runs the actual `u2netp` model on a real portrait and asserts that the background corner comes out nearly transparent and the subject's body comes out nearly opaque.

------------------------------------------------------------------------

## 10. New backgrounds without re-running the model

Once the cut-out exists, the background options (Transparent, White, Black, a colour, or Blur) are all drawn from it on the page. Changing them never re-runs the model.

Blur is the interesting one. If you just blur the original photo, the subject smears into the background and leaves a halo. The core first fills the subject's area from its surroundings, using a normalised convolution (a background-weighted blur divided by the blurred weights), and only then blurs the result, three box-blur passes in a row, which approximates a Gaussian. The sharp cut-out is drawn on top. It runs on a copy of at most 720 pixels on the long side and is scaled up, since a blurred background doesn't need detail.

Saving a JPG with a transparent background flattens onto white, and the status line says so; the PNG keeps the transparent edges.

------------------------------------------------------------------------

## 11. Where it struggles, honestly

U2-Net is a *salient object* model: it finds the thing a person would look at first. That's ideal for one person or one product, and it explains the failures the page warns about:

- **Crowds and scenes without one subject** can keep only the most prominent region.
- **Openings**, such as the gap in a mug handle or under an arm, can keep a little background.
- **Very faint results** are detected: if under 1% of pixels end up mostly opaque, the status line suggests a photo with a single clear subject instead of silently handing you an almost empty PNG.
- **Speed varies a lot by device.** The status line reports how long the run took, and says "graphics chip" or "processor". That label is an estimate from how fast the first inference ran (under 400 ms counts as the graphics chip), not a readout from the runtime. I don't quote timings here because they depend entirely on the hardware.

------------------------------------------------------------------------

## 12. Shipping on-device ML on the web: a checklist

1.  **Pick the smallest model that's good enough**, then spend effort on post-processing.
2.  **Run inference in a worker**, and use a job token so a cancelled or superseded run never paints over a newer one.
3.  **Prefer WebGPU when an adapter exists**, and fall back to WebAssembly at every failure point.
4.  **Verify downloaded binaries** (the `\0asm` magic) and cache them in a versioned cache.
5.  **Respect EXIF orientation** and fit large photos to a memory budget before processing.
6.  **Match the model's training preprocessing exactly.**
7.  **Refine edges with a guided filter** only where the mask is uncertain, and upsample coefficients, not the alpha.
8.  **Estimate foreground colours** at soft edges, and zero the colour of fully transparent pixels.

------------------------------------------------------------------------

## Wrapping up

The network is the smallest part of this tool by line count. A 4.4 MB model gives a rough idea of where the subject is; a zoomed second pass, a guided filter confined to the uncertain band, coefficient upsampling and foreground colour estimation turn that rough idea into a cut-out you'd put in a presentation. All of it runs in a worker in your browser tab, on the GPU when there is one.

Try it, or browse the related tools:

- **Background remover:** https://bestlatestapps.com/tools/background-remover/
- **Upscale image (same worker, Real-ESRGAN):** https://bestlatestapps.com/tools/upscale-image/
- **Passport photo maker:** https://bestlatestapps.com/tools/passport-photo-maker/
- **All of my free web tools:** https://bestlatestapps.com/tools/
- **My portfolio:** https://bestlatestapps.com/alok/

Portraits with clear subjects work best today; busy scenes and fine, low-contrast edges are where I expect the next improvements to come from.

*— Alok Kumar Singh*
{% endraw %}
