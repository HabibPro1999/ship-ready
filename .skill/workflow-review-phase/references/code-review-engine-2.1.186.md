# `/code-review max` — verbatim extraction from binary 2.1.186

Source: `~/.local/share/claude/versions/2.1.186` (Mach-O arm64, bun-compiled; JS bundle embedded).
Command region @ bytes 203,393k–203,427k. Decoded from JS string literals (`—`→—, `\xD7`→×, `→`→→).

`/code-review` ships in **two forms that share one source of truth** for the angle texts and verdict ladders:
- **Inline form** (`vZa(level)` → assembled megaprompt to ONE lead agent that fans out "via the **Agent** tool", `${ns}`="Agent").
- **Workflow form** (`DZa()` emits an `export const meta`/`pipeline()` script — used by `ultra`/cloud). This is the
  cleaner artifact for "what is injected per agent", and is reproduced below.

`max` params: **identical structure to `xhigh`** — only the API reasoning-effort knob differs, not the fan-out.

```
LEVEL_PARAMS = {
  high:  { correctnessAngles: 3, perAngle: 6, maxFindings: 10, sweep: false },
  xhigh: { correctnessAngles: 5, perAngle: 8, maxFindings: 15, sweep: true  },
  max:   { correctnessAngles: 5, perAngle: 8, maxFindings: 15, sweep: true  },
}
SWEEP_MAX = 8
```

So `max` = **5 correctness angles (A–E) + 5 cleanup angles = 10 finders × ≤8 candidates → per-candidate verify → sweep (≤8) → synthesize, cap 15.**

Pipeline: `Scope → pipeline(Find → Verify) [no barrier] → Sweep(if xhigh/max) → Synthesize`.
Finder/verifier/sweep angle objects:
- correctness `a4p = EZa.map((e,t)=>({label:`angle-${"ABCDE"[t]}`, text:e}))`  → labels `angle-A..angle-E`
- cleanup `l4p = [{reuse},{simplification},{efficiency},{altitude},{conventions}]`

---

## Phase 0 — Scope agent (runs once; output rides along to every later agent)

> Establish the scope of a code review.
>
> [if TARGET passed] Review target / instructions (passed by the user, verbatim): "<TARGET>". If it names a PR number, branch, ref range, or file path, build the matching git diff command for it; if it is a free-form instruction (e.g. only review certain files, focus on certain areas), honor any scope restriction when building the diff command and start from the current branch diff ('git diff @{upstream}...HEAD', falling back to 'git diff main...HEAD' or 'git diff HEAD~1') for whatever it does not narrow.
> [else] No explicit target — review the current branch: prefer 'git diff @{upstream}...HEAD' (fall back to 'git diff main...HEAD' or 'git diff HEAD~1'), and if there are uncommitted changes also include 'git diff HEAD'.
>
> 1. Determine the exact diff command(s) for the review and run them to confirm they produce a non-empty diff.
> 2. List the changed files.
> 3. Summarize what changed in one paragraph.
> 4. List the CLAUDE.md files that apply to the changed files (the user-level ~/.claude/CLAUDE.md, the repo-root CLAUDE.md, plus any CLAUDE.md or CLAUDE.local.md in a directory that is an ancestor of a changed file). Read each one that exists and note conventions a reviewer should know.
>
> Return diffCommand exactly as a reviewer should run it. Structured output only.

`SCOPE_SCHEMA = { diffCommand, files[], claudeMdFiles[], summary, conventions }`

### SCOPE_BLOCK — prepended to EVERY finder, verifier, and sweep agent
```
## Review scope
Diff command: <diffCommand>
Changed files (N):
  - <file>
  ...
Applicable CLAUDE.md files (M):
  - <path>            (or "  (none)")

## What changed
<summary>

## Conventions
<conventions or "(none noted)">

## User instructions (verbatim)         [only if TARGET]
<TARGET>
Honor any scope restrictions or focus areas stated above — they take precedence
over your angle's default breadth. Do not surface findings the instructions ask to skip.
```

---

## Phase 1 — Finder agents (10 for max). Injected prompt per finder:

```
## Code-review finder — <label>          (e.g. "angle-A", "reuse")

<SCOPE_BLOCK>

Run the diff command above and review ONLY through the lens of your assigned angle:

<ANGLE TEXT>                              ← one of the 10 below, verbatim
<CLEANUP_PRECEDENCE>                      ← appended ONLY for the 5 cleanup angles
Surface up to 8 candidate findings, each with file, line, a one-line summary, and a
concrete failure_scenario — the user-visible consequence (error, wrong output, data
loss), not an intermediate state (value stale, set grows). Pass every candidate with a
nameable failure scenario through — do not silently drop half-believed candidates; an
independent verifier judges them next. If nothing qualifies, return an empty list.

Structured output only.
```
`CANDIDATES_SCHEMA = { candidates: [{ file, line, summary, failure_scenario }] }`

### The 5 correctness angles (verbatim)

**angle-A — line-by-line diff scan**
> Read every hunk in the diff, line by line. Then Read the enclosing function for each hunk — bugs in unchanged lines of a touched function are in scope (the PR re-exposes or fails to fix them). For every line ask: what input, state, timing, or platform makes this line wrong? Look for inverted/wrong conditions, off-by-one, null/undefined deref, missing `await`, falsy-zero checks, wrong-variable copy-paste, error swallowed in catch, unescaped regex metachars.

**angle-B — removed-behavior auditor**
> For every line the diff DELETES or replaces, name the invariant or behavior it enforced, then search the new code for where that invariant is re-established. If you can't find it, that's a candidate: a removed guard, a dropped error path, a narrowed validation, a deleted test that was covering a real case.

**angle-C — cross-file tracer**
> For each function the diff changes, find its callers (Grep for the symbol) and check whether the change breaks any call site: a new precondition, a changed return shape, a new exception, a timing/ordering dependency. Also check callees: does a parallel change in the same PR make a call unsafe?

**angle-D — language-pitfall specialist**
> Scan for the classic pitfalls of the diff's language/framework — for example: JS falsy-zero, `==` coercion, closure-captured loop var; Python mutable default args, late-binding closures; Go nil-map write, range-var capture; SQL injection; timezone/DST drift; float equality. Flag any instance the diff introduces.

**angle-E — wrapper/proxy correctness**
> When the PR adds or modifies a type that wraps another (cache, proxy, decorator, adapter): check that every method routes to the wrapped instance and not back through a registry/session/global — e.g. a caching provider holding a `delegate` field that resolves IDs via `session.get(...)` instead of `delegate.get(...)` will re-enter the cache or recurse. Also check that the wrapper forwards all the methods the callers actually use.

### The 5 cleanup angles (verbatim)

**reuse**
> Flag new code that re-implements something the codebase already has — Grep shared/utility modules and files adjacent to the change, and name the existing helper to call instead.

**simplification**
> Flag unnecessary complexity the diff adds: redundant or derivable state, copy-paste with slight variation, deep nesting, dead code left behind. Name the simpler form that does the same job.

**efficiency**
> Flag wasted work the diff introduces: redundant computation or repeated I/O, independent operations run sequentially, blocking work added to startup or hot paths. Also flag long-lived objects built from closures or captured environments — they keep the entire enclosing scope alive for the object's lifetime (a memory leak when that scope holds large values); prefer a class/struct that copies only the fields it needs. Name the cheaper alternative.

**altitude**
> Check that each change is implemented at the right depth, not as a fragile bandaid. Special cases layered on shared infrastructure are a sign the fix isn't deep enough — prefer generalizing the underlying mechanism over adding special cases.

**conventions (CLAUDE.md)**
> Find the CLAUDE.md files that govern the changed code: the user-level ~/.claude/CLAUDE.md, the repo-root CLAUDE.md, plus any CLAUDE.md or CLAUDE.local.md in a directory that is an ancestor of a changed file (a directory's CLAUDE.md only applies to files at or below it). Read each one that exists, then check the diff for clear violations of the rules they state.
> Only flag a violation when you can quote the exact rule and the exact line that breaks it — no style preferences, no vague "spirit of the doc" inferences. In the finding, name the CLAUDE.md path and quote the rule so the report can cite it. If no CLAUDE.md applies, return nothing for this angle.

### CLEANUP_PRECEDENCE (appended to the 5 cleanup finders only)
> Cleanup, altitude, and conventions candidates use the same `file`/`line`/`summary` shape; in `failure_scenario`, state the concrete cost (what is duplicated, wasted, harder to maintain, or which CLAUDE.md rule is broken) instead of a crash. Correctness bugs always outrank cleanup, altitude, and conventions findings when the output cap forces a cut.

---

## Phase 2 — Verifier agent (ONE per surviving candidate; no pre-verify dedup)

```
## Code-review verifier

<SCOPE_BLOCK>

## Candidate finding
File: <file>:<line>
Summary: <summary>
Failure scenario: <failure_scenario>

Run the diff command above, read the relevant file(s), and return exactly one verdict:

<VERDICT_LADDER>

<VERDICT_LADDER_RECALL>

Structured output only. Evidence must quote or cite the relevant line(s).
```
`VERDICT_SCHEMA = { verdict: CONFIRMED|PLAUSIBLE|REFUTED, evidence }`

**VERDICT_LADDER**
> - **CONFIRMED** — can name the inputs/state that trigger it and the wrong output or crash. Quote the line.
> - **PLAUSIBLE** — mechanism is real, trigger is uncertain (timing, env, config). State what would confirm it.
> - **REFUTED** — factually wrong (code doesn't say that) or guarded elsewhere. Quote the line that proves it.

**VERDICT_LADDER_RECALL**
> **PLAUSIBLE by default** — do not refute a candidate for being "speculative" or "depends on runtime state" when the state is realistic: concurrency races, nil/undefined on a rare-but-reachable path (error handler, cold cache, missing optional field), falsy-zero treated as missing, off-by-one on a boundary the code does not exclude, retry storms / partial failures, regex/allowlist that lost an anchor. These are PLAUSIBLE.
>
> **REFUTED** only when constructible from the code: factually wrong (quote the actual line); provably impossible (type/constant/invariant — show it); already handled in this diff (cite the guard); or pure style with no observable effect.

Keep CONFIRMED + PLAUSIBLE. Drop REFUTED.

---

## Phase 3 — Sweep agent (xhigh/max only; ONE fresh finder, then each gap verified)

```
## Code-review sweep — gaps only

<SCOPE_BLOCK>

## Already-found candidates (do NOT re-derive or re-confirm these)
- <file>:<line> — <summary>
  ...                                    (or "(none)")

Re-read the diff and the enclosing functions looking ONLY for defects not already listed.
Focus on what the first pass tends to miss: <SWEEP_GAP_FOCUS>

Surface up to 8 additional candidates. If nothing new, return an empty list — do not pad.

Structured output only.
```

**SWEEP_GAP_FOCUS**
> moved/extracted code that dropped a guard or anchor; second-tier footguns (dataclass default evaluated once, `hash()` non-determinism, lock-scope shrink, predicate methods with side effects); setup/teardown asymmetry in tests; config defaults flipped.

---

## Phase 4 — Synthesize agent (ONE; ranks/merges/caps by index)

```
## Synthesis: final code-review report

<K> findings survived independent verification (max-effort review). They are numbered [0]-[K-1] below.

### [i] <file>:<line> (<verdict>[, cleanup])
<summary>
Failure scenario: <failure_scenario>
Verifier evidence: <evidence>
...

## Instructions
Return decisions about findings BY INDEX — never re-emit finding text.
1. For each distinct defect, emit one decision with its index. When several findings describe the
   same defect (same root cause), keep one entry and list the others in its merge array.
2. Order decisions most-severe first. Correctness bugs always outrank cleanup findings.
3. Keep at most 15 decisions; omit the least severe beyond the cap.
4. Write a 2-3 sentence summary of the review.

Structured output only.
```
`REPORT_SCHEMA = { summary, decisions: [{ index, merge:[int] }] }`
Ranking key: `rank = (kind==='cleanup'?2:0) + (verdict==='PLAUSIBLE'?1:0)` → CONFIRMED-correctness first, PLAUSIBLE-cleanup last.

---

## Notes that matter for assay Option A
- The **finder tool** the inline form names is **"Agent"** (`${ns}`); the workflow form uses `agent()`/`pipeline()`/`parallel()`.
- **No `claudeMdFiles`/conventions gating** — every finder gets them via SCOPE_BLOCK, but the `conventions` angle is **advisory cleanup**, must **quote the exact rule**, returns nothing if no CLAUDE.md applies.
- **No pre-verify dedup**: every candidate gets its own verifier; dedup happens once, at synthesis, **by index**.
- **Verify is recall-biased** (PLAUSIBLE-by-default) — it kills only provably-wrong candidates; it is NOT a precision gate.
- **Sweep candidates are also verified** (verifyCandidate over the sliced sweep set), then concatenated.
- `failure_scenario` is explicitly **user-visible consequence**, not intermediate state — worth copying verbatim into assay's FINDINGS schema guidance.
