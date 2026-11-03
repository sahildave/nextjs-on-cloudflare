---
title: Image processing without sharp on Cloudflare Workers
description: "Decode, resize, re-encode to WebP and rasterise SVG on Workers with three WASM packages, EXIF orientation and a 40-megapixel guard included."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers, images, wasm, sharp, webp, photon, resvg, jsquash, payload]
date: 2026-11-03
path: 05-app-features/images-without-sharp.md
---
# Image processing without sharp on Cloudflare Workers

`sharp` is a native addon, and workerd cannot load it. If your Next.js app resizes or re-encodes images on the server (uploads, URL imports, OG cards, Payload upload sizes), that code has to move to WASM. This recipe gives you a small codec module (`decodeImage`, `resizeImage`, `coverImage`, `containImage`, `encodeWebp`, `svgToPng`) that runs on Workers and in Node, so the same unit tests cover both.

## Prerequisites

- A Next.js app on Workers with OpenNext: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md). Versions used here: Next 16.3.8, `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.x, `nodejs_compat`.
- pnpm (the vendored package below uses `link:`).
- Payload only: [./payload-cms-on-workers.md](./payload-cms-on-workers.md) for the rest of the CMS port.

Packages (exact versions used in production):

| Job | Package |
|---|---|
| Decode PNG, JPEG, WebP, GIF, ICO; resize; PNG encode | `@cf-wasm/photon` 0.4.0 |
| SVG to PNG | `@cf-wasm/resvg` 0.4.0 |
| Lossy WebP encode | `@jsquash/webp` 1.5.0 plus a local wrapper, `vendor/webp-enc/` |
| Test reference only | `sharp` (devDependency, never imported by app code) |

## Steps

1. Install the packages.
   ```bash
   pnpm add @cf-wasm/photon@0.4.0 @cf-wasm/resvg@0.4.0 @jsquash/webp@1.5.0
   pnpm add -D sharp
   ```
2. Create the local wrapper package `vendor/webp-enc/` (files under Code). Copy `webp_enc_simd.wasm` unchanged from `node_modules/@jsquash/webp/codec/enc/webp_enc_simd.wasm`. Keep it in step with the `@jsquash/webp` version.
3. Link it: add `"@your-app/webp-enc": "link:vendor/webp-enc"` to `dependencies` and run `pnpm install`.
4. In `next.config.mjs`, list the four WASM packages in `serverExternalPackages` so webpack leaves their `.wasm` imports alone (they resolve unbundled).
5. Add `src/lib/media/image-header.ts` and `src/lib/media/image-codec.ts` (under Code). Change the `@your-app/webp-enc` import if you named the package differently.
6. Replace every `sharp(...)` call in app code with the codec. The pipeline shape is: `decodeImage` → `computeBoundedBoxSize` → `resizeImage` → `encodeWebp`.
7. Payload: do not pass a `sharp` option to `buildConfig`, and set no `imageSizes` on upload collections. Process the image yourself before `payload.create`.
8. Set `images: { unoptimized: true }` in `next.config.mjs`. `next/image` otherwise tries to load `sharp` at request time.
9. Add the test file and run it in Node.

## Code

### `vendor/webp-enc/package.json`

```json
{
  "name": "@your-app/webp-enc",
  "version": "1.5.0",
  "private": true,
  "type": "module",
  "exports": {
    ".": {
      "workerd": { "types": "./index.d.ts", "default": "./workerd.js" },
      "node": { "types": "./index.d.ts", "default": "./node.js" },
      "default": { "types": "./index.d.ts", "default": "./node.js" }
    }
  }
}
```

### `vendor/webp-enc/workerd.js`

```js
// wrangler bundles a relative `.wasm` import as a compiled WebAssembly.Module.
import wasm from "./webp_enc_simd.wasm";

export default wasm;
```

### `vendor/webp-enc/node.js`

```js
import { readFile } from "node:fs/promises";

export default await WebAssembly.compile(
  await readFile(new URL("./webp_enc_simd.wasm", import.meta.url)),
);
```

### `vendor/webp-enc/index.d.ts`

```ts
/** jSquash's libwebp SIMD encoder, compiled. */
declare const module: WebAssembly.Module;
export default module;
```

### `next.config.mjs` (the relevant keys)

```js
const nextConfig = {
  // Workers: these ship workerd builds that only resolve unbundled.
  serverExternalPackages: [
    "@cf-wasm/photon",
    "@cf-wasm/resvg",
    "@jsquash/webp",
    "@your-app/webp-enc",
  ],
  images: { unoptimized: true },
};
export default nextConfig;
```

### `src/lib/media/image-header.ts`

Reads width, height and EXIF orientation from the first bytes, so the size check runs before the decoder allocates `width x height x 4` bytes.

```ts
export type ImageHeader = {
  width: number;
  height: number;
  orientation: number;
};

export function readImageHeader(bytes: Uint8Array): ImageHeader | null {
  const view = new DataView(bytes.buffer, bytes.byteOffset, bytes.byteLength);
  const ascii = (at: number, length: number) =>
    String.fromCharCode(...bytes.subarray(at, at + length));
  try {
    // PNG: signature, then the IHDR chunk.
    if (bytes[0] === 0x89 && ascii(1, 3) === "PNG" && ascii(12, 4) === "IHDR")
      return {
        width: view.getUint32(16),
        height: view.getUint32(20),
        orientation: 1,
      };
    if (ascii(0, 4) === "GIF8")
      return {
        width: view.getUint16(6, true),
        height: view.getUint16(8, true),
        orientation: 1,
      };
    if (ascii(0, 4) === "RIFF" && ascii(8, 4) === "WEBP")
      return readWebp(view, ascii(12, 4));
    if (bytes[0] === 0xff && bytes[1] === 0xd8) return readJpeg(bytes, view);
    // ICO: entries of at most 256x256 (a 0 byte means 256); the largest wins.
    if (view.getUint32(0, true) === 0x00010000 && view.getUint16(4, true) > 0) {
      let width = 0;
      let height = 0;
      for (let i = 0; i < view.getUint16(4, true); i++) {
        width = Math.max(width, bytes[6 + i * 16] || 256);
        height = Math.max(height, bytes[7 + i * 16] || 256);
      }
      return { width, height, orientation: 1 };
    }
  } catch {
    // A header cut short reads past the end.
  }
  return null;
}

function readWebp(view: DataView, chunk: string): ImageHeader | null {
  if (chunk === "VP8 ")
    return {
      width: view.getUint16(26, true) & 0x3fff,
      height: view.getUint16(28, true) & 0x3fff,
      orientation: 1,
    };
  if (chunk === "VP8L") {
    const bits = view.getUint32(21, true);
    return {
      width: (bits & 0x3fff) + 1,
      height: ((bits >> 14) & 0x3fff) + 1,
      orientation: 1,
    };
  }
  if (chunk === "VP8X") {
    const u24 = (at: number) =>
      view.getUint16(at, true) | (view.getUint8(at + 2) << 16);
    return { width: u24(24) + 1, height: u24(27) + 1, orientation: 1 };
  }
  return null;
}

/** Walks the JPEG markers to the frame header, reading the EXIF orientation on the way. */
function readJpeg(bytes: Uint8Array, view: DataView): ImageHeader | null {
  let orientation = 1;
  let at = 2;
  while (at + 4 <= bytes.length) {
    if (bytes[at] !== 0xff) return null;
    const marker = bytes[at + 1];
    if (marker === 0xff) {
      at += 1;
      continue;
    }
    const length = view.getUint16(at + 2);
    if (
      marker === 0xe1 &&
      String.fromCharCode(...bytes.subarray(at + 4, at + 8)) === "Exif"
    )
      orientation = exifOrientation(view, at + 10) ?? orientation;
    // SOF0-SOF15, except DHT (C4), JPG (C8) and DAC (CC).
    if (
      marker >= 0xc0 &&
      marker <= 0xcf &&
      marker !== 0xc4 &&
      marker !== 0xc8 &&
      marker !== 0xcc
    )
      return {
        height: view.getUint16(at + 5),
        width: view.getUint16(at + 7),
        orientation,
      };
    if (marker === 0xda) return null;
    at += 2 + length;
  }
  return null;
}

/** Tag 0x0112 in IFD0 of the TIFF block at `tiff`. */
function exifOrientation(view: DataView, tiff: number): number | undefined {
  const little = view.getUint16(tiff) === 0x4949;
  const ifd = tiff + view.getUint32(tiff + 4, little);
  const count = view.getUint16(ifd, little);
  for (let i = 0; i < count; i++) {
    const entry = ifd + 2 + i * 12;
    if (view.getUint16(entry, little) === 0x0112) {
      const value = view.getUint16(entry + 8, little);
      return value >= 1 && value <= 8 ? value : undefined;
    }
  }
  return undefined;
}
```

### `src/lib/media/image-codec.ts`

```ts
import { PhotonImage, resize, SamplingFilter } from '@cf-wasm/photon';

import { readImageHeader } from './image-header';

/**
 * The raster steps the media pipeline needs, on WASM because Workers cannot load
 * sharp's native addon: Photon decodes and resizes, jSquash (libwebp) encodes lossy
 * WebP, resvg draws SVG. Pixels are straight-alpha RGBA, row-major. Re-encoding from
 * pixels is what strips EXIF/GPS: no metadata is carried over.
 */
export type Raster = { data: Uint8Array; width: number; height: number };

/** The most pixels decodeImage will allocate: 40 MP is 160 MB of RGBA. */
export const MAX_DECODE_PIXELS = 40_000_000;

/**
 * Decode PNG, JPEG, WebP, GIF or ICO bytes, upright: Photon ignores the JPEG EXIF
 * orientation, so a phone photo would come out on its side. The header is read
 * first, and a source over 40 megapixels is refused before Photon allocates it.
 * Throws when the bytes are not an image this reads.
 */
export function decodeImage(bytes: Uint8Array): Raster {
  const header = readImageHeader(bytes);
  if (!header) throw new Error('Unsupported image format.');
  if (header.width * header.height > MAX_DECODE_PIXELS)
    throw new Error(`Image is ${header.width}x${header.height}, over 40 megapixels.`);
  const image = PhotonImage.new_from_byteslice(bytes);
  let raster: Raster;
  try {
    raster = { data: image.get_raw_pixels(), width: image.get_width(), height: image.get_height() };
  } finally {
    image.free();
  }
  return orient(raster, header.orientation);
}

/**
 * Apply an EXIF orientation (1–8) to decoded pixels. For each source pixel,
 * `place` gives its upright position; 5–8 swap width and height.
 */
function orient(image: Raster, orientation: number): Raster {
  if (orientation < 2 || orientation > 8) return image;
  const { width: w, height: h } = image;
  const swap = orientation >= 5;
  const outWidth = swap ? h : w;
  const place: (x: number, y: number) => [number, number] = {
    2: (x: number, y: number) => [w - 1 - x, y],
    3: (x: number, y: number) => [w - 1 - x, h - 1 - y],
    4: (x: number, y: number) => [x, h - 1 - y],
    5: (x: number, y: number) => [y, x],
    6: (x: number, y: number) => [h - 1 - y, x],
    7: (x: number, y: number) => [h - 1 - y, w - 1 - x],
    8: (x: number, y: number) => [y, w - 1 - x],
  }[orientation] as (x: number, y: number) => [number, number];
  const source = image.data.byteOffset % 4 === 0 ? image.data : image.data.slice();
  const pixels = new Uint32Array(source.buffer, source.byteOffset, w * h);
  const out = new Uint32Array(w * h);
  for (let y = 0; y < h; y++)
    for (let x = 0; x < w; x++) {
      const [ox, oy] = place(x, y);
      out[oy * outWidth + ox] = pixels[y * w + x];
    }
  return {
    data: new Uint8Array(out.buffer),
    width: outWidth,
    height: swap ? w : h,
  };
}

/**
 * Resize to exactly `width`×`height` (Lanczos3). Resampled premultiplied, as sharp
 * does, so transparent pixels do not bleed a dark fringe into the edges.
 */
export function resizeImage(image: Raster, width: number, height: number): Raster {
  if (image.width === width && image.height === height) return image;
  const pixels = new Uint8Array(image.data);
  for (let i = 0; i < pixels.length; i += 4) {
    const a = pixels[i + 3];
    if (a === 255) continue;
    pixels[i] = Math.round((pixels[i] * a) / 255);
    pixels[i + 1] = Math.round((pixels[i + 1] * a) / 255);
    pixels[i + 2] = Math.round((pixels[i + 2] * a) / 255);
  }
  const source = new PhotonImage(pixels, image.width, image.height);
  const resized = resize(source, width, height, SamplingFilter.Lanczos3);
  source.free();
  const data = resized.get_raw_pixels();
  resized.free();
  for (let i = 0; i < data.length; i += 4) {
    const a = data[i + 3];
    if (a === 255) continue;
    if (a === 0) {
      data[i] = data[i + 1] = data[i + 2] = 0;
      continue;
    }
    data[i] = Math.min(255, Math.round((data[i] * 255) / a));
    data[i + 1] = Math.min(255, Math.round((data[i + 1] * 255) / a));
    data[i + 2] = Math.min(255, Math.round((data[i + 2] * 255) / a));
  }
  return { data, width, height };
}

/** Scale to fill `width`×`height`, then crop the overflow evenly (sharp `fit: 'cover'`, centred). */
export function coverImage(image: Raster, width: number, height: number): Raster {
  const scale = Math.max(width / image.width, height / image.height);
  const scaled = resizeImage(
    image,
    Math.max(width, Math.round(image.width * scale)),
    Math.max(height, Math.round(image.height * scale)),
  );
  const left = Math.floor((scaled.width - width) / 2);
  const top = Math.floor((scaled.height - height) / 2);
  const data = new Uint8Array(width * height * 4);
  for (let y = 0; y < height; y++) {
    const start = ((top + y) * scaled.width + left) * 4;
    data.set(scaled.data.subarray(start, start + width * 4), y * width * 4);
  }
  return { data, width, height };
}

/** Scale to fit inside `width`×`height`, centred on transparent padding (sharp `fit: 'contain'`). */
export function containImage(image: Raster, width: number, height: number): Raster {
  const scale = Math.min(width / image.width, height / image.height);
  const w = Math.max(1, Math.round(image.width * scale));
  const h = Math.max(1, Math.round(image.height * scale));
  const fitted = resizeImage(image, w, h);
  const left = Math.floor((width - w) / 2);
  const top = Math.floor((height - h) / 2);
  const data = new Uint8Array(width * height * 4);
  for (let y = 0; y < h; y++) {
    data.set(fitted.data.subarray(y * w * 4, (y + 1) * w * 4), ((top + y) * width + left) * 4);
  }
  return { data, width, height };
}

type WebpEncoder = typeof import('@jsquash/webp/encode.js');
let webpEncoder: Promise<WebpEncoder> | undefined;

/**
 * jSquash's own loader fetches its `.wasm` by URL, which neither Node (`file:` fetch)
 * nor workerd (no runtime WASM compile) allows, so it gets the compiled module from
 * `vendor/webp-enc`: its workerd entry imports the wasm relatively, which wrangler
 * bundles as a `WebAssembly.Module`; its Node entry compiles the file.
 */
function loadWebpEncoder(): Promise<WebpEncoder> {
  webpEncoder ??= (async () => {
    const encoder = await import('@jsquash/webp/encode.js');
    const { default: module } = await import('@your-app/webp-enc');
    await encoder.init(module);
    return encoder;
  })();
  webpEncoder.catch(() => {
    webpEncoder = undefined;
  });
  return webpEncoder;
}

/** Lossy WebP at `quality` (0–100), alpha kept at full quality, no metadata. */
export async function encodeWebp(image: Raster, quality: number): Promise<Buffer> {
  const { default: encode } = await loadWebpEncoder();
  const data = new Uint8ClampedArray(image.data.buffer, image.data.byteOffset, image.data.byteLength);
  const webp = await encode({ data, width: image.width, height: image.height } as ImageData, { quality });
  return Buffer.from(webp);
}

/** PNG bytes for an SVG drawn with its longest side at `side` pixels. */
export async function svgToPng(svg: Uint8Array, side: number): Promise<Buffer> {
  const { Resvg } = await import('@cf-wasm/resvg');
  const natural = await Resvg.async(svg);
  const longest = Math.max(natural.width, natural.height) || 24;
  natural.free();
  const resvg = await Resvg.async(svg, { fitTo: { mode: 'zoom', value: side / longest } });
  try {
    return Buffer.from(resvg.render().asPng());
  } finally {
    resvg.free();
  }
}
```

### Pipeline example: `src/lib/media/process-image.ts`

From a production `store.ts` and `small-surface-images.ts`, trimmed. `computeBoundedBoxSize` scales into a box and never upscales.

```ts
import { decodeImage, encodeWebp, resizeImage, coverImage } from "./image-codec";

const LONG_EDGE_MAX = 1920;
const WEBP_QUALITY = 85;
const MAX_BYTES = 2 * 1024 * 1024;

function computeBoundedBoxSize(width: number, height: number, max = LONG_EDGE_MAX) {
  if (width <= 0 || height <= 0) return { width, height };
  const scale = Math.min(1, max / Math.max(width, height));
  return {
    width: Math.max(1, Math.round(width * scale)),
    height: Math.max(1, Math.round(height * scale)),
  };
}

/** Any supported image in, EXIF-free WebP out. */
export async function processFeedImage(input: Buffer) {
  const image = decodeImage(input);
  const target = computeBoundedBoxSize(image.width, image.height);
  const output = await encodeWebp(
    resizeImage(image, target.width, target.height),
    WEBP_QUALITY,
  );
  if (output.byteLength > MAX_BYTES) throw new Error("Image is too large after processing.");
  return { buffer: output, ...target };
}

/** A square avatar: scale to fill, crop the overflow evenly. */
export async function processAvatar(input: Buffer, size = 512) {
  const fitted = coverImage(decodeImage(input), size, size);
  return { buffer: await encodeWebp(fitted, WEBP_QUALITY), width: size, height: size };
}
```

### `src/lib/media/image-codec.test.ts`

`sharp` is the reference here and runs only in Node. The test builds a JPEG for each EXIF orientation and checks the codec lands the same quadrant colours as `sharp().rotate()`.

```ts
import sharp from "sharp";
import { describe, expect, it } from "vitest";

import { type Raster, decodeImage } from "./image-codec";

// Quadrants red | green over blue | white, so every flip and rotation lands a
// different colour in the top-left corner.
const QUADRANTS = [
  [255, 0, 0],
  [0, 255, 0],
  [0, 0, 255],
  [255, 255, 255],
];
async function quadrantJpeg(orientation: number): Promise<Buffer> {
  const width = 64;
  const height = 32;
  const pixels = Buffer.alloc(width * height * 3);
  for (let y = 0; y < height; y++)
    for (let x = 0; x < width; x++) {
      const q = (y < height / 2 ? 0 : 2) + (x < width / 2 ? 0 : 1);
      pixels.set(QUADRANTS[q], (y * width + x) * 3);
    }
  return sharp(pixels, { raw: { width, height, channels: 3 } })
    .jpeg({ quality: 100 })
    .withMetadata({ orientation })
    .toBuffer();
}

/** Which quadrant colour a pixel is closest to. */
function quadrantAt(image: Raster, x: number, y: number, channels = 4): number {
  const i = (y * image.width + x) * channels;
  const px = [image.data[i], image.data[i + 1], image.data[i + 2]];
  const distance = (c: number[]) =>
    c.reduce((sum, v, k) => sum + Math.abs(v - px[k]), 0);
  return QUADRANTS.map(distance).reduce(
    (best, d, q, all) => (d < all[best] ? q : best),
    0,
  );
}

describe("decodeImage", () => {
  // sharp's auto-orient (`rotate()` with no angle) is the reference.
  for (const orientation of [1, 2, 3, 4, 5, 6, 7, 8])
    it(`turns an EXIF orientation ${orientation} JPEG upright`, async () => {
      const jpeg = await quadrantJpeg(orientation);
      const { data, info } = await sharp(jpeg)
        .rotate()
        .raw()
        .toBuffer({ resolveWithObject: true });
      const expected = {
        data: new Uint8Array(data),
        width: info.width,
        height: info.height,
      };
      const image = decodeImage(jpeg);
      expect([image.width, image.height]).toEqual([
        expected.width,
        expected.height,
      ]);
      const probes = [
        [8, 4],
        [image.width - 8, 4],
        [8, image.height - 4],
        [image.width - 8, image.height - 4],
      ];
      expect(probes.map(([x, y]) => quadrantAt(image, x, y))).toEqual(
        probes.map(([x, y]) => quadrantAt(expected, x, y, info.channels)),
      );
    });

  it("refuses a source over 40 megapixels before decoding it", () => {
    // A PNG signature and IHDR for 8000x6000 (48 MP), no pixel data.
    const png = Buffer.alloc(33);
    png.set([0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a, 0, 0, 0, 13]);
    png.write("IHDR", 12, "ascii");
    png.writeUInt32BE(8000, 16);
    png.writeUInt32BE(6000, 20);
    png.set([8, 6, 0, 0, 0], 24);
    expect(() => decodeImage(png)).toThrow(/over 40 megapixels/);
  });
});
```

## Verify

1. Unit tests, in Node (EXIF 1 to 8, 40 MP guard):
   ```bash
   pnpm exec vitest run src/lib/media/image-codec.test.ts
   ```
   Expect: 9 tests passed.
2. Build and run in the real runtime. The Node entry of `webp-enc` is not what ships; workerd takes the `workerd` export.
   ```bash
   pnpm exec opennextjs-cloudflare build
   pnpm exec wrangler dev --local
   ```
   Post a JPEG to the route that calls `processFeedImage` and check the response is `image/webp`:
   ```bash
   curl -s -o out.webp -w '%{content_type} %{size_download}\n' \
     -F file=@photo.jpg http://localhost:8787/api/your-upload-route
   file out.webp   # RIFF (little-endian) data, Web/P image
   ```
3. Strip check: `exiftool out.webp` shows no GPS or camera tags.
4. Grep the build for a leftover import: `rg "from ['\"]sharp['\"]" src` returns only test files.

## Gotchas

- Symptom: build or first request fails with a native module or `sharp` error. Cause: `sharp` imported in app code, or a framework reaching for it (`next/image` optimisation, Payload `sharp` option). Fix: remove the imports, `images.unoptimized: true`, no `sharp` in `buildConfig`.
- Symptom: `@jsquash/webp` fails to initialise on Node or on Workers. Cause: jSquash's own loader fetches its `.wasm` by URL. Node has no `file:` fetch and workerd cannot compile WASM at runtime. Fix: pass it a precompiled `WebAssembly.Module` from the `webp-enc` wrapper, as `loadWebpEncoder` does.
- Symptom: a bare `.wasm` import breaks in the OpenNext bundle. Cause: OpenNext rewrites bare `.wasm` imports into a path next to the importer. Fix: packages whose workerd entry imports `./x.wasm` relatively work (Photon, resvg). `@jsquash/webp` does not, hence the local wrapper with a `workerd` export condition.
- Symptom: webpack refuses `.wasm` in server code. Fix: list the package in `serverExternalPackages` (and use `/* webpackIgnore: true */` on a direct import).
- Symptom: phone photos come out on their side. Cause: Photon ignores the JPEG EXIF orientation. Fix: `decodeImage` reads the orientation with `readImageHeader` and rotates the pixels itself (`orient`). The first version shipped without this. Browser uploads were already rotated, so only URL-fetched camera JPEGs hit it.
- Risk: a large image exhausts the isolate. Cause: workerd gives 128 MB per isolate and RGBA is 4 bytes per pixel, so a 40 MP source is 160 MB. No crash was reproduced; the guard is a precaution. Fix: read the header first and refuse over 40 megapixels before Photon allocates. Resizing copies the buffer, so tune the cap down if you see memory errors.
- Symptom: transparent icons get a dark fringe after resize. Cause: resampling straight-alpha pixels bleeds the colour of transparent pixels into the edge. Fix: `resizeImage` premultiplies before Lanczos3 and un-premultiplies after, as sharp does.
- Symptom: `<text>` in an SVG icon disappears. Cause: resvg in WASM loads no fonts. Fix: none in this recipe; icons with text render blank. Convert text to paths upstream if it matters.
- Symptom: AVIF input fails deep in the pipeline. Cause: Photon cannot read AVIF. Fix: refuse it at the fetch step with a clear message (for example "AVIF images are not supported. Use PNG, JPEG or WebP.").
- Metadata: EXIF and GPS are dropped because the WebP is encoded from raw pixels. No extra strip step.
- Speed: WASM is slower than sharp. There is no per-image timing for the codec alone here; measure on your own images before promising an SLA, and keep large work off the request path if you can (resize in the browser before upload).
