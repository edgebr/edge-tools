---
name: pr-helper
description: "Prepares and opens the Pull Request for a piece of work (feature/task/subtask/fix/refactor/docs) following the team's branch and PR-template convention: confirms the current branch is the right one (or helps create the right one), organizes uncommitted changes into atomic, logical commits, reads the commit log and diff to draft the content, builds the description in the official template (Change Type, PR Purpose, Quality Checklist, Test Evidence, Additional Notes), and pushes the PR via GitHub CLI (gh pr create) once you've reviewed it. Use when the dev says 'push the PR for this activity', 'open a PR for EDGE-XXXX', 'I'm done with this task, prep the PR', or asks for help opening/formatting a Pull Request following the team's convention. Don't use it to review code on an already-open PR, nor for merges/approvals."
preferred_model: sonnet
---

# pr-helper

Prepares and opens the Pull Request for a piece of work, from the branch's current state to the
PR open on GitHub, following the team's naming convention and template. Never pushes or opens
the PR without showing the content to the dev to review first.

## When to invoke

- "Push the PR for this activity/task/subtask"
- "Open a PR for EDGE-XXXX", "finish pushing this branch"
- Request for help formatting/filling in a PR description following the team's template

## When NOT to invoke

- Code review of an already-existing PR (that's a different flow)
- Merge, approval, or conflict resolution on an already-open PR

## Branch naming convention

Every branch tells the story of the work: type + Jira code + short description.

| Type | Pattern | Example | When to use |
|---|---|---|---|
| Feature | `feature/CODE-XXXX-description` | `feature/EDGE-1234-history-screen` | Full User Story |
| Task | `task/CODE-XXXX-XXX-description` | `task/EDGE-1234-001-backend-api` | Larger technical split of a US |
| Subtask | `subtask/CODE-XXXX-XXX-X-description` | `subtask/EDGE-1234-001-login-endpoint` | Atomic work within a Task |
| Fix | `fix/CODE-XXXX-description` | `fix/EDGE-1234-timeout-fix` | Fix during development |
| Refactor | `refactor/CODE-XXXX-description` | `refactor/EDGE-1234-simplify-auth` | Improvement with no functional change |
| Docs | `docs/CODE-XXXX-description` | `docs/EDGE-1234-api-documentation` | Documentation |
| Other | `other/CODE-XXXX-description` | `other/EDGE-1234-setup-ci` | Technical tasks that don't fit above |

`CODE` is the Jira project prefix (e.g., `EDGE`, `REPTT`) — don't assume a fixed project,
identify it from the activity's context or ask.

## Procedure

### Step 1 — Check the current branch

Run `git branch --show-current` (or equivalent) and tell the dev which branch they're on. Check
whether it matches the pattern above **and** corresponds to the activity they want to push
(e.g., correct Jira code, correct type).

- **If it matches:** confirm with the dev and move to Step 2.
- **If it doesn't match:** help create the right branch:
  1. Ask for what's missing to build the name: change type (feature/task/subtask/fix/refactor/
     docs/other), Jira code, and a short description — only what isn't already clear from the
     conversation's context.
  2. Ask which branch this one should be created from (usually the base branch from Step 4, but
     confirm — it could be another in-progress feature branch, in the case of a subtask).
  3. Update that source branch (`git pull`), create the new one with `git checkout -b <name>`,
     and confirm the dev is on it before continuing.

A freshly created branch starts with no commits — all of the dev's work is still uncommitted
change in the working tree. This always leads to Step 2 before any commit log exists.

### Step 2 — Organize uncommitted changes into atomic commits

Before looking at any existing commit, run `git status` and `git diff` to see what's pending. If
**there's nothing pending** (clean working tree, everything already committed), skip straight to
Step 3.

If there are pending changes, **don't** bundle everything into a single commit — that hides the
story of the work and makes review harder. Instead, think the way the dev themself would when
organizing work for someone else's review, and propose a logical split into multiple smaller
commits, for example:

- By layer (e.g., backend first, then frontend, then the integration between them)
- By responsibility (e.g., setup/config → implementation → tests → documentation)
- By unit of functionality, when the diff covers clearly independent parts

Don't force a fixed number of commits — it depends on how many genuinely independent changes
exist in the diff. A small, cohesive change can become a single commit; a diff that mixes
several responsibilities deserves more.

For each proposed group, put together: the files that go into it and an objective commit
message (ask the dev whether the team follows a specific message convention, like Conventional
Commits — `feat:`, `fix:`, `test:`, `docs:`, `chore:` — or use whatever already appears in the
repository's existing commits as a reference).

Show the grouping proposal to the dev before committing anything:

> "I split your changes into N commits to make it easier to review. Take a look before I
> commit."

Only after it's confirmed (or adjusted per feedback), run `git add <group-files>` followed by
`git commit -m "<message>"` for each group, in the proposed order.

### Step 3 — Gather what changed

Get the relevant commit log (`git log <base>..HEAD --oneline` — ask for the base if it hasn't
been defined yet) and the diff (`git diff <base>...HEAD`). Use this — including the commits just
organized in Step 2 — to understand what was done: don't invent functionality that isn't in the
commits/diff. If the diff is too large, summarize by area/file instead of trying to process
everything literally.

### Step 4 — Ask for the base branch (always, don't assume)

Ask which branch the PR will be opened against (e.g., `main`, `develop`, or the parent feature
branch, in the case of a subtask). Don't reuse an answer from an earlier PR in the same
conversation — each activity can target a different base.

### Step 5 — Build the description in the template

Use **exactly** this template — don't change the structure, section headers, or HTML comments:

```markdown
# [JIRA code (Project+number, e.g. REPTT-36] Descriptive title of what this PR is about
## Change Type
<!--Uncomment the type of change being made -->
- [ ] Feature
- [ ] Bugfix
- [ ] Hotfix
- [ ] Refactor
- [ ] Documentation
- [ ] Other (specify)
## PR Purpose
<!--Describe the main goal of this PR. Explain the problem being solved or the functionality being added -->
## Quality Checklist
- [ ] Code formatted (Black/isort) and pylint score ≥ 8.0
- [ ] Unit tests added/updated
- [ ] Test coverage ≥ 80% (product target ≥ 90%)
- [ ] API contract tests passing
- [ ] Critical E2E tests passing
- [ ] Documentation updated (if needed)
- [ ] Evolution tag (prealpha/alpha/beta or rc) created and pushed (when applicable)
## Test Evidence
<!--Add screenshots, logs, or evidence of tests run -->
## Additional Notes
<!--Any additional information relevant to reviewers -->
```

Fill in each section like this:

- **Title (H1):** Jira code extracted from the branch (or asked for) + a descriptive, short
  title focused on what the PR actually delivers — draft it from the commits, but it's just a
  draft for the dev to review, not the final version.
- **Change Type:** check the box (`- [x]`) that matches the branch prefix (feature → Feature,
  fix → Bugfix, refactor → Refactor, docs → Documentation; task/subtask/other require looking at
  the diff's actual content to decide Feature/Bugfix/Refactor/Other, since they don't map 1:1).
  If unsure, ask instead of guessing — this field matters for the team's review process.
- **PR Purpose:** draft 1-3 sentences from the commits and diff, explaining the problem solved
  or the functionality delivered.
- **Quality Checklist:** only check an item if you actually verified it (e.g., ran lint/tests
  and saw them pass, or saw a new test in the diff). Items you couldn't confirm stay unchecked —
  don't check them optimistically. Tell the dev which ones were left without automatic
  verification.
- **Test Evidence:** if you ran something (lint, tests, coverage) during this process, paste the
  result here. Otherwise, leave the comment placeholder for the dev to fill in by hand.
- **Additional Notes:** only fill in when there's something relevant and verifiable (e.g., the
  branch is a subtask that will be merged directly into a feature, with no split into tasks) —
  don't invent process context you have no way of confirming from Jira/the branch.

### Step 6 — Review gate (mandatory, never skipped)

Show the full description to the dev before pushing or opening the PR:

> "Here's the PR draft. Take a look before I push — the title, change type, and checklist
> especially, since I only check what I can actually confirm."

Only move to Step 7 with explicit confirmation.

### Step 7 — Push and open the PR

1. `git push -u origin <current-branch>`.
2. Open the PR with GitHub CLI:
   ```
   gh pr create --base <base-branch> --title "[JIRA-CODE] Title" --body "<template content>"
   ```
3. Return the created PR's link to the dev.

If `gh` isn't installed/authenticated, say so and offer the alternative: just deliver the ready
markdown description for the dev to paste manually into the GitHub interface.

## Edge Cases

| Case | Behavior |
|---|---|
| Current branch doesn't follow the convention and/or doesn't match the activity | Helps create the right branch (Step 1) before continuing |
| New branch or with uncommitted changes | Organizes into atomic, logical commits (Step 2) before building the PR — never pushes everything as a single commit |
| `gh` CLI missing or not authenticated | Says so and delivers only the ready markdown description to paste manually |
| A PR is already open for this branch | Tells the dev and asks whether to update the existing one (`gh pr edit`) instead of creating a new one |
| Diff too large to process in full | Summarizes by area/file instead of trying to reproduce everything literally |
| Ambiguous change type (branch is `task`/`subtask`/`other`, or mixed diff) | Asks instead of checking a box by guessing |
| Nothing to commit/push (clean working tree, no new commits relative to base) | Says so before trying to proceed with the PR |

## Anti-patterns

- Don't bundle changes from multiple responsibilities into one giant commit — split into
  smaller, logical commits (Step 2), always showing the grouping before committing
- Don't check a Quality Checklist item without having actually verified it (ran the test/lint or
  saw it in the diff)
- Don't push or `gh pr create` before the Step 6 review gate
- Don't assume the base branch from an earlier PR — ask again for every activity
- Don't change the template's structure (section headers, comments, order) — only fill in the
  content
- Don't invent Purpose, Additional Notes, or Test Evidence that doesn't come from the diff/
  commits or from something you actually ran
