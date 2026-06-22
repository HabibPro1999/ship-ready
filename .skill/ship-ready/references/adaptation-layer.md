# Adaptation Layer

This is what makes ship-ready *universal*. It runs as the **Adapt** phase of the Workflow (a `parallel()`
fan-out, one detection per target — the systems the PRD touches) and produces a `ProjectProfile` per repo. Nothing downstream
hardcodes a stack — the plan, the gates, and the review lenses are all functions of what's detected here.

Precedence, applied per target: **config file → auto-detect → ask only on genuine ambiguity.**

## Table of contents

1. [The ProjectProfile](#1-the-projectprofile) — what Adapt produces
2. [Config file (first)](#2-config-file-first) — `.claude/ship-ready.json`
3. [Auto-detect (default)](#3-auto-detect-default) — stack, commands, conventions, surfaces
4. [Surface → lens table](#4-surface--lens-table) — which reviewers to fan out
5. [Tiered gates](#5-tiered-gates) — what must run for "green"
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
    "verify": "pnpm typecheck && pnpm -r test",   // the fix-loop's confirm command
    "typecheck": "pnpm typecheck",
    "lint": "pnpm lint",
    "format": "pnpm prettier --check .",
    "build": "pnpm build",
    "unit": "pnpm test",
    "integration": "pnpm test:integration",        // null if none
    "e2e": "pnpm test:e2e",                         // null if none
    "contract": "pnpm pact:verify",                 // null if none
    "uiSmoke": null                                  // e.g. "playwright test smoke" for a web UI
  },
  "ci": [".github/workflows/ci.yml"],               // source of truth for the above
  "conventions": ["CLAUDE.md", "packages/orders/CLAUDE.md", ".eslintrc.cjs", ".prettierrc"],
  "surfaces": ["api", "db"],                          // drives the lens set
  "lenses": ["bugs", "claude-md", "security", "concurrency", "yagni", "api-contract", "git-history"],
  "preflightWarnings": ["imports ./orders.service (file absent)", "build needs tsconfig.build.json (absent)"]
}
```

`preflightWarnings` is the repo-integrity signal (see §3) — referenced-but-absent files/config the work must
create. Empty is the happy path; non-empty means "this doesn't build as-is, fold it into scope."

## 2. Config file (first)

If `.claude/ship-ready.json` exists at a target's root, **trust it** — it pins commands, lenses, surfaces,
and a definition-of-done, overriding detection. It makes runs deterministic and fast. Shape mirrors the
`commands` / `surfaces` / `lenses` blocks above, plus an optional `definitionOfDone` note.

ship-ready maintains this as a **write-through cache**: after a first successful detection, offer to write
the resolved profile to `.claude/ship-ready.json` so the next run skips detection (and so a human can
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
These feed both the implement phase (write code that matches) and the `claude-md` review lens.

### Surfaces
Infer the work's surfaces from dependencies + directory shape — they decide the lenses and the UI gates:

- **UI deps** (React/Vue/Svelte/Angular/SwiftUI/Jetpack Compose/**Flutter**) → `ui` surface.
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

## 4. Surface → lens table

The base lenses always run; surface lenses are added when that surface is present. (Lens prompts live in
the `workflow-review-phase` skill's `references/code-review-roster.md` — reuse them; add the surface-
specific ones below.) A lens with nothing to say returns an empty findings array — that's a good result,
not a prompt to invent issues.

| Surface | Add these lenses | Add these gates |
|---|---|---|
| *(base, always)* | `bugs`, `claude-md`, `security`, `concurrency`, `yagni`, `git-history`, `prior-prs`, `code-comments` | typecheck, lint, format, unit |
| `api` | `api-contract` (request/response shape, status codes, versioning, back-compat) | contract tests if present |
| `ui` | `a11y` (labels, roles, focus, contrast), `visual/state` (loading/empty/error states) | build + boot/route smoke (browser MCP) |
| `db` | `data-integrity` (migrations reversible, constraints, nullability, index impact) | integration if present |
| `infra` | `infra-safety` (least-privilege, secrets, idempotent applies, blast radius) | plan/dry-run if present |
| `async` | (concurrency lens already covers most) idempotency/retry emphasis | integration if present |
| `library` | `public-api` (semver, breaking changes, docs on exported surface) | build + downstream-type-check |
| *(multi-repo)* | `integration` (cross-target contract conformance — see multi-repo-contracts.md) | contract/e2e across targets |

Scale the roster to the diff: a tiny change runs a subset (`bugs`, `claude-md`, plus the one relevant
surface lens); a large/risky change runs the full applicable set, optionally doubling `bugs` and taking the
union.

The **`yagni`** lens is ship-ready-defined (not in the roster file). It reviews the diff for over-engineering
*only* — correctness/security/perf are out of scope (other lenses own those) — and is deletion-biased: its best
outcome is a shorter diff. One line per finding, tagged: `delete:` (dead code, speculative feature — replaced by
nothing), `stdlib:` (hand-rolled what the standard library ships — name the function), `native:` (a dependency or
app-code doing what the platform/framework already does — `<input type=date>` over a picker lib, `Intl` over
moment, a DB constraint over app code), `yagni:` (an abstraction with one implementation, config nobody sets, a
layer with one caller — inline it until a second caller exists), `shrink:` (same logic, fewer lines — show the
short form). End with `net: −<N> lines possible`; nothing to cut ⇒ `Lean already.` Never flag the one runnable
check a `ponytail:` shortcut leaves behind — that's the minimum, not bloat. (Pattern adapted from
[`ponytail`](https://github.com/DietrichGebert/ponytail) `ponytail-review`; encoded as a lens, not pulled in as a
skill — it pairs with the `ponytail:` markers the Implement phase leaves and the ledger the gate harvests.)

**Two review emphases worth making explicit** (they recur and are easy to under-specify):

- **"Existing behavior unchanged" ⇒ a regression test, not a note.** When a constraint says an existing
  shape/endpoint/behavior must keep working, the work isn't done until a *test asserts the old behavior*.
  "Don't break X" is only real when X is machine-checked — fold the regression test into scope and the gate.
- **Public/guest mutations must not trust the client for authority.** For any endpoint reachable without
  authentication (guest checkout, public forms) or any money path, the server resolves authoritative values
  (price, totals, ownership, identity) itself — never from client-supplied fields. The `security` lens
  should flag a client-trusted amount/price/owner on such a route as a concrete tampering/IDOR finding.

## 5. Tiered gates

The fix-loop's `verify` confirms quick (typecheck + unit). The **ship-gate** runs the full applicable tier
set and records each as ran/passed/failed/**skipped-and-why**:

- **Tier 0 (always):** typecheck/compile, lint, format-check, unit tests.
- **Tier 1 (if detected):** integration tests, contract tests.
- **Tier 2 (if detected):** e2e; for `ui` surfaces, a production build **plus** a boot/route smoke check.
- **Skip honestly:** if a suite needs services the environment can't start (Docker, a live DB), mark it
  `skipped` with the reason. A skipped gate is never counted as green.

## 6. When to ask

Detection covers most projects; ask only when a wrong guess would be expensive and the repo genuinely
doesn't disambiguate — e.g. two plausible test commands with different scopes, an integration suite whose
service dependencies you can't tell how to start, or an ambiguous definition-of-done. Batch these into a
single question rather than interrupting repeatedly. Everything you *can* detect, detect — don't ask what
the repo already answers.
