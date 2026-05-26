# Livepeer Modules Suite

An umbrella repository that aggregates the **Livepeer Modules** — the productized
features that enable the Livepeer protocol — as git submodules, and provides a single
high-level overview of the whole suite and how its parts fit together.

This repo holds **documentation and submodule pointers**, not module source code. Each
module's code lives in its own repository under [`modules/`](modules/README.md).

## Start here

- **[`AGENTS.md`](AGENTS.md)** — the map: what this repo is and where everything lives.
- **[`ARCHITECTURE.md`](ARCHITECTURE.md)** — how a unit of work flows across the suite.
- **[`docs/glossary.md`](docs/glossary.md)** — shared vocabulary.
- **[`docs/product-specs/`](docs/product-specs/index.md)** — one overview per capability.
- **[`docs/repos/`](docs/repos/index.md)** — one doc per onboarded submodule.
- **[`docs/guides/git-submodules-primer.md`](docs/guides/git-submodules-primer.md)** —
  how to clone, update, and pin submodules to network releases.

## The capabilities

Gateways · Orchestrators · Runners · Pools · Payment · Service Registry · Discover ·
Payment Clearinghouse (+ SDKs) · Reference Apps. See
[`docs/product-specs/index.md`](docs/product-specs/index.md) for status.

## Onboarded repos

- **[livepeer-network-modules](docs/repos/livepeer-network-modules.md)** — the
  supply-side core (workload-agnostic capability broker + payment, pools, registry,
  discovery, protocol, trust spine). The first abstraction layer over the smart contracts.
- **[livepeer-open-clearinghouse](docs/repos/livepeer-open-clearinghouse.md)** — the
  demand-side, non-custodial Payment Clearinghouse (auth, prepaid credit, mint-on-behalf,
  handoff-mode jobs/sessions). Consumes the network-modules daemons.

## Getting the code

```bash
git clone --recurse-submodules <this-repo-url>
# or, if already cloned:
git submodule update --init --recursive
```

## How this repo is organized

It follows an agent-first, documentation-as-system-of-record approach (see
[`docs/references/openai-harness-engineer.md`](docs/references/openai-harness-engineer.md)
for the inspiration): a short map (`AGENTS.md`) points into a structured `docs/`
directory, and every claim is meant to be verifiable in-repo. Operating principles are
in [`docs/design-docs/core-beliefs.md`](docs/design-docs/core-beliefs.md).

> **Status:** first repo onboarded. Structure and workflow are in place and the
> supply-side capabilities are documented from `livepeer-network-modules`; Gateways,
> Runners, and Reference Apps await their own repos.
