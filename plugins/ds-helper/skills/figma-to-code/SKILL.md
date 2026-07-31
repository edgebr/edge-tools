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

In a Tour flow: scaffold only what the **approved plan** covers. If the design implies work
outside the plan, stop and ask for re-approval of that slice (Tour's rule).

### 4. HAND OFF to verification

After scaffolding, tell the dev the next step is `figma-fidelity`:
- **CAPTURE** a snapshot when the design is stable, and
- **VERIFY** (headless) in REVIEW / the quality-gate.

`figma-to-code` gets the code design-system-correct at authoring time; `figma-fidelity` proves it
stayed that way. A deterministic linter (e.g. `stylelint`) still blocks literals — this skill
complements it.

## Hard rules

- No mapping in Code Connect and no confident design-system component → **STOP and ask**. Never
  guess a component to keep the scaffold moving.
- Never emit a literal where a design-system token exists.
- Never recreate a component the design system already provides.
- Don't write implementation code ahead of an approved plan (Tour: IMPLEMENT follows APPROVAL).
- Don't freeze/guess the component API here — confirm it in the Storybook.

## Anti-patterns

- ❌ Inferring a design-system component from a Figma layer name when Code Connect has no mapping.
- ❌ Scaffolding with `#hex`/`rgb()`/magic `px` instead of a `profile.tokenPrefix` variable.
- ❌ Legacy control-flow idioms when the framework has modern ones (Angular: `*ngFor`/`*ngIf`).
- ❌ Recreating a design-system component "because it's simpler than importing it".
- ❌ Overriding a design-system component's internal style to match the design instead of using its
  props/variants.
- ❌ Writing business logic as if it were part of the scaffold.
- ❌ Treating the scaffold as verified — verification is `figma-fidelity`'s job.

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

## Recommended model

Sonnet covers the read → map → scaffold flow (procedural, backed by the MCP + Code Connect).
Consider Opus for screens with ambiguous composition or where several design-system components
could fit and the mapping needs more judgment.
