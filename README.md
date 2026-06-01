# NBA Trade Video Generator

A tool for turning an NBA trade into a short social video.

**This repo is at the DATA step.** Right now it proves one thing: it can drive
the real NBA Trade Machine in loop/generation mode and read back the resulting
legal `tradeTeams` object. No animation yet.

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

## Use it

1. Describe a scenario the way the loop expects — pick player(s) to move, a
   **to** team, and/or a **from** team, plus optional excludes. (E.g. *packages
   for Star Wing to the Bolts* = player `Star Wing` + to `Bolts`.)
2. Click **Generate via Trade Machine**.
3. You'll see the loop params being driven, the engine's verdict, a structured
   per-team summary (out / in / picks), and the **raw `tradeTeams` JSON** dumped
   from the engine.

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

This step is data-only; the JSON dump is the deliverable. Animation comes next.

## Deploy to GitHub Pages

(Not yet — no deploy requested.) When ready: Settings → Pages → Deploy from a
branch, root folder. `index.html` is at the repo root; it's a single static file
with no build step.
