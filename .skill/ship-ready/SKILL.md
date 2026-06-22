---
name: ship-ready
description: >-
  Turn any unit of work — a freeform feature request OR a Jira/Linear/GitHub issue — into ship-ready,
  reviewed, tested code with no bugs and no tech debt, by running the full delivery loop end to end:
  grill requirements to zero ambiguity, detect the project's stack/commands/conventions, plan, research
  best practices, implement, then review → triage → fix in a loop until clean, behind a structured
  ship-readiness gate. Project-agnostic — backend, frontend, full-stack, mobile (incl. Flutter), CLI,
  library, or infra; monolith or microservices; one repo or a feature spanning several repos/services.
  Invoke it to build, implement, ship, deliver, or finish a feature, or to work a ticket/issue end to
  end (e.g. "build the X feature", "work ABC-123", "take this to a merge-ready PR") — for a whole unit
  of work, not a one-line edit, a review, or a pure question.
disable-model-invocation: true
---

# ship-ready

Turn a unit of work into code you'd actually merge — correct, tested, reviewed, idiomatic, and complete
against its acceptance criteria — without a human babysitting the middle. This skill is a **universal
orchestrator**: it detects the project it's running in and adapts, instead of assuming any stack, shape, or
toolchain.

The headline idea: **resolve all ambiguity up front with an interview, then run a single autonomous loop
that can't declare victory until the machine says it's clean.** A human is in the loop in exactly two
places — the grilling at the start, and a confirm before anything outward (commit / PR / ticket write) at
the end. Everything between is autonomous and structurally gated.

## The shape (three stages)

```
INLINE  ── Intake → GRILL (grill-me) → Synthesize PRD → save .prd/ ──┐  (the only human-facing stage)
                                                                      ▼  author Workflow (inline prdPath + targets)
WORKFLOW ── Adapt → Plan → Research → Implement → ⟲loop(Review→Triage→Fix→Gate→Cover) → Ship-gate ──┐
                                                              (single canonical Workflow, autonomous) ▼
INLINE  ── present ShipVerdict → confirm-first commit / PR / ticket write-back
```

Each stage is described below. The depth lives in `references/` — read the relevant one when you reach
that stage rather than loading everything at once.

## Non-negotiable: grill first

**Always run `grill-me` before anything else** — it is the load-bearing reason the rest can be autonomous.
A Workflow phase-agent cannot stop to ask the user a question mid-run, so every ambiguity has to die before
the Workflow starts. Invoke it explicitly:

```
Skill({ skill: 'grill-me' })
```

`grill-me` is the interview entry point — it runs a `/grilling` session under the hood. It is installed as a
normally-invocable skill, so ship-ready (and you) can call it directly via the Skill tool. (ship-ready itself
is `disable-model-invocation`: the **user** starts it with `/ship-ready`; you cannot auto-launch it for them.)

Seed it first with whatever you already know (the ticket body, the user's request, what you found scanning
the repo) so the interview targets real gaps instead of re-asking the obvious. It self-scales: a crisp,
fully-specified ticket yields a short confirmation; a vague one-liner yields a deep interview. The grilling
must also establish **which repos/services the work touches** (the target set) — that decision shapes
everything downstream — and **whether each target actually builds today** (a missing imported file or absent
build/test config is in-scope work or a blocker, not something to discover at the gate). See
`references/intake-and-spec.md`.

When the interview reaches shared understanding, **synthesize a PRD** — a lean, business-level "what to
build" document — and **save it to `.prd/<slug>.md`** in the repo you're working in (`<slug>` = the ticket
id, else a kebab title). The PRD is pure **WHAT, not HOW**: problem, solution, user stories, acceptance
criteria, scope, and which systems are touched. It deliberately leaves all technical planning — contracts,
module/interface design, test strategy, sequencing — to the Workflow, so planning happens once, in one
place, and the PRD stays a reviewable product artifact. (We write our own PRD; we don't use Pocock's
`to-prd`, which is a *technical* PRD that would duplicate the Workflow's Plan.)

- **`acceptanceCriteria` in EARS form** (`WHEN <trigger>, THE SYSTEM SHALL <response>`), numbered `AC1..ACn`,
  so the Workflow's coverage gate reads them back from the PRD file and checks completeness mechanically —
  this is what makes "done" objective rather than vibes.
- **Systems touched** — the repos/services the work spans (each with its local path), established by the
  grill. This is product scope; the Workflow's **Plan** derives the precise contracts + dependency order
  from it (that's HOW — it does not belong in the PRD). One system ⇒ the multi-repo machinery collapses away.

Then author the Workflow with `prdPath` + `targets` **inlined as literals** (you also know `{ title, source,
id }`): the **PRD file is the source of truth** for criteria and scope, and `targets` drives Adapt. Don't
pass them through the Workflow `args` channel — a stringified or empty `args` crashes the script on first access.

## Stage 2 — the canonical Workflow (autonomous)

Launch **one** Workflow, authoring its script with `prdPath` + `targets` **inlined** (not via the fragile
`args` channel). Use the script in
`references/canonical-workflow.md` as the template — it is already wired with the schemas, the loop, and
the splice points. The phases:

1. **Adapt** — fan out (`parallel()`, one per target) to detect each repo's **ProjectProfile**: stack,
   package manager, the real test/lint/build/format/verify commands, CI, conventions, surfaces, and the
   review lenses those surfaces imply. **Read CI config as ground truth** for the gating commands — it's
   more reliable than guessing from scripts. Detail + the surface→lens table: `references/adaptation-layer.md`.
2. **Plan** — `pipeline`: first a thin **unified contract/sequencing plan** that owns only the seams
   (contracts + the cross-cutting acceptance criteria + the dependency DAG), then `parallel()` per-target
   sub-plans that conform to that contract *and* each repo's own conventions. Why two levels: a single
   monolithic plan ignores per-repo conventions; N disconnected plans drift apart at the seam. See
   `references/multi-repo-contracts.md`.
3. **Research** — pull current best practices for the *actual* libraries in use via **Context7**
   (`mcp__plugin_context7_context7__*`, already installed) plus web where needed, and reconcile them with
   the repo's existing conventions. Front-loading idiomatic patterns here means review isn't fighting
   nonidiomatic code later.
4. **Implement** — per target, in the DAG order (independent targets in parallel, dependents sequenced),
   against the **frozen contract**. **Build lazy (YAGNI):** simplest thing that works — stdlib → native
   platform/framework feature → installed dep → one line → only then new code; no unrequested abstractions;
   never simplify away validation/error-handling/security/anything the PRD requires. Mark each deliberate
   shortcut with a `ponytail: <ceiling>, <upgrade trigger>` comment (the gate harvests these into the verdict's
   debt ledger). Isolate parallel writers (`isolation:'worktree'` within one repo, or a distinct repo path per
   target).
5. **Loop until clean** — the heart of the rigor (see next section).
6. **Ship-gate** — a final judgment agent emits a structured `ShipVerdict` (see *Contracts* below).

Return the `ShipVerdict` from the Workflow so the inline stage can branch on it.

### Loop until clean — what makes "no bugs / no debt" real

The loop runs Review → Triage → Fix → Gate → Cover, then **the script (not a model) decides whether to
loop again**. Default rigor (Standard): exit when a pass is clean on *both* axes, cap at ~3 iterations,
and on hitting the cap return a `blocked` verdict listing residuals — **never a silent green**.

- **Review → Triage → Fix is spliced from the `workflow-review-phase` skill** — do not re-invent it.
  Invoke that skill while authoring the Workflow and paste its segment, passing it this target's
  re-derived diff, detected `VERIFY_CMD`, and surface-selected `LENSES`. Keep its two validated mechanics
  exactly: thermo-nuclear runs as a **real `Skill` call inside one phase-agent**; code-review is the
  **script's own `parallel()` fan-out** (phase-agents can't spawn subagents). For multi-repo, add a
  **cross-cutting integration lens** that checks contract conformance across targets — the one thing no
  single-repo reviewer can see.
- **Triage is the gate** — the loop keys its exit off *triage accepted = 0*, never a confidence score. A
  score floor lets the weakest agent silently kill real findings before the strongest sees them.
- **Two exit axes, both machine-checked:**
  - *Correctness/quality* — `triage.accepted == 0` **and** the tiered gates are green.
  - *Completeness* — every EARS acceptance criterion **in the PRD** is satisfied. After each pass, read the
    criteria back from `.prd/<slug>.md` and reconcile the diff against them (a converge-style pass); treat
    any uncovered criterion as remaining work. The no-debt / actually-done axis, distinct from "found no bugs."
- **Tiered gates actually run.** Always: typecheck/compile + lint + format-check + unit. If present:
  integration / e2e / contract tests (and for UI surfaces: build + a boot/route smoke check — the browser
  MCP is available). If the environment can't bring a suite up (e.g. Docker services), mark it
  **skipped-and-why** — reporting green on a skipped gate is a lie the verdict must never tell.
- **Deliberate simplifications are surfaced, never silent.** After the loop, the gate harvests the `ponytail:`
  shortcut markers into a debt ledger on the `ShipVerdict` (`simplifications`) — each with its ceiling and
  upgrade trigger, any marker with no trigger flagged as a rot risk. This is *accepted* debt: it does **not**
  gate the exit (a clean run still ships with it listed), it just keeps a deferral from quietly becoming
  permanent. Same honesty as skipped-and-why gates, applied to code shortcuts.

## Stage 3 — confirm-first close-out

Present the `ShipVerdict` to the user. Then, and only with confirmation, perform outward actions: commit /
push / open a PR, and (for ticket-sourced work) post a summary comment and transition status. Drafting
these is fine; **publishing them is outward-facing and requires an explicit go** (see the operating rules
on outward actions). `blocked` / `needs-human` verdicts stop here and surface exactly what's unresolved.

## Model usage — assess per phase, don't hardcode

Choose each phase's model/effort yourself based on **stakes × complexity × blast-radius**; the table in
`references/canonical-workflow.md` is a *prior*, not a rule. Heuristic: deep-judgment phases (plan,
security/concurrency lenses, triage, ship-gate) get the top tier; implement/fix/research sit mid; purely
mechanical steps (re-verify, optional scoring, format) go cheap. Escalate mid-loop when signal demands it
(e.g. triage keeps surfacing criticals → raise the fix tier). In multi-repo, assess **per target** — a
security-critical service earns deeper review than a CSS tweak in the mobile app.

## Guardrails (these are load-bearing — keep them)

- **Grill first, always.** No skipping to implementation, however clear the request looks.
- **Triage is the gate, never a threshold.** Send every finding to triage; let the strongest agent decide.
- **Re-derive the diff from git each loop pass.** Fix-agents sometimes mis-report which files they changed;
  trust `git`, not their summary. In a git worktree, `git add -A` before diffing — worktrees symlink
  `node_modules` and an unstaged diff is full of noise.
- **Never report green on a skipped gate.** Enumerate every gate as ran / passed / failed / skipped-and-why.
  And `green` is **computed by the script** from per-gate statuses — never a single boolean a model asserts.
- **A non-building repo is never done.** A missing imported file or absent build/test config is scope or a
  blocker — surface it at intake/adapt (the `preflightWarnings`), don't let it first appear at the gate.
- **`needs-human` is first-class.** If the Workflow hits something only a human can resolve (a contradictory
  criterion surfaced during implementation), return `needs-human`; the inline stage asks the user and
  **resumes** the Workflow (`resumeFromRunId`) rather than restarting.
- **Preserve the phase-agent split.** Thermo-nuclear = real Skill call in one phase-agent; code-review =
  script-level `parallel()`. This is validated runtime behavior, not a style choice.
- **Confirm before any outward write.** Commit, push, PR, ticket comment, status transition.

## Dependencies (deliberately lean — don't add skills)

Hard dependencies, all already present: **`grill-me` + `grilling`** (the interview), **Context7** (library
docs, an MCP), and the review stack — **`workflow-review-phase`**, **`thermo-nuclear-code-quality-review`**,
and the **`/code-review`** engine it reproduces. Everything else from the ecosystem (superpowers, cc-sdd,
Pocock's `tdd`/`diagnosing-bugs`, Pact/Specmatic) is **detect-and-use-if-present**, never installed —
over-skilling bloats every session's context and causes mis-triggering. Patterns we liked (spec-coverage
reconcile, EARS, contract-first, repro→root-cause debugging, and [`ponytail`](https://github.com/DietrichGebert/ponytail)'s
YAGNI/lean-build ladder + its deliberate-simplification ledger) are *encoded here* — the `yagni` review lens,
the lazy-build clause in Implement, and the `ponytail:`-marker harvest — **not** pulled in as skills.

## Reference files

Read the one for the stage you're in:

- `references/intake-and-spec.md` — grill-me handoff, ticket adapters (Jira / Linear / GitHub / freeform),
  PRD synthesis (pure-WHAT, saved to `.prd/<slug>.md`), EARS criteria, systems-touched, confirm-first write-back.
- `references/adaptation-layer.md` — project detection, CI-as-ground-truth, the surface→lens table, tiered
  gate derivation, and the optional `.claude/ship-ready.json` config (a write-through cache of detection).
- `references/canonical-workflow.md` — the full Workflow script template: `meta`, schemas, phases, the
  loop, the harness-owned exit predicate, the `workflow-review-phase` splice, and the model-tier prior.
- `references/multi-repo-contracts.md` — the WorkGraph, contract-first two-level planning, per-target
  fan-out, the integration lens, optional Pact/Specmatic, and how a single repo collapses the machinery.
