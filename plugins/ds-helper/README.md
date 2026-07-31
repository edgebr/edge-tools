# ds-helper — design-system-bound design & development in Figma, via Claude Code

`ds-helper` keeps UI work **bound to a design system** from design to code to review. It bundles
three skills, one per phase of a UI feature — plus a helper to wire them into Tour:

| Phase | Skill | What it does | Mostly used by |
|---|---|---|---|
| Design | `design-builder` | Creates/edits a Figma screen bound **strictly** to the design system (only library instances/variables). | Designers |
| Implement | `figma-to-code` | Turns a Figma design node into **design-system-bound code** — maps components via Code Connect, imports the DS package, binds tokens. | Devs |
| Review | `figma-fidelity` | Verifies the implemented code uses the **same design-system tokens** as the design (capture snapshot → headless verify). | Devs |
| — | `tour-setup` | Wires the suite into projects that use **Tour** (materializes the frontend pattern). | Devs |

## Design-system-agnostic via profiles

The skills carry **no** design system hardcoded. Everything specific — token prefix, naming
convention, code package/framework, Storybook/docs URLs, token allowlist/values, and the Tour
pattern — lives in a **profile** under `profiles/<id>/`. The engine (the locked, verifiable
workflow — discovery → mapping gate → execution → audit — plus Code Connect mapping and the
snapshot verify) is the same for every design system.

**Ships with:** the `dell` profile — Dell Design System v3 (`@dds/angular`, `--dds-*` tokens).

### How a skill resolves the active profile

1. If the consuming repo has a `ds-helper.json` with a `"profile"` field → use it.
2. Else if exactly **one** profile is bundled → use it (so a single-profile install just works).
3. Else → the skill asks which profile to use.

To pin a profile explicitly in a project, add at the repo root:

```json
// ds-helper.json
{ "profile": "dell" }
```

### Adding another design system (later)

Drop a new folder under `profiles/<id>/`:

```
profiles/<id>/
├── profile.json                 # displayName, tokenPrefix, namingConvention, code.{framework,package}, sources.{storybook,docs}, tokens.{css,names}, tour.{pattern,patternName,indexSummary}
├── tokens/tokens.css            # token values (alias → value), for offline resolution
├── tokens/tokens.css-names.txt  # token allowlist
└── tour/frontend-pattern.md     # the Tour frontend pattern for this design system
```

No skill code changes — the skills read every design-system detail from `profile.json`.

## Installation

```bash
/plugin marketplace add edgebr/edge-tools
/plugin install ds-helper@edge-tools
```

This enables `/ds-helper:design-builder`, `/ds-helper:figma-to-code`, `/ds-helper:figma-fidelity`,
and `/ds-helper:tour-setup` across **all your projects**. A private repo works — it uses the git
credentials you already have.

> If a command fails, your Claude Code version may be older (`claude --version`).

**Updating:** automatic by default; or force it with `/plugin marketplace update edge-tools`.

## Usage

Invoke the skill for the phase you're in:

- **`/ds-helper:design-builder`** — describe the screen; it confirms the design-system library is
  linked, enumerates the real components/tokens, gets your map approved, draws, and audits
  conformance.
- **`/ds-helper:figma-to-code`** — give it the Figma node URL while implementing; it reads the
  design, maps each component to the DS package via Code Connect, and scaffolds the component with
  tokens bound. Stops and asks when a component has no mapping.
- **`/ds-helper:figma-fidelity`** — at review time, captures a token snapshot of the design and
  verifies (headless) that the code matches.

Each skill prefers to **stop and ask** over improvising when the design, the design system, or a
mapping is ambiguous.

## Tour integration (optional)

If your project uses **Tour**, one extra step wires the mentor to these skills by phase:

```bash
tour init             # only if the project isn't a Tour project yet
/ds-helper:tour-setup # materializes the active profile's frontend pattern into .agent/patterns/
```

It's **idempotent** — re-run it to re-sync after the profile's pattern changes. Projects that don't
use Tour skip this entirely; the three skills work without it.

## Prerequisites

- **Figma MCP** connected in the session.
- The **`/figma-use`** skill available — the Figma MCP requires it before `use_figma`
  (`design-builder`).
- **`design-builder`:** the target Figma page with the **profile's design-system library linked and
  enabled**.
- **`figma-to-code`:** the Figma node URL, the **profile's design-system package installed** in the
  repo, and **Code Connect** configured for reliable mapping (degrades to name-matching + Storybook
  if not).
- **`figma-fidelity`:** components styled with the profile's tokens in the consuming repo.

## Repository structure (plugin)

| Path | Purpose |
|---|---|
| `.claude-plugin/plugin.json` | Plugin manifest (`ds-helper`) |
| `profiles/dell/profile.json` | Dell profile — all design-system specifics |
| `profiles/dell/tokens/…` | Dell token values + allowlist (offline resolution) |
| `profiles/dell/tour/frontend-pattern.md` | Dell Tour frontend pattern (materialized per project) |
| `skills/design-builder/SKILL.md` | Design-time: create the Figma screen |
| `skills/figma-to-code/SKILL.md` | Implement-time: Figma → design-system-bound code |
| `skills/figma-fidelity/SKILL.md` | Review-time: verify code vs design |
| `skills/tour-setup/SKILL.md` | Helper: materialize the profile's pattern into a Tour project |
| `README.md` | This file |

## Development (contributing to the plugin)

To iterate locally without publishing, register the local repo as a marketplace:

```bash
/plugin marketplace add /path/to/edge-tools
/plugin install ds-helper@edge-tools
```

Reload skill changes without restarting: `/reload-plugins`. Quick-dev alternative (one-off
session, no install): `claude --plugin-dir /path/to/edge-tools/plugins/ds-helper`.

When editing, keep the manifests valid JSON, keep every design-system detail in `profile.json` (not
in the skills), and keep this README's skill list in sync.
