# Canonical Workflow

The template for the single autonomous Workflow ship-ready launches after the grill. ship-ready **authors
this script fresh each run**, inlining the PRD path + the systems-touched targets as literals in the INPUTS
block — it does **not** rely on the Workflow `args` channel, which crashes the script the instant it arrives
stringified or empty. The script is **resumable** (the same script replays cached agent results, so a
`needs-human` bounce resumes rather than restarts), and since it can't read files itself, it passes
`prdPath` to the agents (Plan, Cover, ship-gate), which read the PRD.

This is a **template to adapt**, not a frozen script. The review→triage→fix block is **spliced from the
`workflow-review-phase` skill** — invoke `Skill({ skill: 'workflow-review-phase' })` while authoring and
paste its segment at the marked point, rather than hand-rolling reviewers. Keep its validated mechanics:
thermo-nuclear runs as a real `Skill` call inside one phase-agent; code-review is the script's own
`parallel()` fan-out (phase-agents cannot spawn subagents).

## Model tiers (a prior — the orchestrator reassesses per phase, per target)

| Phase | Prior | Why |
|---|---|---|
| Adapt | sonnet | Detection is mechanical-ish; cheap. |
| Plan · contract/unified | **opus** | Cross-cutting design; the seam decisions are costly to get wrong. |
| Plan · per-target sub-plan | sonnet | Conform to a frozen contract + local conventions. |
| Research | sonnet (+ Context7) | Synthesis of docs against conventions. |
| Implement | sonnet | Competent codegen from a precise plan. |
| Review · security / concurrency / integration | **opus** | Subtle, high-stakes reasoning. |
| Review · other lenses | sonnet | Breadth. |
| Triage | **opus** | The gate — decides what actually changes. |
| Fix | sonnet (escalate on repeated criticals) | Implementation from a precise finding. |
| Gate / re-verify / coverage | haiku–sonnet | Runs commands, checks criteria; mostly mechanical. |
| Ship-gate | **opus** | Final judgment + structured verdict. |

Scale by stakes × complexity × blast-radius; in multi-repo, set per target. See the model-assessment
heuristic in `SKILL.md`.

## Schemas

Reuse `FINDINGS`, `TRIAGE`, `FIXED` verbatim from the `workflow-review-phase` skill. Add:

```js
const PROFILE = { /* the ProjectProfile shape from adaptation-layer.md */ }

const PLAN = { type:'object', additionalProperties:false,
  required:['steps','risks','filesLikely'], properties:{
    steps:{ type:'array', items:{ type:'string' } },
    risks:{ type:'array', items:{ type:'string' } },
    filesLikely:{ type:'array', items:{ type:'string' } },
    conformsTo:{ type:['string','null'] } } }            // contract id this sub-plan implements

const CONTRACT_PLAN = { type:'object', additionalProperties:false,
  required:['contracts','sequence','crossCuttingCriteria'], properties:{
    contracts:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['id','between','shape'], properties:{
        id:{type:'string'}, between:{type:'array',items:{type:'string'}},
        shape:{type:'string'}, ownerTarget:{type:'string'} } } },
    sequence:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['target','dependsOn'], properties:{
        target:{type:'string'}, dependsOn:{type:'array',items:{type:'string'}} } } },
    crossCuttingCriteria:{ type:'array', items:{ type:'string' } } } }

const COVERAGE = { type:'object', additionalProperties:false,
  required:['covered','uncovered'], properties:{
    covered:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['id','evidence'], properties:{ id:{type:'string'}, evidence:{type:'string'} } } },
    uncovered:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['id','whatsMissing'], properties:{ id:{type:'string'}, whatsMissing:{type:'string'} } } } } }

// Gate REPORTS per-gate status; the script computes green from it (harness owns the gate, not the model).
const GATE_RESULT = { type:'object', additionalProperties:false, required:['target','gates'], properties:{
  target:{type:'string'},
  gates:{ type:'array', items:{ type:'object', additionalProperties:false,
    required:['name','tier','status','detail'], properties:{
      name:{type:'string'}, tier:{enum:[0,1,2]}, status:{enum:['passed','failed','skipped']},
      detail:{type:'string'} } } } } }

const SHIP_VERDICT = { type:'object', additionalProperties:false,
  required:['status','criteria','gates','perTarget','recommendation'], properties:{
    status:{ enum:['ship','blocked','needs-human'] },
    criteria:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['id','met','evidence'], properties:{
        id:{type:'string'}, met:{type:'boolean'}, evidence:{type:'string'} } } },
    gates:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['name','status','detail'], properties:{
        name:{type:'string'}, status:{enum:['passed','failed','skipped']}, detail:{type:'string'} } } },
    perTarget:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['target','green','residualFindings'], properties:{
        target:{type:'string'}, green:{type:'boolean'},
        residualFindings:{type:'array',items:{type:'string'}} } } },
    integration:{ type:['object','null'], additionalProperties:true },
    risk:{ type:['string','null'] },
    recommendation:{ type:'string' } } }
```

## The script

```js
export const meta = {
  name: 'ship-ready',
  description: 'Autonomous delivery loop: adapt → plan → research → implement → review-loop → ship-gate',
  phases: [
    { title: 'Adapt' }, { title: 'Plan' }, { title: 'Research' }, { title: 'Implement' },
    { title: 'Review' }, { title: 'Triage' }, { title: 'Fix' }, { title: 'Gate' }, { title: 'Ship-gate' },
  ],
}

// ── INPUTS — the orchestrator INLINES these literals when it authors the script. Do NOT read from `args`:
//    a fresh-authored workflow doesn't need it, and a stringified/missing `args` crashes the script on the
//    first nested access. Inlining is also resume-safe (the same script replays identically).
const prdPath    = '.prd/<slug>.md'           // the PRD file (agents read it; the script can't read files)
const targets    = [                          // from the PRD's "Systems touched" — one entry per system
  { name: '<svc>', repoPath: '/abs/path/<svc>', surfacesHint: ['api'] },
]
const multiRepo  = targets.length > 1
const CAP        = 3                            // Standard rigor: max loop iterations
const byName     = (arr, k='target') => Object.fromEntries(arr.filter(Boolean).map(x => [x[k], x]))

// ── ADAPT ── detect a ProjectProfile per target (single-repo ⇒ one) ──────────
// Resilient: retry a transient failure, then fall back to a conservative profile rather than aborting the
// whole run. A fallback has empty `commands` → the gate can't go green → the run ends `blocked`, never
// crashed and never falsely green. (agent() already retries terminal API errors; this guards the null it
// returns after a sustained overload — one blip must not nuke a multi-minute build.)
phase('Adapt')
const adaptPrompt = t =>
  `Detect the ProjectProfile for the repo at ${t.repoPath} (surfaces hint: ${t.surfacesHint}). `
  + `Follow adaptation-layer.md: read CI as the source of truth for commands, detect stack/monorepo/`
  + `conventions/surfaces, and select review lenses from the surface→lens table. Run the repo-integrity `
  + `preflight: resolve imports of the files the change touches and record any referenced-but-absent `
  + `module/config in preflightWarnings (these become in-scope work — never plan as if a non-building tree `
  + `compiles). Return the profile.`
const fallbackProfile = t => ({
  target: t.name, stack: { language: 'unknown' }, commands: {},          // empty commands ⇒ gate can't pass
  surfaces: t.surfacesHint ?? [], lenses: ['bugs', 'claude-md', 'security'],
  preflightWarnings: [`Adapt could not profile ${t.name} (transient failure) — using a fallback; the run will block until verified`],
})
const profiles = byName(await parallel(targets.map(t => async () => {
  for (let attempt = 0; attempt < 3; attempt++) {
    const p = await agent(adaptPrompt(t), { label:`adapt:${t.name}`, phase:'Adapt', model:'sonnet', schema: PROFILE })
    if (p) return { ...p, target: t.name }
  }
  log(`adapt:${t.name} failed after retries — using fallback profile (run will end blocked, not crashed)`)
  return fallbackProfile(t)                                              // never throw; degrade instead
})))

// ── PLAN ── thin unified contract plan, then conforming per-target sub-plans ──
phase('Plan')
const contractPlan = multiRepo
  ? await agent(
      `Read the PRD at ${prdPath}. Produce the UNIFIED contract/sequencing plan across targets `
    + `${targets.map(t=>t.name)}: own ONLY the seams — freeze each contract's shape, list the cross-cutting `
    + `acceptance criteria, and emit the dependency sequence (what must precede what; contract-bearing changes `
    + `first). The PRD gives WHAT and which systems; you derive the technical contracts + DAG here.`,
      { label:'plan:contract', phase:'Plan', model:'opus', effort:'high', schema: CONTRACT_PLAN })
  : { contracts:[], sequence:[{target:targets[0].name, dependsOn:[]}], crossCuttingCriteria:[] }

const subPlans = byName(await parallel(targets.map(t => () =>
  agent(`Plan the implementation for target ${t.name}. Read the PRD at ${prdPath} for scope + acceptance `
      + `criteria. Conform to the frozen contracts ${JSON.stringify(contractPlan.contracts)} and to this `
      + `repo's conventions (${profiles[t.name]?.conventions}).`,
    { label:`plan:${t.name}`, phase:'Plan', model:'sonnet', schema: PLAN }).then(p => ({...p, target:t.name})))))

// ── RESEARCH ── best practices for the libs in use, reconciled with conventions ─
phase('Research')
const research = byName(await parallel(targets.map(t => () =>
  agent(`For target ${t.name} (${profiles[t.name]?.stack?.frameworks}), use Context7 `
      + `(mcp__plugin_context7_context7__*) to pull current best practices for the libraries this change `
      + `touches, plus web where needed. Reconcile with the repo's existing patterns; return concise guidance.`,
    { label:`research:${t.name}`, phase:'Research', model:'sonnet' }).then(r => ({ target:t.name, notes:r })))))

// ── IMPLEMENT ── DAG-ordered; independents in parallel, dependents sequenced ───
phase('Implement')
await implementInDagOrder(targets, contractPlan.sequence, t =>
  agent(`Implement target ${t.name} per its sub-plan ${JSON.stringify(subPlans[t.name])} and research `
      + `${JSON.stringify(research[t.name]?.notes)}, against the FROZEN contract. Match surrounding code and `
      + `the repo's conventions. Write tests-first where this project is test-driven. Do NOT commit. `
      + `Work in repo ${t.repoPath}; if using a git worktree, stage with \`git add -A\` so the diff is real.`,
    { label:`impl:${t.name}`, phase:'Implement', model:'sonnet', effort:'high',
      isolation: multiRepo ? 'worktree' : undefined, schema: FIXED }))

// ── LOOP UNTIL CLEAN ── review → triage → fix → gate → cover; harness owns exit ─
let iteration = 0, lastVerdictInputs = null
while (true) {
  iteration++
  log(`loop iteration ${iteration}/${CAP}`)

  // Per-target review→triage→fix = the workflow-review-phase SPLICE.
  // <<< PASTE the workflow-review-phase segment here, once per target, with:
  //     DIFF = reDeriveDiff(t.repoPath)  VERIFY_CMD = profiles[t.name].commands.verify
  //     LENSES = profiles[t.name].lenses  SCOPE='review+fix'  SCORE_FLOOR=null >>>
  const perTarget = await parallel(targets.map(t => () => runReviewTailForTarget(t, profiles[t.name])))

  // Multi-repo only: a cross-cutting integration lens over the seam (contract conformance).
  const integration = multiRepo
    ? await agent(`Integration review across targets. Verify each side conforms to the frozen contracts `
        + `${JSON.stringify(contractPlan.contracts)} — request/response shapes, event payloads, shared types, `
        + `versioning/back-compat. Flag concrete drift with the exact mismatch.`,
        { label:'review:integration', phase:'Review', model:'opus', effort:'high', schema: FINDINGS })
    : { findings: [] }

  // GATE: re-derive each diff from git (don't trust fix reports) and run the tiered gates.
  // The agent REPORTS per-gate status; the SCRIPT computes green below (the harness owns the gate).
  phase('Gate')
  const gateResults = await parallel(targets.map(t => () =>
    agent(`From ${t.repoPath}, re-derive the diff from git, then run the tiered gates for this profile `
        + `(${JSON.stringify(profiles[t.name]?.commands)}): tier 0 always (typecheck/compile, lint, format, `
        + `unit); tier 1/2 if present (integration, contract, e2e, UI build+smoke). FIRST confirm the target `
        + `compiles — a typecheck/analyze failure caused by a missing imported file is a tier-0 'failed', never `
        + `a 'skipped'. Return every gate with its tier and passed/failed/skipped-and-why. Change nothing.`,
      { label:`gate:${t.name}`, phase:'Gate', model:'haiku', schema: GATE_RESULT })))

  // COVER: completeness axis — reconcile the cumulative diff vs the EARS criteria + the always-on AC0.
  const coverage = await agent(
    `Read the acceptance criteria from the PRD at ${prdPath}. Reconcile the implementation against EACH AC `
  + `(by its id) and the cross-cutting criteria ${JSON.stringify(contractPlan.crossCuttingCriteria)}, PLUS the `
  + `always-on AC0 "the target builds and tier-0 is green". For each, decide covered (evidence: the file/test/`
  + `endpoint that satisfies it) or uncovered (what's missing). Nothing counts as covered on a tree that does `
  + `not compile.`,
    { label:'cover', phase:'Gate', model:'sonnet', effort:'high', schema: COVERAGE })

  // HARNESS-OWNED EXIT — machine-checked, fail-safe. `clean` requires POSITIVE evidence of success, so a
  // missing/null result (a crashed or overloaded agent) reads as NOT clean — never a silent green. Note
  // `[].every()` is `true`, so every "all passed" check is paired with a "results actually exist" check.
  const okGates  = gateResults.filter(Boolean)
  const okReview = perTarget.filter(Boolean)
  const integrationFindings = (integration?.findings ?? []).length
  const triageAccepted = okReview.reduce((n, pt) => n + (pt?.acceptedCount ?? 0), 0)
  const reviewComplete = okReview.length === targets.length             // every target was actually reviewed
  const gatesGreen = okGates.length === targets.length                  // every target produced a gate result…
    && okGates.every(gr => {
         const gs = gr.gates ?? []
         const tier0Ran = gs.some(g => g.tier === 0 && g.status !== 'skipped')   // …with tier-0 actually run
         return tier0Ran && gs.every(g => g.status === 'passed' || (g.status === 'skipped' && g.tier !== 0))
       })
  const allCovered = !!coverage && Array.isArray(coverage.uncovered)    // coverage actually ran…
    && coverage.uncovered.length === 0                                  // …and nothing is left uncovered
  const clean = reviewComplete && triageAccepted === 0 && integrationFindings === 0 && gatesGreen && allCovered

  lastVerdictInputs = { perTarget, integration, gates: gateResults, coverage }
  if (clean) { log('clean on both axes — exiting loop'); break }
  if (iteration >= CAP) { log(`hit cap with residuals — verdict will be blocked`); break }
  // else: uncovered criteria + surviving findings feed the next Fix pass (the review splice applies them).
}

// ── SHIP-GATE ── structured verdict the inline stage branches on ──────────────
phase('Ship-gate')
const verdict = await agent(
  `You are the ship-readiness gate. Read the PRD at ${prdPath}, then decide the verdict from the evidence. status='ship' `
+ `ONLY if every acceptance criterion is met, all applicable gates are green (skips justified), no unresolved `
+ `critical/high findings remain, and no new tech debt was introduced. status='blocked' if residuals remain `
+ `after the loop cap; status='needs-human' if a criterion is contradictory or a decision exceeds the PRD. `
+ `Be concrete: cite evidence per criterion and per gate. Evidence:\n${JSON.stringify(lastVerdictInputs)}`,
  { label:'ship-gate', phase:'Ship-gate', model:'opus', effort:'high', schema: SHIP_VERDICT })

return verdict
```

## Helpers the orchestrator fills in

- `runReviewTailForTarget(t, profile)` — the pasted `workflow-review-phase` segment for one target: the
  thermo-nuclear Skill call + the code-review `parallel()` fan-out over `profile.lenses` + triage + fix +
  re-verify, scoped to `reDeriveDiff(t.repoPath)` and `profile.commands.verify`. It should return at least
  `{ acceptedCount, green, residualFindings }` so the harness exit predicate can read it.
- `reDeriveDiff(repoPath)` — **execute** git and return the diff *text* (not a command string). In a
  worktree, `git add -A` first, then diff against the merge-base:
  `git -C <repoPath> add -A && git -C <repoPath> diff --staged $(git -C <repoPath> merge-base <base> HEAD)`.
  Don't combine `--staged` with a three-dot range, and don't pass an un-executed command string into a
  prompt. Never derive the diff from a fix-agent's reported file list.
- `implementInDagOrder(targets, sequence, fn)` — run targets with no unmet deps via `parallel()`, then the
  next wave, etc. With a single target or an empty DAG it's one `parallel()` of one.

### Authoring rules for the emitted script (avoid the common breakages)

- **Declare every schema you reference, in-file.** A Workflow script is a single module — you cannot
  `require('./schemas/…')`. Inline the schemas you change; for the ones reused verbatim from
  `workflow-review-phase` (`FINDINGS`, `TRIAGE`, `FIXED`) paste them once near the top and reference by
  name, rather than re-inlining their bodies at each use.
- **Every `phase:` string must be a member of `meta.phases`.** Mismatched phase labels (e.g. `'Review'`
  vs `'Review · code-review'`) scatter the progress display; pick one set and use it everywhere.
- **Inline the inputs; don't depend on `args`.** ship-ready authors the script fresh, so bake `prdPath` and
  `targets` in as literals (the INPUTS block). The Workflow `args` channel crashes the script the instant it
  arrives stringified or empty (`args.targets` → `undefined.map`), and inlining is resume-safe.
- **Make Adapt (and any phase) resilient, never fatal.** A transient agent failure — e.g. a 529 overload —
  must not abort the run: retry, then fall back to a conservative profile with empty `commands`, which keeps
  the gate from going green so the run ends `blocked`, not crashed and not falsely green. Don't `throw` on a
  missing `profiles[t.name]`; degrade to a result that *fails* the exit predicate instead.
- **The exit must be fail-safe.** `clean` requires positive evidence (every target reviewed and gated, tier-0
  actually run, coverage actually ran). Empty/null results read as NOT clean — remember `[].every()` is
  `true`, so pair every "all passed" check with a "results actually exist" check.
- **`needs-human` resumes, not restarts.** When the gate returns `needs-human`, the inline stage asks the
  user, then relaunches with `Workflow({ scriptPath, resumeFromRunId })` so completed phases replay from
  cache and only the unresolved tail re-runs.

## Why the exit is the script's job

Letting a model decide "are we done?" invites optimism — it'll call a 90%-done change shipped. The harness
computes `clean` from objective signals (triage accepted count, gate pass/fail, uncovered-criteria count),
so the loop can't terminate on vibes. The ship-gate agent then *explains* the verdict; it doesn't get to
*invent* a green one. This is the difference between "reviewed once" and "ship-ready."
