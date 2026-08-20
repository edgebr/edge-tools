---
name: figma-fidelity
description: "Verifies design-system token conformance (color, spacing, typography) AND rendered box geometry (position/size of key elements) between the implemented UI and the Figma design. Two modes: CAPTURE (dev-assisted, produces a committed snapshot of the design's tokens and layout boxes) and VERIFY (headless, compares the code against the snapshot — runs in the quality-gate/review). Token prefix and values come from the active profile. It is NOT pixel-perfect image diffing: it's token conformance plus box-level layout conformance."
preferred_model: sonnet
---

# figma-fidelity

Checks whether the implementation uses the **same design-system tokens** as the Figma design
(color, spacing, typography), **and** whether the rendered layout occupies the **same boxes**
(position/size of each mapped element) that the Figma auto-layout produces. It works at the level
of **token/value conformance plus box-geometry conformance** — not full pixel/image comparison.

Token conformance alone is not sufficient: a component can use every correct token and still
render with a visibly different layout, because CSS box-model defaults (flex `stretch`, native
element margins, etc.) sit *between* "tokens are bound correctly" and "the box ends up the right
size and place". Catching that requires comparing actual rendered geometry, not just which
variables were referenced — see CAPTURE/VERIFY step on box geometry below.

Operating model (snapshot):
- **CAPTURE** (occasional, dev-assisted): when a screen's design stabilizes, extract from Figma
  the tokens it uses **and** the bounding box (x/y/width/height) of each mapped element, and write
  a **committed snapshot**.
- **VERIFY** (recurring, headless): compare the component's style **and** its rendered bounding
  boxes against the snapshot. This is what runs in the quality-gate/review — no Figma open.

## Active profile (resolve before anything)

This skill is **design-system-agnostic**. The token prefix and the token values/allowlist come from
a **profile** bundled with the plugin. Resolve it first:

1. If the consuming repo has a `ds-helper.json` with a `"profile"` field → use that profile.
2. Else if exactly one profile exists under `${CLAUDE_PLUGIN_ROOT}/profiles/` → use it.
3. Else → **ask the dev** which profile to use.

Load `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/profile.json`. Token prefix = `profile.tokenPrefix`;
values source = `profile.tokens.css` (resolve alias→value offline); allowlist =
`profile.tokens.names`. **Do not** freeze token values in the skill — resolve them via the
profile's token file or the snapshot.
> Example (Dell profile): token prefix `--dds-`, values from the profile's `tokens/tokens.css`
> (a snapshot of the `@dds/angular` v3 tokens).

## When to invoke

- **UI** work that has a Figma design — at the end of IMPLEMENT / inside the quality-gate / at
  review.
- When a screen's design **stabilizes** → run CAPTURE mode (or update the snapshot).

## When NOT to invoke

- Non-UI work (logic, backend, config).
- A screen **without** a Figma design (nothing to compare against — see anti-patterns).
- As a linter substitute: a deterministic linter already blocks literals; this skill checks
  **adherence to the design**.

## Before comparing anything: confirm the runtime mode/theme is pinned

Skip straight to CAPTURE/VERIFY only after confirming the consuming repo pins a theme (see
`figma-to-code`'s prerequisite step 5 — per profile, e.g. `dds--light-mode` on the document root
for Dell). If it doesn't, **both modes are "correct" simultaneously** and any comparison you run
depends on whichever mode the runtime happens to resolve to (OS/browser preference) — a mismatch
found this way may be a real bug, or it may just mean you ran VERIFY under the mode the snapshot
wasn't captured in. If the theme isn't pinned:
- Don't report a fail based on the observed mismatch alone — first check whether forcing the
  snapshot's captured mode makes the comparison pass.
- Flag the missing pin itself as a project-level gap (separately from any component-level finding)
  so it gets fixed once, not rediscovered per screen.

## Procedure — CAPTURE (dev-assisted)

1. The dev provides the screen's Figma **node URL** (from the activity's design link) and has the
   file open/selected.
2. Extract the tokens the node uses via the Figma MCP (`get_variable_defs`) → a `{ css, value }`
   set.
3. Extract the **bounding box of each mapped element** via `get_metadata` on the same node (x, y,
   width, height, in the node's local coordinate space — i.e. relative to the top-level frame
   being captured, not absolute canvas coordinates). Capture this for every element that has its
   own code counterpart (the container, and each direct child worth asserting on) — not just leaf
   text nodes. **Always record `x`/`y` alongside `w`/`h` when the element's placement relative to
   its siblings is part of the design intent** (e.g. a button group meant to sit right-aligned in
   a footer, not just sized correctly) — recording only `w`/`h` lets a left-vs-right or
   top-vs-bottom placement bug through even when the exploratory measurement already showed the
   real position, simply because that position was never written into the comparison point.
   For a **text-bearing element inside a container whose own size the library pins independent of
   content** (e.g. a header row whose `flex-basis` matches a sibling icon button, so it stays the
   same height whether the text is 16px or 24px) — also record the expected `fontSize`,
   `fontWeight`, and `lineHeight` on that box entry (see schema below) and note in a comment which
   container this applies to. The outer box will pass geometry comparison either way; only a
   computed-style check on the text node itself catches a typography regression there.
4. Normalize tokens against the profile's token file (`profile.tokens.css`): resolve alias → value;
   record light/dark mode when applicable.
5. Write the snapshot **co-located** with the component: `<component>.design.json` (committed),
   in the format:
   ```json
   {
     "figma": { "fileKey": "...", "nodeId": "..." },
     "capturedAt": "<ISO>",
     "tokens": [
       { "css": "<prefix>color-text-primary", "type": "COLOR", "value": {"light":"#1d2c3b","dark":"#ebf1f6"} },
       { "css": "<prefix>spacing-lg", "type": "FLOAT", "value": 16 }
     ],
     "boxes": [
       { "selector": ".admin-availability__header", "x": 0, "y": 0, "w": 1366, "h": 188 },
       { "selector": ".admin-availability__header [ddsTag]", "x": 48, "y": 32, "w": 102, "h": 32 },
       {
         "selector": "dds-modal-header h2",
         "x": 24, "y": 24, "w": 704, "h": 32,
         "fontSize": 24, "fontWeight": 400, "lineHeight": 32,
         "note": "Container height is pinned by the library to match the close button (32px) regardless of the text's font-size — box geometry alone can't catch a typography regression here; fontSize/fontWeight/lineHeight are compared via getComputedStyle in VERIFY step 4b."
       }
     ]
   }
   ```
   `selector` is the CSS selector in the consuming repo that identifies the equivalent element —
   record it while the mapping is fresh (right after `figma-to-code` or manual implementation),
   don't reverse-engineer it later during VERIFY. `fontSize`/`fontWeight`/`lineHeight` are optional
   — add them only for a text-bearing box whose container size doesn't reflect its own typography
   (see the note above); most `boxes` entries don't need them, since a wrong font-size on unpinned
   content usually already shows up as a wrong `h`.
6. Present to the dev what was captured for **validation** before committing (is the design
   stable?).

## Procedure — VERIFY (headless)

1. Locate the snapshot `<component>.design.json`. **No snapshot → don't fail; record "no reference
   design" and exit** (so it doesn't give a false "conformant").
2. Collect the tokens the component uses: parse the SCSS/CSS/template for `profile.tokenPrefix`
   variables **and** for **literals** (`#hex`, `rgb()`, `px` outside the profile's spacing scale).
3. Compare (set level):
   - **Literal in code** where the design expects a design-system token → **blocking divergence**.
   - **Snapshot token missing** in the code → **warning** (design expects it, code didn't apply).
   - **Token in code outside the snapshot** → **warning** (may be a legitimate extra; doesn't
     block on its own).
   - Resolve values via `profile.tokens.css` and respect **mode** (don't flag a difference that is
     only light vs dark).
4. **Box geometry:** for each `boxes` entry in the snapshot, render the component (or use an
   already-running dev server) and read the real rendered box for that `selector`
   (`getBoundingClientRect()`, e.g. via the project's browser tooling). Compare against the
   snapshot's `x/y/w/h` with a small tolerance (a few px, for subpixel/font-rendering rounding —
   not for real drift):
   - A rendered box outside tolerance on **width or height** → **blocking divergence**. This is
     what catches a chip/badge stretched to fill its container (missing `align-items`), or a block
     taller than expected (unreset native margins stacking on top of a flex `gap`) — cases where
     every individual token is correct but the box model isn't.
   - A rendered box outside tolerance on **position only** (correct size, shifted x/y), where the
     parent chain itself has no divergence → usually downstream of a sibling's size divergence
     above it; report it but don't double-count it as a separate root cause.
   - Skip this step (report "no geometry reference", not a false pass) if the snapshot has no
     `boxes`, or if the theme/mode isn't pinned (see the mode-pinning check above) and forcing the
     snapshot's mode isn't feasible in this run.
   - A `boxes` entry that records `x`/`y` and the rendered element sits at a **different position
     than expected while its own `w`/`h` are correct** → **blocking divergence**, even if nothing
     else in the box looks wrong. This is what catches a right-aligned button group rendering
     flush-left (or vice versa) — the group's own size can be exactly right while its position
     relative to its container is not; don't skip comparing `x`/`y` just because `w`/`h` passed.
4b. **Typography-on-fixed-container check:** for any `boxes` entry that also records `fontSize`/
    `fontWeight`/`lineHeight`, read the real computed style of that exact selector
    (`getComputedStyle(el).fontSize` etc.) and compare against those values, **independently of
    the box-geometry comparison**. A container whose own size the library pins to a sibling (e.g.
    a header row sized to match its close button) will pass the box check at any font-size — this
    step exists specifically to catch what that check structurally cannot.
5. **Verdict:** `pass` if there is no blocking divergence (token, geometry, or typography);
   otherwise `fail` with the list.
6. **Report:** per token — expected (design) vs found (code); per box — expected vs rendered
   x/y/w/h and which CSS property is the likely cause (missing `align-items`/`justify-content`,
   unreset margin, wrong `width`, etc.); per typography check — expected vs computed
   fontSize/fontWeight/lineHeight — with the file/line when possible.

## Scope

- **Checks:** color, spacing, typography (design-system tokens); rendered box geometry
  (position **and** size) of elements recorded in the snapshot's `boxes`; computed
  fontSize/fontWeight/lineHeight of any `boxes` entry that records them (for text inside a
  library-pinned container, where box geometry alone can't reveal a typography regression).
- **Does NOT check:** pixel/image-level comparison (font antialiasing, exact curve rendering,
  gradients), dynamic content, layout arrangement beyond the boxes explicitly captured (e.g. it
  won't notice a reordered child unless that child's own box is in `boxes`).
- **v2 (future):** per-property binding (via `get_design_context`) for a fuller token↔property
  trace, and full computed-style diffing (not just the box-affecting subset) via a headless
  browser driver.

## Flow integration

- Runs as **one quality-gate / review check** on UI work. It complements — does not replace — a
  deterministic linter (which blocks literals) and the project's design-system pattern (knowledge).
- The snapshot is a **committed** artifact of the activity; verification is **deterministic** and
  runs without Figma.

## Anti-patterns

- ❌ Returning "conformant" with no snapshot. It must warn that there is **no reference design**.
- ❌ Using full screenshot/image-diff as a criterion — box geometry is compared via
  `getBoundingClientRect()`-style numbers, not pixel images.
- ❌ Freezing token values in the skill instead of resolving via `profile.tokens.css`/snapshot.
- ❌ Capturing a snapshot and committing **without the dev confirming** the design is stable.
- ❌ Treating "extra token in code" as an automatic error (it's a warning, not a block).
- ❌ Flagging a divergence that is only a **mode** difference (light vs dark).
- ❌ Declaring "conformant" from token checks alone when the snapshot has `boxes` recorded — a
  token-only pass with an untested box divergence is a false pass (this is exactly the gap that
  let a stretched chip and a doubled-height header through as "no divergence found").
- ❌ Reporting a mode-driven mismatch as a component bug without first checking whether the
  runtime's theme is pinned at all (see the mode-pinning check).
- ❌ Recording only `w`/`h` for an element whose alignment relative to its siblings is part of
  the design intent — this silently drops the one comparison point that would have caught a
  left-vs-right (or top-vs-bottom) placement bug, even when the exploratory measurement already
  had the real `x`/`y` in hand. If you measured the position, write it into the snapshot.
- ❌ Trusting a container's box geometry as proof that its content's typography is correct when
  the library pins that container's own size independent of content (e.g. `flex-basis` matched to
  a sibling icon/button). The box passes at any font-size; only a computed-style check on the text
  node itself (fontSize/fontWeight/lineHeight) catches a regression there.

## Dependencies

- **Figma MCP** — CAPTURE mode only (`get_variable_defs` for tokens, `get_metadata` for boxes).
- **Profile (bundled, shared):** `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/tokens/` —
  `profile.tokens.css` (+ `profile.tokens.names`) to resolve alias→value offline. Same source as
  `design-builder`/`figma-to-code`.
- Components styled in SCSS/CSS using `profile.tokenPrefix` variables (in the consuming repo).
- **A running instance of the component** (dev server, or the project's browser/preview tooling) —
  VERIFY's box-geometry step needs a real rendered DOM to read `getBoundingClientRect()` from; it
  can't be done from static source alone the way the token check can.

## How to validate the skill (evidence)

1. Take a component + a design-system node; run CAPTURE → generate the `<comp>.design.json` with
   both `tokens` and `boxes`.
2. Run VERIFY on **clean** code → `pass` (tokens and geometry both within tolerance).
3. Introduce a deliberate token deviation (`#hex` literal or a swapped `spacing`) → VERIFY must
   **catch** it and `fail`, pointing at the right token.
4. Introduce a deliberate **geometry** deviation with all tokens still correct (e.g. remove
   `align-items: flex-start` so a child stretches full-width, or remove a `margin: 0` reset so a
   heading gains extra height) → VERIFY must **catch** it via the box comparison and `fail`,
   naming the affected selector and the likely CSS cause — this is the case a token-only check
   silently passes.
5. Rename/remove the snapshot → VERIFY must **warn "no reference design"**, not pass.
6. Run VERIFY under the mode opposite the snapshot's captured mode, on an unpinned project →
   VERIFY must flag the missing theme pin rather than reporting the mode-driven color/box diffs as
   component-level fidelity bugs.
7. Take an element meant to be right-aligned in its container (all tokens correct, box `w`/`h`
   correct) and flip its CSS to `justify-content: flex-start` (or drop the alignment rule
   entirely) → VERIFY must **catch** it via the `x` comparison and `fail`, naming the selector —
   this is the case a `w`/`h`-only snapshot entry passes silently even though the position is
   wrong.
8. Take a text element inside a container whose size the library pins independent of content
   (e.g. a header row sized to match a sibling icon button), and remove its heading class so it
   renders as plain body text instead of the intended larger/bolder style → VERIFY must **catch**
   it via the fontSize/fontWeight computed-style check (step 4b) and `fail`, even though the
   container's own box stays within tolerance — this is the case a geometry-only check on the
   container passes silently.

## Recommended model

Sonnet (orchestration + parsing). The Figma extraction (CAPTURE) uses the MCP.
