---
name: tour-setup
description: "Materializes the ds-helper Tour integration into the current project: copies the active profile's frontend pattern into .agent/patterns/ and registers it in .agent/PROJECT_PATTERNS.md, so the Tour mentor routes to the design-builder/figma-to-code/figma-fidelity skills by phase. Pattern and index summary come from the active design-system profile. Idempotent — safe to re-run to sync the pattern with the plugin's version. Only needed for projects that use Tour; the three skills work without it. Use when setting up ds-helper in a new project that uses Tour, or to update the pattern after the plugin changes it."
preferred_model: sonnet
---

# tour-setup

One-time (and re-runnable) setup that plugs the `ds-helper` skills into a project's **Tour** flow.
It takes the active design-system profile's frontend pattern and materializes it into the project's
Tour catalog, so the mentor knows — by phase — to reach for `design-builder`, `figma-to-code`, and
`figma-fidelity`.

> This is **only** for projects that use Tour. The three skills work standalone without it — the
> pattern just wires the *mentor routing* + design-system coding knowledge into the phase flow. If
> the project doesn't use Tour, there's nothing to do here.

The profile is the **source of truth** for the pattern (`${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/`);
each project holds a **materialized copy** in `.agent/patterns/`. Re-running this skill re-syncs
that copy.

## Active profile (resolve before anything)

The profile provides the pattern file and how to register it. Resolve it first:

1. If the dev names a profile explicitly (e.g. "set up the Dell profile") → use that.
2. Else if the repo has a `ds-helper.json` with a `"profile"` → use it.
3. Else if exactly one profile exists under `${CLAUDE_PLUGIN_ROOT}/profiles/` → use it.
4. Else → **ask the dev** which profile to use, listing what's under `profiles/`.

Load `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/profile.json`. Pattern source =
`profile.tour.pattern`; materialized name = `profile.tour.patternName`; index row summary =
`profile.tour.indexSummary`; label = `profile.displayName`.
> Example (Dell profile): patternName `frontend-dds`, source `tour/frontend-pattern.md`.

## When to invoke

- Setting up `ds-helper` in a **new project that uses Tour**.
- After the profile's frontend pattern changes, to **re-sync** the project's copy.
- The dev asks to "wire up the design system into Tour" / "install the DS pattern".

## When NOT to invoke

- The project **doesn't use Tour** (no `.agent/`) → the skills work directly; nothing to set up.
- Just wanting to run a skill (`/ds-helper:design-builder` etc.) — those need no setup.

## Procedure

1. **Check the project uses Tour.**
   - Look for `.agent/patterns/` and `.agent/PROJECT_PATTERNS.md`.
   - If `.agent/` is absent → tell the dev this isn't a Tour project: the skills work as-is,
     and if they want the mentor integration they should run `tour init` first. **Stop.**

2. **Materialize the pattern.**
   - Source: `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/${profile.tour.pattern}`.
   - Target: `.agent/patterns/${profile.tour.patternName}.md`.
   - If the target **doesn't exist** → copy it.
   - If it **exists and is identical** → report "already up to date", do nothing.
   - If it **exists and differs** → show a short diff summary and **ask** before overwriting (the
     project may have local edits worth keeping). Default to not clobbering silently.

3. **Register in the index.**
   - Ensure `.agent/PROJECT_PATTERNS.md` has a Frontend entry linking
     `patterns/${profile.tour.patternName}.md` (the index summary is what cues the mentor to load
     it by phase).
   - If missing → add the row (under a "Frontend" section, creating it if needed), using
     `profile.displayName` as the label and `profile.tour.indexSummary` as the description:
     `| **<displayName>** | <indexSummary> | [<patternName>.md](patterns/<patternName>.md) |`
   - If present → leave it.

4. **Report** exactly what changed (copied / synced / already current; index row added or not),
   and confirm the mentor will now surface the skills in SPEC/PLAN/IMPLEMENT/REVIEW of UI work.

## Hard rules

- Never overwrite a project-local materialized pattern that differs **without asking** — the
  project may have intentionally diverged.
- Touch only `.agent/patterns/<patternName>.md` and `.agent/PROJECT_PATTERNS.md`. Nothing else.
- Don't create `.agent/` yourself — if it's absent, that's the "not a Tour project" branch (step
  1), not something to scaffold.

## Anti-patterns

- ❌ Running in a non-Tour project and scaffolding `.agent/` from scratch (that's `tour init`'s
  job, not this skill's).
- ❌ Silently overwriting a diverged local pattern.
- ❌ Copying the pattern but forgetting the index row (without it, the mentor won't load the
  pattern by phase).
- ❌ Editing other patterns or Tour config.

## Dependencies

- **Profile (bundled, source of truth):** `${CLAUDE_PLUGIN_ROOT}/profiles/<profile>/` — the
  `profile.json` and the frontend pattern (`profile.tour.pattern`).
- A project initialized with **Tour** (`.agent/` present) — otherwise the skill no-ops with an
  explanation.

## Recommended model

Sonnet — it's a small, deterministic copy + index-edit with an overwrite guard.
