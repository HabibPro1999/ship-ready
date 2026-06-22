# Intake & Spec

How a request or ticket becomes an unambiguous PRD the Workflow can run autonomously. This stage is
**inline in the main conversation** — it's the one place a human is involved before the close-out, because
the grilling needs back-and-forth and a Workflow phase-agent can't ask questions mid-run.

## Table of contents

1. [Intake adapters](#1-intake-adapters) — pull context from a ticket or a freeform request
2. [Grill](#2-grill) — drive the interview to zero ambiguity
3. [Synthesize the PRD](#3-synthesize-the-prd) — PRD template (saved to `.prd/`), EARS criteria, systems touched
4. [Confirm-first write-back](#4-confirm-first-write-back) — closing the loop on a ticket

---

## 1. Intake adapters

Normalize every source into the same seed context, then hand it to the grill. **Read, don't write, at this
stage** (no ticket mutations until close-out).

- **Freeform request** — the user's message *is* the seed. Note what's explicit vs. assumed; the gaps are
  what the grill will resolve.
- **Jira** — use the Atlassian MCP. `getJiraIssue` for the issue (summary, description, acceptance
  criteria, labels, components, fix-version), `getJiraIssueRemoteIssueLinks` and the linked Confluence
  pages (`getConfluencePage`) for design docs, and read comments for clarifications and prior decisions.
  Labels/components are strong **surface hints** (e.g. `mobile`, `api`, `payments`) — carry them into the
  PRD's *Systems touched*.
- **GitHub issue** — `gh issue view <n> --comments` (prefer `rtk gh ...` per the project's CLAUDE.md).
  Read the body, the acceptance checklist, linked PRs, and the labels.
- **Linear** — if a Linear MCP is connected, fetch the issue + sub-issues + comments; otherwise ask the
  user to paste the issue.

Whatever the source, extract: **goal, context, explicit acceptance criteria, constraints, out-of-scope,
linked docs, and which systems/repos it appears to touch.** Hand all of it to the grill as the opening
context so it never re-asks what the ticket already answered.

## 2. Grill

Invoke `Skill({ skill: 'grill-me' })` with the seed context (it's installed normally-invocable, and runs a
`/grilling` session under the hood). The interview (one question at a time, each with a recommended answer)
walks the design tree until shared understanding. Make sure it nails, at minimum:

- **Scope & boundaries** — what's in, what's explicitly out. Vague scope is the #1 source of rework.
- **The target set** — *which repos/services does this touch?* One service? Two services + a mobile app?
  A shared types package? For each target the grill should establish **where it's checked out** (a path),
  because Adapt needs to profile each one. If a target repo isn't local, decide now: clone it, or treat it
  as out-of-scope for this run.
- **Repo integrity** — does each target build *today*? Note any referenced-but-missing files (an imported
  module, an auth guard, a build/test config) as in-scope work or a blocker. The classic failure mode is
  planning as if a skeletal repo compiles, then hitting a tier-0 typecheck wall at the first gate.
- **The contracts between targets** — if the work spans services, what's the API/event/shared-type shape
  at each seam, and who owns it? (Detail in `multi-repo-contracts.md`.)
- **Acceptance criteria** — concrete, testable conditions for "done". If the ticket's criteria are vague
  ("make checkout faster"), grill them into measurable ones ("p95 checkout < 800ms").
- **Non-obvious constraints** — back-compat, feature flags, data migration, rollout order, performance
  budgets, security/compliance requirements.

The grilling is the human gate. When it ends, ambiguity should be effectively zero — everything after runs
without stopping to ask.

## 3. Synthesize the PRD

Synthesize the conversation into a **PRD** — a lean, business-level document of *what to build*, not *how*.
Write it yourself (lean-core: we don't use Pocock's `to-prd`, which is a technical PRD that would duplicate
the Workflow's Plan). **Save it to `.prd/<slug>.md`** in the repo you're working in — `<slug>` is the ticket
id (e.g. `ACME-412-guest-checkout`) or, for a freeform request, a kebab-cased title. Create `.prd/` if absent.

The PRD stays pure WHAT. It does **not** contain contracts, module/interface design, schema, test strategy,
or sequencing — those are the Workflow's job, and deciding them twice (here and in Plan) is how the two drift.

### PRD template (`.prd/<slug>.md`)

```markdown
# <Title>

- **Source:** <jira|github|linear|freeform> · <id or "—"> · <link or "—">

## Problem
The problem the user faces, from the user's perspective.

## Solution
What changes for the user — the intended outcome, not how it's built.

## User stories
1. As a <actor>, I want <capability>, so that <benefit>.

## Acceptance criteria
EARS form (see below), numbered — the Workflow's coverage gate reads these back to check completeness.
- [ ] **AC1:** WHEN <trigger>, THE SYSTEM SHALL <response>.
- [ ] **AC2:** IF <condition>, THEN THE SYSTEM SHALL <response>.

## Constraints
Back-compat, feature flags, data migration, rollout order, performance/security budgets.

## Out of scope
What this PRD explicitly does not cover.

## Systems touched
Product scope — the repos/services the work spans (the Workflow's Plan turns these into the technical
seam/contracts). One per line: `<name> — <local path> — <surfaces hint>`.
```

Then **author the Workflow** with these values **inlined as literals** in its INPUTS block (`prdPath` and
`targets` are the load-bearing ones; the **PRD file is the source of truth** for criteria and scope). Do
**not** pass them through the Workflow `args` channel — a stringified or empty `args` crashes the script on
its first nested access. The values:

```jsonc
{
  "prdPath": ".prd/ACME-412-guest-checkout.md",
  "title":   "Guest checkout",
  "source":  "jira", "id": "ACME-412",
  "targets": [          // from "Systems touched" — drives the Adapt fan-out. One node ⇒ single-repo.
    { "name": "orders-svc", "repoPath": "/abs/path/orders-svc", "surfacesHint": ["api", "db"] },
    { "name": "mobile",     "repoPath": "/abs/path/mobile-app",  "surfacesHint": ["ui", "flutter"] }
  ]
}
```

### EARS acceptance criteria

Write each criterion in **EARS** (Easy Approach to Requirements Syntax) so it's unambiguous and testable —
this is what lets the completeness axis of the loop be machine-checked instead of judged. The common forms:

- **Ubiquitous:** `THE SYSTEM SHALL <response>` (always true).
- **Event-driven:** `WHEN <trigger>, THE SYSTEM SHALL <response>`.
- **State-driven:** `WHILE <state>, THE SYSTEM SHALL <response>`.
- **Unwanted/error:** `IF <condition>, THEN THE SYSTEM SHALL <response>`.
- **Optional:** `WHERE <feature is present>, THE SYSTEM SHALL <response>`.

**Example 1 (backend):** Input: "users should be able to reset their password"
→ `AC1: WHEN a user requests a reset for a registered email, THE SYSTEM SHALL send a single-use reset link valid for 60 minutes.`
→ `AC2: IF the email is not registered, THEN THE SYSTEM SHALL return 200 without revealing whether the account exists.`

**Example 2 (mobile/UI):** Input: "show a loading state on the profile screen"
→ `AC1: WHILE the profile request is in flight, THE SYSTEM SHALL display a skeleton placeholder and disable the edit button.`

Prefer criteria that map to an observable check (an HTTP response, a rendered state, a row written). Where
the stack already has e2e/acceptance tests, the gate can assert a real test for the criterion; where it
doesn't, the ship-gate verifies the criterion against the diff and any unit/integration evidence. (We do
not *force* writing new acceptance tests — that rigor step was intentionally left optional.)

### Systems touched → targets (not contracts)

The PRD's **Systems touched** list becomes `targets` in the handle, and drives the Adapt fan-out (a
ProjectProfile per node) and the per-target implement/review. It is product scope only. The **contracts**
between systems and the **dependency order** are *derived by the Workflow's Plan* from these targets — they
are HOW, so they stay out of the PRD (see `multi-repo-contracts.md`). One target ⇒ no contracts/DAG, and
every multi-repo step degenerates to its single-repo form.

## 4. Confirm-first write-back

When the Workflow returns a `ShipVerdict` and the user has confirmed the outward actions, close the loop on
a ticket source:

- **Draft** a comment summarizing what shipped: the change per target, gates run (pass/skip), criteria
  coverage, residual risk, and a link to the branch/PR. Show it to the user.
- **Only after an explicit go**, post it (`addCommentToJiraIssue` / `gh issue comment`) and transition
  status if asked (`getTransitionsForJiraIssue` → `transitionJiraIssue`). Publishing to a tracker is
  outward-facing — never automatic.
- On a `blocked` / `needs-human` verdict, don't transition to done; summarize what's unresolved so the
  ticket reflects reality.
