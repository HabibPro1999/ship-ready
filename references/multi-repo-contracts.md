# Multi-Repo & Contract-First

How ship-ready handles a unit of work that spans more than one repo/service — two backend services and a
mobile app, a service plus a shared types package, a monorepo with several affected packages. The design
principle, validated against current research (the "spec is the contract" line of work, and multi-agent
API-first systems): **make the contract the source of truth, freeze it first, then let each target be built
against it independently.**

## The WorkGraph (assembled across stages)

The "WorkGraph" is the targets + contracts + dependency order. It isn't one pre-baked object — it's assembled
in two places, matching the WHAT/HOW split:

- **`targets`** — one node per repo/service, each with a `repoPath` and `surfacesHint`. Comes from the
  **PRD's *Systems touched*** (product scope; see `intake-and-spec.md`) and reaches the Workflow as
  `args.targets`. Drives the Adapt fan-out (a ProjectProfile each) and the per-target implement/review.
- **`contracts`** — the seams between targets: an API path (OpenAPI/GraphQL/gRPC), an event/message payload
  (AsyncAPI/Avro/JSON-schema), or a shared type/DTO, each naming the targets it spans and an `owner`.
  **Derived by the Workflow's Plan** (that's HOW) — deliberately not in the PRD.
- **`dag`** — the dependency order, also **derived by Plan**. Contract-bearing changes come first; consumers
  depend on producers unless they build against the frozen spec/a mock.

## Two-level planning (the answer to "unified vs per-repo plan")

Neither extreme works: one monolithic plan ignores each repo's conventions and structure; N disconnected
plans drift apart at the seam. So plan at **two levels**:

1. **Thin unified contract/sequencing plan** (one Opus agent, sees all profiles). Owns *only*:
   - the **frozen shape** of each contract (the exact request/response, event payload, or type),
   - the **cross-cutting acceptance criteria** — the end-to-end story across targets ("WHEN a user taps Pay
     in mobile, THE SYSTEM SHALL create an order in orders-svc AND emit an `order.created` event consumed
     by billing-svc"),
   - the **dependency sequence**.
2. **Thick per-target sub-plans** (one per node, in parallel). Each implements its slice **conforming to the
   frozen contract** *and* to that repo's own conventions/structure.

The contract layer is small; the per-repo layers carry the real work. **Treat a contract change as its own
first step** — ideally landing in a shared schema/types package both sides consume — so once it's frozen,
the consuming targets can proceed in parallel against a stable interface.

When the seam is reachable **without authentication** (a guest flow) or carries money, freeze it
*server-authoritative*: the contract makes the server resolve price/totals/ownership/identity from its own
data, never from client-supplied fields (see the security review-emphasis in `adaptation-layer.md`). A
client-trusted amount or owner id on a public route is a tampering/IDOR bug — the contract is where you
prevent it, by shape.

## Implement in DAG order

`implementInDagOrder` runs targets with no unmet dependencies concurrently, then the next wave. Independent
targets (both just consuming a frozen contract) go in parallel; a genuine ordering (a DB migration before
the code that reads it; a producer before a consumer that can't mock) is serialized. Isolate parallel
writers — `isolation:'worktree'` for packages in one repo, or distinct `repoPath`s for separate repos.

> **Contract-first lets you parallelize what looks sequential.** If the consumer can build against the
> frozen spec (or a generated mock/stub), producer and consumer proceed at the same time and meet at the
> integration gate — you don't have to wait for the producer to finish first.

## The cross-cutting integration lens

Per-target review can't see the seam: each side can be internally green while the *combination* is broken
(mobile's client expects `amount` as a number; orders-svc now returns it as a string). So every loop
iteration adds one **integration review agent** (Opus) over all targets that checks **contract conformance**:
request/response shapes, event payloads, shared-type alignment, status codes, and versioning/back-compat.
It flags the exact mismatch and which side diverges. Its findings join triage like any other — and the
harness exit predicate counts them, so an unresolved seam mismatch keeps the loop running.

## The integration gate (optional tooling — detect, don't install)

For the *gate* (not just review), use real contract-testing tools **when the project already has them** —
they catch cross-service drift in CI without standing up a shared environment:

- **Pact** (consumer-driven) — the consumer publishes expectations; the provider verifies against them.
- **Specmatic** — validate each service against an OpenAPI/AsyncAPI contract as executable tests.

If `profiles[t].commands.contract` was detected, run it as a Tier-1 gate. If the project has *no* contract
tooling, **do not introduce it** (lean-core; and forcing a new test framework into someone's repo is a big,
unrequested change) — rely on the integration review lens plus any existing e2e tests. The verdict records
which it was, honestly.

## Where the targets live

The grill resolves each target's `repoPath`. Three common shapes:

- **Monorepo** — targets are packages under one root; detect the workspace and scope gates/lint per package.
- **Sibling repos** — targets are separate checkouts under a parent dir; each has its own git root and CI.
- **A target not checked out locally** — decided during the grill: clone it to a known path, or declare it
  out-of-scope for this run (and say so in the verdict, so no one assumes it was touched).

## Single-target collapse (the important property)

A single-repo feature is just a WorkGraph with **one node, no contracts, an empty DAG**. Then:

- the contract plan is a no-op (`contracts:[]`, trivial sequence),
- there's no integration lens and no integration gate,
- the Adapt/Plan/Implement/Review fan-outs are each a `parallel()` of one.

So the multi-repo machinery **disappears cleanly** for the common case — you don't pay for it when you don't
need it, and there's no separate single-repo code path to maintain. The same script handles "add a button"
and "ship a feature across three services."
