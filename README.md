# FrameForge

**Eyewear virtual try-on platform.** It turns photos of real frames into accurate 3D models, then lets shoppers try them on live in the browser. Retailers embed it with one script tag.

![FrameForge try-on demo](assets/demo.gif)

<!-- Uncomment once the demo is deployed:
▶️ **[Live demo](https://glasses.tryon.kessaldev.com/try-on/demo)** (needs camera access; nothing leaves your browser)
-->

> The source code is private because this is a commercial project. Code walkthroughs are available to employers on request: harounekessal89@gmail.com

---

## What it does

**For brands: digitize frames automatically**
- Turns a capture session (studio rig, phone + laptop screen at home, or a quick **front + side photo** mode) into an articulated, physically based **glTF** model with working temple hinges, lenses, and materials.
- Reconstructs lens width and bridge to within about 1 mm in home-capture tests.
- Includes a review studio where brand reviewers check each model, then approve and publish it to the CDN.

**For shoppers: AR try-on in the browser**
- Live face tracking with no app to install. Works on desktop and mobile.
- Glasses stay put when a hand covers the face and follow the head when looking down.
- Measures **pupillary distance** with a standard bank card and gives a fit hint ("Good fit", "Runs narrow").

**For retailers: one-line embed**
```html
<script async src="https://tryon.example.com/widget/v1.js" data-key="ff_pk_..."></script>
<button data-frameforge-sku="ACME-1234-C2" hidden>Try on</button>
```
- "Try on" buttons appear only when a SKU has a published 3D model.
- The try-on runs in a sandboxed iframe, so camera frames never reach the store's page. Only origins registered on the key can embed it.
- Events (`open`, `snapshot`, `add_to_cart`, …) plug into the store's cart and analytics.

## Architecture

![Architecture](assets/architecture.png)

```
capture station ──presigned PUT──▶ S3 captures (private) ◀── worker (digitizer)
        │                                   ▲
        └──▶ catalog-api ──jobs──▶ PostgreSQL ┘
                  │
     on publish:  ▼
             S3 assets (public) ──▶ CDN ──▶ web try-on ◀── retailer page + widget
```

| Component | Stack |
|---|---|
| Digitizer pipeline + worker | Python, NumPy/SciPy, OpenCV, trimesh, glTF-Transform; GPU backend with 2D Gaussian Splatting (PyTorch, CUDA) |
| Catalog / studio API | FastAPI, PostgreSQL, S3 (presigned uploads) |
| Try-on client and widget | Next.js, TypeScript, MediaPipe, Three.js |
| Infrastructure | Docker Compose, SeaweedFS (S3), CPU and GPU worker profiles |

## Engineering highlights

- **The job queue is PostgreSQL alone.** Workers claim jobs with `FOR UPDATE SKIP LOCKED`, wake up on `LISTEN/NOTIFY`, and send heartbeats so a dead worker's job is requeued. Each job names its backend, so CPU workers take visual-hull jobs and GPU workers take 2DGS jobs. No Redis or message broker needed.
- **Reconstruction keeps thin parts.** Silhouette carving runs coarse-to-fine from 4 mm down to 0.3 mm voxels, testing voxel bounding spheres against a distance transform, so thin temples and bridges never get carved away. The hull also seeds the 2DGS stage and crops its floaters.
- **A custom glTF exporter** controls the hinge pivot nodes, the extensions, and the fit metadata. Each level of detail is compressed with meshopt and WebP until it fits the size budget.
- **Tracking quality is measured, not guessed.** An evaluation harness with **17 gates** runs in CI: static jitter (< 0.4 px), lag (< 1 px), occlusion recovery (back within 15 frames), head-down pose, and PD accuracy. A change that buys smoothness with lag fails the build.
- **Security by design:** the try-on runs in an iframe on its own origin, publishable keys are bound to registered origins, review files stay in a private bucket, and only approved assets reach the public CDN.

## Testing

- 59 digitizer tests on realistic synthetic captures, compared against ground truth: geometry, size, materials, rig faults, home capture, two-photo mode, and the worker queue on real PostgreSQL.
- The 17-gate tracking and PD harness, plus API tests, run on every push.

---

Built by **Haroune Kessal** · [GitHub](https://github.com/HarouneKESSAL) · harounekessal89@gmail.com
