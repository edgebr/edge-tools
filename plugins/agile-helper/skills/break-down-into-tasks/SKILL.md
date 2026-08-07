---
name: break-down-into-tasks
description: "Breaks down User Stories already in Jira into technical Subtasks linked directly to the Story itself (one per acceptance criterion, with a [front-end]/[back-end]/[integration] tag at the start of the title and mirrored in the issue's Labels field, visible right on the Kanban). Creates Tasks (technical enabler work with no relation to the system's end user — e.g., infrastructure, architecture) only when the dev explicitly asks for it. Also suggests next sprint composition based on Subtasks/Tasks already created, their Priority, and story points, for the PM to review before applying to the board. Use when the dev/PO/PM asks to 'break this story into subtasks/tasks', 'create the dev subtasks for this US', 'prep the subtasks for the team', 'put together the next sprint', 'suggest what goes into the sprint', or asks for help with sprint planning after the Stories are already ready. Don't use it to create the User Story itself or its acceptance criteria (that's the qa-assistant skill's scope) nor for simple, one-off Jira operations that don't involve breaking down work or sprint planning."
preferred_model: sonnet
---

# break-down-into-tasks

Two independent functions — handle each request for what it asks, don't force the other
function.

**Function 1 — Breaking a Story into Subtasks (and, on request, Tasks):** starts from one or
more Stories already in Jira and generates, by default, technical Subtasks **linked directly to
the Story itself** (one per acceptance criterion, tagged by layer when applicable). Only creates
a Task when the dev explicitly asks for it — a Task is a technical enabler activity with no
relation to the system's end user (see "Concept" below).

**Function 2 — Sprint Suggestion:** looks at the Subtasks/Tasks already created (from one or
more Stories) and suggests which ones go into the next sprint, based on criticality (Priority)
and story points already filled in by the team — never applying this to Jira on its own. It's
always a suggestion for the PM to review.

## When to invoke

- "Break this story into subtasks", "create the dev subtasks for US X"
- "Prep the subtasks for the team" / "get this ready for sprint planning" (once the Story is
  already ready — if it doesn't have acceptance criteria yet, see Edge Cases)
- "I want a task for..." / "create a task for..." — only in that specific case, see the "Task
  (Enabler) — only on explicit request" section
- "Put together the next sprint", "suggest what goes into the sprint", "do the story points
  match the sprint?"

## When NOT to invoke

- Creating the User Story itself, its Epic, or its acceptance criteria — that's
  `qa-assistant`. If the Story doesn't exist yet or has no clear acceptance criteria,
  point that out and suggest running that skill first.
- Quick, one-off operational question about Jira, with no breakdown of work involved.

---

## Function 1: Breaking a Story into Subtasks (and, on request, Tasks)

### Concept: Subtask directly on the Story (default) vs. Task (on request)

**Default:** breaking down a User Story becomes **Subtasks linked directly to the Story itself**
— native Jira hierarchy (Subtask sits one level below Story, no Issue Link or Advanced Roadmaps
needed). In practice, **each acceptance criterion (Gherkin scenario) becomes a Subtask**. When a
criterion requires work in more than one layer (e.g., client-side AND server-side validation),
split it into one Subtask per layer — but this is no longer its own Task, it's just a **tag at
the start of the title**, mirrored in the Jira **Labels** field: `front-end`, `back-end`, or
`integration` (all lowercase; leave untagged when the Subtask isn't clearly one of these three,
e.g. a cross-cutting test). The tag goes in the title, not the description — that's what
guarantees it shows up on the Kanban/board card without opening the issue.

**A Task is only created when the dev explicitly asks for it** ("I want a task", "create a task
for..."). A Task represents a **technical enabler** — a development activity **with no relation
to the system's end user**: it doesn't implement any specific acceptance criterion of the Story,
but enables the Story to be built (architecture, infrastructure, environment setup,
prototyping). Aligned with agile practices and PMBOK 7th edition. Never create a Task as an
automatic consequence of "breaking down the story" without that explicit request — the default
is always a Subtask directly on the Story.

### Step 0 — Determine the scope

Ask (if unclear) which Stories are in scope: one specific story, a list, or "all ready Stories of
an Epic". Don't assume the Jira project — confirm every time, even if it's already been used
earlier in the same conversation.

### Step 1 — Fetch the Story and understand what needs to be built

Use `getJiraIssue` to get the Story's full description. If it was created by
`qa-assistant`, you'll find: the As a/I want/So that panel, Functional Requirements
(FR), Non-Functional Requirements (NFR), Business Rules (BR), and the Acceptance Criteria in
Gherkin — this last one is the unit that becomes a Subtask. Also check whether the description
contains a **Figma link** (design reference) — if present, note it for step 3, since the
Subtask description should reflect what the design actually shows, not just the criterion's
text. Use this content to decide what each Subtask needs to cover — don't invent functionality
that isn't described.

If the Story has no identifiable acceptance criteria, warn the dev/PM before continuing: without
them, the Subtasks have nothing to reference for what they cover, and the breakdown becomes less
reliable. Ask if they want to proceed anyway (in that case, the Subtask references only the
corresponding FR) or run `qa-assistant` first.

### Step 2 — Turn each acceptance criterion into a Subtask

For each Gherkin scenario in the Story's Acceptance Criteria, decide which layer(s) it touches:

- **front-end** — when the scenario involves a visible interface/interaction (screen, component,
  user feedback).
- **back-end** — when it involves server-side logic: endpoint, business rule, validation,
  persistence.
- **integration** — when the scenario depends on the contract between Frontend and Backend being
  defined/changed because of this Story (don't use this tag just because there's an existing,
  unchanged API call already in place — that's already `back-end`/`front-end`).
- **No tag** — when the scenario doesn't clearly fit one of these three (e.g., a cross-cutting
  test, documentation).

A scenario can become **more than one Subtask** when it requires work in more than one layer
(e.g., a validation that needs both client-side AND server-side checks becomes two Subtasks, one
`front-end` and one `back-end`). Don't force a tag on a Subtask that isn't clearly one of the
three layers.

### Step 3 — Write each Subtask: short, concrete title + functionality-focused description

- **Title:** the layer tag when applicable, **at the start of the title itself** (not the
  description) — `[front-end] `, `[back-end] `, or `[integration] `, **all lowercase** —
  followed by a **short, concrete** name for the actual piece of UI/functionality being built
  (e.g., "the scheduling modal", "the weekly availability grid component", "the reservation
  endpoint") rather than a restatement of the scenario. The tag in the title is what guarantees
  visibility on the Kanban/board without opening the description — never put the tag only in
  the description.
- **Description (ADF):** start with a short **"hook"** — a 2-4 word heading naming the piece of
  work (e.g., "Scheduling modal", "Weekly availability grid") — followed by a paragraph that's
  genuinely descriptive of the **functionality being built**, not just a restatement of the
  acceptance criterion. Use the criterion as the factual base (what must happen), but describe
  it as a feature: what the user/system sees and does, and what it's for. If the Story's
  description contains a **Figma link**, open/inspect it (via the available Figma MCP tools) and
  let the visual/interaction details you find there inform the description — don't invent
  details the Figma file doesn't show, and don't block on this if there's no Figma link or the
  tool isn't available (just describe from the criterion in that case). Close, when it makes
  sense, with 1-2 bullets of that step's specific completion criteria (DoD) — don't restate the
  Story's full DoD, just what's particular to this Subtask.
- **Labels:** set the Jira `labels` field to match the title tag — `front-end`, `back-end`, or
  `integration` (same lowercase values as the title tag; no label when the Subtask is untagged).
- **Priority:** inherits the Story's Priority, unless the dev asks for a different value.
- **Story Points:** don't estimate or fill in. Leave blank — the dev team estimates it after
  creation. (This only comes into play in Function 2, once the points are already filled in.)

### Step 4 — Review gate (mandatory, never skipped)

Show all proposed Subtasks, by Story, before touching Jira:

> "Here's the proposed breakdown for [Story]. Take a look before I create it in Jira."

Only move to step 5 with explicit confirmation. A vague answer or silence doesn't count as
confirmation.

### Step 5 — Create in Jira

- Confirm the issue types available in the project (`getJiraProjectIssueTypesMetadata`) before
  creating — names and hierarchy vary by project. Don't assume every project has the same
  hierarchy setup across sessions, even if you've already confirmed it before in a specific
  project.
- **Subtask:** create with `parent` pointing directly to the Story (native Jira hierarchy —
  Subtask sits one level below any standard-level issue, including Story).
- Set the `labels` field per Step 3.
- Build the description in ADF (not markdown) following the structure from step 3 — no Subtask
  is left without a description.

---

## Task (Enabler) — only on explicit request

Use this section **only** when the dev explicitly asks for a Task (e.g., "I want a task to set
up X", "create an infrastructure task"). Never trigger this as part of the default Step 0-5 flow
above.

Characteristics of a Task:
- Associated with an Epic or a User Story, but **is not its functionality** — it doesn't
  implement any specific acceptance criterion, has no relation to the system's end user.
- Has a clear, specific technical scope (architecture, infrastructure, configuration,
  prototyping).
- Is self-contained enough to be tracked and completed within a sprint.

**Writing the Task:**
- **Title:** `Task – [short description of the enabler]`.
- **Description (ADF)**, following the enabler template:
  1. Opening paragraph: "This Task is a technical enabler activity, needed so that [the User
     Story / the Epic] '[name]' can be implemented."
  2. "Scope" heading + bulletList with the concrete technical points the Task covers.
  3. Fixed paragraph: "Doesn't deliver direct value to the end user, but enables the related
     story to be implemented with quality and security."
  4. "Acceptance Criteria (DoD)" heading + bulletList with objective, verifiable completion
     conditions.
  5. "Relationship" heading + paragraph with the originating Epic and/or User Story (name/key).
- **Priority:** inherits the Priority of the related Story/Epic, unless the dev asks for a
  different value.
- **Story Points:** don't estimate or fill in.

**Task's Subtasks:** same rules as Steps 2-3 above (short, concrete description), but with no
relation to the system's end user — they're technical steps of the enabler itself (e.g.,
"Provision the staging environment", "Configure the vault environment variable"), not the
Story's acceptance criteria. `parent` points to the Task.

**Link to the Story/Epic:** since a Task is usually at the same hierarchy level as the
Story/Epic (varies by project — confirm with `getJiraProjectIssueTypesMetadata`), use an Issue
Link (`relates to`) to connect the Task to the related Story or Epic.

---

## Function 2: Sprint Suggestion

### Step 0 — Determine the scope

Ask for the universe of candidate Subtasks/Tasks: a project, a specific Epic, or a list of
Stories. Also ask the sprint's capacity every time the function is used — never assume a number
from an earlier time:

> "What's the sprint's capacity (total story points the team delivers) and how many days/weeks
> does it last?"

### Step 1 — Fetch the candidate Subtasks/Tasks

Use `searchJiraIssuesUsingJql` to get Subtasks (linked directly to the Story) and Tasks (when
they exist) with no sprint assigned, within the defined scope, usually in "To Do"/"Ready"
status. Bring back the fields: Summary, Priority, Story Points, and the parent Story.

Separate the ones **without Story Points filled in** — they don't count toward the capacity
math, but list them separately with a warning: the team needs to estimate them before they go
into a sprint.

### Step 2 — Sort and put together the suggestion

Sort the estimated Subtasks/Tasks by Priority (most critical first) and add up Story Points
until you approach the given capacity, without going over. When deciding order within the same
Priority, consider natural dependencies where they exist (e.g., a `back-end`/`integration`
Subtask of a Story before the `front-end` one that consumes the contract) — this is a heuristic,
not a strict rule, so flag it when you notice a dependency but aren't sure.

Put together the suggestion as a table: Subtask/Task | Parent Story | Priority | Story Points |
Running Total. Close with the suggested total vs. the given capacity, and the separate list of
items left out for lack of an estimate.

### Step 3 — Show it to the PM for review

Present the suggestion in the chat. Make it clear it's only a suggestion — the skill **never**
moves, assigns a sprint to, or changes any issue in Jira on its own, not even after it's
approved. The PM applies it to the board.

---

## Edge Cases

| Case | Behavior |
|---|---|
| Story with no clear acceptance criteria | Warn before breaking into Subtasks; suggest running `qa-assistant` first, or proceed referencing only the FRs if the dev agrees |
| Gherkin scenario doesn't clearly fit front-end/back-end/integration | Create the Subtask without a tag, instead of forcing one of the three |
| Dev asks for a Task, but the project has Task at the same hierarchy level as Story (same `hierarchyLevel`) | Warn and use an Issue Link (`relates to`) to connect the Task to the Story/Epic — confirm with `getJiraProjectIssueTypesMetadata` every time |
| Subtask/Task with no Story Points filled in, requested for the sprint suggestion | Goes into the "unestimated" list, out of the capacity math |
| Ambiguous request about which Stories are in scope | Ask objectively before fetching from Jira |
| Dev asks to skip the review gate | Still show the proposal before creating anything in Jira — only the review speed changes |
| Story description has no Figma link | Just describe the Subtask from the acceptance criterion, without inventing visual details |

## Anti-patterns

- Don't create a Task without an explicit request from the dev — the default is always a Subtask directly on the Story
- Don't force the `front-end`/`back-end`/`integration` tag on a Subtask that isn't clearly one of those three layers
- Don't create an `integration` Subtask by default — only when the contract between Frontend and Backend is actually changing
- Don't estimate Story Points for Subtasks/Tasks — that's the team's job, only count them once they're already filled in
- Don't move, assign a sprint to, or edit anything in Jira in Function 2 — it's always just a suggestion
- Don't create anything in Jira before the Function 1 review gate, even if the request seems simple
- Don't assume the sprint capacity from an earlier time — ask again every time
- Don't assume a fixed Jira project or hierarchy across sessions — confirm every time
- Don't invent Figma visual details that the file doesn't actually show — if you can't access it or it says nothing relevant, describe from the criterion alone

## Recommended model

Sonnet.
