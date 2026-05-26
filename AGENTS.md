# AGENTS.md

> **Map, not manual.** This file is the table of contents for the Livepeer Modules
> Suite. It points to the system of record in `docs/`. Keep it short (~100 lines).
> When in doubt, **link — don't inline.** Operating principles live in
> [`docs/design-docs/core-beliefs.md`](docs/design-docs/core-beliefs.md).

## What this repository is

The **Livepeer Modules Suite** is an umbrella (meta) repository. It aggregates the
individual Livepeer Module repositories as **git submodules** and provides a single,
high-level, agent-legible overview of the whole suite and how its parts fit together.

- This repo holds **documentation and submodule pointers** — not module source code.
  Each module's code lives in its own repository, mounted under `modules/<name>/`.
- Goal: an agent or human can understand the entire suite and the relationships
  between modules **directly from this repository**.
- The Livepeer Modules are the productized features that enable the Livepeer
  protocol. They are developed independently and released on the network's cadence;
  this suite tracks specific submodule revisions so the overview always corresponds
  to a known, reproducible set of releases.

## Two axes: capabilities and repos

- **Capabilities** (the "what") — the conceptual feature areas, one spec each in
  [`docs/product-specs/`](docs/product-specs/index.md).
- **Repos** (the "where") — the actual submodules, one doc each in
  [`docs/repos/`](docs/repos/index.md). **A single repo can implement several
  capabilities** (e.g. `livepeer-network-modules` covers most of the supply side).

New terms? Start with the [`docs/glossary.md`](docs/glossary.md).

## The capabilities (the suite)

One line each. Full specs in [`docs/product-specs/`](docs/product-specs/index.md).

| Capability | One-liner | Spec |
| --- | --- | --- |
| Gateways | Demand-side entry point: discover, pay, forward work | [gateways](docs/product-specs/gateways.md) |
| Orchestrators | Supply side: a workload-agnostic broker + daemons that serve paid work | [orchestrators](docs/product-specs/orchestrators.md) |
| Runners | The backends a broker dispatches to (the provided capability) | [runners](docs/product-specs/runners.md) |
| Pools | Control plane aggregating member backends behind one orch identity | [pools](docs/product-specs/pools.md) |
| Payment | Probabilistic micropayment tickets exchanged for work | [payment](docs/product-specs/payment.md) |
| Service Registry | On-chain pointer + off-chain signed manifest of capabilities | [service-registry](docs/product-specs/service-registry.md) |
| Discover | Resolver API gateways use to find and select orchestrators | [discover](docs/product-specs/discover.md) |
| Payment Clearinghouse | Settlement/distribution of payments (+ SDKs) | [payment-clearinghouse](docs/product-specs/payment-clearinghouse.md) |
| SDKs | Client/integration libraries spanning the suite | [sdks](docs/product-specs/sdks.md) |
| Reference Apps | Example gateway apps showing how to build on the suite | [reference-apps](docs/product-specs/reference-apps.md) |

## Repositories (submodules)

| Repo | Implements | Doc |
| --- | --- | --- |
| livepeer-network-modules | Supply-side core: Orchestrators, Runners, Pools, Payment, Service Registry, Discover, Protocol, SDKs | [repos/livepeer-network-modules](docs/repos/livepeer-network-modules.md) |
| livepeer-open-clearinghouse | Demand-side control plane: Payment Clearinghouse, Payment issuance, SDKs, Discover proxy, gateway control-plane half. Consumes network-modules daemons. | [repos/livepeer-open-clearinghouse](docs/repos/livepeer-open-clearinghouse.md) |

The end-to-end picture (how a unit of work flows through these capabilities) is in
[`ARCHITECTURE.md`](ARCHITECTURE.md).

## Repository map

```text
AGENTS.md            <- you are here: the map
ARCHITECTURE.md      <- top-level domain map + job-flow across modules
modules/             <- git submodules, one per repo (see modules/README.md)
docs/
├── glossary.md      <- shared vocabulary — start here for unfamiliar terms
├── design-docs/     <- operating principles + design index (core-beliefs.md, index.md)
├── product-specs/   <- capability axis: one overview per capability
├── repos/           <- repo axis: one doc per submodule (index.md + <repo>.md)
├── guides/          <- how-tos for maintaining this repo (git-submodules-primer.md)
├── references/      <- external reference material (e.g. the harness-engineering post)
├── exec-plans/      <- active/, completed/, tech-debt-tracker.md
└── generated/       <- machine-generated docs (e.g. submodule manifest)
```

## Working in this repository

- **Onboarding a repo (user hands you one):** add it under `modules/<name>/`, read it,
  then (1) write/refresh `docs/repos/<name>.md` with a component map + pinned revision,
  (2) update the capability specs in `docs/product-specs/` it implements — replacing
  hedged language with confirmed facts and **resolving open questions as the code
  answers them**, (3) add new terms to [`docs/glossary.md`](docs/glossary.md), and
  (4) update the index tables here and in `docs/repos/`. Document only what the repo
  actually shows.
- **Submodules (clone, update, pin to releases):** see
  [`docs/guides/git-submodules-primer.md`](docs/guides/git-submodules-primer.md).
- **Planning larger work:** capture it in [`docs/exec-plans/`](docs/exec-plans/);
  track shortcuts in [`tech-debt-tracker.md`](docs/exec-plans/tech-debt-tracker.md).

## Conventions

- **Progressive disclosure:** start here, follow links. Don't grow this file into an
  encyclopedia — push detail into `docs/`.
- **Agent legibility:** if it isn't in this repo as versioned markdown (or a tracked
  submodule), it effectively doesn't exist. Capture decisions here, not in chat.
- **Honesty over completeness:** mark unverified module claims as drafts. A correct
  "we don't know yet" beats a confident guess.
- **Keep the map current:** when a module's status changes, update both its spec and
  the table above.

## Status

Two repos onboarded: **`livepeer-network-modules`** (supply-side core) and
**`livepeer-open-clearinghouse`** (demand-side, non-custodial Payment Clearinghouse that
consumes the network-modules daemons). Most capabilities are now 🟠 Documented; a
standalone full gateway shell and Reference Apps await their own repos. Per-capability
status is in [`docs/product-specs/index.md`](docs/product-specs/index.md); per-repo status
in [`docs/repos/index.md`](docs/repos/index.md).
