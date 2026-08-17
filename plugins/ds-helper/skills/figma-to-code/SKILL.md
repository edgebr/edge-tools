---
name: figma-to-code
description: "During IMPLEMENT, translates a Figma design node into design-system-bound code. Extracts the components and tokens the node uses (get_design_context / get_variable_defs), maps each Figma component to its code counterpart via Code Connect (get_code_connect_map), and scaffolds the component already importing the design-system package and binding its tokens — modern framework idioms, zero literals. Framework, package and token prefix come from the active profile. Only real design-system components/tokens; if Code Connect has no mapping for a component, STOP and ask instead of guessing. Use when a dev is implementing UI that has a Figma design and wants the design-system-correct scaffold. This WRITES code in the consuming repo (dev-facing) — the counterpart of design-builder (which writes the design in Figma) and figma-fidelity (which verifies code vs design)."
preferred_model: sonnet
---

# figma-to-code

Turns a **Figma design node into design-system-bound code**. It is the active bridge in the
IMPLEMENT phase: instead of leaving the dev to translate the design by hand (and only catching
mistakes later in review), it reads the design, maps it to the real design-system components via
**Code Connect**, and scaffolds code that already imports the right components and binds the right
tokens.

It is the middle link of the trio — and only makes sense **after** the design exists and the plan
is approved:

> `design-builder` (creates the design in Figma) → **`figma-to-code`** (design → bound code) →
> `figma-fidelity` (verifies code vs design). This skill **writes code in the consuming repo**;
> the other two write to Figma / read the code.

It assists the scaffold — it does **not** invent business logic, and it does **not** replace the
project's design-system coding rules; it applies them.

## Active profile (resolve before anything)

This skill is **design-system-agnostic**. The framework, package name, token prefix, allowlist and
live sources come from a **profile** bundled with the plugin. Resolve it first:

1. If the consuming repo has a `ds-helper.json` with a `"profile"` field → use that profile.
2. Else if exactly one profile exists under `${CLAUDE_PLUGIN_ROOT}/profiles/` → use it.
3. Else → **ask the dev** which profile to use.

Load `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/profile.json`. Throughout this skill: framework =
`profile.code.framework`; design-system package = `profile.code.package`; token prefix =
`profile.tokenPrefix`; token allowlist = `profile.tokens.names`; live component API =
`profile.sources.storybook`. **Never hardcode a specific design system or framework here.**
> Example (Dell profile): framework `angular`, package `@dds/angular`, token prefix `--dds-`,
> Storybook `https://angular.delldesignsystem.com`. Under this profile the scaffold is standalone
> Angular with native control flow (`@for`/`@if`) and `signal()`.

## When to invoke

- A dev is in IMPLEMENT on a UI task that **has a Figma design node**, and wants the
  design-system-correct scaffold (component skeleton, imports, token bindings) instead of writing
  it by hand.
- Turning a specific Figma frame/component into a design-system-based component or template.

## When NOT to invoke

- No Figma design exists for the screen → there is nothing to translate. Build from the project's
  design-system pattern by hand, or create the design first with `design-builder`.
- **Verifying** already-written code against the design → that is `figma-fidelity`.
- Non-UI work (logic, backend, config).
- The plan isn't approved yet (in a Tour flow, IMPLEMENT comes after APPROVAL — don't scaffold
  implementation code ahead of the gate).

## Prerequisites (check before anything)

1. **The target Figma node URL** (the activity's design link). If the dev didn't provide it, ask.
2. **Figma MCP** connected (`get_design_context`, `get_variable_defs`, `get_code_connect_map`,
   `get_metadata`, `get_screenshot`).
3. **The profile's design-system package installed** in the consuming repo, with the design-system
   tokens imported. Confirm before binding `profile.tokenPrefix` variables.
4. Read the project's local design-system knowledge if present (e.g. a Tour pattern in
   `.agent/patterns/`); otherwise rely on the profile's bundled allowlist + the live Storybook.
5. **Theme/mode is pinned in the consuming repo.** Many design systems resolve semantic tokens via
   `prefers-color-scheme` when no explicit theme class is set (e.g. `@dds/angular` on `--dds-*`
   tokens). Figma captures are single-mode (usually light). If the app's root (`index.html` /
   root layout) has no theme-locking class/attribute, the exact same token-correct code renders
   with completely different resolved colors depending on the OS/browser color scheme — a false
   "it doesn't match the design" that isn't actually a code bug. Check once per project
   (skip if already confirmed this session):
   - Look for a theme class/attribute on the document root (per profile — Dell: `dds--light-mode`
     / `dds--dark-mode` on `<html>`/`<body>`).
   - If missing → **stop and ask the dev** whether to pin a theme (and which one) before
     scaffolding, rather than silently scaffolding on top of an unpinned theme and letting the
     mismatch surface later as a "fidelity bug".

## Procedure — locked workflow

Run **in this order**. Steps 1–2 (read + map) come **before** writing any code.

### 1. READ the design node

1. `get_design_context` on the node → the structure and the design-system components it uses.
2. `get_variable_defs` → the exact tokens (color, spacing, typography, radius) the node binds.
3. `get_screenshot` (optional) → visual reference for the dev.

Cross-check the tokens against the profile's allowlist (`profile.tokens.names`) and the naming
convention (`profile.namingConvention`).

### 2. MAP Figma → design-system code (Code Connect)

For each design-system component in the node:

1. `get_code_connect_map` (and `get_code_connect_suggestions` when needed) → the mapping from the
   Figma component to its code component (from `profile.code.package`) + props.
2. If Code Connect **has a mapping** → use it verbatim (component name, prop/variant bindings).
3. If Code Connect has **no mapping** for a component → **STOP and ask**. Don't guess a component
   from the name. Options to offer the dev: pick the right design-system component together
   (confirm via Storybook), adjust the design, or flag that Code Connect needs the mapping added.

**Present the element → code map to the dev before writing code.** This is the gate: a
missing/ambiguous mapping is resolved here, not improvised in the scaffold.

### 3. SCAFFOLD the code

Generate the component honoring the profile's coding rules — no exceptions:

- **Import design-system components from `profile.code.package`**; never recreate a component that
  exists in the design system.
- **Bind tokens, never literals** — every color/spacing/radius/typography goes through a
  `profile.tokenPrefix` variable. Zero `#hex`, zero `rgb()`, zero magic `px` (profile spacing
  scale).
- **Modern framework idioms** for `profile.code.framework`. (Dell/Angular: native control flow
  `@for`/`@if`, standalone `imports`, `signal()`; no `*ngFor`/`*ngIf`/`CommonModule` for flow.)
- **Configure via props/variants**; never override a design-system component's internal style.
- Leave business logic as clearly-marked stubs (`/* business logic */`) — the skill scaffolds
  structure + design-system wiring, not behavior.
- Confirm exact selector/inputs/outputs against the profile's **Storybook**
  (`profile.sources.storybook`) — this skill does not freeze the component API.
- **Match the Figma auto-layout's box behavior, not just its tokens.** When a Figma auto-layout
  frame's children are scaffolded as native block elements (`h1`, `p`, headings/paragraphs bound
  to design-system typography utility classes) inside a flex/grid container:
  - Design-system typography utility classes (e.g. `dds__heading--2`, `dds__subtitle--1`) commonly
    set only `font-*`/`line-height` — they do **not** reset the browser's default element margins.
    Explicitly zero the margin on those elements (or on the container's children generically) so
    the container's own `gap`/token-bound spacing is the only source of vertical spacing. Otherwise
    UA margins stack on top of the intended gap and the block ends up visibly taller than the
    design, even though every spacing token used is individually correct.
  - Set the container's cross-axis alignment (`align-items`) to match the Figma auto-layout's
    alignment (e.g. "Hug contents" / left-aligned → `align-items: flex-start`) instead of leaving
    the flexbox default (`stretch`). A `stretch` default silently expands non-full-width children
    (e.g. a tag/chip/badge with no intrinsic width) to fill the container's cross-axis, distorting
    it into a full-width bar that reads as a completely different visual result even though its own
    tokens (color, padding, radius) are all correct.
- **Cross-check the resolved component-variant token against the Figma capture.** After picking a
  design-system component variant via Code Connect (step 2), diff the variant's actual bound
  tokens (readable from the installed package's compiled CSS/source, e.g. which
  `--<prefix>-color-*` variable a `--variant` class hardcodes) against the tokens `get_variable_defs`
  returned for that exact Figma instance. If they resolve to different tokens (e.g. the design uses
  a `-pale` border token but the shipped component's variant hardcodes a `-subtle` one), that is a
  **design-system/library drift**, not a scaffolding mistake — do not invent an inline override to
  force pixel-match. Surface it to the dev as a named gap (which token differs, in which variant)
  and let them decide: accept the library's shipped value as source of truth, or file it as a
  design-system defect upstream. This mirrors `design-builder`'s "stop and report the gap" rule,
  applied at code-generation time instead of design time.

In a Tour flow: scaffold only what the **approved plan** covers. If the design implies work
outside the plan, stop and ask for re-approval of that slice (Tour's rule).

### 4. HAND OFF to verification

Immediately after scaffolding, **invoke `figma-fidelity` in VERIFY mode automatically** against
the just-scaffolded code — don't wait for the dev to ask or for the quality-gate/REVIEW to run it.
Do this without asking for confirmation first; VERIFY is headless and read-only (no Figma file
state, no commit), so there's nothing to gate.

- If VERIFY comes back **pass** → say so briefly and move on; no further action needed.
- If VERIFY comes back **fail** → fix the flagged token/geometry divergence in the scaffold before
  considering this step done (this closes the loop this skill's own anti-patterns and hard rules
  set up — e.g. an unreset margin or a `stretch`-ed child, see step 3) — then re-run VERIFY once.
  Don't loop indefinitely on a genuine design-system/library drift (see step 3's variant-drift
  case): surface that to the dev instead of retrying.
- If VERIFY reports **"no reference design"** (no snapshot yet) → that's expected for a first
  scaffold. Tell the dev the next step is `figma-fidelity` **CAPTURE** once the design is stable —
  CAPTURE stays dev-assisted and requires their confirmation before writing/committing the
  snapshot (it needs the Figma file open/selected, and shouldn't freeze a design that might still
  change). Don't run CAPTURE automatically or without that confirmation.

`figma-to-code` gets the code design-system-correct at authoring time; `figma-fidelity` proves it
stayed that way — automatically for VERIFY, dev-gated for CAPTURE. A deterministic linter (e.g.
`stylelint`) still blocks literals — this skill complements it.

## Hard rules

- No mapping in Code Connect and no confident design-system component → **STOP and ask**. Never
  guess a component to keep the scaffold moving.
- Never emit a literal where a design-system token exists.
- Never recreate a component the design system already provides.
- Don't write implementation code ahead of an approved plan (Tour: IMPLEMENT follows APPROVAL).
- Don't freeze/guess the component API here — confirm it in the Storybook.
- Always run `figma-fidelity` VERIFY right after scaffolding — don't leave it for the dev to
  remember or for REVIEW to catch. Never auto-run `figma-fidelity` CAPTURE — that stays dev-gated.

## Anti-patterns

- ❌ Inferring a design-system component from a Figma layer name when Code Connect has no mapping.
- ❌ Scaffolding with `#hex`/`rgb()`/magic `px` instead of a `profile.tokenPrefix` variable.
- ❌ Legacy control-flow idioms when the framework has modern ones (Angular: `*ngFor`/`*ngIf`).
- ❌ Recreating a design-system component "because it's simpler than importing it".
- ❌ Overriding a design-system component's internal style to match the design instead of using its
  props/variants.
- ❌ Writing business logic as if it were part of the scaffold.
- ❌ Treating the scaffold as verified — verification is `figma-fidelity`'s job.
- ❌ Scaffolding UI on a project with no pinned theme/mode and assuming the rendered result will
  match a single-mode Figma capture.
- ❌ Leaving native heading/paragraph elements with their browser default margins when the parent
  auto-layout already carries the intended spacing via `gap`.
- ❌ Leaving a flex/grid container's children on the default `stretch` alignment when the Figma
  auto-layout hugs its content — an unstyled child (chip, badge, icon button) will silently expand
  to fill the axis.
- ❌ Silently overriding a design-system component's internal token to force a pixel-match with
  Figma when the shipped variant resolves a different token — that's a drift to report, not patch
  around invisibly.

## Dependencies

- **Figma MCP** (required): `get_design_context`, `get_variable_defs`, `get_code_connect_map`,
  `get_code_connect_suggestions`, `get_metadata`, `get_screenshot`.
  > Tool names come prefixed with the Figma MCP server id (`mcp__<server-id>__…`), which varies
  > per installation — discover the real tools in the session.
- **Code Connect** configured for the design system in Figma — this is what makes the mapping
  reliable. Without it, the skill degrades to name-matching + Storybook confirmation and must flag
  lower confidence (see step 2).
- **Profile (bundled, shared):** `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/` — allowlist
  (`profile.tokens.names`) + values (`profile.tokens.css`), package/framework (`profile.code.*`),
  live sources (`profile.sources.*`).
- **The profile's design-system package** installed in the consuming repo, tokens imported.
- **If the project uses Tour:** the local design-system pattern (`.agent/patterns/`) carries the
  repo's decisions — read it, but the skill doesn't depend on it.

## How to validate the skill (evidence)

1. Figma node with a Button + a Text Input that HAVE Code Connect mappings → scaffold must import
   the mapped design-system components and bind the node's tokens, with no literals.
2. Node with a component that has **no** Code Connect mapping → step 2 must **STOP and ask**, not
   guess.
3. A token in the node that's **not** in the allowlist → flag it instead of emitting it.
4. Run `figma-fidelity` VERIFY on the scaffolded code → should pass (proves the hand-off closes
   the loop).
5. Render the scaffold with the project's theme unpinned (default OS light) and again forced dark
   (or vice versa) → the rendered result must only change if the project intentionally supports
   both modes; an unpinned project must be caught at prerequisite step 5, not discovered here.
6. A container whose Figma auto-layout hugs its content, with a non-full-width child (tag/badge) →
   the scaffolded child must render at its intrinsic width, not stretched to the container's
   cross-axis.

## Recommended model

Sonnet covers the read → map → scaffold flow (procedural, backed by the MCP + Code Connect).
Consider Opus for screens with ambiguous composition or where several design-system components
could fit and the mapping needs more judgment.
