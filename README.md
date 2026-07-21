# KP2 Birmingham (Kharis Church) — Website

Static website for **KP2 Birmingham / Kharis Church**: homepage, service information,
and photo/video galleries of church events (Sunday services, midweek services, fasting
services, and other events).

## Tech at a glance

- Plain **static HTML/CSS/JS** — no build step, no framework. Pages are served/opened directly.
- **All media (photos + videos) is served from a bunny.net CDN**, not from this repo.
- Deployment target: **Netlify**, from the GitHub repo `Musa-Mundia/KP2-Birmingham-website`.

---

## The shift in approach: media lives on the CDN, not in the repo

Originally the site loaded photos and videos from local folders sitting next to the HTML
(several thousand images plus videos). Those assets were **migrated to bunny.net**, and the
repository now contains **code only** — the media folders are git-ignored.

| | |
|---|---|
| Pull (delivery) zone | `https://kp2-website-pz.b-cdn.net/` |
| Media path prefix | everything lives under `/media/`, e.g. `https://kp2-website-pz.b-cdn.net/media/Service%20photos/...` |
| Storage zone (origin, for uploads/listing) | `kp2-web-assets` — UK region, host `uk.storage.bunnycdn.com`, accessed via bunny's HTTP Storage API with an `AccessKey` header |
| Local mirror | `C:\HTML Files\KP2-web-media\media\` — a **backup only**, verified identical to the CDN |

### Rules of thumb

- **Every** image/video `src` in the HTML must be a `https://kp2-website-pz.b-cdn.net/media/...` URL.
- Spaces in paths are URL-encoded as `%20`.
- **Reference and list images from the CDN, never from the local mirror.** The mirror is a backup;
  the CDN is the single source of truth.

---

## Gallery pages — how they work now

Event galleries (e.g. `Album-gallery-Event-Services/.../*.HTML`) show a grid of thumbnails plus a
full-screen lightbox/modal, with a "Download all photos" ZIP button.

**Key change:** the modal/lightbox is now **auto-generated from the thumbnails at runtime** by
`buildModalSlidesFromGallery()`. There are **no hardcoded modal image paths** anymore. Each
thumbnail tile looks like this:

```html
<div class="column">
  <img class="demo"
       src="https://kp2-website-pz.b-cdn.net/media/.../NAME.jpg"
       onclick="openModal();currentSlide(N)" alt="Service Photo">
</div>
```

Because the modal, the slide counter, and the ZIP download all read from `.gallery-row .demo`,
**you only ever edit the thumbnail grid** — everything else stays in sync automatically.

---

## Procedure: batch-replacing a gallery's images

Use this when repointing a gallery to a new CDN folder (as done for
*June 2026 Fasting 2 / Midweek Services*).

1. **Get the authoritative file list from the CDN** (not the local mirror). List the target folder
   via the Storage API:
   ```
   GET https://uk.storage.bunnycdn.com/kp2-web-assets/media/<path>/
   Header:  AccessKey: <storage-zone password>
   ```
   The JSON `ObjectName` fields are the filenames. Sort them for a stable order and confirm the count.

2. **Build one thumbnail tile per image**, numbered sequentially in document order:
   ```html
   <div class="column">
     <img class="demo"
          src="https://kp2-website-pz.b-cdn.net/media/<path>/<NAME>"
          onclick="openModal();currentSlide(<n>)" alt="Service Photo">
   </div>
   ```
   `currentSlide(n)` runs `1..N`. Encode spaces as `%20`.

3. **Replace the existing tiles manually, in batches of ~25** (no bulk find/replace scripts run
   against the file), inside the `<div class="gallery-row"> … </div>` block. If the new image
   count is smaller than the old one, delete the surplus tiles; if larger, add tiles and keep the
   `currentSlide` numbering contiguous.

4. **Do NOT touch the modal/lightbox** — it regenerates itself from the thumbnails. No separate
   modal edits are needed (older pages that still have hardcoded modal slides can have them removed).

5. **Verify before finishing:**
   - tile count == image count; `currentSlide` runs `1..N` with no gaps or duplicates;
   - the filenames in the HTML exactly match the CDN listing (diff them);
   - every `src` URL returns **HTTP 200** (spot-check, or check all).

6. **Remove redundant sections** — leftover tiles, stale hardcoded modal slides, or references to a
   different event's folder.

### Gotchas

- The **hostname is case-insensitive** (`KP2-Website-PZ` == `kp2-website-pz`), but the **path after
  `/media/` is case-sensitive**.
- bunny caches files for **30 days**. If you overwrite a file under the same name, **purge the CDN
  cache** for that URL; uploading under a new path avoids this entirely.
- Photos are stored at **2048px** with no separate thumbnail files — the gallery CSS scales them.

---

## Repo layout (code only)

- `Index.html` — homepage
- `Album-gallery-Event-Services/`, `Gallery-Selection/` — event galleries and selection pages
- `External_demo.css` and per-page `<style>` blocks — styling
- Media folders — **git-ignored**; served from the CDN

> For CDN migration history, zone details, and upload lessons, see `CDN-Migration-Report.docx`.
