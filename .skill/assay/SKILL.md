---
name: assay
description: >-
  Turn any unit of work — a freeform feature request OR a Jira/Linear/GitHub issue — into shippable, assayed,
  reviewed, tested code with no bugs and no tech debt. Grill requirements to zero ambiguity and detect the
  project (stack/commands/conventions) at intake, then run one autonomous Workflow: plan, research, implement,
  converge on correctness+completeness behind a machine gate, take ONE advisory maintainability pass, and emit
  a structured ship verdict. Project-agnostic — backend, frontend, full-stack, mobile (incl. Flutter), CLI,
  library, or infra; monolith or microservices; one repo or a feature spanning several. Invoke it to build,
  implement, ship, deliver, or finish a feature, or to work a ticket/issue end to end (e.g. "build the X
  feature", "work ABC-123", "take this to a merge-ready PR") — for a whole unit of work, not a one-line edit,
  a review, or a pure question.
disable-model-invocation: true
---

# assay

Turn a unit of work into code you'd actually merge — correct, tested, reviewed, idiomatic, and complete
against its acceptance criteria — without a human babysitting the middle, and **without the loop burning
tokens re-checking work that's already clean.** This skill is a **universal orchestrator**: it detects the
project it's running in and adapts, instead of assuming any stack, shape, or toolchain.

The headline idea: **resolve all ambiguity up front with an interview, then run an autonomous build that
converges on correctness behind a machine gate — and treats code-cleanliness as advice, not a gate.** A human
is in the loop in exactly two places: the grilling at the start, and a confirm before anything outward
(commit / PR / ticket write) at the end. Everything between is autonomous and structurally gated.

## The shape (three stages)

```
INLINE  ── Intake → GRILL (grill-me) → PRD (WHAT) + ProjectProfile (detected here) → save .prd/ ──┐
                                                          (the only human-facing stage)            ▼  author Workflow (CONFIG inlined)
WORKFLOW ── Plan → Research → Implement → ⟲ A: converge(correctness+completeness) → B: polish×1 (advisory) → Ship-gate ──┐
                                                              (one Workflow, autonomous, machine-gated)                  ▼
INLINE  ── present ShipVerdict (+ simplifications ledger) → confirm-first commit / PR / ticket write-back
```

Two things changed from the obvious design, both because a real run proved them wasteful:

- **There is no separate "Adapt" phase.** The inline agent is already reading the repo to grill it, so it
  produces the `ProjectProfile` *there* and inlines it as Workflow config. (Detection-as-a-phase only survives
  for multi-repo targets the inline stage didn't read.)
- **The "loop until clean" is split into two clocks.** Correctness+completeness *converge fast and gate*;
  maintainability/DRY is *bottomless and must not gate*. So **Mechanism A** converges the first; **Mechanism B**
  is a single non-blocking polish pass that emits an advisory ledger.

Depth lives in `references/` — read the one for the stage you're in.

## Requirements & degraded fallback

Stage 2 runs as a Claude Code **Workflow** (the `agent()`/`parallel()`/`pipeline()`/`phase()` primitives), which
requires **Claude Code ≥ 2.1.154 on a paid plan**. The skills install from the bundle's `.skill/` into the
auto-discovered `~/.claude/skills/` (`cp -R .skill/* ~/.claude/skills/`) — the model can't load them from `.skill/`
in place. **Context7** (an MCP, for Research) and **`/code-review`** (ships with Claude Code) are also expected.

If the Workflow tool is unavailable (older CLI, free plan), **degrade — don't fail**: drive the same
Plan → Research → Implement → converge → polish → ship-gate flow from a single top-level `Agent`-tool orchestrator
that spawns the review fan-out itself (nested subagents shipped in **v2.1.172**, 5-level limit) and invokes
thermo-nuclear directly. You lose the *script-decided* exit (a model judges "converged" instead of the harness), so
say which path ran in the verdict — but the work still ships. (The phase-agent-can't-nest constraint that motivates
the script-level `parallel()` fan-out applies *inside* a Workflow; the top-level Agent fallback **can** nest, which
is exactly why it works.)

## Non-negotiable: grill first

**Always run `grill-me` before anything else** — it is the load-bearing reason the rest can be autonomous. A
Workflow phase-agent cannot stop to ask the user a question mid-run, so every ambiguity has to die before the
Workflow starts:

```
Skill({ skill: 'grill-me' })
```

`grill-me` runs a `/grilling` session under the hood; it's normally-invocable, so call it via the Skill tool.
(assay itself is `disable-model-invocation`: the **user** starts it with `/assay`.) Seed it with
everything you know (ticket body, request, repo scan) so it targets real gaps. It self-scales. It must nail:
**scope**, **the target set** (which repos/services, each with a local path), **repo integrity** (does each
target build *today* — missing imports/config are in-scope work, not gate surprises), the **contracts** between
targets, **acceptance criteria**, and **non-obvious constraints**.

## Intake also produces the ProjectProfile (no Adapt phase)

While grilling, the inline agent is already reading the repo — so **detect the `ProjectProfile` here** and inline
it into the Workflow config: stack, package manager, the **real gate commands read from CI as ground truth**,
**which commands actually exist** (so a missing lint script becomes `not-applicable`, never a phantom gate
failure), conventions (the sibling modules to mirror), surfaces, and a `file→lens` map. Detail:
`references/adaptation-layer.md`. For multi-repo, splice a lean parallel Adapt into the Workflow only for targets
not profiled at intake.

## Synthesize the PRD (pure WHAT) — with testable criteria

Synthesize the interview into a lean, business-level PRD and **save it to `.prd/<slug>.md`**. Pure **WHAT, not
HOW**: problem, solution, user stories, acceptance criteria, scope, systems touched. Technical planning
(contracts, interfaces, sequencing) stays in the Workflow's Plan so it's decided once. (We write our own PRD;
we don't pull Pocock's *technical* `to-prd`.)

- **`acceptanceCriteria` in EARS form**, numbered `AC1..ACn`. Each AC will be mapped to **≥1 test assertion** in
  Implement, so completeness is **machine-checked by running the AC-tagged tests** — not re-judged by an agent
  every loop. Where a criterion genuinely can't be asserted on this stack, it's flagged `untestable` and the
  ship-gate judges it (it never silently passes).
- **Systems touched** — the repos/services the work spans (each with its local path). One system ⇒ the
  multi-repo machinery collapses away.

Then author the Workflow, filling **only the CONFIG block** (`prdPath`, `targets` *with their profiles*,
contracts, caps) and pasting the mechanism verbatim. Don't use the `args` channel — a stringified/empty `args`
crashes the script on first access, and inlining is resume-safe.

## Stage 2 — the canonical Workflow (autonomous)

Launch **one** Workflow from the template in `references/canonical-workflow.md`. It is a **frozen mechanism +
a CONFIG block**: you fill config, you paste the loop/predicate/disposition-cache/gate verbatim (re-deriving
them per run is what produced the historical footguns). Phases:

1. **Plan** — (multi-repo) freeze the contracts + dependency DAG, then per-target plans; **map each AC to a
   concrete test assertion**. Conform to each repo's conventions — *the code is the oracle; CLAUDE.md is a hint
   and the code wins on conflict.*
2. **Research** — current best practices for the *actual* libraries via **Context7**, reconciled with repo
   conventions, so review isn't fighting nonidiomatic code later.
3. **Implement** — per target in DAG order, against the frozen contract. **Build lazy (YAGNI):** stdlib →
   native → installed dep → one line → only then new code; no unrequested abstractions; never simplify away
   validation/error-handling/security/PRD requirements. **Mirror the sibling code.** Write the AC-tagged tests.
   Mark deliberate shortcuts with a `ponytail: <ceiling>, <upgrade>` comment.
4. **Mechanism A — converge correctness + completeness** (the gate; see below).
5. **Mechanism B — one advisory polish pass** (maintainability; never gates; see below).
6. **Ship-gate** — a final agent emits a structured `ShipVerdict`.

Return the `ShipVerdict` so the inline stage can branch on it.

### Mechanism A — converge correctness + completeness (the real gate)

A loop of **Stage → Review → Triage → Fix → Gate**, where **the script (not a model) decides whether to loop
again.** What makes it converge fast *and* without waste:

- **Delta- and surface-scoped review.** Pass 1: the focused fan-out over the surfaces present. Pass 2+: review
  **only the files the last fix changed**, with **only the lenses whose surface those files touch.** (A DRY-only
  fix re-runs no security/data reviewers.) Review is `parallel()` in the script; thermo-nuclear is **not** in
  this loop (it's Mechanism B).
- **Reality-anchored lenses only.** The five `/code-review` correctness angles `angle-A`…`angle-E` and
  `test-integrity` are the always-on core (every pass, every target, every lap); `security`/`data-integrity`/
  `concurrency`/`infra-safety`/`api-contract`/`public-api` are SMART-selected by a static surface/`fileLensMap`
  **floor** (fail-safe) UNION a cheap **lens-router** that may only ADD to the floor, never remove; `public-api`
  on the library surface; `integration` across targets when multi-repo. `a11y`/`visual-state`/`git-history` are
  cut entirely (`angle-B` owns git-history's removed-behavior job); the artifact-anchored lenses are gone from the
  gate (see *Guardrails*). The A–E angles live in the `CORRECTNESS_ANGLES` registry (verbatim from the engine) and
  the specialist keys in `LENS_DEF`, so the review prompt carries real oracle guidance, not a bare adjective. After
  the finders, a 3-state recall-biased verify drops only REFUTED candidates before triage.
- **Incremental triage with memory.** A `disposed` cache carries verdicts across passes; **only new findings
  reach triage.** This stops the loop re-litigating the same nits (and re-accepting a finding it earlier
  rejected). **Triage is the gate** — keyed off `accepted == 0`, never a confidence score.
- **A mechanical, N/A-aware gate + test-backed completeness.** Tier-0 (typecheck/lint/format/unit) always; tier
  1/2 if present; **plus the AC-tagged tests** (completeness folded in — no separate coverage agent). The agent
  *reports* per-gate status (`passed/failed/skipped/not-applicable`); the **script computes `green`**: a missing
  command is `not-applicable` and never blocks, a tier-0 command the env can't run is `failed` and never
  silently green.
- **Exit (harness-owned, fail-safe):** `accepted == 0` **and** tier-0 actually passed **and** no AC test is
  failing. Positive evidence required — a null/empty result reads as *not* clean. Cap is a backstop; the
  predicate exits first. Hitting the cap returns `blocked` with residuals — never a silent green.

### Mechanism B — the advisory polish pass (one, non-blocking)

After A is green, run maintainability **once**: thermo-nuclear + `yagni`/simplify on the full diff, plus
`efficiency` (wasted work / repeated I/O / sequential-independent ops / hot-path-blocking / closure-capture memory
leaks), a reverse-direction **doc-drift** check (CLAUDE.md/comments the code now contradicts — the code is right,
the doc is stale), and a forward-direction **conventions** check (code that breaks a quoted exact CLAUDE.md rule;
advisory only, never gates, consistent with `claude-md` being cut from the gate). Apply
the cheap, obviously-worth-it cleanups (DRY extractions), re-verify tier-0, and **roll everything else into an
advisory ledger** on the verdict: the `ponytail:` markers (with ceiling + upgrade trigger), deferred findings,
and drift notes. This is **accepted, tracked debt** — it does not gate the exit (a clean run ships with it
listed), it just keeps a deferral from silently becoming permanent.

## Stage 3 — confirm-first close-out

Present the `ShipVerdict` (with its simplifications ledger). Then, **only with confirmation**, perform outward
actions: commit / push / open a PR, and (for ticket work) post a summary and transition status. Drafting is
fine; publishing is outward-facing and needs an explicit go. `blocked` / `needs-human` stop here and surface
exactly what's unresolved (`needs-human` *resumes* the Workflow via `resumeFromRunId`, it doesn't restart).

## Model usage — assess per phase, don't hardcode

Choose each phase's model/effort by **stakes × complexity × blast-radius**; the table in
`references/canonical-workflow.md` is a *prior*. Deep judgment (plan, security/data/concurrency lenses, triage,
ship-gate) → top tier; implement → top tier (codegen is the product); fix/research → mid; gate/re-verify/stage →
cheap. Escalate mid-loop if signal demands it.

**Cheap mode for small tasks.** assay is a heavyweight orchestrator; a real unit of work can run into the
millions of tokens. Cost already scales down on its own for small single-repo changes (surface-gating runs fewer
specialists; delta laps shrink), but for a low-blast-radius task you can also drop the whole tier prior a notch —
a one-file change doesn't need an Opus triage every lap. Reserve the heavy tiers for genuine blast-radius, and
steer one-line edits to a plain prompt rather than this skill.

## Guardrails (load-bearing — keep them)

- **Grill first, always.** No skipping to implementation, however clear the request looks.
- **Two clocks: gate correctness, advise on cleanliness.** Correctness+completeness block ship; DRY/maintainability
  never does — it's one capped pass plus a ledger. Don't let the bottomless axis hold the cheap one hostage.
- **Triage is the gate, never a threshold — and it has memory.** Every *new* finding goes to triage; dispositioned
  findings never re-enter. The strongest agent decides; a score floor lets the weakest silently kill a real one.
- **Review the delta, not the world.** After pass 1, review only what the last fix changed, with only the
  surface-relevant lenses. Re-scanning unchanged code is the single biggest measured waste.
- **A lens is only as good as its oracle.** Keep lenses that judge the code against execution semantics (the
  `angle-A`…`angle-E` correctness core / `api-contract`/`security`/`data-integrity`/`concurrency`). Drop the ones
  anchored on lagging artifacts: `prior-prs` (unrelated/hallucinated in a team), `code-comments` (comments rot),
  and `claude-md` *as a gate* (docs lag the code); `a11y`/`visual-state`/`git-history` are cut entirely. Enforce
  conventions by **prevention** (Implement mirrors the sibling code; code wins over CLAUDE.md); surface
  reverse-direction doc/comment **drift** and forward-direction `conventions` (quote-the-exact-rule) as advisory
  Mechanism-B notes, never gate findings.
- **Completeness is test-backed — and the tests are audited.** Each AC has ≥1 tagged assertion; the gate runs
  them. Because the same agent writes the code *and* the test, a `test-integrity` lens checks each `@AC` test
  actually exercises its criterion (a tautology becomes a triage finding), and the `untestable`-AC ratio is
  capped: past `MAX_UNTESTABLE_RATIO` the script downgrades `ship` to `needs-human`. Untestable criteria defer
  to the ship-gate, flagged — never a silent pass.
- **A blocking mid-build unknown is asked, not guessed.** The grill kills *known* ambiguity up front; an
  irreversible or product-semantic unknown *discovered* during plan/implement that the PRD doesn't resolve
  returns a `needs-human` verdict with the question **immediately** (carrying `resumeFromRunId`), instead of
  guessing or deferring to the end. Reversible, non-semantic choices are resolved from the sibling code and
  flagged with a `ponytail:` marker.
- **The gate is mechanical and honest about N/A.** Real commands run; the script computes `green`; a command that
  doesn't exist is `not-applicable` (never blocks); a tier-0 command the env can't run is `failed` (never green).
- **Re-derive the diff from git each pass.** Trust git, not a fix-agent's reported file list. In a worktree,
  `git add -A` before diffing.
- **Frozen mechanism, config-only authoring.** Fill the CONFIG block; paste the loop/predicate/cache/gate verbatim.
- **A non-building repo is never done; `needs-human` resumes; confirm before any outward write.**

## Dependencies (deliberately lean — don't add skills)

Hard dependencies, all present: **`grill-me` + `grilling`** (the interview), **Context7** (library docs, an MCP),
and the review stack — **`workflow-review-phase`** (the spliced review→triage→fix interior),
**`thermo-nuclear-code-quality-review`** (Mechanism B), and the CURRENT **`/code-review`** angles (A–E) + 3-state
recall-biased verify, which Mechanism A MIRRORS at the script level (source of truth:
`workflow-review-phase/references/code-review-engine-2.1.186.md`).
Everything else (superpowers, cc-sdd, Pocock's `tdd`/`diagnosing-bugs`, Pact/Specmatic) is
**detect-and-use-if-present**, never installed. Patterns we liked are *encoded here, not pulled in as skills*:
spec-coverage reconcile, EARS, contract-first, repro→root-cause debugging, and
[`ponytail`](https://github.com/DietrichGebert/ponytail)'s YAGNI ladder + deliberate-simplification ledger
(the `yagni` lens, the lazy-build clause in Implement, and the `ponytail:`-marker harvest).

## Reference files

- `references/intake-and-spec.md` — grill-me handoff, ticket adapters, **ProjectProfile detection at intake**,
  PRD synthesis (pure-WHAT, `.prd/<slug>.md`), EARS criteria + **AC→test mapping**, confirm-first write-back.
- `references/adaptation-layer.md` — project detection, CI-as-ground-truth, the **oracle-reliability lens ranking**,
  the surface→lens map, `cmdExists`/tiered gates, and the optional `.claude/assay.json` cache.
- `references/canonical-workflow.md` — the v2 Workflow template: CONFIG + frozen mechanism, schemas, **Mechanism A
  (converge) and B (polish)**, the disposition cache, delta-scoping, the mechanical gate, and the model-tier prior.
- `references/multi-repo-contracts.md` — the WorkGraph, contract-first two-level planning, per-target fan-out, the
  integration lens, and how a single repo collapses the machinery.
