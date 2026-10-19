# ScannaLite

A tiny document scanner that lives at a URL. No App Store, no install, no account.

Point the phone at a page → it **auto-captures** when the page is steady → it
**straightens** the perspective, **cleans it up** like a real scan → tap **Done**
and the PDF opens straight into the iPhone share sheet (AirDrop, Mail, Files,
Messages).

Everything happens on the device. Nothing is uploaded to any server.

---

## Using it (iPhone)

1. Open the URL in **Safari**.
2. Tap **Scan document** → allow the camera.
3. Hold the phone over the page. The white outline snaps to the paper; hold still
   for a moment and it captures itself (there's a small chime + haptic). The
   outline dims to show that page is done — move to the next one and repeat.
4. Tap **Done (n)**. The PDF goes straight to the share sheet → **Save to Files**,
   AirDrop, or Mail.

### Nice to have: make it feel like a real app
Open the URL in Safari → **Share** → **Add to Home Screen**. It then launches
full screen with its own icon and works offline (the service worker caches it).

### Buttons that matter
| Control | What it does |
| --- | --- |
| **Looks: B&W / Gray / Color** | how new pages are processed. B&W is the crisp scanner look; Color keeps photos and stamps as they are. |
| **AUTO / MAN** (camera screen) | auto-capture on/off. Turn it off to tap the shutter yourself. |
| **☀** (camera screen) | camera light, when the phone exposes it. |
| **Add photos** | scan an existing photo instead of using the camera — it still auto-crops. |
| Tap a thumbnail | view it full size / delete it. |
| **Save PDF · n** | shares/downloads the PDF again at any time. |

Scans are kept in the browser's local database, so closing the tab (or the app)
by accident does **not** lose them. **Clear all** wipes them.

---

## Hosting it (pick one — all free)

The camera **requires HTTPS**, so a plain `file://` copy of `index.html` will
show the "Camera unavailable" panel (it still works via the "use the camera app"
fallback). Any of these give you a proper URL:

### A. Fastest: Netlify Drop
1. Go to <https://app.netlify.com/drop>.
2. Drag this whole **folder** onto the page.
3. You get a URL like `https://something.netlify.app` — send that to your friend.
   (Free account required to keep it permanently.)

### B. GitHub Pages
```powershell
cd <this folder>
git init -b main
git add .
git commit -m "ScannaLite"
gh repo create scannalite --public --source . --push
gh api repos/:owner/scannalite/pages -f 'source[branch]=main' -f 'source[path]=/'
```
Then share `https://<username>.github.io/scannalite/`.

### C. Only for a phone on your Tailscale network
```powershell
# serve the folder locally, then expose it over HTTPS on your tailnet
npx serve . -l 8080        # in one terminal
tailscale serve --bg --https=8443 http://127.0.0.1:8080
# open https://<machine>.<tailnet>.ts.net:8443 on the phone
```

---

## What's inside

| File | Purpose |
| --- | --- |
| `index.html` | the whole app — UI, camera, edge detection, clean-up, PDF writer. No dependencies, no CDN. |
| `manifest.webmanifest` | "Add to Home Screen" metadata. |
| `sw.js` | service worker, so it also works offline after the first load. |
| `icon-*.png`, `apple-touch-icon.png` | home screen icons. |

Implementation notes, in case it needs tweaking:

- **Edge detection** works on a 200px grey copy of the frame: Otsu threshold (plus
  brighter fallback thresholds, because paper is the brightest thing in shot) →
  flood fill the largest bright blob → extremes of that blob give the four
  corners. Corners are smoothed frame to frame so the outline doesn't jitter.
- **Auto-capture** fires when the quad has barely moved for ~0.65s and the frame
  has enough detail to not be blurred. A captured page is remembered, so it won't
  shoot the same page twice — move the page or tap the shutter to capture again.
- **Straightening** is a real perspective transform (8-equation homography solved
  per capture, bilinear sampling), not just a rotate/crop.
- **Clean-up**: auto-levels to stretch contrast, then either grayscale or an
  integral-image adaptive threshold (that's the "scanned document" look — it
  survives shadows and uneven light).
- **PDF**: written by hand as raw PDF bytes with each page embedded as a JPEG
  (`/DCTDecode`) — that's why there is no library and no internet needed. Verified
  to open in Acrobat, Preview, Chrome, Safari and pdf.js.

### Tuning knobs (top of the `<script>` in `index.html`)
| Knob | Default | Meaning |
| --- | --- | --- |
| `state.maxSide` | `1700` | long side of a stored page in pixels. Raise to 2400 for sharper text, lower to 1200 for smaller files/faster captures. |
| `JPEG quality` | `0.86` | in `commitPage()` — raise for quality, lower for size. |
| auto-capture dwell | `650` | ms of steadiness before it fires (`state.stableMs / 650`). |
| `state.filter` | `'bw'` | default look: `bw`, `gray` or `color`. |
