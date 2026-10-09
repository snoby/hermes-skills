---
name: animated-scenario-diagrams
description: Animated dark-mode scenario diagrams as one HTML file.
version: 0.1.0
author: Snoby (snoby), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [visualization, svg, animation, html, diagrams, ci]
    related_skills: [p5js, claude-design]
---

# Animated Scenario Diagrams (single-file HTML)

Build interactive, multi-scenario pipeline/flow diagrams as ONE self-contained HTML file — dark mode, animated state machine (blue pulse → green settle / red halt), zero dependencies, opens via `file://`. Start from `templates/diagram-skeleton.html`; it contains the working engine and every fix listed in Pitfalls already applied.

## When to Use
- User wants an animated walkthrough of "what happens when X fails" — CI pipelines, deploy flows, incident paths, approval gates
- Multiple scenarios over the same graph (happy path + failure modes + cancel/queue semantics)
- Don't use for: static diagrams (use excalidraw / architecture-diagram), data charts (use p5js), multi-page web apps

## Architecture (the pattern)
One HTML file, five parts:

1. **Theme**: `:root` vars — `--bg/--panel/--card/--border/--fg/--muted/--accent/--accent2/--ok/--bad/--warn/--skip`. Dark values: ok `#3fd97f`, bad `#ff5c66`, warn `#e0a53f`, skip `#6e7f95`, accent `#4d9fff`, accent2 `#7c5cff`, border `#233044`.
2. **DATA**: `NODES = [{id,x,y,w,h,t,s,chip,gate}]`, `EDGES = [{d:"M..C..",from,to,cls,label,lx,ly}]`. Every edge `d` is a **hand-routed path** — never auto-route; plan corridors so nothing crosses boxes or text.
3. **Build**: SVG constructed at load — node = rect + title + sub (+ optional NEW-RUNNER chip), edge = path with `marker-end` arrow, plus lane labels and dashed plane outlines.
4. **State machine**: one class on each node group drives ALL visuals: `st-waiting / st-running / st-pass / st-fail / st-skip / st-cancel / st-pending`.
5. **Engine**: `SCENARIOS = {id: [beats]}`; beat `{at, n, st, log, g:[ids], cg:[ids], e, to}`. `play()` = reset + one `setTimeout` per beat; `exec()` applies log AND state together — never short-circuit on log.

## Visual grammar
| State | Visual |
|---|---|
| running | accent border + pulse animation |
| pass | green border (2px) |
| fail | red border + shake + burst + red flash on inbound edges + STOP tag |
| pending | amber blink + QUEUED… tag (queued-not-skipped semantics) |
| skip | grey dashed border, dimmed |
| cancel | amber border + CANCELLED tag; whole-run cancel also flips still-running jobs |
| edge flowing | **dotted green, marching ants** |
| edge completed | **solid green** |
| edge blocked | red dashed flash, 900–1100 ms transient |

## Procedure
1. Agree the job graph + gate semantics with the user FIRST (who blocks whom, what skips vs waits vs cancels). Write the scenario spec to a file.
2. Layout: main spine left→right across the top; secondary lanes hang below in columns. Data flows left→right into each box (inputs enter LEFT edge, outputs exit RIGHT edge).
3. Build the file from `templates/diagram-skeleton.html` — fill NODES/EDGES/SCENARIOS, keep the CSS/engine as-is.
4. **Geometry pass** (in-page): call `window.__verifyGeometry()` via `browser_exec` `js()` — must return 0 edge-through-box collisions, 0 label-overlaps, 0 out-of-viewBox items. Fix by re-routing edges through free corridors; re-run until clean.
5. **Per-scenario state pass**: run each scenario in the browser; assert every node's `getComputedStyle(rect).stroke` matches the expected palette color; assert end-state tags are unique per position.
6. **Timing probes**: mid-run samples (e.g. t≈4s) to prove dotted-green edges flow and completed edges are solid, and tag replacement works (QUEUED… → STOP, never stacked).
7. **Visual sign-off**: `capture_screenshot()` + `vision_analyze`. If vision times out (it does, repeatedly), fall back to DOM assertions — they are the ground truth.
8. **External semantics review**: for domain-critical logic (e.g. GitHub Actions skip/cancel propagation), pipe the spec file into `claude -p "<role> … Return ONLY: verdict + fixes" --max-turns 1`, then apply fixes to both diagram and spec.

## Pitfalls (all hit for real — skeleton has fixes applied)
1. **CSS specificity**: a base rule WITH id prefix `#app .node-rect` (1,1,0) beats an unprefixed state rule `.st-pass .node-rect` (0,3,0) — colors silently never apply. Prefix every state rule: `#app .node.st-pass .node-rect`.
2. **CSS transitions freeze in throttled/background tabs**: stroke/opacity transitions never advance, so state colors stick at their start value forever. Use `transition: none` for anything state-driven and express life via keyframe `animation` (those run).
3. **`classList.add("blocked warn")` throws InvalidCharacterError** and silently aborts the REST of the fx function mid-cascade. Always separate `add()` calls.
4. **Beat offsets are ABSOLUTE ms** from run start. `{at:100}` after a 4000 ms beat still fires at t=100. Chain nothing implicitly.
5. **Tags stack**: QUEUED… and STOP render at identical coords and overlap. One tag per node — new state replaces, skip clears.
6. **SVG elements**: `.className` is an `SVGAnimatedString`, not a string — use `classList` / `getAttribute('class')`.
7. **Stale page after edits**: the browser may hold the old file — `goto_url` fresh before probing, and verify the page's inline script contains your latest marker string.
8. **Edge endpoints must land ON node borders** (±3 px) — an arrowhead floating in dead space reads as "doesn't even connect" (user will flag it).
9. **Labels near the viewBox edge**: right-anchor (`text-anchor="end"`) or they clip; check with `getBBox()` against viewBox width.
10. **exec() short-circuit**: checking `log` first and returning skips the state change on beats that carry both. Apply both, then branch.

## Verification
- `__verifyGeometry()` → `{edgeCollisions:[], labelOverlaps:[], outOfView:[]}` all empty
- Every scenario's end state matches the agreed semantics, node by node (computed strokes, not class names alone)
- No two tags share the same (x,y) at any sampled time; exactly one tag per halt point
- Visual screenshot pass, or DOM-assert fallback if vision_analyze times out

## Sharing the skill
- Personal install lives under `~/.hermes/skills/` (per-profile: `~/.hermes/profiles/<name>/skills/`); `/reload-skills` or a new session loads it — the current session's loader is cached at start.
- Publish for anyone: `hermes skills publish <skill-dir>` → registry; others `hermes skills install <id>`.
- Or host the skill dir in a GitHub repo and share the direct URL: `hermes skills install https://raw.githubusercontent.com/<org>/<repo>/main/<skill>/SKILL.md`. A whole repo can be added as a source: `hermes skills tap add <org>/<repo>`.
- Keep finished diagrams as project artifacts (next to the doc they illustrate), not inside the skill.