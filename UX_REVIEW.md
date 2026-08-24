# Joke Jar — UX/UI design review

Review of the app as it stood at `666986c` ("Finish PWA: service worker +
offline cache"), plus the changes made in response.

Scope agreed before starting: keep EN/RU (fix the docs rather than restore
Armenian), and tighten the existing dark blue-accent identity rather than
restyle it.

---

## 1. The centrepiece never rendered

**The WebGL jar and the GPU particles have never been visible.** The stage has
been an empty rectangle since the WebGL work began.

The 4×4 matrix helper multiplied as though its matrices were row-major:

```js
o[i*4+j] = a[i*4]*b[j] + a[i*4+1]*b[4+j] + a[i*4+2]*b[8+j] + a[i*4+3]*b[12+j];
```

Every matrix the app builds is column-major (`translate()` puts x, y, z at
indices 12–14), and that is also what `gl.uniformMatrix4fv(..., false, ...)`
expects. Feeding column-major data through a row-major multiply produced a
transform with `clip.w == 0` for almost every vertex:

| vertex | before | after |
|---|---|---|
| `(0.43, -0.56, 0)` | `w=0.000` → undefined NDC | `w=2.403`, NDC `(0.565, -0.939)` |
| `(-0.43, -0.56, 0)` | `w=0.000` → undefined NDC | `w=2.697`, NDC `(-0.503, -0.836)` |
| `(0.33, 0.58, 0)` | `w=0.000` → undefined NDC | `w=2.437`, NDC `(0.427, 0.833)` |
| `(0, 0.24, 0.43)` | `w=-0.430` (behind camera) | `w=2.146`, NDC `(-0.231, 0.350)` |

Nothing errored, because a degenerate transform is not a GL error —
`drawElements` ran ~60 times a second and produced nothing. That is why five
successive commits could add spring-damper physics, a multi-axis shake model
and a transform-feedback particle system without any of it becoming visible.

Fixed by making `mul()` a proper column-major multiply. The existing call sites
(`mul(proj, mul(view, model))`) then read correctly as `proj · view · model`,
so no call site changed.

Two follow-ons were needed once geometry appeared:

- The context is `premultipliedAlpha: true` but the shaders emitted
  straight alpha. Now they output `vec4(col * a, a)` with
  `blendFuncSeparate(ONE, ONE_MINUS_SRC_ALPHA, …)`, so the canvas composites
  correctly onto the page.
- At ~0.9 alpha with depth writes on, the jar self-occluded and looked like a
  solid plastic churn. Body alpha is now ~0.32 with a strong fresnel rim and
  `depthMask(false)` for the glass pass, so the far wall shows through and the
  particle burst is visible *inside* the jar.

---

## 2. Accessibility blockers

### Keyboard users could not get a joke at all
The only trigger for the first joke was `canvas.addEventListener('click', …)`.
A `<canvas>` is not focusable and has no role, so there was no keyboard path
and nothing for a screen reader to operate. The "Next joke" button only existed
*after* a joke had been drawn, so it could never be reached first.

→ The jar is now a real `<button>` wrapping the canvas: focusable, labelled,
and operable with Enter or Space.

### Denying motion permission bricked the app
`#perm` was a full-screen overlay shown before anything else. It was dismissed
only on `'granted'`; on denial the branch did nothing, so the wall stayed up
permanently with no explanation and no way past it.

→ The blocking overlay is gone. The app works by tapping from the first frame,
and motion is offered as an optional upgrade via a small "Enable shake" pill.
Denial hides the pill, shows a short explanation, and the app keeps working.
This also removes a full-screen interstitial that every iOS user had to
dismiss before seeing the product.

### Pinch-zoom was disabled
`user-scalable=no, maximum-scale=1` — a WCAG 1.4.4 failure. Removed.

### Jokes were never announced
`jokeEl.textContent = joke.text` with no live region. Note that simply adding
`aria-live` to the joke element is *not* enough here: the card is `display:none`
until the first draw, and a live region that is populated and revealed in the
same task does not reliably announce. The live region is therefore a separate,
always-rendered visually-hidden element outside the card.

### No reduced-motion support
A full-screen shaking 3D object plus a particle burst, unconditionally. Now
`prefers-reduced-motion: reduce` stops the render loop entirely (a single static
frame is drawn), suppresses the impulse and the burst, drops the haptic, and
replaces the card's rise animation. The app remains fully functional.

### Text scaling clipped content
`body { overflow: hidden }` with fixed heights meant enlarged text was cut off
with no way to scroll. The output region now scrolls.

Also added: visible focus rings, `aria-pressed` on the language buttons (the
selected language was previously conveyed by colour alone), `<html lang>` and
per-element `lang` so screen readers switch voice, and a `prefers-contrast: more`
block.

---

## 3. Bugs

| Issue | Detail |
|---|---|
| **Manifest referenced missing icons** | `icon-192.png` and `icon-512.png` were listed but never existed in the repo. Both are now generated from `icon.svg`, plus a properly padded `icon-maskable-512.png` — the 512 was previously declared `"any maskable"` without safe-zone padding, so a platform mask would crop it. |
| **Layout thrash every frame** | `resize()` ran inside the render loop, reading `canvas.clientWidth` 60×/sec and forcing synchronous layout. Now driven by a `ResizeObserver` flag. |
| **Render loop never stopped** | `requestAnimationFrame` ran forever, including on a hidden tab. Now parked on `visibilitychange`, and the transform-feedback pass is skipped entirely when no particles are alive. |
| **Constant repeats** | 7 jokes picked with `Math.random()`, guarding only an immediate repeat. Replaced with a shuffled deck that exhausts before reshuffling, and won't hand back the joke just shown. |
| **Language switch left the UI bilingual** | Switching language re-labelled the chrome but left the previous language's joke on screen. It now redraws. |
| **Dead control** | The "Interesting" button set `opacity: 0.5` and did nothing else. Replaced with Copy / Share (Web Share API where available, clipboard otherwise), with toast confirmation. |
| **Cooldown was a silent no-op** | Taps during the 1100 ms cooldown did nothing with no feedback. Cooldown is now 850 ms and visibly disables the button. |
| **Wrong hint on desktop** | The hint always read "Shake the device", including where no motion sensor exists. It now reflects the actual available input, and only claims shaking works after a real motion event has been observed. |
| **Stale app for returning users** | The service worker was cache-first for navigations, so a returning visitor could stay on an old build. Navigations are now network-first with a cached-shell fallback. Its `catch` also returned `index.html` for *any* failed request, so a failed image resolved to HTML. |
| **No WebGL2 fallback** | `console.warn` and an empty stage. Now falls back to the jar icon so the stage is never blank. |

---

## 4. Interface

- **Version chip removed.** `v4.3` in the header and `<title>` is developer
  metadata; it tells a user nothing and dates the product on every screenshot.
- **Follow-up questions restyled.** They were accent-coloured and prefixed "→",
  which reads as tappable — an affordance lie, since they are not interactive.
  Now quiet secondary text under a "Think about it" label.
- **Light theme added**, with the palette keyed off a resolved `data-resolved`
  attribute so "auto" needs no duplicated token block. The jar shader takes a
  `uDark` uniform, and the canvas clears transparent so the themed page shows
  through instead of the canvas painting its own hard-coded dark background.
- **Type scale** consolidated into four fluid steps rather than a dozen
  hard-coded rem values.
- **Landscape** no longer pushes the card off-screen on short viewports.
- Contrast checked: muted-on-background is 6.5:1 (dark) and 5.9:1 (light);
  accent button text is 10.9:1 (dark) and 5.5:1 (light). All pass AA.

---

## 5. Verified

Driven in headless Chromium at 390×844 with a real WebGL2 context:

```
PASS  WebGL2 context created
PASS  Jar is a focusable <button>
PASS  Keyboard (Enter) draws a joke
PASS  Live region announces, outside card
PASS  Deck cycles without repeats — 7 distinct of 7 draws
PASS  Language switch is complete
PASS  Pinch-zoom allowed
PASS  Light theme applies
PASS  Dark theme applies
PASS  Render loop stops when hidden
PASS  Reduced motion: still works
No console errors. No failed requests. All 4 manifest icons resolve 200.
```

---

## 6. Not addressed

- **Armenian.** The README promised Armenian / English / Russian; commit
  `8bae688` deliberately dropped it. Per the decision taken for this pass, the
  README and manifest were corrected to EN/RU rather than the language being
  restored. Re-adding it is a content task: UI strings, a joke set, and a font
  stack with Armenian coverage.
- **Content depth.** Seven jokes per language means a user sees the whole jar
  in about a minute. This is the biggest remaining product limitation, and no
  amount of interface work compensates for it.
- **Branch hygiene.** `prod` holds only the initial README and a
  "Hello, world" CI stub, and has diverged from `main` (which is what actually
  deploys). Worth deleting or resetting to avoid future work being branched
  from it — this review was very nearly written against that empty tree.
