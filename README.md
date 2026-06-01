# NBA Trade Video Generator

Turns an NBA trade into a short, animated social video (9:16 or 1:1).

The trade and its verdict come **only** from the real NBA Trade Machine (loaded
and run in a same-origin iframe). This tool never builds or validates a trade —
it consumes the engine's `tradeTeams` and animates it.

## Design rule: no legality logic here

This tool contains **no CBA / trade-legality logic of its own**. Every trade and
every verdict is produced by the **real Trade Machine**
([TransactionMaster](https://github.com/jsierrahoopshype/TransactionMaster)).
This tool only *consumes* the result. It never builds or validates a trade, and
it never recomputes a verdict. If it can't read a trade from the engine, it
**fails closed** — it shows an error and refuses to proceed. It never fabricates
a trade or a verdict.

## How it drives the engine

The Trade Machine supports a loop/generation mode via URL params:

```
?player=<slugs>&to=<teamslug>&from=<teamslug>&exclude=<slugs>&loop=1
```

Slugs are the player/team name with non-alphanumerics replaced by hyphens. In
that mode the engine runs `generateUnifiedTrade()` and stores the result in its
global `tradeTeams` (an array of
`{ teamName, playersOut, playersIn, picksOut, rightsOut, cashOut, tpeUsed }`).

Because the engine reads its salary CSV from `window.__INJECTED_CSV_DATA__`
(normally injected by the Cloudflare Worker) rather than fetching it, this tool:

1. Fetches the **live CSV** from `/api/salaries` and the **live Trade Machine
   source** from `raw.githubusercontent.com` (both send permissive CORS).
2. Injects the CSV as `window.__INJECTED_CSV_DATA__` and appends a tiny **bridge
   script**, then loads the whole thing in a **same-origin blob-URL iframe**.
   (Blob URLs inherit the host page's origin, so the iframe is same-origin and
   its globals are readable — the reliable, verifiable method when this tool
   isn't served from the engine's own origin. If it ever *is* served from
   `hoopsmatic.com`/the worker, a plain same-origin iframe to the live URL works
   too.)
3. The bridge runs **inside the engine's scope**: it resolves the scenario the
   same way `setupTradeLoopFromUrl()` does, calls the engine's own
   `generateUnifiedTrade()` / `redistributePlayers()` / `validateTrade()`, and
   returns the structured `tradeTeams` plus the engine's verdict message.

If the source can't be fetched, the iframe can't be read (e.g. a genuine
cross-origin situation), or `generateUnifiedTrade()` returns an error/empty —
the UI shows a red error and stops.

The live `/api/salaries` feed is still used for the **player/team pickers only**
(names, salaries, headshots) — never for legality. Headshots load through the
image proxy with `crossOrigin="anonymous"`.

## The animated renderer

Once a legal `tradeTeams` is captured, the **Render video** card animates it on a
canvas and records a downloadable WebM via `MediaRecorder`. The look is matched
to the [Bar Chart Race generator](https://github.com/jsierrahoopshype/bar-chart-race):

- **Poppins** font, brand gradient `#0f0c29 → #302b63`, vignette + film noise.
- NBA **team-color gradient bars** (chips) with a top highlight strip and soft
  shadow — the same bar treatment as the race.
- The race's easing: `ease_out_cubic` for chips flying in, `ease_in_out_cubic`
  for the meter, and **linear** value interpolation for the salary count-ups
  (constant growth, no micro-pauses).

Sequence (~9.5s, +~2s in quiz mode):

1. **Hook** — team marks/headshots animate on over "WOULD YOU DO THIS TRADE?".
2. **Team A chips** fly in one at a time (staggered); the team's outgoing-salary
   counter ticks up linearly as each lands.
3. *(quiz mode)* **freeze gate** — holds on "What completes this trade? <Team B>".
4. **Team B chips** fly in the same way.
5. **Salary-match meter** fills continuously in response to the running totals.
6. **Verdict stamp** lands (`LEGAL` / `NOT LEGAL`) with the Trade Machine's own
   message as the reason line — the verdict is never recomputed here.
7. **End card / CTA** poll prompt.

Two aspect ratios behind the `aspect` flag:
- **9:16** (1080×1920) — Team A block on top, Team B on the bottom, meter centered.
- **1:1** (1080×1080) — Team A left, Team B right, meter across the bottom.

Headshots load through the image proxy with `crossOrigin="anonymous"`, so the
canvas is never tainted (`toDataURL`/`MediaRecorder` keep working — verified).

## Use it

1. Describe a scenario the way the loop expects — pick player(s) to move, a
   **to** team, and/or a **from** team, plus optional excludes.
2. Click **Generate via Trade Machine** — you get the engine's verdict, a
   per-team summary, and the raw `tradeTeams` JSON (collapsible).
3. In **Render video**, choose the aspect ratio and (optionally) quiz mode, then
   **Preview** to watch it on the canvas or **Record & download** to save the
   WebM.

> Output is WebM (what `MediaRecorder` produces in-browser). Convert if a
> platform needs MP4: `ffmpeg -i trade-9x16.webm trade-9x16.mp4`.

## Run locally

Serve over `http://` (not `file://`) — blob-URL iframes only inherit the page
origin over http(s), and the cross-origin `fetch`es need a real origin.

```bash
# from the repo root
python3 -m http.server 8000
# then open http://localhost:8000/
```

Any static server works (`npx serve`, `php -S localhost:8000`, …). Use a recent
Chrome/Edge/Firefox.

### What to test

- **Happy path:** pick a star + a destination team → Generate → you get a green
  banner, a verdict from the engine, and a `tradeTeams` JSON dump with non-empty
  `playersOut`/`playersIn` on both teams.
- **Fail closed:** temporarily break your network (or block
  `raw.githubusercontent.com`) and Generate → you get a red error and **no**
  trade. Ask for an impossible package (e.g. exclude every realistic return
  piece) → the engine returns an error and the tool surfaces it without
  inventing anything.
- **Render both ratios:** after a trade, switch the aspect to **9:16** and to
  **1:1** and click **Record & download** for each → a `.webm` downloads and
  plays back in the preview, with chips flying in, counters ticking, the meter
  filling, and the verdict stamp landing. Toggle **Quiz mode** to see the freeze
  gate before Team B's reveal.

## Deploy to GitHub Pages

(Not yet — no deploy requested.) When ready: Settings → Pages → Deploy from a
branch, root folder. `index.html` is at the repo root; it's a single static file
with no build step.
