---
name: figma-fidelity
description: "Verifies design-system token conformance (color, spacing, typography) between the implemented UI and the Figma design. Two modes: CAPTURE (dev-assisted, produces a committed snapshot of the design) and VERIFY (headless, compares the code against the snapshot — runs in the quality-gate/review). Token prefix and values come from the active profile. It is NOT pixel-perfect: it's token conformance."
preferred_model: sonnet
---

# figma-fidelity

Checks whether the implementation uses the **same design-system tokens** as the Figma design
(color, spacing, typography). It works at the level of **token/value conformance**, not pixel
comparison.

Operating model (snapshot):
- **CAPTURE** (occasional, dev-assisted): when a screen's design stabilizes, extract from Figma
  the tokens it uses and write a **committed snapshot**.
- **VERIFY** (recurring, headless): compare the component's style against the snapshot. This is
  what runs in the quality-gate/review — no Figma open.

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

## Procedure — CAPTURE (dev-assisted)

1. The dev provides the screen's Figma **node URL** (from the activity's design link) and has the
   file open/selected.
2. Extract the tokens the node uses via the Figma MCP (`get_variable_defs`) → a `{ css, value }`
   set.
3. Normalize against the profile's token file (`profile.tokens.css`): resolve alias → value;
   record light/dark mode when applicable.
4. Write the snapshot **co-located** with the component: `<component>.design.json` (committed),
   in the format:
   ```json
   {
     "figma": { "fileKey": "...", "nodeId": "..." },
     "capturedAt": "<ISO>",
     "tokens": [
       { "css": "<prefix>color-text-primary", "type": "COLOR", "value": {"light":"#1d2c3b","dark":"#ebf1f6"} },
       { "css": "<prefix>spacing-lg", "type": "FLOAT", "value": 16 }
     ]
   }
   ```
5. Present to the dev what was captured for **validation** before committing (is the design
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
4. **Verdict:** `pass` if there is no blocking divergence; otherwise `fail` with the list.
5. **Report:** per token — expected (design) vs found (code), with the file/line when possible.

## Scope

- **Checks:** color, spacing, typography (design-system tokens).
- **Does NOT check (v1):** layout arrangement (flex/grid), positioning, pixel-perfect, dynamic
  content.
- **v2 (future):** per-property binding (via `get_design_context`), and checking the rendered
  **computed value** (Playwright) against the design.

## Flow integration

- Runs as **one quality-gate / review check** on UI work. It complements — does not replace — a
  deterministic linter (which blocks literals) and the project's design-system pattern (knowledge).
- The snapshot is a **committed** artifact of the activity; verification is **deterministic** and
  runs without Figma.

## Anti-patterns

- ❌ Returning "conformant" with no snapshot. It must warn that there is **no reference design**.
- ❌ Using screenshot/pixel-diff as a v1 criterion.
- ❌ Freezing token values in the skill instead of resolving via `profile.tokens.css`/snapshot.
- ❌ Capturing a snapshot and committing **without the dev confirming** the design is stable.
- ❌ Treating "extra token in code" as an automatic error (it's a warning, not a block).
- ❌ Flagging a divergence that is only a **mode** difference (light vs dark).

## Dependencies

- **Figma MCP** — CAPTURE mode only.
- **Profile (bundled, shared):** `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/tokens/` —
  `profile.tokens.css` (+ `profile.tokens.names`) to resolve alias→value offline. Same source as
  `design-builder`/`figma-to-code`.
- Components styled in SCSS/CSS using `profile.tokenPrefix` variables (in the consuming repo).

## How to validate the skill (evidence)

1. Take a component + a design-system node; run CAPTURE → generate the `<comp>.design.json`.
2. Run VERIFY on **clean** code → `pass`.
3. Introduce a deliberate deviation (`#hex` literal or a swapped `spacing`) → VERIFY must **catch**
   it and `fail`, pointing at the right token.
4. Rename/remove the snapshot → VERIFY must **warn "no reference design"**, not pass.

## Recommended model

Sonnet (orchestration + parsing). The Figma extraction (CAPTURE) uses the MCP.
