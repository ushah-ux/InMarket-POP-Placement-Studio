# InMarket DOOH Proof of Placement Mock up tool

Drops a campaign's creative onto the screens in out-of-home placement photography, at the
right perspective, and builds the proof-of-placement deck from the result. Pin each screen's
four corners once; after that every future campaign is a drag-and-drop and a download.

Everything runs in the browser. No server, no accounts, no upload — artwork is read from
disk with `FileReader` and warped on the GPU.

## Using it

The live copy is on GitHub Pages — open the link under **About** on the repository page and
it runs, no install. That's the one to send people.

To turn Pages on, or if the link is missing: **Settings → Pages → Source: Deploy from a
branch → Branch: `main`, folder: `/ (root)` → Save.** The first build takes a minute or two.
The `.nojekyll` file in this repository is what stops GitHub's templating from interfering;
leave it there.

Requires WebGL, which every current browser has.

To run it on your own machine instead:

```sh
git clone <this repo>
cd <this repo>
python3 -m http.server 8000
# open http://localhost:8000
```

It has to be served over HTTP either way. Opening `index.html` straight off the disk fails:
browsers treat `file://` images as cross-origin, which taints the canvas and makes the WebGL
texture upload throw.

## What's in it

```
index.html              the whole app — markup, styles, WebGL warp, deck writer
p/slide-01..35.jpg      placement photographs at full resolution
t/slide-01..35.jpg      gallery thumbnails
template/shell.b64.txt  the 2025 InMarket deck template, stripped to master +
                        layouts + theme + cover, base64'd
inmarket-logo.png       header mark (@2x alongside)
```

The 35 placements and their 38 pinned screens are baked into `index.html` as the `DATA`
constant, so a fresh clone opens ready to work.

## How it works

**Perspective-correct warping.** Each screen is four corners in the photo. The app solves
the homography onto that quad and renders it with WebGL, so the artwork converges the way
the real screen does — not a skew or a free transform. Checked against an OpenCV reference
of the same pipeline: mean absolute difference of one to three levels out of 255, confined
to the anti-aliased screen edge.

**Exposure matching.** The warped artwork is nudged toward the luminance the real screen was
emitting, so it reads as a lit display rather than a pasted rectangle. A gamma-damped gain,
clamped to ±22%.

**Creative assignment.** Files are grouped by aspect ratio and dealt round-robin to screens
whose shape matches, so two portrait files alternate across the portrait screens. Any screen
can be pinned to a specific file with the dropdown under its preview.

**Quality.** Artwork is pre-downscaled on the CPU to about 1.35× the destination quad before
it becomes a texture, which avoids the shimmer a single large minifying warp produces. Edges
are anti-aliased by MSAA. Output is the photo's native resolution.

## The library

Placements are grouped in the grid by the DOOH size they approximate — 1920×1080, 1280×960,
1080×1920 and so on — worked out from the pinned quad rather than the photo, so the heading
reflects the screen.

Adding a photo runs a 16×16 luminance signature against everything already there and refuses
a close match, naming the placement it duplicates. **Scan for duplicates** sweeps the whole
library for near-identical pairs at a looser threshold and pre-selects the later copy.

Removal is soft: removed placements drop out of the grid and every export but stay in
storage, and **Show removed** brings them back.

## Outputs

- **Download JPGs** — the finished composites at native resolution.
- **Layer PNGs** — just the warped artwork on transparency, to composite in Photoshop.
- **Swap into a deck** — drop in an existing `.pptx` and every matching screenshot is
  replaced in place. Images are matched by content, not slide position: each image in the
  deck is compared against the library by exact pixel dimensions plus a luminance signature,
  so reordered or duplicated slides still land correctly. Layout, crops and slide structure
  are untouched.
- **Build InMarket deck** — a new deck on the 2025 template: a cover carrying the title and
  subhead you type, then one slide per selected placement, each mockup in a rounded card with
  the template's drop shadow and an eyebrow naming the size and placement.

## Storage

Served from a plain web server the app keeps pins, selection, removals and added photos in
`localStorage`, and the added photos' image data in IndexedDB — all of it per-browser, so
pin edits made on one machine stay there. Running inside the Claude artifact the same code
picks up the hosted document store instead and everything is shared across the team; the
shim at the top of `index.html` is what switches between the two.

## Updating the pins for everyone

Pin edits live in browser storage. To make one permanent for every clone, edit the `DATA`
constant near the top of the script block in `index.html` — `pins` maps a placement id to an
array of screens, each eight numbers: top-left x, top-left y, top-right x, top-right y,
bottom-right, bottom-left, as fractions of the photo's dimensions. "Top-left" means the
corner where the artwork's top-left should land, which is what fixes the artwork's rotation.

## Known gaps

- `slide-04` is a 4:3 screen and currently takes whichever creative the assignment picks; the
  4:3 Booster Ball stretches about 26% horizontally on a 16:9 screen if that pairing comes up.
- `slide-33` has a second ceiling-mounted screen that has never been pinned.

## Faces

The placement photography was shot in live stores, so shoppers and staff are in frame. Every
recognisable face outside the advertising screens has been obscured — 26 of them across 13
photos, detected with YuNet and then checked by eye, pixelated and blurred under a feathered
elliptical mask. Faces that are part of the advertising artwork, on a screen or a lightbox
poster, are left alone, because they are the creative and blurring them would wreck the
mockup.

If you add a placement photo of your own, check it for faces before committing it.

## Licence and content

© inMarket. The tool and the placement photography are inMarket's; the campaign artwork on
the screens is inMarket-owned creative. Everything in here is cleared for this repository —
but a photograph taken in a client's store is still a photograph taken in a client's store,
so check with the account team before reusing the imagery outside proof-of-placement work.
