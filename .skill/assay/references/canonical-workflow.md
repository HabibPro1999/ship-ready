# Canonical Workflow (v2)

The single autonomous Workflow assay launches after the grill. **v2 is a rearchitecture** driven by
measured waste on a real run (a 3.7M-token, 2-hour build whose loop re-scanned a clean feature three times).
The redesign is built on six load-bearing decisions, each tied to evidence:

1. **Two clocks, not one loop.** Correctness+completeness *converge fast and must gate*; maintainability/DRY
   is *bottomless and must not gate*. They are split into **Mechanism A** (a blocking convergence loop) and
   **Mechanism B** (one non-blocking advisory polish pass).
2. **No Adapt phase (single-repo).** The `ProjectProfile` is detected at **intake** (the inline agent is
   already reading the repo to grill) and **inlined as config**. A lean per-target Adapt survives only for
   multi-repo targets the inline stage didn't profile.
3. **Delta- and surface-scoped review.** Lap 1 runs the focused fan-out; lap 2+ reviews **only the files the
   last fix changed**, with **only the lenses whose surface those files touch**.
4. **Incremental triage with memory.** A `disposed` cache carries verdicts across passes; only *new* findings
   reach triage. (On the studied run the same false positive was re-found 3× and wrongly accepted on lap 3 —
   memory prevents both.) The cache is **per-target** (a file path can collide across repos).
5. **Test-backed completeness + a mechanical, N/A-aware gate.** Each EARS AC carries ≥1 tagged assertion;
   "complete" = the AC-tagged tests pass, folded into the gate. The script computes `green`; a command that
   doesn't exist is `not-applicable` (never blocks), an env-blocked tier-0 is `failed` (never silently green).
   A `test-integrity` lens audits that each `@AC` test actually *exercises* its criterion (so a tautological
   green can't ship), and the `untestable`-AC ratio is a script-owned guard, not a free escape hatch.
6. **Reality-anchored lenses.** A lens is only as good as its oracle. Keep the lenses that judge the code
   against *execution semantics*; cut the ones anchored on lagging artifacts (`prior-prs`, `code-comments`)
   and the surface-narrow UI/blame lenses (`a11y`, `visual-state`, `git-history` — `git-history`'s "this diff
   dropped a guard" job is now owned by always-on `angle-B`); re-anchor convention-checking from CLAUDE.md to
   **the sibling code** and move it to *prevention* in Implement.

Two safety mechanisms ride on top of the six decisions: a **per-target convergence loop** (Mechanism A runs
once per target each round, plus a cross-target **integration pass** when `multiRepo` — so a 3-service feature
is reviewed and gated on *every* side, not just the first), and a **mid-build open-question channel** (an
autonomous agent that hits a genuinely irreversible/product-semantic unknown stops the run *early* with a
`needs-human` verdict carrying the question, instead of guessing or aborting at the very end — see below).

> **Frozen mechanism + config.** The orchestrator fills **only the CONFIG block** (PRD path, targets+profiles,
> contracts, caps) and pastes the rest **verbatim**. The loop, the exit predicate, the disposition cache, the
> delta-derivation, `implementInDagOrder`, `convergeTarget`, `integrationPass`, and the gate computation are
> tested mechanism — re-deriving them per run is what produced the historical footguns (the `args`-channel
> crash, the `[].every()` trap, the skipped-vs-N/A gate bug, the missing-helper-body footgun).
> The `review→triage→fix` interior is still **spliced from `workflow-review-phase`** for its schemas
> (`FINDINGS`/`TRIAGE`/`FIXED`); Mechanism A drives the review with **inline, oracle-anchored prompts from the
> `CORRECTNESS_ANGLES` (A–E, verbatim from `/code-review`) + `LENS_DEF` (specialists/completeness) registries**
> (so every emitted lens key has guidance without depending on that skill's default roster, which still ships
> the cut lenses). The lens set per pass = a static surface/`fileLensMap` **floor** (fail-safe) UNION a cheap
> **lens-router**'s ADD-only specialist picks; the router selects reviewers but never decides done-ness. Mechanism
> A mirrors the CURRENT `/code-review` angles + 3-state recall-biased verify at the script level (source of truth:
> `../../workflow-review-phase/references/code-review-engine-2.1.186.md`). Keep its two validated mechanics:
> thermo-nuclear is a real `Skill` call in one phase-agent, code-review is the script's own `parallel()` fan-out
> (phase-agents can't spawn subagents).

## Model tiers (a prior — reassess per phase × blast-radius)

| Phase | Prior | Why |
|---|---|---|
| Plan · contract/unified | **opus** | Cross-cutting seam decisions; costly to get wrong. |
| Plan · per-target | sonnet | Conform to a frozen contract + local conventions. |
| Research | sonnet (+Context7) | Doc synthesis. |
| Implement | **opus** | Codegen quality is the product; mirrors sibling code, writes AC tests. |
| Review · lens-router | sonnet · **low effort** | Cheap ADD-only specialist selector; reads the change, extends the floor, never gates. |
| Review · angle-A…E (always-on correctness core) / test-integrity | sonnet | A–E technique angles + completeness, every pass, every target, every lap. |
| Review · api-contract / public-api | sonnet | Surface-gated specialists; breadth on the change. |
| Review · security / data-integrity / concurrency / infra-safety / integration | **opus** | Subtle, high-blast-radius — floor- or router-selected on the relevant surface. |
| Triage | **opus** | The gate — decides what changes. Only *new* findings. |
| Fix | sonnet (escalate on repeated criticals) | Implement a precise finding. |
| Gate / re-verify / stage | haiku | Runs commands; the **script** computes green. |
| Polish (thermo + yagni + efficiency + doc-drift + conventions) | sonnet–opus, **once** | Advisory; never loops; never gates. |
| Ship-gate | **opus** | Final judgment + structured verdict. |

## The lens roster — reality-anchored, surface-gated

A lens earns its place by **(blast-radius of a miss) × (reliability of its oracle)**, gated to the surfaces
where the miss is possible — not by raw yield. (See `adaptation-layer.md` §4 for the oracle-reliability
ranking and the full rationale for each keep/cut.) **Every key below is defined once in the `LENS_DEF`
registry** in the script — the docs describe that registry, they don't independently re-name lenses.

| Bucket | Lenses | When |
|---|---|---|
| **Correctness core** (always-on) | `angle-A`…`angle-E` (the five `/code-review` technique angles, verbatim) | EVERY pass, EVERY target, every lap — they replace the old vague `bugs` lens |
| **Completeness core** (always-on) | `test-integrity` | always (completeness is always gated) |
| **Specialists** (SMART-selected) | `security`, `data-integrity`, `concurrency`, `infra-safety`, `api-contract`, `public-api` | the static **floor** (surface/`fileLensMap`) UNION the cheap **lens-router**'s ADD-only picks |
| **Multi-repo** | `integration` (cross-target contract conformance) | spans ≥2 targets — runs once per round, after the per-target passes |
| **Polish** (advisory, non-gating) | `thermo-nuclear`, `yagni`, `efficiency`, `doc-drift` (reverse) + `conventions` (forward) | Mechanism B, once |
| **Cut entirely** | ~~`git-history`~~, ~~`a11y`~~, ~~`visual-state`~~ | removed everywhere — `angle-B` (removed-behavior) absorbs git-history's "this diff dropped a guard" job |
| **Cut from the gate** | ~~`prior-prs`~~, ~~`code-comments`~~, ~~`claude-md`~~ | weak/rotting oracle — see below |

Convention-following is handled by **prevention** (Implement mirrors the sibling code; CLAUDE.md is a hint and
**the code wins on conflict**), and doc/comment **drift** is surfaced as an advisory ledger note in Mechanism B
— never as a blocking gate finding. `efficiency` adds the one quality-clock capability gap thermo+yagni don't
cover (wasted work, sequential-independent ops, hot-path-blocking, closure-capture memory leaks); `conventions`
is the forward pair to `doc-drift` — it flags code that breaks a **quoted exact** CLAUDE.md rule, advisory only,
consistent with `claude-md` being banned from the gate but allowed as a note.

## Schemas

Reuse `FINDINGS`, `TRIAGE`, `FIXED` verbatim from `workflow-review-phase`. Add:

```js
// A stable signature so a dispositioned finding never re-enters triage (incremental triage). Namespaced per target.
const DISPOSITION = { type:'object', additionalProperties:false, required:['key','verdict'], properties:{
  key:{type:'string'},                       // `${file}:${normalizedTitle}` — survives across passes
  verdict:{ enum:['accepted','rejected'] },
  reason:{type:['string','null']} } }

// A mid-build unknown an autonomous agent must NOT guess on. Propagates up to an early `needs-human` exit.
const OPEN_QUESTION = { type:'object', additionalProperties:false, required:['question','target','blocking','recommendation'], properties:{
  question:{type:'string'}, target:{type:'string'}, location:{type:['string','null']},
  options:{type:'array',items:{type:'string'}}, recommendation:{type:'string'},
  blocking:{type:'boolean'} } }     // blocking ⇒ irreversible / product-semantic / security-data-money ⇒ stop and ask

// Plan output: the AC→test map (completeness contract) + derived contracts + any blocking unknowns surfaced while planning.
const PLAN = { type:'object', additionalProperties:false, required:['plan','acMap'], properties:{
  plan:{type:'string'},
  acMap:{ type:'array', items:{ type:'object', additionalProperties:false, required:['ac','test'], properties:{
    ac:{type:'string'}, test:{type:'string'}, untestable:{type:'boolean'} } } },
  contracts:{ type:['array','null'], items:{type:'object', additionalProperties:true} },
  sequence:{ type:['array','null'], items:{type:'object', additionalProperties:true} },
  openQuestions:{ type:'array', items: OPEN_QUESTION } } }

// Implement result = FIXED + any mid-build open questions the implementer hit.
const IMPL_RESULT = { type:'object', additionalProperties:false, required:['applied','verified','filesChanged','notes'], properties:{
  applied:{type:'array',items:{type:'string'}}, verified:{type:'boolean'},
  filesChanged:{type:'array',items:{type:'string'}}, notes:{type:'string'},
  openQuestions:{ type:'array', items: OPEN_QUESTION } } }

// Gate REPORTS raw status; the SCRIPT computes green. `not-applicable` ⇒ the command doesn't exist (never blocks).
const GATE_RESULT = { type:'object', additionalProperties:false, required:['target','gates'], properties:{
  target:{type:'string'},
  gates:{ type:'array', items:{ type:'object', additionalProperties:false,
    required:['name','tier','status','detail'], properties:{
      name:{type:'string'}, tier:{enum:[0,1,2]},
      status:{enum:['passed','failed','skipped','not-applicable']},   // skipped = exists-but-env-blocked
      detail:{type:'string'} } } },
  acTests:{ type:['object','null'], additionalProperties:false,        // completeness, folded into the gate
    properties:{ passed:{type:'array',items:{type:'string'}},          // AC ids whose tagged tests pass
      failing:{type:'array',items:{type:'string'}},                    // AC ids whose tests fail
      untestable:{type:'array',items:{type:'string'}} } } } }          // ACs with no machine assertion (ship-gate judges)

const LEDGER = { type:'object', additionalProperties:false, required:['markers'], properties:{
  markers:{ type:'array', items:{ type:'object', additionalProperties:false,
    required:['kind','location','note','rotRisk'], properties:{
      kind:{enum:['ponytail','doc-drift','convention-gap','deferred-finding']},  // shortcut · stale doc (code right) · code breaks a quoted CLAUDE.md rule · deferred polish
      location:{type:'string'}, note:{type:'string'},
      ceiling:{type:['string','null']}, upgrade:{type:['string','null']}, rotRisk:{type:'boolean'} } } } } }

const SHIP_VERDICT = { type:'object', additionalProperties:false,
  required:['status','criteria','gates','perTarget','recommendation'], properties:{
    status:{ enum:['ship','blocked','needs-human'] },
    criteria:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['id','met','evidence'], properties:{
        id:{type:'string'}, met:{type:'boolean'}, evidence:{type:'string'} } } },
    gates:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['name','status','detail'], properties:{
        name:{type:'string'}, status:{enum:['passed','failed','skipped','not-applicable']}, detail:{type:'string'} } } },
    perTarget:{ type:'array', items:{ type:'object', additionalProperties:false,
      required:['target','green','residualFindings'], properties:{
        target:{type:'string'}, green:{type:'boolean'}, residualFindings:{type:'array',items:{type:'string'}} } } },
    simplifications:{ type:['array','null'], items:{ type:'object', additionalProperties:false,   // the LEDGER (advisory)
      required:['kind','location','note','rotRisk'], properties:{
        kind:{type:'string'}, location:{type:'string'}, note:{type:'string'},
        ceiling:{type:['string','null']}, upgrade:{type:['string','null']}, rotRisk:{type:'boolean'} } } },
    openQuestions:{ type:['array','null'], items: OPEN_QUESTION },     // present on a needs-human verdict
    integration:{ type:['object','null'], additionalProperties:true },
    risk:{ type:['string','null'] }, recommendation:{ type:'string' } } }
```

`simplifications` is **accepted, tracked debt** — it does NOT affect the exit predicate or `status` (a clean
run ships with it listed). It only keeps a deferral from silently becoming permanent.

## The script

```js
export const meta = {
  name: 'assay',
  description: 'Autonomous delivery: plan → research → implement → converge → polish → ship-gate',
  phases: [
    { title: 'Plan' }, { title: 'Research' }, { title: 'Implement' },
    { title: 'Converge' }, { title: 'Polish' }, { title: 'Ship-gate' },
  ],
}

/* ═══════════ CONFIG — the orchestrator fills ONLY this block (from intake). ═══════════
   Everything below it is the FROZEN MECHANISM: paste verbatim. The ProjectProfile is detected at
   INTAKE and inlined here — there is no Adapt phase for single-repo. (Multi-repo: splice a lean
   parallel Adapt before Plan for any target whose profile isn't already inlined.) */
const prdPath = '.prd/<slug>.md'
const targets = [
  { name:'<svc>', repoPath:'/abs/<svc>', surfaces:['api','db'],
    profile:{
      commands:{ typecheck:'…', lint:'…', format:'…', unit:'…', acTests:'…', integration:'…', contract:'…', verify:'…' },
      cmdExists:{ typecheck:true, lint:true, format:false, unit:true, acTests:true, integration:true, contract:false }, // false ⇒ not-applicable
      conventions:'mirror <sibling modules>; the CODE wins over CLAUDE.md on conflict',
      fileLensMap:[ ['controller', ['api-contract','security']], ['schema', ['data-integrity']],
                    ['migration', ['data-integrity']], ['async', ['concurrency']], ['txn', ['concurrency']] ],
                    // path-substr → SPECIALIST lenses ONLY. A–E + test-integrity are the always-on core (the floor adds them);
                    // never list them here. A '' catch-all is forbidden — the core already covers every file.
      buildsClean:true } },
]
const contracts = []                                   // multi-repo seams (Plan derives); [] single-repo
const sequence  = [{ target: targets[0].name, dependsOn: [] }]
const CAP_A  = 4                                        // correctness-convergence backstop (predicate exits first)
const POLISH = 1                                        // maintainability passes — hard cap, NON-blocking
const MAX_UNTESTABLE_RATIO = 0.34                       // > this ⇒ ship-gate can't return 'ship' (untestable escape-hatch guard)
const SMALL_DELTA = 3                                   // a delta lap touching ≤ this many files is "small" ⇒ skip VERIFY (triage is still the precision gate)
const SWEEP_MAX   = 8                                   // gap-only sweep candidate cap (mirrors /code-review SWEEP_MAX)

/* ═══════════ FROZEN MECHANISM (paste verbatim) ═══════════ */
const multiRepo = targets.length > 1
const byName = (a, k='name') => Object.fromEntries(a.filter(Boolean).map(x => [x[k], x]))
async function aretry(prompt, opts, tries=3) {        // one transient 529 must not abort a multi-minute build
  let r = null
  for (let i=0;i<tries;i++){ r = await agent(i? `${prompt}\n\n(retry ${i+1}/${tries} after empty result)`:prompt, opts); if (r) return r }
  return r
}

// ── CORRECTNESS_ANGLES: the FIVE /code-review technique angles, VERBATIM from code-review-engine-2.1.186.md.
//    Always-on core — they run on EVERY pass, EVERY target, every lap. They REPLACE the old single vague 'bugs' lens. ──
const CORRECTNESS_ANGLES = {
  'angle-A':'Read every hunk in the diff, line by line. Then Read the enclosing function for each hunk — bugs in unchanged lines of a touched function are in scope (the PR re-exposes or fails to fix them). For every line ask: what input, state, timing, or platform makes this line wrong? Look for inverted/wrong conditions, off-by-one, null/undefined deref, missing `await`, falsy-zero checks, wrong-variable copy-paste, error swallowed in catch, unescaped regex metachars.',
  'angle-B':'For every line the diff DELETES or replaces, name the invariant or behavior it enforced, then search the new code for where that invariant is re-established. If you can\'t find it, that\'s a candidate: a removed guard, a dropped error path, a narrowed validation, a deleted test that was covering a real case.',
  'angle-C':'For each function the diff changes, find its callers (Grep for the symbol) and check whether the change breaks any call site: a new precondition, a changed return shape, a new exception, a timing/ordering dependency. Also check callees: does a parallel change in the same PR make a call unsafe?',
  'angle-D':'Scan for the classic pitfalls of the diff\'s language/framework — for example: JS falsy-zero, `==` coercion, closure-captured loop var; Python mutable default args, late-binding closures; Go nil-map write, range-var capture; SQL injection; timezone/DST drift; float equality. Flag any instance the diff introduces.',
  'angle-E':'When the PR adds or modifies a type that wraps another (cache, proxy, decorator, adapter): check that every method routes to the wrapped instance and not back through a registry/session/global — e.g. a caching provider holding a `delegate` field that resolves IDs via `session.get(...)` instead of `delegate.get(...)` will re-enter the cache or recurse. Also check that the wrapper forwards all the methods the callers actually use.',
}

// ── LENS_DEF: SPECIALIST + completeness lenses only (no 'bugs' — A–E own correctness now; no a11y/visual-state/git-history — cut). ──
const LENS_DEF = {
  'api-contract':'request/response shapes, status codes, error contracts, and back-compat of the changed API surface',
  security:'authz/ownership, injection (SQL/cmd/path), secrets/tokens, SSRF, unsafe deserialization, and trust-boundary input validation; for any public/guest or money route the server must resolve price/totals/owner/identity from its own data, never trust client-supplied fields (flag a client-trusted amount/owner as a tampering/IDOR finding)',
  'data-integrity':'schema/migration safety, constraints, transactional atomicity, lost-update and partial-write hazards',
  concurrency:'races, TOCTOU, lock-ordering, non-atomic read-modify-write, idempotency/retry hazards — reason explicitly about interleavings',
  'infra-safety':'destructive or irreversible infra changes, state drift, secrets in state, and the blast radius of the plan/apply',
  'public-api':'exported/library API surface: stability, semver impact, accidental breaking changes',
  integration:'CROSS-TARGET contract conformance — request/response shapes, event payloads, shared-type alignment, status codes, versioning/back-compat between targets; name the exact mismatch and WHICH side diverges',
  'test-integrity':'for each @AC-tagged test: does it invoke the REAL code path, assert the SPECIFIC behavior the AC names (not a tautology, not just "is defined", not asserting a value against itself), and would it FAIL if that behavior regressed? Flag vacuous / self-referential AC tests as high findings',
}
// shared methodology note so specialists inherit A–E rigor (hunt by tracing callers / auditing deletions / scanning line-by-line)
const SPECIALIST_METHOD = 'Hunt like the correctness angles: trace callers (Grep the symbol), audit every deleted/replaced line for a dropped guard, and scan changed lines one by one — do not eyeball.'
const OPUS_LENSES = new Set(['security','concurrency','data-integrity','infra-safety','integration'])
const lensObj = k => ({ key:k, model: OPUS_LENSES.has(k)?'opus':'sonnet' })

// ── always-on core + specialist candidates ──
const CORE_KEYS = [...Object.keys(CORRECTNESS_ANGLES), 'test-integrity']   // A–E + completeness, ALWAYS on
const SPECIALIST_KEYS = ['security','concurrency','data-integrity','infra-safety','api-contract','public-api']

// reality-anchored, surface-gated SPECIALIST selection (no ui mapping — a11y/visual-state cut)
const SURFACE_LENS = { api:['api-contract','security'], db:['data-integrity'], async:['concurrency'],
  infra:['infra-safety'], library:['public-api'] }

// FLOOR = fail-safe static selection. A sensitive surface can NEVER be under-reviewed. The router may ADD to this, never cut it.
function floorFor(surfaces) {                          // lap-1: core + surface-gated specialists
  const keys = new Set(CORE_KEYS)
  surfaces.flatMap(s => SURFACE_LENS[s] || []).forEach(k => keys.add(k))
  return keys
}
function floorForFiles(changedFiles, profile) {        // delta laps: core + specialists whose path-substr the change touches
  const keys = new Set(CORE_KEYS)
  for (const f of changedFiles) for (const [sub, ls] of profile.fileLensMap) if (sub !== '' && f.includes(sub)) ls.forEach(k=>keys.add(k))
  return keys
}
// lensesFor/lensesForFiles now return the floor Set (core + floor specialists) for the router to extend.
function lensesFor(surfaces)         { return floorFor(surfaces) }
function lensesForFiles(cf, profile) { return floorForFiles(cf, profile) }

// ── lens ROUTER: a CHEAP agent reads the actual change and returns the specialists THIS change needs.
//    It may ONLY ADD to the floor (union); it can NEVER remove a floor lens, so a wrong router can't make the run unsafe.
//    It justifies omissions ("when we need them and when we don't"). NOT a gate — triage + the mechanical gate decide done. ──
async function routeLenses(t, changed, floorSet, plan, pass) {
  const router = await aretry(
    `LENS ROUTER (cheap, fail-safe ADD-only). In ${t.repoPath}, look at the changed files ${JSON.stringify(changed)} `
  + `and a one-paragraph diff summary you derive (read-only). Decide which of these SPECIALIST lenses THIS change needs `
  + `BEYOND the always-on floor ${JSON.stringify([...floorSet])}: ${JSON.stringify(SPECIALIST_KEYS)}. `
  + `ADD any the static floor missed (e.g. a migration in a file not named 'migration', a money route in a generic handler, `
  + `a txn/lock in a file not named 'async'). For each specialist you OMIT, give a one-line justification of when it WOULD be `
  + `needed and why it isn't here. You may ONLY return additions — the floor always runs regardless of what you say. `
  + `Plan context: ${JSON.stringify(plan && plan.plan ? plan.plan.slice(0,800) : '')}.`,
    { label:`route:${t.name}:${pass}`, phase:'Converge', model:'sonnet', effort:'low',
      schema:{ type:'object', additionalProperties:false, required:['add','omitJustification'], properties:{
        add:{ type:'array', items:{ enum: SPECIALIST_KEYS } },
        omitJustification:{ type:'array', items:{ type:'object', additionalProperties:false, required:['lens','reason'],
          properties:{ lens:{ enum: SPECIALIST_KEYS }, reason:{ type:'string' } } } } } } })
  return new Set((router && router.add) || [])           // null router ⇒ empty add ⇒ floor still runs (fail-safe)
}

// union the floor (always) with the router's additions; map to lens objects with model tiers
function unionLenses(floorSet, routerAdd) {
  const keys = new Set(floorSet)
  for (const k of routerAdd) keys.add(k)                 // ADD-only union — floor is never reduced
  return [...keys].map(lensObj)
}
const dispKey = f => `${f.file}:${(f.title||'').toLowerCase().replace(/[^a-z0-9]+/g,' ').trim().slice(0,80)}`

// ── 3-state recall-biased verify (verbatim ladder from code-review 2.1.186). A scalable PRE-FILTER, not the gate. ──
const VERDICT = { type:'object', additionalProperties:false, required:['verdict','evidence'], properties:{
  verdict:{ enum:['CONFIRMED','PLAUSIBLE','REFUTED'] }, evidence:{type:'string'} } }
const VERIFY_LADDER =
  `Return exactly one verdict:\n`
+ `- CONFIRMED — you can name the inputs/state that trigger it and the wrong output or crash. Quote the line.\n`
+ `- PLAUSIBLE — mechanism is real, trigger is uncertain (timing, env, config). State what would confirm it.\n`
+ `- REFUTED — factually wrong (code doesn't say that) or guarded elsewhere. Quote the line that proves it.\n\n`
+ `PLAUSIBLE BY DEFAULT — do NOT refute a candidate for being "speculative" or "depends on runtime state" when the state is realistic: `
+ `concurrency races, nil/undefined on a rare-but-reachable path (error handler, cold cache, missing optional field), falsy-zero treated as missing, `
+ `off-by-one on a boundary the code does not exclude, retry storms / partial failures, regex/allowlist that lost an anchor — these are PLAUSIBLE. `
+ `REFUTED ONLY when constructible from the code: factually wrong (quote the actual line); provably impossible (type/constant/invariant — show it); `
+ `already handled in this diff (cite the guard); or pure style with no observable effect. Keep CONFIRMED + PLAUSIBLE; drop REFUTED.`

// One verifier per candidate (no pre-verify dedup — dedup happens at triage by dispKey). Survivors = CONFIRMED + PLAUSIBLE.
async function verifyCandidates(cands, repoPath, label) {
  if (!cands.length) return cands
  const verdicts = await parallel(cands.map(f => () => agent(
    `Verify ONE candidate finding against the diff in ${repoPath}. Read the relevant file(s) and judge it.\n`
  + `Candidate — ${f.file}:${f.line||''} — ${f.title||f.summary||''}\nFailure scenario: ${f.failure_scenario||'(none stated)'}\n\n`
  + VERIFY_LADDER + ` Evidence must quote or cite the relevant line(s).`,
    { label, phase:'Converge', model:'sonnet', effort:'medium', schema: VERDICT })))
  return cands.filter((f,i) => { const v = verdicts[i]; return v && (v.verdict==='CONFIRMED' || v.verdict==='PLAUSIBLE') })
}

// ── DAG-ordered implement: run targets with no unmet deps concurrently, wave by wave; returns {target → result}. ──
async function implementInDagOrder(targets, sequence, fn) {
  const byTarget = byName(targets, 'name')
  const seq = (sequence && sequence.length) ? sequence : targets.map(t => ({ target:t.name, dependsOn:[] }))
  const done = new Set(), results = {}
  let remaining = seq.slice()
  while (remaining.length) {
    const wave = remaining.filter(s => (s.dependsOn||[]).every(d => done.has(d)))
    if (!wave.length) throw new Error('assay: cyclic/unsatisfiable DAG: ' + JSON.stringify(remaining))
    const out = await parallel(wave.map(s => () => fn(byTarget[s.target])))   // one target / empty DAG ⇒ parallel() of one
    wave.forEach((s,i) => { results[s.target] = out[i]; done.add(s.target) })
    remaining = remaining.filter(s => !wave.includes(s))
  }
  return results
}

// ── one convergence pass for ONE target (extracted so the loop can run N of them per round) ──
async function convergeTarget(t, disposed, cursorMap, pass) {
  phase('Converge')
  const cursor = cursorMap.get(t.name)

  // STAGE + derive the diff to review. Pass 1 for this target: whole change. Pass 2+: ONLY what its last fix changed.
  const view = await aretry(
    `In ${t.repoPath}: \`git add -A\`, then return (a) the full staged file list, and (b) the files `
  + (cursor ? `changed SINCE ${cursor} (this target's last fix).` : `(first pass for this target — all staged files).`)
  + ` Read-only except the add. Return {changedFiles:[...], head:'<sha>'}.`,
    { label:`stage:${t.name}:${pass}`, phase:'Converge', model:'haiku', effort:'low',
      schema:{type:'object',additionalProperties:false,required:['changedFiles','head'],
        properties:{changedFiles:{type:'array',items:{type:'string'}},head:{type:'string'}}} })
  const changed = (view && view.changedFiles) || []

  // REVIEW — SMART selection: static FLOOR (fail-safe) UNION the cheap router's ADD-only specialist picks; A–E + test-integrity always.
  const floorSet = cursor ? lensesForFiles(changed, t.profile) : lensesFor(t.surfaces)
  const routerAdd = await routeLenses(t, changed, floorSet, plan, pass)   // ADD-only; floor never reduced
  const lenses = unionLenses(floorSet, routerAdd)                          // [{key,model}] core + floor specialists + router specialists
  const skipList = [...disposed.keys()].slice(0, 120)
  const reviewed = await parallel(lenses.map(l => () => agent(
    `Review ${cursor?'ONLY these changed files: '+JSON.stringify(changed):'the staged change'} in ${t.repoPath} `
  + `through the ${l.key} lens — ${CORRECTNESS_ANGLES[l.key] || (LENS_DEF[l.key]+' '+SPECIALIST_METHOD)}. `
  + `Judge the CODE against execution semantics (not docs/comments, which rot). `
  + `Each finding's failure_scenario must be the USER-VISIBLE consequence (an error, wrong output, or data loss), `
  + `NOT an intermediate state (a value going stale, a set growing). Pass every candidate with a nameable failure `
  + `scenario through — do not pre-drop half-believed candidates; an independent verifier judges them next. `
  + `Do NOT re-report anything matching these already-decided items: ${JSON.stringify(skipList)}. `
  + `Empty findings is a good result; verify framework defaults before asserting (e.g. status codes).`,
    { label:`review:${l.key}:${t.name}:${pass}`, phase:'Converge', model:l.model, effort:l.model==='opus'?'high':'medium', schema: FINDINGS })))
  let fresh = reviewed.filter(Boolean).flatMap(r=>r.findings||[]).filter(f => !disposed.has(dispKey(f)))

  // VERIFY — 3-state recall-biased pre-filter (CONFIRMED/PLAUSIBLE kept, REFUTED dropped). GATED: it is a scalable
  // pre-filter, NOT the precision gate (triage is). Run on lap-1 (whole change) and on LARGE delta laps; SKIP small
  // delta laps where the candidate set is already tiny and triage alone is cheap enough.
  const runVerify = !cursor || changed.length > SMALL_DELTA
  if (runVerify && fresh.length) fresh = await verifyCandidates(fresh, t.repoPath, `verify:${t.name}:${pass}`)

  // SWEEP — lap-1 ONLY: one fresh gap-only finder for what the angled pass tends to miss; its candidates are ALSO
  // verified, then merged into the fresh set (triage's dispKey dedup absorbs any overlap). No synthesize stage.
  if (!cursor) {
    const already = fresh.map(f => `${f.file}:${f.line||''} — ${f.title||f.summary||''}`).slice(0, 120)
    const swept = await aretry(
      `Re-read the staged diff in ${t.repoPath} and its enclosing functions looking ONLY for defects NOT already listed.\n`
    + `Already-found (do NOT re-derive or re-confirm): ${already.length?JSON.stringify(already):'(none)'}.\n`
    + `Focus on what the first pass misses: moved/extracted code that dropped a guard or anchor; second-tier footguns `
    + `(default evaluated once, hash() non-determinism, lock-scope shrink, predicate methods with side effects); `
    + `setup/teardown asymmetry in tests; config defaults flipped. Surface up to ${SWEEP_MAX} additional findings; `
    + `if nothing new, return an empty list — do not pad.`,
      { label:`sweep:${t.name}:${pass}`, phase:'Converge', model:'sonnet', effort:'medium', schema: FINDINGS })
    const sweepFresh = ((swept&&swept.findings)||[]).slice(0, SWEEP_MAX).filter(f => !disposed.has(dispKey(f)))
    const sweepKept = sweepFresh.length ? await verifyCandidates(sweepFresh, t.repoPath, `verify:sweep:${t.name}:${pass}`) : []
    fresh = fresh.concat(sweepKept)
  }

  // TRIAGE = THE GATE — only verify-survivors (+ verified sweep) reach it. Record every verdict into the disposition cache.
  let accepted = []
  if (fresh.length) {
    const tri = await aretry(
      `Triage these NEW findings against the diff in ${t.repoPath}. Dedupe; drop pre-existing issues, nitpicks, `
    + `false premises (verify the claim against the actual code/framework), and anything a typechecker/linter/CI `
    + `catches. Keep only real, in-diff, worth-changing items. Findings:\n${JSON.stringify(fresh)}`,
      { label:`triage:${t.name}:${pass}`, phase:'Converge', model:'opus', effort:'high', schema: TRIAGE })
    accepted = (tri && tri.accepted) || []
    for (const f of fresh) disposed.set(dispKey(f), {verdict:'rejected'})
    for (const a of accepted) disposed.set(dispKey(a), {verdict:'accepted'})
    if (accepted.length) await aretry(
      `Apply these accepted findings to ${t.repoPath} coherently and minimally — match surrounding code, no new behavior. `
    + `Then run \`${t.profile.commands.verify}\` until tier-0 passes. Do NOT commit. Findings:\n${JSON.stringify(accepted)}`,
      { label:`fix:${t.name}:${pass}`, phase:'Converge', model:'sonnet', effort:'high', schema: FIXED })
  }

  // GATE — mechanical, N/A-aware, completeness folded in (AC-tagged tests). Agent REPORTS; script COMPUTES green.
  const gate = await aretry(
    `From ${t.repoPath}, re-derive the diff (read-only) and run the tiered gates using ${JSON.stringify(t.profile.commands)} `
  + `and existence map ${JSON.stringify(t.profile.cmdExists)}: a command that does NOT exist → status 'not-applicable'; `
  + `a tier-0 command that exists but the env can't run → 'failed' (never 'skipped'); env-blocked tier-1 (Docker, contract) → 'skipped'. `
  + `If commands.contract exists, run it as a tier-1 gate. ALSO run the AC-tagged tests and report, per AC id, passed / failing / untestable. Change nothing.`,
    { label:`gate:${t.name}:${pass}`, phase:'Converge', model:'haiku', effort:'low', schema: GATE_RESULT })

  // COMPUTE green for this target — positive evidence required; null reads as NOT clean.
  const gs = (gate && gate.gates) || []
  const tier0Ran = gs.some(g => g.tier===0 && g.status==='passed')
  const gatesGreen = tier0Ran && gs.every(g => g.status==='passed' || g.status==='not-applicable' || (g.status==='skipped' && g.tier!==0))
  const ac = gate && gate.acTests
  const complete = !!ac && (ac.failing||[]).length===0       // untestable ACs defer to the ship-gate, don't block here
  if (view && view.head) cursorMap.set(t.name, view.head)
  return { target:t.name, accepted, gate, acTests: ac, gatesGreen, complete,
           clean: accepted.length===0 && gatesGreen && complete }
}

// ── cross-target integration pass (multi-repo only): one Opus review of all seams; findings flow through triage→fix ──
async function integrationPass(targets, contracts, disposed) {
  phase('Converge')
  const reviewed = await aretry(
    `INTEGRATION review across targets ${JSON.stringify(targets.map(t=>({name:t.name,repoPath:t.repoPath})))} against the frozen `
  + `contracts ${JSON.stringify(contracts)} — ${LENS_DEF.integration}. Read each side's staged diff. Empty findings is a good result.`,
    { label:'review:integration', phase:'Converge', model:'opus', effort:'high', schema: FINDINGS })
  const fresh = ((reviewed&&reviewed.findings)||[]).filter(f => !disposed.has('integration:'+dispKey(f)))
  let accepted = []
  if (fresh.length) {
    const tri = await aretry(
      `Triage these cross-target seam findings against the targets' diffs. Keep only real contract/shape/payload/version `
    + `mismatches that break the integration; name which side diverges. Findings:\n${JSON.stringify(fresh)}`,
      { label:'triage:integration', phase:'Converge', model:'opus', effort:'high', schema: TRIAGE })
    accepted = (tri && tri.accepted) || []
    for (const f of fresh) disposed.set('integration:'+dispKey(f), {verdict:'rejected'})
    for (const a of accepted) disposed.set('integration:'+dispKey(a), {verdict:'accepted'})
    if (accepted.length) await aretry(
      `Fix these seam mismatches in the diverging target(s); match each repo's conventions; do NOT commit. ${JSON.stringify(accepted)}`,
      { label:'fix:integration', phase:'Converge', model:'sonnet', effort:'high', schema: FIXED })
  }
  return { accepted }
}

// ── PLAN ── (multi-repo: contracts+DAG first, then per-target) ──────────────────
phase('Plan')
const plan = await aretry(
  `Read the PRD at ${prdPath}. Produce the implementation plan for ${targets.map(t=>t.name)}. Conform to each `
+ `repo's conventions (mirror the sibling modules named in its profile; the CODE is the convention oracle — `
+ `if CLAUDE.md conflicts with the prevailing code pattern, follow the code and note the doc drift). `
+ (multiRepo ? `Freeze the cross-target contracts ${JSON.stringify(contracts)} and the dependency sequence first. ` : ``)
+ `CRITICAL: map EACH acceptance criterion (AC id) to at least one concrete test assertion to write in Implement; `
+ `where the stack genuinely cannot assert a criterion, mark it untestable (the ship-gate will judge it). `
+ `If planning surfaces an IRREVERSIBLE or product-semantic unknown the PRD doesn't resolve (a contract that can't hold, `
+ `a contradictory criterion), record it in openQuestions with blocking:true rather than guessing.`,
  { label:'plan', phase:'Plan', model:'opus', effort:'high', schema: PLAN })

// ── RESEARCH ── current best practices for the libs actually in use ─────────────
phase('Research')
const research = await aretry(
  `Via Context7 (mcp__plugin_context7_context7__*) + web where needed, pull current best practices for the `
+ `libraries this change touches; reconcile with the repo's existing patterns. Concise, actionable.`,
  { label:'research', phase:'Research', model:'sonnet' })

// ── IMPLEMENT ── DAG order; lazy build; mirror sibling code; AC→test; ponytail markers ─
phase('Implement')
const implResults = await implementInDagOrder(targets, sequence, t => aretry(
  `Implement ${t.name} per the plan ${JSON.stringify(plan)} and research ${JSON.stringify(research)}. `
+ `MIRROR the sibling code named in conventions (${t.profile.conventions}) — the code is the oracle, CLAUDE.md a hint. `
+ `Build LAZY (YAGNI): stdlib → native → installed dep → one line → only then new code; no unrequested abstractions; `
+ `NEVER simplify away validation/error-handling/security/anything the PRD requires. Write ≥1 tagged test per AC `
+ `(name it so the gate can run AC-tagged tests; the test must actually exercise the AC, not just assert truthiness). `
+ `Mark deliberate shortcuts \`ponytail: <ceiling>, <upgrade>\`. Do NOT commit. In a worktree, \`git add -A\` so the diff is real. `
+ `If you hit an IRREVERSIBLE or product-semantic unknown not resolved by the plan/PRD, do NOT guess: return it in `
+ `openQuestions with blocking:true (otherwise resolve reversible choices via the sibling code and flag with a ponytail marker).`,
  { label:`impl:${t.name}`, phase:'Implement', model:'opus', effort:'high',
    isolation: multiRepo?'worktree':undefined, schema: IMPL_RESULT }))

// ── H4: stop EARLY on a blocking mid-build unknown — don't guess, don't burn the loop, don't defer to the ship-gate ──
const openQuestions = [
  ...((plan&&plan.openQuestions)||[]),
  ...Object.values(implResults||{}).filter(Boolean).flatMap(r => (r&&r.openQuestions)||[]),
]
const blocking = openQuestions.filter(q => q && q.blocking)
if (blocking.length) {
  log(`needs-human: ${blocking.length} blocking open question(s) surfaced mid-build`)
  return { status:'needs-human', criteria:[], gates:[],
    perTarget: targets.map(t=>({ target:t.name, green:false, residualFindings:['blocked on open question'] })),
    openQuestions: blocking, integration:null, risk:'mid-build ambiguity',
    recommendation:'Answer the open question(s), fold the decision into the PRD/CONFIG, then RESUME the run (resumeFromRunId) — do not restart.' }
}

// ══════════ MECHANISM A — converge correctness + completeness, PER TARGET + integration (BLOCKING) ══════════
const disposed   = new Map(targets.map(t => [t.name, new Map()]))   // per-target: dispKey → {verdict} (paths collide across repos)
const integDisp  = new Map()                                        // integration findings (multi-repo)
const cursorMap  = new Map(targets.map(t => [t.name, null]))        // per-target delta cursor
const converged  = new Set()
let pass = 0, lastByTarget = {}, integLast = { accepted: [] }
while (true) {
  pass++; log(`converge round ${pass}/${CAP_A} — ${converged.size}/${targets.length} targets green`)
  const active = targets.filter(t => !converged.has(t.name))
  const passes = await parallel(active.map(t => () => convergeTarget(t, disposed.get(t.name), cursorMap, pass)))
  passes.forEach((r,i) => { if (r) { lastByTarget[active[i].name] = r; if (r.clean) converged.add(active[i].name) } })

  // cross-target seam check (multi-repo); an accepted seam fix un-converges the sides so they re-gate against it
  if (multiRepo) {
    integLast = (await integrationPass(targets, contracts, integDisp)) || { accepted: [] }
    if ((integLast.accepted||[]).length) targets.forEach(t => converged.delete(t.name))
  }

  const clean = converged.size === targets.length && (integLast.accepted||[]).length===0
  if (clean) { log('converged — every target correct + complete; seams aligned'); break }
  if (pass >= CAP_A) { log('hit convergence cap with residuals — verdict will be blocked'); break }
  // else: next round reviews ONLY each unconverged target's fix delta with surface-relevant lenses.
}

// ══════════ MECHANISM B — polish ONCE per target (NON-blocking, advisory) ══════════
// Quality clock: thermo-nuclear (structure/altitude) + yagni (deletion) SUBSUME code-review's
// reuse/simplification/altitude. Plus the two real gaps — `efficiency` (wasted work / leaks) and a
// FORWARD-direction `conventions` lens (quote the exact CLAUDE.md rule), the mirror of reverse-direction
// `doc-drift`. All advisory: they write the ledger, never the gate. One pass (POLISH).
phase('Polish')
let ledger = { markers: [] }
for (let i=0;i<POLISH;i++){
  const perRepo = await parallel(targets.map(t => () => aretry(
    `Invoke Skill({ skill:'thermo-nuclear-code-quality-review' }) and apply the \`yagni\` lens to the FULL diff in `
  + `${t.repoPath}: find over-engineering/duplication to DELETE (delete/stdlib/native/yagni/shrink; net −lines). `
  + `ALSO run the \`efficiency\` lens: flag wasted work the diff introduces: redundant computation or repeated I/O, `
  + `independent operations run sequentially, blocking work added to startup or hot paths. Also flag long-lived objects `
  + `built from closures or captured environments — they keep the entire enclosing scope alive for the object's lifetime `
  + `(a memory leak when that scope holds large values); prefer a class/struct that copies only the fields it needs. `
  + `Name the cheaper alternative. `
  + `ALSO run \`doc-drift\` (REVERSE direction): any CLAUDE.md rule or code comment the implementation now contradicts `
  + `(the code is right — the doc is stale). `
  + `ALSO run \`conventions\` (FORWARD direction): find the CLAUDE.md files that govern the changed code (user-level `
  + `~/.claude/CLAUDE.md, the repo-root CLAUDE.md, plus any CLAUDE.md/CLAUDE.local.md in a directory that is an ancestor `
  + `of a changed file — a directory's CLAUDE.md only applies to files at or below it). Read each that exists, then check `
  + `the diff for clear violations. Only flag a violation when you can QUOTE THE EXACT RULE and the exact line that breaks `
  + `it — no style preferences, no vague "spirit of the doc" inferences. Name the CLAUDE.md path and quote the rule. If no `
  + `CLAUDE.md applies, return nothing for this lens. (conventions is ADVISORY — it never gates; the code still wins on a `
  + `genuine code-vs-doc conflict, which is what doc-drift records.) `
  + `For EVERY finding, state the CONCRETE COST in failure_scenario — what is duplicated / wasted / leaked, or which exact `
  + `CLAUDE.md rule is broken — not a vague crash and not an intermediate state. Return findings.`,
    { label:`polish:review:${t.name}`, phase:'Polish', model:'opus', effort:'high', schema: FINDINGS })))
  const allQ = perRepo.filter(Boolean).flatMap(r => r.findings||[])
  const qa = await aretry(`Triage ${JSON.stringify(allQ)}: accept only cheap, clearly-worth-it cleanups (obvious DRY extractions, `
  + `a sequential pair trivially made concurrent, a closure leak with a one-line fix). Each kept item must name its concrete cost `
  + `(what is duplicated/wasted/leaked, or the exact CLAUDE.md rule broken) — drop vague ones. A \`conventions\` item with no exact `
  + `quoted rule is noise: reject it. Everything else is deferred, not rejected. Findings:\n${JSON.stringify(allQ)}`,
    { label:'polish:triage', phase:'Polish', model:'sonnet', effort:'medium', schema: TRIAGE })
  const acc = (qa&&qa.accepted)||[]
  if (acc.length) for (const t of targets) await aretry(`Apply any of these cheap cleanups that belong to ${t.repoPath}, then run \`${t.profile.commands.verify}\` (must stay tier-0 green). Do NOT commit. ${JSON.stringify(acc)}`,
    { label:`polish:fix:${t.name}`, phase:'Polish', model:'sonnet', effort:'medium', schema: FIXED })
  const harvested = await aretry(
    `Across ${JSON.stringify(targets.map(t=>t.repoPath))}, grep for \`(#|//|--) ?ponytail:\` markers (skip node_modules/.git/build). Each → a ledger row `
  + `(kind:'ponytail', location, note, ceiling, upgrade; rotRisk=true if no upgrade trigger). From this deferred/rejected evidence `
  + `${JSON.stringify((qa&&qa.rejected)||[])}: add the deferred efficiency/yagni/thermo polish findings (kind:'deferred-finding'), the `
  + `reverse-direction doc-drift items (kind:'doc-drift'), and the forward-direction convention violations (kind:'convention-gap', `
  + `note = the quoted CLAUDE.md rule + the breaking line; rotRisk=false). Every note states the concrete cost, not a vague summary.`,
    { label:'polish:ledger', phase:'Polish', model:'haiku', effort:'low', schema: LEDGER })
  if (harvested && harvested.markers) ledger = harvested
}

// ══════════ SHIP-GATE — structured verdict (explains; cannot invent green) ══════════
phase('Ship-gate')
// H3 guard: the untestable escape hatch can't silently ship a half-untestable build (script-owned, not a model call).
const acAll = Object.values(lastByTarget).map(r=>r&&r.acTests).filter(Boolean)
const totalAc = acAll.reduce((n,a)=> n + (a.passed||[]).length + (a.failing||[]).length + (a.untestable||[]).length, 0)
const untestableN = acAll.reduce((n,a)=> n + (a.untestable||[]).length, 0)
const untestableRatio = totalAc ? untestableN/totalAc : 0

const verdict = await aretry(
  `You are the ship-readiness gate. Read the PRD at ${prdPath} and decide from the evidence. status='ship' ONLY if every AC `
+ `is met (AC-tagged tests pass; judge 'untestable' ACs against the diff), all tier-0 gates are green (N/A and env-skipped `
+ `tier-1 are fine), and no unresolved critical/high finding remains. Set \`simplifications\` to the harvested ledger VERBATIM — `
+ `accepted, tracked debt, NOT a reason to block; a clean run ships WITH it listed; raise \`risk\` for any rotRisk entry. `
+ `${untestableRatio>MAX_UNTESTABLE_RATIO ? `NOTE: ${(untestableRatio*100).toFixed(0)}% of ACs are untestable (> ${(MAX_UNTESTABLE_RATIO*100).toFixed(0)}% threshold) — set risk and do NOT return 'ship'. ` : ``}`
+ `status='blocked' if residuals remain after the convergence cap; 'needs-human' if a criterion is contradictory. `
+ `Evidence:\n${JSON.stringify({ converge: lastByTarget, integration: integLast, ledger, untestableRatio })}`,
  { label:'ship-gate', phase:'Ship-gate', model:'opus', effort:'high', schema: SHIP_VERDICT })

// script-owned override: a model can't vote 'ship' past the untestable guard
let v = verdict || { status:'blocked', criteria:[], gates:[],
  perTarget: targets.map(t=>({target:t.name,green:false,residualFindings:['ship-gate returned no result']})),
  recommendation:'Re-run the ship-gate.' }
if (v.status==='ship' && untestableRatio>MAX_UNTESTABLE_RATIO)
  v = { ...v, status:'needs-human', risk:`${(untestableRatio*100).toFixed(0)}% of ACs untestable — needs human sign-off on completeness`,
        recommendation:'Confirm the untestable ACs are genuinely unassertable on this stack, then resume; or add assertions.' }
return v
```

## Helpers the orchestrator pastes (part of the frozen mechanism)

- `implementInDagOrder(targets, sequence, fn)` — wave execution: each wave is the targets whose `dependsOn` are
  all already done, run via `parallel()`; cycles throw (a clean failed run, not a hang). One target / empty
  `sequence` ⇒ one wave of one ⇒ a `parallel()` of one. Returns `{target → implResult}` so the loop can read
  per-target Implement output (and the open-question channel). **It now has a real body — do not re-derive it.**
- `convergeTarget(t, disposed, cursorMap, pass)` — one Mechanism-A pass (stage → review → triage → fix → gate)
  for a single target; returns `{accepted, gate, acTests, gatesGreen, complete, clean}`. The loop runs it once
  per unconverged target each round.
- `integrationPass(targets, contracts, disposed)` — the multi-repo seam review→triage→fix; an accepted seam fix
  un-converges the sides so they re-gate. No-op path for single-repo (never called when `!multiRepo`).
- The per-target disposition caches (`disposed`), per-target delta cursors (`cursorMap`), `converged` set, the
  `CORRECTNESS_ANGLES`/`LENS_DEF` registries, the static `floorFor*` floor, the ADD-only `routeLenses` router, and
  the 3-state `verifyCandidates` pre-filter are the v2 mechanism — they are why A–E + test-integrity always run, a
  sensitive surface can never be under-reviewed (floor), the router can only widen coverage (never make a run
  unsafe), REFUTED candidates are dropped before triage, review cost shrinks each pass, triage stops re-litigating
  ghosts, and every emitted lens key has guidance. Do not "simplify" them away.

## Authoring rules (v2)

- **Fill only the CONFIG block; paste the mechanism verbatim.** This is what makes the loop/predicate/cache
  reliable and kills the historical footgun class. Don't read from `args`.
- **Every `phase:` string ∈ `meta.phases`.** The whole convergence loop (per-target + integration) uses
  `phase('Converge')`; polish uses `phase('Polish')`. One label set, used everywhere.
- **The exit is the script's job, fail-safe.** Each target's `clean` needs positive evidence (tier-0 actually
  `passed`, AC tests not failing, `accepted===0`); a null/empty result reads as NOT clean. Remember `[].every()`
  is `true`, so pair every "all passed" with a "tier-0 actually ran" check. The global `clean` also requires
  every target converged **and** no accepted integration finding.
- **Completeness is the AC-tagged tests, audited by `test-integrity`, with an untestable-ratio guard.** Where a
  criterion is genuinely untestable it defers to the ship-gate (flagged); a build that declares more than
  `MAX_UNTESTABLE_RATIO` of ACs untestable cannot return `ship` (the script downgrades to `needs-human`).
- **A blocking mid-build unknown stops the run early.** An irreversible/product-semantic question returns
  `needs-human` with the question *before* the converge loop spends laps — it does not get guessed or deferred
  to the ship-gate. `needs-human` resumes (`resumeFromRunId`), it doesn't restart.
- **Polish never gates and never loops past `POLISH`.** It applies cheap wins and writes the ledger; everything
  else is advisory.

## Why the exit is the script's job

A model deciding "are we done?" calls a 90%-done change shipped. The harness computes `clean` from objective
signals — per-target `accepted===0`, tier-0 `passed`, AC tests not failing, no open seam — so the loop can't
terminate on vibes, and the ship-gate *explains* the verdict without getting to *invent* a green one. v2 keeps
that and adds: the loop can't **waste** either — it reviews only the delta, triages only what's new, and stops
the bottomless quality axis from holding the cheap correctness axis hostage. The two newest guards close the
remaining holes a self-interested loop could slip through: a `test-integrity` lens + untestable-ratio cap so a
tautological green can't ship, and an early `needs-human` exit so a mid-build unknown is asked, not guessed.
