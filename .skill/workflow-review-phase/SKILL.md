---
name: workflow-review-phase
description: >-
  Emits a ready-to-splice Workflow review→verify→triage→fix segment that runs BOTH a thermo-nuclear
  code-quality review (a real Skill call inside a phase-agent) AND a code-review fan-out that mirrors the
  CURRENT /code-review engine (the five A–E correctness angles always-on + smart-selected specialists, then
  a 3-state recall-biased verify), then triages real-vs-noise and fixes the surviving findings against the
  diff. Use this WHENEVER you are authoring or editing a
  Workflow (the Workflow tool / an "ultracode" run) and need a review phase — when the user says "add the
  review phase", "use the review skill as a phase", "review and fix the diff in the workflow", "run
  thermo-nuclear and code-review", or otherwise wants the changed code reviewed and fixed inside an
  orchestration. Also use for a standalone (non-workflow) review of a branch diff via the nested-subagent
  mode. Prefer this over hand-rolling review agents: it encodes the validated runtime mechanics (a workflow
  phase-agent CAN invoke a skill but CANNOT spawn subagents, so thermo-nuclear runs as a real Skill call
  while code-review must be a script-level parallel() fan-out) and the correct per-stage model tiers.
---

# Workflow Review Phase

This skill gives you a **drop-in Workflow segment** that reviews a diff with two complementary engines,
keeps only the findings that survive scrutiny, and fixes them — all as deterministic `phase()` steps you
splice into whatever Workflow you are writing.

It is a **recipe**, not a runner. Your deliverable when this skill triggers is correct workflow JavaScript
appearing in the script you are authoring (or a complete standalone script), with the rubrics below wired in.

## Why it is shaped this way (validated, not assumed)

A probe across all workflow agent types established two facts that dictate the design:

1. **A workflow phase-agent CAN invoke the `Skill` tool.** So `thermo-nuclear-code-quality-review` — which
   is a self-contained rubric needing no helpers — runs faithfully as a *real Skill call inside one
   phase-agent*. Don't reimplement it; call it.
2. **A workflow phase-agent CANNOT spawn its own subagents** (no `Agent`/`Task` tool in any agent type).
   `code-review` fans out to many independent finder angles + per-candidate verifiers, so it **cannot** run
   inside a single phase-agent — it would stall or silently collapse to one in-context pass. Therefore
   code-review is reproduced as the **workflow script's own `parallel()` fan-out**, where the script (not a
   phase-agent) is the orchestrator. The always-on A–E angle texts, the specialist oracles, and the verbatim
   3-state verdict ladder live in `references/code-review-roster.md` (verbatim source: `references/code-review-engine-2.1.186.md`).

Keep these two mechanics exactly. They are the whole reason the segment is split the way it is.

## Parameters (decide these at the splice site)

| Param | Default | Meaning |
|---|---|---|
| `DIFF` | `'develop...HEAD'` plus uncommitted | What to review. A git range, or the changed-file list from a preceding implement phase (e.g. `impl.filesChanged`). |
| `VERIFY_CMD` | project typecheck + unit tests (e.g. `rtk pnpm typecheck && rtk pnpm -r test`) | Used to confirm fixes; integration/Docker suites excluded unless asked. |
| `SCOPE` | `'review+fix'` | `'review'` stops after triage and returns findings; `'review+fix'` also applies + re-verifies. |
| `ANGLES` | `angle-A…angle-E` (always-on) | The five `/code-review` correctness technique angles, verbatim from the roster. Always run — they replace the old single `bugs` lens. Don't dial these down. |
| `SPECIALISTS` | surface-relevant set | The smart-selected specialists to run alongside the angles: `security`, `data-integrity`, `concurrency`, `infra-safety`, `api-contract`, `public-api` (+ `integration` cross-target). Pick by the diff's surface — or compute a static floor and let a cheap lens-router ADD to it (see roster). `test-integrity` is always-on when the diff carries `@AC` tests. |
| `RUN_VERIFY` | `true` | The 3-state recall-biased verify pre-filter (keep CONFIRMED+PLAUSIBLE, drop REFUTED). Set `false` only for a tiny diff where triage alone is cheap enough. Verify is a scalable pre-filter, **not** the gate — triage is. |
| `RUN_SWEEP` | `true` | One fresh gap-only finder after verify; its candidates are also verified. Turn off for trivial diffs. |
| `MODELS` | the table below | Per-stage model override. |

## Model tiers (defaults — Opus on the judgment, cheap on the mechanical)

| Stage | Model | Why |
|---|---|---|
| Thermo-nuclear review | **opus** | Deep maintainability judgment + ambitious "code-judo" restructuring. The important one. |
| Code-review · security / data-integrity / concurrency / infra-safety / integration | **opus** | High-stakes, subtle reasoning — a missed authz/secret, lost update, or race is costly. |
| Code-review · A–E angles + breadth specialists (`api-contract`, `public-api`, `test-integrity`) | **sonnet** | Faithful to `/code-review` (Sonnet finders); breadth over depth. |
| Verify (3-state, one per candidate) | **sonnet** | Recall-biased pre-filter; reads the file and refutes only the provably-wrong. Cheap, parallel. |
| Triage (real-vs-noise, dedupe) | **opus** | Decides what actually gets changed; the precision gate. |
| Fix-apply ("impl") | **sonnet** | Implementation work — competent codegen from a precise finding. |
| Re-verify | **haiku** | Runs the verify command and reports pass/fail. |

Override per call via `opts.model`; never hardcode a tier that contradicts this table without a reason.

## The segment (splice this into your Workflow script)

Define the schemas once near the top of your script, then paste the phases where the review belongs —
**after** whatever produced the diff (an implement phase, or just the current branch). Read
`references/code-review-roster.md` to build `finderPrompt(key, kind)`, `verifyPrompt(finding)` (the verbatim
3-state ladder), and `sweepPrompt(already)`, and `references/thermo-nuclear-rubric.md` for the fallback rubric
if the skill isn't installed.

```js
// ── SCHEMAS (define once) ───────────────────────────────────────────────
const FINDINGS = { type:'object', additionalProperties:false, required:['findings'], properties:{
  findings:{ type:'array', items:{ type:'object', additionalProperties:false,
    required:['title','file','severity','category','rationale'], properties:{
      title:{type:'string'}, file:{type:'string'}, line:{type:['number','null']},
      severity:{enum:['low','medium','high','critical']}, category:{type:'string'},
      rationale:{type:'string'}, suggestedFix:{type:'string'},
      // failure_scenario = the USER-VISIBLE consequence (error, wrong output, data loss), NOT an
      // intermediate state (value stale, set grows). Carried into VERIFY; optional so any producer stays valid.
      failure_scenario:{type:['string','null']} } } } } }
// VERIFY verdict (3-state, recall-biased) — replaces the old 0–100 SCORE. Ladder text in code-review-roster.md.
const VERDICT = { type:'object', additionalProperties:false, required:['verdict','evidence'],
  properties:{ verdict:{enum:['CONFIRMED','PLAUSIBLE','REFUTED']}, evidence:{type:'string'} } }
const TRIAGE = { type:'object', additionalProperties:false, required:['accepted','rejected','summary'],
  properties:{ accepted:{type:'array',items:FINDINGS.properties.findings.items},
    rejected:{type:'array',items:{type:'object',additionalProperties:false,
      required:['title','reason'],properties:{title:{type:'string'},reason:{type:'string'}}}},
    summary:{type:'string'} } }
const FIXED = { type:'object', additionalProperties:false, required:['applied','verified','filesChanged','notes'],
  properties:{ applied:{type:'array',items:{type:'string'}}, verified:{type:'boolean'},
    filesChanged:{type:'array',items:{type:'string'}}, notes:{type:'string'} } }

// ── INPUTS (set at the splice site) ─────────────────────────────────────
const DIFF = 'develop...HEAD'                    // or: impl.filesChanged.join(', ')
const VERIFY_CMD = 'rtk pnpm typecheck && rtk pnpm -r test'
const SCOPE = 'review+fix'
const RUN_VERIFY = true                           // 3-state recall-biased pre-filter; false only for a tiny diff
const RUN_SWEEP  = true                           // one gap-only finder after verify (its candidates also verified)
// FINDERS: the five A–E correctness angles are ALWAYS-ON (they replace the old `bugs` lens); add the
// smart-selected specialists for this diff's surface (a static floor; a cheap router may ADD to it — see roster).
// Angle texts + specialist oracles + the verdict ladder live in references/code-review-roster.md.
const ANGLES = ['angle-A','angle-B','angle-C','angle-D','angle-E']
const SPECIALISTS = ['security','data-integrity','concurrency','api-contract']   // pick by surface; + `test-integrity` when the diff has @AC tests
const OPUS_KEYS = new Set(['security','data-integrity','concurrency','infra-safety','integration'])
const FINDERS = [
  ...ANGLES.map(k => ({ key:k, kind:'angle', model:'sonnet' })),
  ...SPECIALISTS.map(k => ({ key:k, kind:'specialist', model: OPUS_KEYS.has(k)?'opus':'sonnet' })),
]

// ── REVIEW · thermo-nuclear (REAL Skill call inside one phase-agent) ─────
phase('Review · thermo-nuclear')
const tn = await agent(
  `Invoke the Skill tool now: Skill({ skill: 'thermo-nuclear-code-quality-review' }). Apply its standards `
  + `to the diff (${DIFF}); read the changed files and the surrounding code they touch. If that skill is not `
  + `available, apply the rubric in references/thermo-nuclear-rubric.md of the workflow-review-phase skill. `
  + `Return every maintainability/structure finding as the schema (severity reflects how much it hurts the `
  + `codebase; prefer findings that delete complexity).`,
  { label:'review:thermo-nuclear', phase:'Review · thermo-nuclear', model:'opus', effort:'high', schema: FINDINGS },
)

// ── REVIEW · code-review (script-level fan-out — phase-agents can't nest) ─
// A–E angles + specialists run in parallel. `finderPrompt(key, kind, DIFF)` builds the prompt from the angle
// or specialist-oracle text in references/code-review-roster.md, and tells each finder its failure_scenario
// must be the USER-VISIBLE consequence (error, wrong output, data loss), not an intermediate state.
phase('Review · code-review')
const reviewed = await parallel(FINDERS.map(f => () =>
  agent(finderPrompt(f.key, f.kind, DIFF), { label:`review:${f.key}`, phase:'Review · code-review', model:f.model, schema: FINDINGS })))
let candidates = reviewed.filter(Boolean).flatMap(r => r.findings)

// ── VERIFY (3-state recall-biased; keep CONFIRMED+PLAUSIBLE, drop REFUTED) ─
// A scalable PRE-FILTER that kills only the provably-wrong — NOT the gate (triage is). One verifier per
// candidate, no pre-verify dedup (triage dedups). `verifyPrompt` carries the verbatim ladder from the roster.
if (RUN_VERIFY && candidates.length) {
  phase('Verify')
  const verdicts = await parallel(candidates.map(c => () =>
    agent(verifyPrompt(c, DIFF), { label:'verify', phase:'Verify', model:'sonnet', effort:'medium', schema: VERDICT })))
  candidates = candidates.filter((c,i) => { const v = verdicts[i]; return v && (v.verdict === 'CONFIRMED' || v.verdict === 'PLAUSIBLE') })
}

// ── SWEEP (one fresh gap-only finder; its candidates are ALSO verified) ───
if (RUN_SWEEP) {
  phase('Sweep')
  const already = candidates.map(c => `${c.file}:${c.line ?? ''} — ${c.title}`)
  const swept = await agent(sweepPrompt(already, DIFF), { label:'sweep', phase:'Sweep', model:'sonnet', effort:'medium', schema: FINDINGS })
  let gaps = (swept?.findings ?? [])
  if (RUN_VERIFY && gaps.length) {
    const gv = await parallel(gaps.map(c => () => agent(verifyPrompt(c, DIFF), { label:'verify:sweep', phase:'Sweep', model:'sonnet', schema: VERDICT })))
    gaps = gaps.filter((c,i) => { const v = gv[i]; return v && v.verdict !== 'REFUTED' })
  }
  candidates = candidates.concat(gaps)
}

// ── TRIAGE (dedupe + real-vs-noise against the diff — THE GATE) ──────────
phase('Triage')
const allFindings = [ ...(tn?.findings ?? []), ...candidates ]
const triaged = await agent(
  `Triage these review findings against the diff (${DIFF}). Dedupe overlaps (thermo-nuclear and code-review `
  + `often flag the same line from different angles — merge them). Drop anything that is a pre-existing issue, `
  + `a nitpick a senior engineer wouldn't raise, or something a typechecker/linter/CI already catches. Keep `
  + `only findings that are real, in-diff, and worth changing. Findings:\n${JSON.stringify(allFindings, null, 2)}`,
  { label:'triage', phase:'Triage', model:'opus', effort:'high', schema: TRIAGE },
)
if (SCOPE === 'review') return { findings: triaged, fixed: null, green: null }

// ── FIX (apply accepted findings coherently → re-verify) ────────────────
phase('Fix')
const fixed = await agent(
  `Apply these accepted review findings to the working tree, coherently and minimally — match surrounding `
  + `code, don't over-refactor, don't introduce new behavior. Then run \`${VERIFY_CMD}\` and iterate until it `
  + `passes. Do NOT commit. Accepted findings:\n${JSON.stringify(triaged.accepted, null, 2)}`,
  { label:'fix:apply', phase:'Fix', model:'sonnet', effort:'high', schema: FIXED },
)
const reverify = await agent(
  `From the repo root run \`${VERIFY_CMD}\` and report pass/fail with any failing test names. Change nothing.`,
  { label:'fix:verify', phase:'Fix', model:'haiku', effort:'low', schema: FIXED },
)
return { findings: triaged, fixed, green: reverify?.verified === true }
```

### Notes that keep it correct

- **Triage is the gate; verify is a pre-filter — not a score.** The 3-state verify (recall-biased) only drops
  the provably-wrong (REFUTED); CONFIRMED+PLAUSIBLE flow on to triage, which dedups and decides real-vs-noise
  against the code. Precision belongs at triage (Opus), not at a numeric threshold — an earlier 0–100
  `SCORE_FLOOR` design let the weakest agent silently kill real findings (a real `78` dropped by a floor of
  `80`); the recall-biased verify keeps anything realistically triggerable and lets triage make the call.
- **A–E are always-on; specialists are smart-selected.** The five correctness angles run on every diff (they
  replace the old `bugs` lens). Pick `SPECIALISTS` by surface as a fail-safe floor; a cheap router may only
  ADD to it, never cut it, so a sensitive surface can't be under-reviewed.
- **Fix is one coherent agent, not a per-finding `parallel()`** — accepted findings often touch the same files,
  and parallel editors would clobber each other. If the accepted set is provably file-disjoint and large, you
  may `pipeline()` it with `isolation:'worktree'` instead; otherwise keep the single applier.
- **Return a structured contract** (`{ findings, fixed, green }`) so the *host* workflow can branch on it —
  e.g. fail the run or skip a commit phase when `green !== true` or unfixed `critical`s remain.
- **`DIFF` is the only real coupling.** Pass it the upstream implement phase's `filesChanged`, or a git range.
  Everything else is self-contained, so the segment drops into any pipeline position.
- **Don't widen the thermo-nuclear phase into a fan-out.** It's one agent on purpose (it invokes the real
  skill). Only code-review fans out.

## Standalone mode (no host workflow)

When there is no Workflow to splice into, emit a **complete** script: the same `meta`/schemas/phases above
wrapped in a Workflow with `phases` declared in `meta`, reviewing `DIFF = 'develop...HEAD'`. Alternatively,
since nested subagents shipped (Claude Code v2.1.172, subagents may spawn subagents up to 5 levels), you may
instead drive it from a single top-level `Agent`-tool orchestrator subagent that spawns the code-review
reviewers itself and invokes the thermo-nuclear skill directly — useful outside an orchestration. Prefer the
Workflow form when the user is already "in a workflow."

## Reference files

- `references/code-review-roster.md` — the always-on A–E angle texts, the specialist oracles, the **verbatim**
  3-state verdict ladder, the sweep focus, and the false-positive guidance. Read this to build `finderPrompt`,
  `verifyPrompt`, and `sweepPrompt`. **Required.**
- `references/code-review-engine-2.1.186.md` — the verbatim `/code-review max` engine extracted from the
  Claude Code binary (the source of truth the roster mirrors). Re-sync the roster from here when it drifts.
- `references/thermo-nuclear-rubric.md` — a condensed copy of the thermo-nuclear standards, used only as a
  fallback when the real skill isn't installed. The recipe prefers the real Skill call.
