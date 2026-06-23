# Code-Review Roster (diff-scoped fan-out)

This MIRRORS the CURRENT `/code-review` angles (A–E) + 3-state recall-biased verify at the **script** level,
adapted to review a **diff** (not a GitHub PR) so it composes inside a Workflow. Source of truth for every angle
text and verdict ladder below: `./code-review-engine-2.1.186.md`. assay's Mechanism A drives this as its own
`parallel()` fan-out (a Workflow phase-agent can't spawn subagents).

The roster is **not** a flat list of equal lenses anymore. It is:
- an **always-on correctness core** — the five `/code-review` technique angles `angle-A`…`angle-E` (verbatim),
- an **always-on completeness lens** — `test-integrity`,
- **SMART-selected specialists** — `security`, `data-integrity`, `concurrency`, `infra-safety`, `api-contract`,
  `public-api` (`integration` is multi-repo-only), chosen by a static surface/`fileLensMap` **floor** UNION a
  cheap **lens-router** that may only ADD to the floor, never remove from it.

There is **no 0–100 confidence rubric and no `SCORE_FLOOR`** — those are removed. Precision is the recall-biased
3-state VERIFY pre-filter followed by TRIAGE (the gate). There is **no synthesize stage** — assay auto-fixes
real findings and re-checks; triage's `dispKey` dedup absorbs any merge/overlap.

## The pipeline (per target, per lap)

```
finders (A–E + test-integrity + selected specialists)
  → fresh-filter vs the per-target `disposed` cache   (FIRST — drop already-decided findings)
  → VERIFY   (3-state recall-biased; GATED to lap-1 / large delta; SKIP small delta laps)
        keep CONFIRMED + PLAUSIBLE, drop REFUTED
  → SWEEP    (lap-1 ONLY; one fresh gap-only finder; its candidates ALSO verified, then merged)
  → TRIAGE = THE GATE  (only verify-survivors + verified sweep reach it; the only stage that writes the cache)
  → FIX
  → mechanical GATE (script computes green; unchanged)
```

## The five correctness angles (always-on; verbatim)

| Angle | What it hunts |
|---|---|
| `angle-A` — line-by-line | Read every hunk in the diff, line by line. Then Read the enclosing function for each hunk — bugs in unchanged lines of a touched function are in scope (the PR re-exposes or fails to fix them). For every line ask: what input, state, timing, or platform makes this line wrong? Look for inverted/wrong conditions, off-by-one, null/undefined deref, missing `await`, falsy-zero checks, wrong-variable copy-paste, error swallowed in catch, unescaped regex metachars. |
| `angle-B` — removed-behavior | For every line the diff DELETES or replaces, name the invariant or behavior it enforced, then search the new code for where that invariant is re-established. If you can't find it, that's a candidate: a removed guard, a dropped error path, a narrowed validation, a deleted test that was covering a real case. (This absorbs the job the cut `git-history` lens used to do.) |
| `angle-C` — cross-file tracer | For each function the diff changes, find its callers (Grep for the symbol) and check whether the change breaks any call site: a new precondition, a changed return shape, a new exception, a timing/ordering dependency. Also check callees: does a parallel change in the same PR make a call unsafe? |
| `angle-D` — language-pitfall | Scan for the classic pitfalls of the diff's language/framework — for example: JS falsy-zero, `==` coercion, closure-captured loop var; Python mutable default args, late-binding closures; Go nil-map write, range-var capture; SQL injection; timezone/DST drift; float equality. Flag any instance the diff introduces. |
| `angle-E` — wrapper/proxy | When the PR adds or modifies a type that wraps another (cache, proxy, decorator, adapter): check that every method routes to the wrapped instance and not back through a registry/session/global — e.g. a caching provider holding a `delegate` field that resolves IDs via `session.get(...)` instead of `delegate.get(...)` will re-enter the cache or recurse. Also check that the wrapper forwards all the methods the callers actually use. |

`angle-C` overlaps `api-contract`/`public-api` and `angle-B` overlaps the old git-history job — the overlap is
**intentional**. Do not delete a specialist to avoid it; triage's `dispKey` dedup absorbs the double-reports.

Each finder surfaces up to ~8 candidates, each with `file`, `line`, a one-line summary, and a concrete
`failure_scenario` — the **user-visible consequence** (error, wrong output, data loss), NOT an intermediate state
(value stale, set grows). Pass every candidate with a nameable failure scenario through — do not silently drop
half-believed candidates; the verifier judges them next. If nothing qualifies, return an empty list.

## The specialist lenses (SMART-selected)

`security`, `data-integrity`, `concurrency`, `infra-safety`, `api-contract`, `public-api` carry the oracle-anchored
definitions in the script's `LENS_DEF` registry; `integration` is cross-target only. They inherit A–E rigor via a
shared methodology note (hunt by tracing callers / auditing deletions / scanning line-by-line). The static floor is
the fail-safe (a sensitive surface can NEVER be under-reviewed); the router can only widen coverage, never make a
run unsafe; selection is NOT gating — triage + the mechanical build/AC gate decide "done".

## 3-state recall-biased verify (verbatim ladder)

One verifier per surviving candidate (no pre-verify dedup). Return exactly one verdict:

- **CONFIRMED** — you can name the inputs/state that trigger it and the wrong output or crash. Quote the line.
- **PLAUSIBLE** — mechanism is real, trigger is uncertain (timing, env, config). State what would confirm it.
- **REFUTED** — factually wrong (code doesn't say that) or guarded elsewhere. Quote the line that proves it.

**PLAUSIBLE by default** — do not refute a candidate for being "speculative" or "depends on runtime state" when the
state is realistic: concurrency races, nil/undefined on a rare-but-reachable path (error handler, cold cache,
missing optional field), falsy-zero treated as missing, off-by-one on a boundary the code does not exclude, retry
storms / partial failures, regex/allowlist that lost an anchor. These are PLAUSIBLE. **REFUTED** only when
constructible from the code: factually wrong (quote the actual line); provably impossible (type/constant/invariant
— show it); already handled in this diff (cite the guard); or pure style with no observable effect.

**Keep CONFIRMED + PLAUSIBLE. Drop REFUTED.** Verify is a scalable PRE-FILTER, not the precision gate — triage is.
It is GATED: run it on lap-1 (whole change) and on large delta laps; SKIP small delta laps where the candidate set
is already tiny. REFUTED candidates are NOT written to the `disposed` cache (a recall-biased refute is cheap to
re-derive on a later, higher-coverage lap); only triage writes the cache.

## Sweep (lap-1 only)

One fresh gap-only finder re-reads the diff and enclosing functions for defects NOT already found. Focus on what
the first pass misses: moved/extracted code that dropped a guard or anchor; second-tier footguns (default evaluated
once, `hash()` non-determinism, lock-scope shrink, predicate methods with side effects); setup/teardown asymmetry in
tests; config defaults flipped. Up to `SWEEP_MAX` (8) candidates; its candidates go through the SAME verifier before
joining the fresh set. If nothing new, return an empty list — do not pad.

## Empty is a good result

If the codebase isn't one where a lens applies (e.g. no auth surface in the diff), that lens returns an empty
findings array rather than inventing something. Empty is a valid, good result.

## False positives to discard (verbatim guidance — re-anchored to the verifier/triage)

Tell the finders, the verifier, and triage to treat these as non-issues:

- Pre-existing issues (not introduced by this change).
- Something that looks like a bug but is not actually a bug.
- Pedantic nitpicks a senior engineer wouldn't call out.
- Issues a linter, typechecker, or compiler would catch (missing imports, type errors, broken tests, formatting,
  pedantic style). Assume CI runs these separately.
- General code-quality gripes (test coverage, documentation, vague "security" hand-waving) — the dedicated
  `security`/`concurrency`/`data-integrity` specialists DO surface concrete, in-diff bugs; that is different from a
  generic "add more tests" nit.

## Mechanism B (advisory) cleanup angles — for cross-reference

assay's quality clock keeps thermo-nuclear (structure/altitude) + yagni (deletion), which subsume
`/code-review`'s reuse/simplification/altitude cleanup angles, and ADDS the two real gaps: `efficiency` (wasted
work / repeated I/O / sequential-independent ops / hot-path-blocking / closure-capture memory leaks) and
forward-direction `conventions` (quote-the-exact-CLAUDE.md-rule, advisory, returns nothing if no CLAUDE.md applies),
which pairs with reverse-direction `doc-drift`. All of Mechanism B is advisory and never gates; it lives in
`../../assay/references/canonical-workflow.md`, not here.
