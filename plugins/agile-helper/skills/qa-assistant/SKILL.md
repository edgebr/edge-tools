---
name: qa-assistant
description: "Creates and maintains QA backlog items (Epic, INVEST User Stories, Gherkin Acceptance Criteria, Test Cases) from a requirement — scoped to exactly what's asked (just the epic, just breaking into stories, just improving a description, full cascade, etc.), never assuming more than requested. Also edits/enriches epics and stories that already exist in Jira, not just creates new ones. Creates or updates the Jira issue formatted in ADF (info panel, Gherkin code blocks). Use when the dev/PO asks to create, break down, refine, complete, or improve a story/epic/criteria/test cases, prepare items for sprint planning, or register/update this in Jira. Don't use for quick operational questions about Jira, nor to review/test already-implemented code."
preferred_model: sonnet
---

# qa-assistant

Generates and maintains QA backlog items (Epic → Story → Acceptance Criteria → Test Cases) from
a requirement, scoped to exactly what's requested — doesn't force the full cascade when the dev
only wants one part. Creates new issues in Jira or enriches/edits existing ones. Doesn't modify
production code. Doesn't create or edit anything in Jira without the dev/PO reviewing and
confirming the draft/increment first.

## When to invoke

- Dev or PO asks to "create a user story", "create an epic", "write acceptance criteria"
- Request to "refine" a requirement or prepare items for sprint planning/refinement
- Requirements/PRD document attached with a request to break into stories
- Explicit request to "create/register this in Jira"
- Request to **complete or improve** the description of an Epic/Story that **already exists** in
  Jira (without creating anything new)
- Request to **break an existing Epic** in Jira into Stories
- Request to write **only** the Acceptance Criteria, or **only** the Test Cases, of a specific
  Story (existing or newly created)

## When NOT to invoke

- Quick operational question about Jira itself ("how do I change an issue's assignee?")
- Review or testing of already-implemented code (that's a different skill/flow's scope)
- Exploratory architecture discussion with no intention of becoming a backlog item

## Procedure

0. **Determine the scope of the request (before any draft)**

   Don't assume the full cascade (Epic → Stories → Criteria → Test Cases → Jira) by default.
   Identify exactly what was requested and produce only that:

   | Dev's request | Scope | What to do |
   |---|---|---|
   | "Create an epic for X" | Epic only | Step 2 only |
   | "Break this [existing] epic into stories" | Stories only | Fetch the epic from Jira (`getJiraIssue`) to anchor context; step 3 only — don't rewrite the epic nor generate criteria/test cases unless asked |
   | "Write the acceptance criteria for this story" | Criteria only | Fetch the story from Jira if it already exists; step 4 only |
   | "Create test cases for this criterion/story" | Test Cases only | Step 5 only, from the criteria that already exist (fetch from Jira if needed) |
   | "Improve/complete the description of this epic/story" | Enrichment (edit) | See "Enriching an existing issue" below — don't recreate from scratch what's already good |
   | "Create the full feature: epic, stories, criteria, tests" | Full cascade | Steps 2-5 in sequence |

   If the request is ambiguous about scope, ask objectively instead of assuming — e.g.: "Do you
   want just the epic, or should I break it into stories too?". Regardless of scope, the review
   gate (step 6) and the ADF/Jira rules (step 7) always apply to whatever is produced.

1. **Understand the requirement**

   Read the attached document (PRD, spec, ticket) or the free-text description. Keep a mental
   checklist of essentials before drafting: persona/who uses it, trigger/what, and business value
   (why). Specific business rules (limits, formats, messages) also count as essential when the
   domain clearly needs them (e.g., password, sensitive data, money).

   If an essential is missing, ask — **don't invent a business rule**. A broad question covers a
   very vague request better than a long list:

   > "To put this together properly: who uses this feature, what triggers it, and what business
   > rules already exist (limits, formats, messages)? Feel free to answer only what you know."

   Accept "I don't know" as a valid answer for a non-critical item and move on; for a truly
   blocking essential (e.g., undefined persona), don't proceed to the next step without it.

2. **[If the step 0 scope includes Epic] Draft the Epic** — see the "Epic" section below.

3. **[If the scope includes Stories] Break into User Stories (INVEST)** — see the "User Story"
   section below. A large epic becomes more than one smaller, testable story. If the epic
   already exists in Jira (wasn't drafted just now in step 2), fetch it first (`getJiraIssue`)
   to anchor the stories in its real context.

4. **[If the scope includes Acceptance Criteria] Write in Gherkin** — see the "Acceptance
   Criteria" section below. Cover positive flow, negative flow, and the relevant non-functional
   requirements. If the story already exists in Jira, fetch it first to write criteria
   consistent with what's already described.

5. **[If the scope includes Test Cases] Derive from each criterion** — see the "Test Cases"
   section below. Numbered CT-XX.YY, tracing back to the originating criterion. Proportional to
   actual complexity — don't force a fixed number of variations.

6. **Review gate (mandatory, never skipped, regardless of scope)**

   Show the draft or increment (only what was produced within the step 0 scope — don't invent
   parts that weren't requested) in the chat before touching Jira:

   > "Here's the draft. Take a look before I [create/update] it in Jira — if anything is
   > different from how you all do it, let me know before I proceed."

   Only move to step 7 with explicit confirmation from the dev/PO. A vague answer or silence
   doesn't count as confirmation — ask again.

7. **Create OR update in Jira (ADF)** — see the "Jira Formatting" section below.
   - Never assume a fixed Jira site/project: discover/ask the destination project every time.
   - Check the project's real issue types and fields before creating/editing (names vary by
     project).
   - Build the description in ADF, not markdown.
   - **New issue:** create the Epic first, then each Story linked to it as a child item.
   - **Existing issue (enrichment):** see "Enriching an existing issue" below — never overwrite
     with `editJiraIssue` without first fetching the current content and showing the proposed
     increment.
   - If there's no Jira/tracking MCP available in the environment, tell the dev and deliver the
     same content as text/markdown in the chat (without the ADF-exclusive elements).

---

## Epic

An epic is a large slice of business value, too big to fit in a single iteration/sprint. It
serves as an umbrella: groups related stories, gives traceability from the initial idea to
delivery, and helps prioritize the backlog.

**Title:** clear, direct, value-oriented — not focused on technical implementation. Good:
"Complete Student Registration Management". Avoid: "Registration" (vague) or "Refactor Student
CRUD" (technique-oriented, not value-oriented).

**Epic description**, answering in this order:
1. What will be delivered (functional scope in 1-2 sentences)
2. Who benefits (roles/personas)
3. What problems it solves (the current pain point)
4. What value it generates for the business (efficiency, compliance, error reduction,
   satisfaction)

**Business justification:** 2 to 5 concrete reasons. Cite quality/compliance standards (GDPR,
ISO/IEC 25010, ISO/TR 25060, etc.) only when they're actually relevant to the requirement —
don't paste the generic list without thinking.

**Dependencies:** split into Frontend and Backend (or other relevant layers): form fields,
buttons, expected visual states; API endpoints, validations, error responses.

**Risks:** 2 to 4 plausible risks that could jeopardize delivery or quality.

**Related stories:** list the stories that make up this epic (short title for each).

### Full example

**Epic Title:** Complete Student Registration Management

**Epic Description:** This epic covers the development of CRUD (Create, Read, Update, Delete)
for students in an academic system. The goal is to let registrars, coordinators, and
administrators register, update, look up, and delete student information securely, intuitively,
and efficiently. The system must offer a friendly interface, real-time validation, clear
messages, and visual feedback.

**Business Justification:**
- Eliminate duplicate and inconsistent records, ensuring the integrity of academic data.
- Speed up the enrollment and service process, reducing registration time.
- Increase the reliability of information used by registrar's offices and coordination teams.
- Meet usability best practices (ISO/TR 25060:2023), product quality (ISO/IEC 25010:2023), and
  data security (GDPR/ISO/IEC 27001).

**Dependencies:**
- Frontend: responsive form with required fields (Name, National ID, Date of Birth, Program);
  Save/Cancel/Delete buttons; status messages (Loading, Success, Error, Validation); colors (red
  for error, green for success).
- Backend: REST API (POST, GET, PUT, DELETE); uniqueness validation for the national ID; change
  logs (audit trail); structured responses (400 invalid fields, 409 duplicate, 500 internal
  error).

**Related User Stories:**
- US01 – Register student (with visual feedback)
- US02 – Edit student (with error messages and confirmation)
- US03 – Look up student (search by name, ID, enrollment number)
- US04 – Delete student (with deletion confirmation)

**Risks:**
- Frontend-backend integration failure.
- Duplicate records if validation is insufficient.
- Usability issues reducing adoption.

---

## User Story

Every story follows the INVEST format:

> **As a** [role/persona who will use it]
> **I want** [concrete action/intent]
> **So that** [business value — why it matters]

This defines, in this order, who will use it, what the person needs to do, and why it matters to
the business. Follow the INVEST principles (Independent, Negotiable, Valuable, Estimable, Small,
Testable): if a story is getting too large to fit in a sprint, break it into smaller stories.

In the chat, write this block as running prose. Only when creating the issue in Jira does it
change format — it becomes an info panel (see "Jira Formatting" below), not running text.

### Functional Requirements (FR)

Each functional requirement is an objective, measurable sentence, numbered (`FR01`, `FR02`,
...), describing a concrete action/behavior the system must have — unambiguous, so any
requirement can be traced back to a specific acceptance criterion. When the requirement stems
from a specific business rule, reference the corresponding `BR` in parentheses at the end of the
sentence (see the "Business Rules" section below). The relationship doesn't need to be 1:1 — an
FR may have no associated BR, and a BR may underlie more than one FR.

```
Functional Requirements (FR)
FR01 – [concrete, measurable action/behavior]. (BR1)
FR02 – [concrete, measurable action/behavior]. (BR2)
```

Cover: main fields/actions, expected validations, integrations with other screens/systems, and
expected error/success behaviors — the same content that used to go in running prose, now
broken into numbered, traceable FR items.

### Non-Functional Requirements (NFR)

Describe how the system should behave in terms of quality — not what it does, but how it behaves
while doing it. Still organized by ISO/IEC 25010 attributes (only develop the ones relevant to
the requirement at hand), but each attribute becomes its own line, with the attribute name as
the label:

```
Non-Functional Requirements (NFR)
Usability – [quality behavior, with a metric when it makes sense]
Performance Efficiency – [response times, loading]
```

Available attributes: Functional Suitability, Performance Efficiency, Compatibility, Usability,
Reliability, Security, Maintainability, Portability, Sustainability — plus Responsiveness when
the requirement involves layout adapting to different screens/resolutions.

Prefer concrete metrics when it makes sense ("response time ≤ 2s") over vague adjectives
("fast").

### Business Rules (BR)

Each business rule is the condition/constraint behind one or more FRs, numbered (`BR1`, `BR2`,
...) and referenced in parentheses in the corresponding FR(s):

```
Business Rules (BR)
BR1 – [business condition or constraint, in one objective sentence]
BR2 – [business condition or constraint, in one objective sentence]
```

Same rule from step 1 applies here: don't invent a BR that didn't come from the requirement or
the dev's answer.

### Risks and Mitigations

2 to 4 plausible risks specific to this story (finer granularity than the Epic's risks — see the
"Epic" section above), each with its mitigation on the same line:

```
Risks and Mitigations
R1 – [concrete risk] → [corresponding mitigation].
R2 – [concrete risk] → [corresponding mitigation].
```

### Acceptance Criteria in Gherkin

Each criterion is an objective rule the system must satisfy for the story to be considered done:

```
Scenario: [short, descriptive scenario name]
  Given [context/precondition]
  When [user action, if any]
  Then [what the system must do]
  And [additional conditions, if needed]
```

Cover both positive and negative flows (errors, validations, exceptions), and include criteria
that verify the relevant non-functional requirements, not just the functional ones.

In Jira, all scenarios of the story go inside **a single** code block highlighted as Gherkin
(not one `codeBlock` per scenario) — see "Jira Formatting" below. The Gherkin keywords
(`Scenario`, `Given`, `When`, `Then`, `And`, `But`) and the step-description phrase are both in
**English**.

### Definition of Ready (DoR)

Close the story with this checklist, marking what's already covered by the generated content and
leaving open what depends on human validation (don't mark as done something only a human can
validate, like UX prototypes or technical complexity assessed by the team):

```
Description and Context
[ ] Story has a clear description in "As a... I want... So that..." format
[ ] Business value is explicit and understandable
[ ] Business rules are documented and validated

Design and UX
[ ] High-fidelity prototypes are available in Figma
[ ] User flows have been validated by the UX team
[ ] Error and loading states are defined

Criteria and Tests
[ ] All acceptance criteria are documented
[ ] BDD scenarios cover positive and negative flows
[ ] Non-functional requirements have measurable criteria

Technical Dependencies
[ ] Required APIs and integrations are identified
[ ] Dependencies on other teams/systems have been mapped
[ ] Test data and environments are available

Estimation and Planning
[ ] Story is sized appropriately for a sprint
[ ] Technical complexity has been assessed by the dev team
[ ] There are no known blockers preventing the start

Status: [ ] Approved for Planning | [ ] Needs adjustments
Notes: [space for notes on open items]
```

### Example (condensed)

**US-01: Student Registration**

> As a system user (administrator or student)
> I want to fill out and submit a registration form with required fields
> So that I can correctly register the student's data with quality and efficiency

**Functional Requirements (FR)**
```
FR01 – Display a form with required fields (First Name, Last Name, National ID, University,
Program) with a visual indication of which are required. (BR1)
FR02 – Validate the national ID in real time, applying the ###.###.###-## mask and checking
the check digit. (BR2)
FR03 – Block submission when any required field is empty or invalid, highlighting them with
contextual error messages. (BR3)
FR04 – Display a clear confirmation after a successful save.
```

**Non-Functional Requirements (NFR)**
```
Performance Efficiency – Response time ≤ 2s on submission.
Usability – Keyboard navigation and WCAG 2.2 AA compliance.
Security – Communication via HTTPS/TLS 1.3+ and masking of the national ID.
Compatibility – Support for the current versions of Chrome/Edge/Firefox/Safari and iOS ≥ 15 /
Android ≥ 10.
```

**Business Rules (BR)**
```
BR1 – First Name, Last Name, National ID, University, and Program are required fields.
BR2 – The national ID is only accepted with a valid check digit per the official algorithm.
BR3 – Submission is only allowed when all required fields are filled in and valid.
```

**Risks and Mitigations**
```
R1 – User loses filled-in data in case of a network failure → save a local draft (autosave)
while the form is open.
R2 – National ID validation diverges between frontend and backend → centralize the check-digit
logic in a shared, tested function.
```

**Acceptance Criteria (examples):**

```
Scenario: Real-time national ID validation
  Given that the user types in the National ID field
  Then the system must apply the ###.###.###-## mask
  And validate the check digit per the official algorithm
  And display visual feedback indicating valid/invalid

Scenario: Prevention of invalid submission
  Given that there are empty or invalid required fields
  Then the submission must be blocked
  And problematic fields must be highlighted with clear messages
```

---

## Test Cases

An acceptance criterion is a *rule* the system needs to follow. A test case is a *script* to
verify that rule is actually being met. A single criterion can spawn several test cases — but
not every criterion deserves the same number of cases.

Before writing formally, think about the questions a QA would ask about that criterion:
- What should happen when the user does it right (positive flow)?
- What should happen when they do something wrong or unexpected (negative flow)?
- Are there relevant variations (other data, screen sizes, browsers, user profiles) worth testing
  separately?

Not every criterion needs browser/device variations — only include those when the original
criterion already talks about responsiveness/compatibility. For a simple field-validation
criterion, two or three cases (positive, negative, one edge case) already cover it well. For a
richer criterion (e.g., mask + validation + accessibility + performance), break it into more
cases, one per testable aspect.

**Written in Gherkin:**

```
CT-XX.YY – [short case name, describing what it verifies]

Scenario: [same name, as the scenario title]
  Given [initial context/state]
  When [action taken, if any]
  Then [expected, verifiable result]
```

Number the test cases referencing the originating criterion (e.g., criterion 5 → CT-05.01,
CT-05.02...), to keep traceability between criterion and test.

Without a test management tool integration (Qase or similar) configured, test cases stay
alongside the User Story itself in Jira — as its own section ("Test Cases") in the description,
right after the acceptance criteria, keeping the CT-XX.YY numbering.

### Applied example

**Originating criterion:** "The system must validate the national ID in real time, applying the
###.###.###-## mask and checking the check digit per the official algorithm."

```
CT-05.01 – Dynamic mask application

Scenario: Dynamic mask applied while typing
  Given that the user types a number in the National ID field
  Then the system must apply the ###.###.###-## mask as digits are entered

CT-05.02 – Rejection of invalid check digit

Scenario: National ID check digit validation
  Given that the user fills in a complete national ID
  Then the system must validate the check digit per the official algorithm
  And reject national IDs with an invalid check digit

CT-05.03 – Visual feedback for valid/invalid national ID

Scenario: Visual indication of invalid national ID
  Given that the user typed an invalid national ID
  Then the system must display a negative visual indicator (e.g., red icon or message)
```

This criterion generated 3 focused test cases (mask, check-digit validation, feedback) — no need
to invent browser or device variations, because the original criterion didn't address that.

---

## Jira Formatting (ADF)

Jira Cloud stores an issue's description in **ADF** (Atlassian Document Format), a tree of JSON
nodes. When the available issue creation/edit tool accepts a content format parameter
(`contentFormat`), use `"adf"` — not `"markdown"` — whenever you need the elements below,
because plain markdown doesn't turn into a colored panel or a code block with highlighted
language.

The whole document is always:

```json
{
  "type": "doc",
  "version": 1,
  "content": [ ]
}
```

(the `content` array holds the block nodes, in the order they appear in the description.)

### 1. "As a / I want / So that" panel

`info`-type panel. Each line of the block is a paragraph inside the panel, with the label in
bold:

```json
{
  "type": "panel",
  "attrs": { "panelType": "info" },
  "content": [
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "As a ", "marks": [{ "type": "strong" }] },
        { "type": "text", "text": "Bulk Processing system user (Dell Specialist)" }
      ]
    },
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "I want ", "marks": [{ "type": "strong" }] },
        { "type": "text", "text": "to fix records with errors directly from the interface, through a Fix modal accessed from the records table" }
      ]
    },
    {
      "type": "paragraph",
      "content": [
        { "type": "text", "text": "So that ", "marks": [{ "type": "strong" }] },
        { "type": "text", "text": "the user can fix errors in a controlled, efficient way, without redoing the entire upload, ensuring the record is reprocessed successfully" }
      ]
    }
  ]
}
```

This panel replaces the running-text "As a.../I want.../So that..." format used in the chat —
in Jira, it always goes inside this info panel.

### 2. Acceptance Criteria in Gherkin (a single code block with all scenarios)

All scenarios of the story go inside a **single** `codeBlock` node, with
`attrs.language: "gherkin"` — don't create one `codeBlock` per scenario. Separate scenarios from
each other with a blank line inside the same `text`. The Gherkin keywords (`Scenario`, `Given`,
`When`, `Then`, `And`, `But`) and the step-description phrase are both in **English**:

```json
{
  "type": "codeBlock",
  "attrs": { "language": "gherkin" },
  "content": [
    {
      "type": "text",
      "text": "Scenario: 1 - Open the Fix Job modal\n  Given the records table is visible\n  And the row status is COMPLETED WITH ERRORS\n  When the user clicks the \"Fix\" button (wrench icon)\n  Then a modal should open in the center of the screen to fix the errors\n\nScenario: 2 - Close the modal without saving\n  Given the Fix Job modal is open\n  When the user clicks \"Cancel\"\n  Then the modal should close without persisting any change"
    }
  ]
}
```

Same pattern for Test Cases — they also all go inside a single `codeBlock`, keeping each case's
`CT-XX.YY` numbering separated by a blank line within the same `text`.

Before actually creating the issue, confirm (by inspecting the tool's fields/options or asking
the user) that the Jira project in question has "Gherkin" enabled as a code block language —
most Jira Cloud projects on the new editor do, but this can vary by instance.

### Heading and normal text

For section titles (e.g., "Functional Requirements", "Test Cases"):

```json
{
  "type": "heading",
  "attrs": { "level": 3 },
  "content": [{ "type": "text", "text": "Functional Requirements" }]
}
```

And for normal text, plain paragraphs:
`{"type": "paragraph", "content": [{"type": "text", "text": "..."}]}`.

### 3. FR / NFR / BR / Risks (bulletList with bold label)

Each Functional Requirements, Non-Functional Requirements, Business Rules, or Risks and
Mitigations item becomes a `listItem` inside a `bulletList`, with the label (`FR01 – `, the NFR
attribute name, `BR1 – `, `R1 – `) in bold and the rest as normal text:

```json
{
  "type": "bulletList",
  "content": [
    {
      "type": "listItem",
      "content": [
        {
          "type": "paragraph",
          "content": [
            { "type": "text", "text": "FR01 – ", "marks": [{ "type": "strong" }] },
            { "type": "text", "text": "requirement text, with (BR1) at the end when there's an associated rule." }
          ]
        }
      ]
    }
  ]
}
```

Same structure for NFR (label is the attribute name, e.g. `"Usability – "`), BR (label
`"BR1 – "`), and Risks (label `"R1 – "`, with the risk and mitigation in the same paragraph,
separated by `→`). Each section (FR, NFR, BR, Risks) is its own heading (level 3) followed by
its own `bulletList` — don't mix items from different sections in the same list.

---

## Enriching an existing issue (instead of creating one)

When the request is to complete or improve an Epic/Story that already exists in Jira (not create
a new one), the flow changes in three points compared to creation:

1. **Fetch before writing.** Use `getJiraIssue` to get the current description (ask for
   `responseContentFormat: "adf"` if you're going to rebuild the whole document, or
   `"markdown"` if you just need to read the content to understand what already exists).

2. **Tell apart what's missing from what's already good.** Compare the current content against
   the expected structure of the corresponding section ("Epic" or "User Story" above). Some
   fields may already be filled in and good — don't rewrite them. List only what's missing,
   incomplete, or visibly outdated.

3. **Propose only the increment, not a brand-new document.** Show, at the review gate (step 6),
   what already exists vs. what you're proposing to add/adjust — not an entirely new draft that
   discards what was already there, unless the dev explicitly asks for a full rewrite.

Once confirmed, assemble the full description in ADF (existing content preserved + approved
increment) and apply it with `editJiraIssue`, always in ADF when the tool accepts
`contentFormat`. Never call `editJiraIssue` without having shown the increment and received
explicit confirmation — even for a small enrichment, the step 6 gate doesn't relax.

## Edge cases

| Case | Behavior |
|---|---|
| Vague requirement (no persona/trigger/rule) | Ask the essentials before drafting (step 1); don't deliver a story with an invented business rule |
| Request covers only part of the cascade (e.g., "just the description", "just the criteria") | Produce only what was requested (see the step 0 scope table); don't move on to the following parts of the cascade on your own |
| Ambiguous request scope | Ask objectively which scope before drafting, instead of assuming the full cascade |
| Request to enrich an Epic/Story that already exists in Jira | Fetch the issue first (`getJiraIssue`), propose only the increment over what already exists — don't rewrite from scratch what's already good (see "Enriching an existing issue") |
| Environment without Jira/tracking MCP configured | Tell the dev and deliver the content in markdown in the chat, without ADF elements |
| Jira project without "Gherkin" as a code block language | Confirm with the dev before creating/editing; use the closest available language and flag the limitation |
| Dev asks to skip the review gate ("just create/edit it directly") | Still show the draft/increment before touching Jira — the gate doesn't relax; only the review's speed changes |
| Requirement too large for a single story | Break into multiple smaller stories (INVEST) instead of one giant story |

## Anti-patterns

- Don't invent a business rule, numeric limit, or error message that didn't come from the requirement or the dev's answer
- Don't create or edit anything in Jira before the step 6 review gate, even if the request seems simple
- Don't force the next steps of the cascade (Epic → Stories → Criteria → Test Cases) when the request covers only one part — produce exactly the requested scope (step 0)
- Don't overwrite the description of an existing issue from scratch when the request is to "improve"/"complete" it — fetch the current content and propose only the increment
- Don't force a fixed number of criteria/test cases — proportional to actual complexity
- Don't use `contentFormat: markdown` when the ADF panel or the Gherkin code block are needed — plain markdown produces neither
- Don't assume a fixed Jira project across sessions — always confirm the destination project

## Recommended model

Sonnet covers the entire flow (content generation + ADF assembly is procedural, doesn't require
deep reasoning). Consider Opus only if the requirement involves multiple simultaneous epics with
many cross-dependencies between stories, where the INVEST breakdown gets genuinely ambiguous.
