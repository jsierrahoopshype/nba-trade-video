# NBA Trade Video Generator

A standalone, single-file web app that turns an NBA trade into a short
(~9-second) social video (vertical 9:16 or square 1:1), with a CBA-checked
**WORKS / DOESN'T WORK** verdict.

It is a separate tool. It reads the existing
[NBA Trade Machine](https://github.com/jsierrahoopshype/TransactionMaster)'s
CBA logic for reference only and never modifies it.

## What it does

- **Live data** — salaries are fetched at runtime from
  `https://nba-trade-calculator.thejorgesierra.workers.dev/api/salaries` and
  parsed with PapaParse. No salary file is bundled.
- **CBA engine** — the salary-matching / trade-legality logic (tiered
  below-tax brackets, tax/apron tiers, hardcaps, aggregation, poison pill,
  trade kicker, min-salary exception, two-way, 10-day, NTC, options) is
  replicated from the trade machine.
- **Headshots** — every image loads through the image proxy with
  `crossOrigin="anonymous"`, so the canvas is **not tainted** and
  MediaRecorder/`toDataURL` keep working.
- **Render** — Canvas + MediaRecorder produce a downloadable `.webm`.
  - `9:16` (1080×1920): Team A on top, Team B on bottom, salary-match meter
    in the center.
  - `1:1` (1080×1080): Team A left, Team B right, meter across the bottom.
- **Timeline (~9s):** hook → Team A assets → Team B assets → meter fills /
  dollars count up → verdict stamp + one-line reason → poll/CTA end card.
- **Quiz mode** — a toggle that inserts a freeze gate between beats 2 and 3:
  *"What does [Team] need to send?"*, pause, then reveal.

## Use it

1. Pick **Team A** and **Team B**.
2. Search each team and click players to send **out** (the other team
   receives them). Remove with the ✕ on a chip.
3. Choose the aspect ratio and (optionally) Quiz mode.
4. **Preview animation** to watch it on the canvas, or **Record & download**
   to save the `.webm`.

> Output is WebM (the format MediaRecorder produces in-browser). Convert to MP4
> if a platform needs it, e.g. `ffmpeg -i trade-9x16.webm trade-9x16.mp4`.

## Run locally

The app needs to be served over `http://` (not opened as a `file://`) so the
`fetch` to the live salary worker behaves like a normal web request.

```bash
# from the repo root
python3 -m http.server 8000
# then open http://localhost:8000/
```

Any static server works (`npx serve`, `php -S localhost:8000`, etc.). Use a
recent Chrome/Edge/Firefox — they support `canvas.captureStream` +
`MediaRecorder`.

## Deploy to GitHub Pages

1. Push this repo to GitHub (`index.html` must be at the repo root).
2. **Settings → Pages → Build and deployment → Source: Deploy from a branch.**
3. Pick the branch (e.g. `main`) and folder `/ (root)`, then **Save**.
4. The site publishes at `https://<user>.github.io/<repo>/`.

No build step — it's one static HTML file. The salary worker and image proxy
already send permissive CORS headers, so it works from any origin.
