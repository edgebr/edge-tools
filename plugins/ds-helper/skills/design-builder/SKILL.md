---
name: design-builder
description: "Creates/edits a Figma screen bound STRICTLY to a design system (the active profile's library, linked on the page). Everything in the design — components, color, spacing, typography, radius — must be an instance/variable from that design-system library; nothing is created from scratch. Locked workflow: DISCOVERY (confirm the library and enumerate what exists) → MAPPING (the dev approves the element-to-DS map before drawing) → EXECUTION → AUDIT (prove node by node that everything traces back to the design system). If something requested does not exist in the design system, STOP and report the gap — do not improvise. Use when the dev asks to create, mock up, generate, or edit a screen/component in Figma using the design system. Do NOT confuse with code-vs-design verification (that is the figma-fidelity skill, if the project has it)."
preferred_model: sonnet
---

# design-builder

Generates or edits a design in Figma **bound to a design-system library** already linked on the
page. The rule the skill is named after: **the design system is the single, absolute source of
truth**. No component is created from scratch, no color/measurement is hardcoded, no new style or
variable is invented — everything is a **component instance** and a **variable/token** from the
design-system library.

What guarantees this is not repeating "use only the design system": it is the **locked workflow**
below. Prohibition phrases don't stop the model from pasting a look-alike `#hex` or detaching an
instance — what stops it is forcing discovery before drawing, an approval gate in the middle, and a
verifiable audit at the end.

> **Relationship with `figma-fidelity`** (if the project has it): that one *verifies* whether the
> code uses the same tokens as the design; this one *produces* the design in Figma. One feeds the
> other — the screen created here becomes the reference `figma-fidelity` captures later. They are
> complementary, not overlapping.

## Active profile (resolve before anything)

This skill is **design-system-agnostic**. Everything design-system-specific — display name, token
prefix, naming convention, spacing scale, breakpoints, live sources — comes from a **profile**
bundled with the plugin. Resolve it first:

1. If the consuming repo has a `ds-helper.json` with a `"profile"` field → use that profile.
2. Else if exactly one profile exists under `${CLAUDE_PLUGIN_ROOT}/profiles/` → use it.
3. Else → **ask the dev** which profile to use.

Load `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/profile.json`. Throughout this skill: "the design
system" = `profile.displayName`; token prefix = `profile.tokenPrefix`; naming convention =
`profile.namingConvention`; spacing scale = `profile.spacingScale`; breakpoints =
`profile.breakpoints`; token allowlist = `profile.tokens.names`; live sources = `profile.sources.*`.
**Never hardcode a specific design system in this skill.**
> Example (Dell profile): displayName `Dell Design System v3`, tokenPrefix `--dds-`, convention
> `--dds-{category}-{type}-{variant}--{state}`, spacing base 4px, breakpoints XS 320 · S 480 ·
> M 768 · L 1024 · XL 1366.

## When to invoke

- Dev/designer asks to **create, mock up, generate, or draw** a screen, flow, or component in
  Figma using the design system.
- A request to **edit/update** an existing Figma screen while keeping design-system adherence.
- A request to turn a description, a wireframe, or a code screen into a Figma design **with the
  real design-system components**.

## When NOT to invoke

- **Verifying** conformance of already-implemented code against the design → that is
  `figma-fidelity`.
- Reading/extracting a design from Figma to generate code → use `figma-to-code`, or the Figma MCP
  read tools directly (`get_design_context`), without this skill.
- Architecture/UX discussion with no intent to produce a Figma design right now.
- The design-system library is **not** linked on the target page (see Prerequisites — don't draw).

## Prerequisites (check before anything)

1. **Design-system library linked and enabled** on the target page/file. Confirmed in step 1
   (DISCOVERY), not assumed.
2. **The `/figma-use` skill is MANDATORY before any `use_figma`** (a requirement of the Figma MCP
   itself). Load it first; if the Figma plugin isn't installed, use the fallback
   `skill://figma/figma-use/SKILL.md`. Do not call `use_figma` without it.
3. **The target Figma URL/node** (where the screen will be created/edited). If the dev didn't
   provide it, ask before drawing.

## Procedure — locked workflow

Run **in this order**. Don't skip a step. Steps 1 and 2 come **before** any stroke in Figma.

### 0. Screen description

Confirm you have a concrete description of what to draw (purpose, sections, fields, states,
target breakpoint). If the request came as "create a screen with this description" but the
description isn't present, **stop and ask** — with no description there is nothing to draw.

### 1. DISCOVERY (prove what exists in the design system, before drawing)

1. `get_libraries` → confirm that the **profile's design-system library** is linked **and enabled**
   on the page. **If it isn't: STOP and tell the dev. Don't draw anything** (not even as a
   "provisional" draft).
2. `search_design_system` for each component the screen needs (button, input, header, card,
   modal, table…). List the **exact names** found in the library. Watch for components that may
   have no implementation in the profile's code target — confirm before planning a screen that
   depends on them.
3. `get_variable_defs` → list the **real tokens/variables** for color, spacing, typography, and
   radius the screen will use. Cross-check against the profile's allowlist (`profile.tokens.names`)
   and the profile's naming convention (`profile.namingConvention`).

Rule for this step: in the design you may only use **what appeared** in the results of
`search_design_system` and `get_variable_defs`. Nothing beyond that.

### 2. MAPPING + approval gate (show before applying)

Build and **present to the dev** an element → design-system-resource map, before touching Figma:

| Screen element | DS component (instance) | Tokens (color / spacing / typography / radius) |
|---|---|---|
| e.g. primary button "Save" | `Button` (variant primary) | brand background token, spacing `md`, type ramp |

- If **any element of the description does not exist in the design system**, **STOP and report the
  gap** — don't invent, don't create a new component/style/variable, don't approximate with a local
  shape+style. Wait for the dev's decision (swap for another DS component, adjust the scope, or
  formally accept the exception).
- Layout must use **auto-layout** with the profile's spacing scale and the profile's breakpoints
  (mobile-first).
- Only move to execution with **explicit confirmation** from the dev. Silence or a vague answer
  doesn't count — ask again.

### 3. EXECUTION (draw, always via the library)

With `/figma-use` already loaded, use `use_figma` to build the screen, honoring:

- **Every component = an instance of the design-system library.** Forbidden: detaching an instance,
  creating a new component, creating a new style/variable.
- **Every visual property = a design-system variable/token** — color, spacing, radius, typography
  all bound. **Zero hardcode**: no `#hex`, no `rgb()`, no `px` outside the profile's spacing scale.
- **Configure via the component's props/variants**; never apply an override that breaks the bond
  to the token or to the instance.
- Typography: the profile's family and ramp, via token — don't hardcode size/weight/font.

### 4. FINAL AUDIT (mandatory — prove, don't assert)

After drawing, **re-read** the created nodes in Figma (`get_metadata` / `get_variable_defs` /
`get_design_context` on the screen node; `get_screenshot` for the visual) and produce a
node-by-node report:

- Is it an **instance** of a design-system component? (yes/no + component name)
- Are all visual properties **bound to design-system variables**? (yes/no)
- Flag **any** detached node, hardcoded color/measurement, local style, or variable outside the
  allowlist.

**Verdict:** only conclude "conformant" if **all** nodes pass. On any "no", fix it in Figma or
report to the dev — **don't close with a silent pending item**.

## Hard rules

- When in doubt between **improvising** and **stopping**, **STOP and ask**. Improvising is failure
  mode number one for this skill.
- Design-system library not linked/enabled → draw nothing.
- `/figma-use` not loaded → don't call `use_figma`.
- No configuration, component, style, or variable created from scratch. The source is the design
  system (the active profile), period.

## Anti-patterns

- ❌ Calling `use_figma` without loading `/figma-use` first.
- ❌ Drawing without confirming via `get_libraries` that the design-system library is linked.
- ❌ Skipping the mapping gate and going straight to creating nodes in Figma.
- ❌ Improvising a component/style/variable when the design system lacks what the description asks
  for (the right move is to STOP and report the gap).
- ❌ Detaching an instance, an override that breaks the bond to the token, or a local style "just
  to make it look similar".
- ❌ Magic `#hex`/`rgb()`/`px` instead of a design-system variable.
- ❌ Closing without the step-4 audit — asserting "I used only the design system" without re-reading
  the nodes and proving it.

## Dependencies

- **Figma MCP** (required): `get_libraries`, `search_design_system`, `get_variable_defs`,
  `use_figma`, `get_metadata`, `get_design_context`, `get_screenshot`.
  > Tool names in the environment come prefixed with the Figma MCP server id
  > (`mcp__<server-id>__get_libraries`, etc.). That id varies per installation — discover the
  > real tools of the Figma server in the session instead of assuming a fixed prefix.
- **The `/figma-use` skill** (mandatory before `use_figma`). Fallback:
  `skill://figma/figma-use/SKILL.md`.
- **Profile (bundled, shared across skills):** `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/` — the
  `profile.json`, the token allowlist (`profile.tokens.names`), and token values
  (`profile.tokens.css`, used by `figma-fidelity`). Versioned with the plugin, so the skill works
  in any repo (not tied to a specific project).
- **Live canonical design-system source:** the profile's `sources.storybook` (real component API)
  and `sources.docs` (tokens/foundations). Use when `search_design_system`/`get_variable_defs`
  aren't enough for the exact name/variant.
- **If the consuming project has local design-system knowledge** (e.g. a Tour pattern in
  `.agent/patterns/`), read it too — it reflects that repo's decisions. But the skill **does not
  depend** on it to work.

## How to validate the skill (evidence)

1. Page with the profile's library linked + a simple description (e.g. a card with title, text,
   and button) → run the flow. Step 1 should list the real `Card`/`Button`; step 4 should close
   everything bound.
2. Ask in the description for a component the design system **doesn't have** → step 2 must **STOP
   and report the gap**, not improvise.
3. Run on a page **without** the library linked → it must stop at the prerequisite/step 1, without
   drawing.
4. Mentally inject a `#hex` in the mapping → step 4 must catch it as non-conformant.

## Recommended model

Sonnet covers the orchestration (discovery + mapping + audit are procedural, backed by the MCP
tools). Consider Opus only for large screens with genuinely ambiguous layout, where choosing among
multiple design-system components and composing the auto-layout demand more reasoning.
