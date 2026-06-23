# Adaptation Layer

This is what makes assay *universal*: nothing downstream hardcodes a stack — the plan, the gates, and the
review lenses are all functions of a detected `ProjectProfile`.

**Where detection runs (v2): at intake, inline — not as a Workflow phase.** The inline agent is already reading
the repo to grill it, so it produces the profile *there* and **inlines it into the Workflow CONFIG block**. This
removes a whole phase (and its failure mode — a run that died on "Adapt failed to resolve a ProjectProfile") at
~no cost, because the reads are shared with grilling. Only **multi-repo** targets the inline stage didn't read
get a lean parallel Adapt spliced into the Workflow before Plan.

Precedence, per target: **config file → auto-detect → ask only on genuine ambiguity.**

## Table of contents

1. [The ProjectProfile](#1-the-projectprofile) — what intake produces and inlines
2. [Config file (first)](#2-config-file-first) — `.claude/assay.json`
3. [Auto-detect (default)](#3-auto-detect-default) — stack, commands, `cmdExists`, conventions, surfaces
4. [The lens roster](#4-the-lens-roster) — oracle-reliability, what earns its place, what's cut
5. [Tiered gates](#5-tiered-gates) — the mechanical, N/A-aware gate + test-backed completeness
6. [Asking](#6-when-to-ask) — the narrow cases where detection isn't enough

---

## 1. The ProjectProfile

```jsonc
{
  "target": "orders-svc",
  "stack":  { "language": "typescript", "frameworks": ["nestjs", "fastify"], "runtime": "node" },
  "pkgMgr": "pnpm",
  "monorepo": { "tool": "pnpm-workspaces", "thisPackage": "packages/orders" },  // or null
  "commands": {
    "verify": "pnpm typecheck && pnpm -r test",   // the fix-loop's quick confirm command
    "typecheck": "pnpm typecheck", "lint": "pnpm lint", "format": "pnpm prettier --check .",
    "build": "pnpm build", "unit": "pnpm test",
    "acTests": "pnpm test -- --grep '@AC'",         // the AC-tagged tests (completeness — see §5)
    "integration": "pnpm test:integration", "e2e": "pnpm test:e2e", "contract": "pnpm pact:verify"
  },
  "cmdExists": {                                     // does the command ACTUALLY exist? false ⇒ 'not-applicable', never blocks
    "typecheck": true, "lint": true, "format": false, "unit": true, "acTests": true, "integration": true
  },
  "ci": [".github/workflows/ci.yml"],               // source of truth for the commands above
  "conventions": "mirror packages/orders + the providers contract; the CODE wins over CLAUDE.md on conflict",
  "surfaces": ["api", "db"],                          // drives the lap-1 lens set
  "fileLensMap": [                                    // delta laps: changed-file path-substr → SPECIALIST lenses to re-run
    ["controller", ["api-contract", "security"]], ["schema", ["data-integrity"]],
    ["migration", ["data-integrity"]], ["async", ["concurrency"]], ["txn", ["concurrency"]]
  ],                                                  // A–E + test-integrity are the always-on core (the floor adds them); no '' catch-all
  "buildsClean": true,                                // did the tree typecheck BEFORE our change?
  "preflightWarnings": ["imports ./orders.service (file absent)", "build needs tsconfig.build.json (absent)"]
}
```

Two v2 fields earn their keep:
- **`cmdExists`** — whether each gate command actually exists in this repo. A missing command becomes a
  `not-applicable` gate (never blocks), instead of a `skipped` one the predicate mistook for not-green. (On the
  studied run there was no lint/format script, and the old loop treated that as a phantom failure.)
- **`fileLensMap`** — maps a changed file to the lenses worth re-running, so delta laps re-review *only the
  surfaces the fix touched* (a DRY-only fix re-runs no security/data reviewers).

`preflightWarnings` + `buildsClean` are the repo-integrity signal (§3) — referenced-but-absent files/config the
work must create. Empty/clean is the happy path; otherwise "this doesn't build as-is, fold it into scope."

## 2. Config file (first)

If `.claude/assay.json` exists at a target's root, **trust it** — it pins commands, lenses, surfaces,
and a definition-of-done, overriding detection. It makes runs deterministic and fast. Shape mirrors the
`commands` / `surfaces` / `lenses` blocks above, plus an optional `definitionOfDone` note.

assay maintains this as a **write-through cache**: after a first successful detection, offer to write
the resolved profile to `.claude/assay.json` so the next run skips detection (and so a human can
correct it once instead of every run). Never write it without asking — it lands in the user's repo.

## 3. Auto-detect (default)

When there's no config, probe the repo. Work outside-in: manifest → commands → conventions → surfaces.

### Stack & package manager
Detect from manifests and lockfiles — presence, not guessing:

| Signal | Stack | Package manager |
|---|---|---|
| `package.json` + `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json` | JS/TS (Node) | pnpm / yarn / npm |
| `pyproject.toml` / `requirements.txt` / `poetry.lock` | Python | poetry / pip / uv |
| `Cargo.toml` | Rust | cargo |
| `go.mod` | Go | go |
| `pubspec.yaml` | Dart / **Flutter** | pub / flutter |
| `pom.xml` / `build.gradle` | Java/Kotlin | maven / gradle |
| `*.csproj` / `*.sln` | .NET | dotnet |
| `Gemfile` | Ruby | bundler |
| `composer.json` | PHP | composer |

Detect **monorepo** from `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, `lerna.json`, workspaces in
`package.json`, Cargo `[workspace]`, or `go.work`. If present, locate *this target's* package within it —
gates and lint usually scope to the package, not the whole repo.

### Commands — read CI as ground truth
The most reliable source for "what command actually gates a merge" is the project's CI, not its docs.
Read, in order of trust:

1. **CI config** — `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`, Azure Pipelines.
   The steps that run on PR *are* the real lint/typecheck/build/test/contract commands and their flags.
2. **Task runners** — `Makefile`, `justfile`, `Taskfile.yml`, `package.json` scripts, `cargo`/`go`
   defaults, `composer` scripts.
3. **Pre-commit** — `.husky/`, `.pre-commit-config.yaml`, `lefthook.yml` — reveals the commit-time gate.

Prefer the project's wrapper (`rtk ...` when the repo's CLAUDE.md asks for it). If two plausible test
commands exist and CI doesn't disambiguate, that's an *ask* (§6).

### Conventions
Load `CLAUDE.md` at the repo root **and** in any directory the change touches (nested CLAUDE.md override),
plus `.editorconfig`, the lint config (eslint/biome/ruff/clippy/golangci-lint), the formatter
(prettier/black/rustfmt/`dart format`/gofmt), and commit conventions (`commitlint`, a `CONTRIBUTING.md`).
These feed the **implement phase** (write code that matches) — convention-following is *prevention*, not a gate
lens. Any doc/comment **drift** the code later contradicts surfaces as an advisory note in Mechanism B, never as
a blocking finding (the `claude-md` lens is cut from the gate — see §4). The forward-direction `conventions` lens
re-enters ONLY as an advisory Mechanism-B lens (quote the exact CLAUDE.md rule; never gates), the mirror of the
reverse-direction `doc-drift`.

### Surfaces
Infer the work's surfaces from dependencies + directory shape — they decide the lenses and the UI gates:

- **UI deps** (React/Vue/Svelte/Angular/SwiftUI/Jetpack Compose/**Flutter**) → `ui` surface (drives the tier-2
  build + boot/route smoke GATE only; it adds no Mechanism-A review lens — `a11y`/`visual-state` are cut).
- **Routes / controllers / GraphQL / OpenAPI / gRPC** → `api` surface.
- **Migrations / ORM / raw SQL** → `db` surface.
- **Dockerfile / k8s / terraform / pulumi** → `infra` surface.
- **Background jobs / queues / schedulers** → `async` surface.
- **A published package** (`bin`, library entrypoint, `exports`) → `library` surface.

### Repo integrity (preflight — do this every time)

Before trusting the repo, confirm it actually builds. Resolve the imports of the files the change will
touch and check that referenced modules and build/test config files exist — an imported `*.service.ts`, an
auth guard, a `tsconfig.build.json`, a test config, a Dart file behind `import '…'`. Record anything
referenced-but-absent in `preflightWarnings`, and **fold creating/stubbing it into scope**. This is the
single most common way a confident plan goes wrong: it plans against a skeletal tree as if it compiles,
then tier-0 typecheck hard-fails on the first gate with no anticipated cause. A missing imported file is
in-scope work or a blocker surfaced now — never a surprise at the gate. If you can cheaply run the
typecheck/analyze command, do; otherwise reason from the imports.

## 4. The lens roster

A review lens checks the code against *some oracle*. **A lens is only as reliable as its oracle.** Rank them:

| Lens | Oracle | Reliability |
|---|---|---|
| `angle-A`…`angle-E` (correctness core), `api-contract`, `security`, `data-integrity`, `concurrency` | **the code + execution semantics** (does it actually break / leak / race) | high — the oracle *is* reality |
| `claude-md` | a doc that **lags** the code | drift-prone |
| `code-comments` | inline comments that **rot silently** | most drift-prone |
| `prior-prs` | external PR threads it usually can't read → **hallucinated** | lowest |

The bottom three share one failure mode: they trust a written artifact that drifts from the code, so they
false-flag correct code that contradicts a stale rule. So the **gate roster is the reality-anchored set**, and
convention-following moves to *prevention*:

| Bucket | Lenses | When |
|---|---|---|
| **Correctness core** (always-on) | `angle-A`…`angle-E` (the five `/code-review` technique angles, verbatim) | EVERY pass, EVERY target, every lap — replaces the old vague `bugs` lens |
| **Completeness core** (always-on) | `test-integrity` | always (completeness is always gated) |
| **Specialists** (SMART-selected) | `security`, `data-integrity`, `concurrency`, `infra-safety`, `api-contract`, `public-api` | the static **floor** (surface/`fileLensMap`) UNION the cheap **lens-router**'s ADD-only picks; the router may only ADD, never remove from the floor; selection is NOT gating |
| **Multi-repo** | `integration` (cross-target contract conformance) | spans ≥2 targets |
| **Polish (advisory, Mechanism B — never gates)** | `thermo-nuclear`, `yagni`, `efficiency`, `doc-drift` (reverse) + `conventions` (forward) | once, after convergence |

**Lap 2+ still runs the always-on A–E core + `test-integrity`, plus only the specialists the fix's surface
touches** (via `fileLensMap`) UNION the router's ADD-only picks; a DRY-only fix re-runs none of the insurance
specialists. A–E's `angle-C` (cross-file) overlaps `api-contract`/`public-api` and `angle-B` (removed-behavior)
covers the "this diff dropped a guard" job; the overlap is intentional — triage's `dispKey` dedup absorbs the
double-reports, so do NOT delete a specialist to avoid overlap. A lens earns its place by **(blast-radius of a
miss) × (oracle reliability)** — that's why `security` stays even at zero findings (catastrophic, reliable) while
the artifact-anchored lenses go.

**Cut from the gate (and why):**
- **`prior-prs`** — weak premise in a team (unrelated PRs), unreadable/hallucinated guidance, overlaps the
  removed-behavior angle (`angle-B`) and `claude-md`, cheap miss.
- **`code-comments`** — the oracle rots; its reliable signal (explicit live invariants like "must hold the lock")
  overlaps the correctness angles (`angle-A`…`angle-E`) and `concurrency`. Fold that one line into the always-on
  angles; don't run it as a lens.
- **`claude-md` as a gate** — docs lag the code. The genuinely-hard rules ("money in millimes", "audit in the
  same tx") are caught by the *code-anchored* lenses checking actual behavior anyway. So enforce conventions by
  **prevention** (Implement mirrors the sibling code; **the code wins over CLAUDE.md on conflict**), and surface
  any **doc/comment drift** as an advisory ledger note in Mechanism B — never a blocking finding. A forward-direction
  `conventions` lens (quote-the-exact-rule) re-enters ONLY as an advisory Mechanism-B lens writing a `convention-gap`
  ledger note — it never gates; `prior-prs` and `code-comments` stay fully cut.
  *(This is a deliberate, contestable tradeoff. A team that holds CLAUDE.md authoritative can promote the forward
  `conventions` lens to a gate finding instead of a ledger note — assay's default keeps it advisory because
  docs lag code, but the knob is there for shops that enforce conventions by policy.)*

**The `yagni` lens (Mechanism B, advisory):** reviews the diff for over-engineering *only* and is
deletion-biased. Tags: `delete:` (dead/speculative), `stdlib:` (hand-rolled what the stdlib ships — name it),
`native:` (a dep or app-code the platform already does — `<input type=date>` over a picker lib, `Intl` over
moment, a DB constraint over app code), `yagni:` (an abstraction with one caller — inline it), `shrink:` (same
logic, fewer lines). End `net: −<N> lines`; nothing to cut ⇒ `Lean already.` Never flag the one check a
`ponytail:` shortcut leaves behind. (From [`ponytail`](https://github.com/DietrichGebert/ponytail) — encoded,
not bundled; pairs with the `ponytail:` markers and the ledger.)

**The `efficiency` lens (Mechanism B, advisory):** flags wasted work the diff introduces — redundant computation
or repeated I/O, independent operations run sequentially, blocking work added to startup or hot paths — and
**closure-capture leaks**: long-lived objects built from closures/captured environments keep the entire enclosing
scope alive for the object's lifetime (a memory leak when that scope holds large values); prefer a class/struct
that copies only the fields it needs. It names the cheaper alternative and states the concrete cost (what is
duplicated/wasted/leaked), never a vague crash. This is the one capability the structure-and-deletion lenses
(thermo + yagni) don't cover. Verbatim from the current `/code-review` `efficiency` cleanup angle.

**The `conventions` lens (Mechanism B, advisory — forward direction):** the mirror of `doc-drift`. `doc-drift` is
*reverse* (the code is right, a CLAUDE.md rule or comment is stale → record it, the code wins). `conventions` is
*forward* (the code violates a CLAUDE.md rule that still holds). It reads the governing CLAUDE.md files (user-level,
repo-root, and any in an ancestor directory of a changed file) and **only flags a violation when it can quote the
EXACT rule and the exact breaking line** — no style preferences, no "spirit of the doc"; if no CLAUDE.md applies it
returns nothing. It is **advisory and never gates**, consistent with §4's decision to cut `claude-md` from the gate:
a quoted-rule violation lands as a `convention-gap` ledger note, never a blocking finding. Verbatim shape from the
current `/code-review` `conventions (CLAUDE.md)` cleanup angle.

**Two emphases worth keeping explicit:**
- **"Existing behavior unchanged" ⇒ a regression test, not a note** — fold it into the AC-tagged tests.
- **Public/guest mutations must not trust the client for authority** — for any unauthenticated or money route the
  server resolves price/totals/ownership/identity itself; the `security` lens flags a client-trusted amount/owner
  as a concrete tampering/IDOR finding.

## 5. Tiered gates — mechanical, N/A-aware, completeness folded in

The gate agent **reports** raw status; the **script computes `green`** (the harness owns the gate, not a model).
Each command resolves to one of `passed / failed / skipped / not-applicable`:

- **Tier 0 (always):** typecheck/compile, lint, format-check, unit tests.
- **Completeness — the AC-tagged tests** (`commands.acTests`): run them and report per-AC `passed / failing /
  untestable`. This *is* the completeness check — no separate coverage agent. Untestable ACs defer to the
  ship-gate (flagged), they never silently pass. Two guards keep the completeness oracle honest, because the
  *same* agent writes both the AC and its test: a **`test-integrity`** lens (lap-1 always; re-run on any
  changed test/spec file) audits that each `@AC` test actually exercises its criterion — a tautological or
  self-referential assertion becomes a triage finding, not a free green; and if more than `MAX_UNTESTABLE_RATIO`
  (~⅓) of ACs are marked `untestable`, the script downgrades a `ship` verdict to `needs-human` so the escape
  hatch is bounded, not unlimited.
- **Tier 1 (if present):** integration, contract tests.
- **Tier 2 (if present):** e2e; for `ui`, a production build **plus** a boot/route smoke (browser MCP).

Status rules (this is the fix to the studied run's phantom gate):
- A command that **doesn't exist** (`cmdExists[x] === false`) → **`not-applicable`** — never blocks. (No lint
  script ⇒ not a failure.)
- A command that **exists but the env can't run** (Docker for Testcontainers) → **`skipped`**; fine at tier 1+,
  but a **tier-0** command that can't run is **`failed`**, never `skipped`.
- `green` = tier-0 actually `passed`, every gate `passed`/`not-applicable`/(tier-1+ `skipped`), and no AC test
  `failing`. A skipped or absent gate is **never** silently counted green.

## 6. When to ask

Detection covers most projects; ask only when a wrong guess would be expensive and the repo genuinely
doesn't disambiguate — e.g. two plausible test commands with different scopes, an integration suite whose
service dependencies you can't tell how to start, or an ambiguous definition-of-done. Batch these into a
single question rather than interrupting repeatedly. Everything you *can* detect, detect — don't ask what
the repo already answers.
