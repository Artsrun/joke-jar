# Joke Jar 🫙

Mobile-first **Joke Jar** — shake your phone, get a short joke in **English / Russian**.

Pure vanilla HTML + CSS + JS + WebGL2. No frameworks, no build step, one file.

## Live

**https://artsrun.github.io/joke-jar/**

## Deploy

Uses GitHub Actions (`.github/workflows/deploy.yml`).

Any push to `main` or manual **workflow_dispatch** deploys the static site to GitHub Pages.

## Features

- Shake detection, with tap and keyboard as equal paths (no gesture is required)
- WebGL2 glass jar with spring-damper physics and a transform-feedback particle burst
- 2 languages: localized UI, jokes and follow-up questions
- Light / dark / system theme
- Respects `prefers-reduced-motion` and `prefers-contrast`
- Haptic feedback, copy / share, installable PWA with offline support

## Accessibility

The jar is a real `<button>`: focusable, labelled, and operable with Enter or
Space. Every drawn joke is announced through a live region. Pinch-zoom is not
blocked. With `prefers-reduced-motion: reduce` the render loop is stopped and
the jar is drawn as a single static frame — the app stays fully usable.

Made for quick laughs.
