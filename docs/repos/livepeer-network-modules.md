# livepeer-network-modules

**Submodule:** `modules/livepeer-network-modules/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-network-modules.git`
**Pinned revision:** `6406a6d` (documented 2026-06-11)
**Status:** 🟠 Onboarded — documented from code

The first and foundational repo in the suite: a **workload-agnostic rearchitecture of
the Livepeer Network supply side**, and the first layer of abstraction over the Livepeer
smart contracts. It is a Go + TypeScript (pnpm) monorepo following the same agent-first
harness pattern as this suite.

## The core idea

The old supply side shipped workload-shaped worker binaries (`openai-worker-node`,
`vtuber-worker-node`, `video-worker-node`) that hard-coded capabilities at build time;
adding a new capability meant forking a worker and cutting a release. This repo replaces
all of that with **one workload-agnostic capability broker** per host that:

- owns its host's `/registry/offerings`,
- reads a single declarative `host-config.yaml`,
- dispatches paid HTTP/streaming/RTMP/session traffic to arbitrary backends, and
- carries **no per-capability code** — only a small fixed set of *interaction modes*.

The orchestrator's day-to-day surface becomes three no-code steps: **define**
capabilities + price, **identify** the backends, **serve**. Adding a capability under an
existing mode is a YAML edit, not a release.

> **Scope note (from the repo, 2026-05-19/20):** the named gateway shells, `gateway-adapters`,
> and the named runner product families were **removed from this repo's working tree**.
> The broker/payment/discovery *contracts* and the `sessionrunner` protocol remain. So
> the **demand-side Gateway, Runners, and reference apps are expected to live in other
> repos**; this repo is the supply-side core + protocol + trust spine.

## The eight-layer architecture

The repo's own [`docs/design-docs/architecture-overview.md`](../../modules/livepeer-network-modules/docs/design-docs/architecture-overview.md)
is the authoritative deep dive. In brief:

1. **Capability broker** — workload-agnostic dispatch, one per host.
2. **Interaction-mode typology** — fixed wire contracts; implemented once per side.
3. **Declarative capability config** — `host-config.yaml` (identity, capabilities, backends).
4. **Discovery** — flat capability-tuple manifest; on-chain pointer → signed off-chain manifest.
5. **Trust spine** — operator-driven, cold-key-signed manifest publication cycle.
6. **Payment** — sender/receiver daemon; opaque capability/work-unit; arithmetic only.
7. **Routing (gateway side)** — resolve tuple → pick mode adapter → wrap headers → forward.
8. **Demand visibility** — comparable Prometheus surfaces on both sides; third-party aggregation.

Host archetypes: `secure-orch` (cold key, firewalled), `orch-coordinator` (public,
key-less publisher), `worker-orch` (broker + receiver + backends), and the gateway
(resolver + sender + adapters).

## Components (15)

Each is a top-level directory with its own `AGENTS.md`/`docs/`. Status reflects the
pinned revision.

| Component | Lang | Purpose | Status |
| --- | --- | --- | --- |
| `capability-broker` | Go | Workload-agnostic per-host dispatch; owns offerings/health; routes to backends; reports usage | Shipped (mode drivers, extractors, metrics, QUIC/WebSocket worker sessions) |
| `payment-daemon` | Go | Sender/receiver micropayment sidecar; ticket validation + on-chain redemption | Shipped (BoltDB sessions, Arbitrum, metrics, payout simulator) |
| `livepeer-network-protocol` | proto+Go | Wire spec: modes, extractors, manifest schema, payment/sessionrunner protos, conformance | Shipped (spec + reference impls + conformance) |
| `proto-contracts` | proto+Go | Generated protobuf bindings shared across daemons | Shipped |
| `orch-coordinator` | Go | Scrapes brokers, builds candidate manifest, publishes signed manifest | Scaffold → building (plan 0018) |
| `secure-orch-console` | Go | Cold-key diff-and-sign console (LAN-only) | Shipped v0.1 (signing, diff, audit log) |
| `protocol-daemon` | Go | On-chain round init, reward, serviceURI writes, orchestrator admin actions | Shipped (gRPC action surface + locked lifecycle) |
| `service-registry-daemon` | Go | Publisher + resolver; fetch/verify/cache signed manifests | Shipped |
| `chain-commons` | Go (lib) | Shared chain/RPC/tx-intent plumbing | Scaffold → building (interfaces + TxIntent shipped) |
| `pool-controller` | Go | Pool control plane: members/backends/offers, broker config, scoring, web admin | Shipped (connected-pool UX + runtime config) |
| `pool-member-agent` | Go | Connected pool member agent: hardware report + outbound broker worker session | Shipped (QUIC preferred, WebSocket fallback) |
| `pool-reconciler` | Go | Round-close accounting producer | Shipped |
| `pool-payout-executor` | Go | Native-ETH member payouts on Arbitrum | Shipped |
| `customer-portal` | TS | Shared SaaS library: API keys, ledger, Stripe top-ups, admin UI | Shipped (library, not a service) |
| `infra` | shell+YAML | Staged compose scenarios + image-build helpers | Working |

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Orchestrators](../product-specs/orchestrators.md) | `capability-broker` + `payment-daemon` (receiver) + `protocol-daemon` + `secure-orch-console` + `orch-coordinator` |
| [Runners](../product-specs/runners.md) | Backends dispatched to by the broker (declared in `host-config.yaml`); `sessionrunner` protocol in `livepeer-network-protocol` |
| [Pools](../product-specs/pools.md) | `pool-controller` + `pool-reconciler` + `pool-payout-executor` |
| [Payment](../product-specs/payment.md) | `payment-daemon` (sender + receiver), on-chain `TicketBroker` |
| [Payment Clearinghouse](../product-specs/payment-clearinghouse.md) | _Not the demand-side clearinghouse_: supply-side receiver settlement + `pool-payout-executor` distribution only |
| [Service Registry](../product-specs/service-registry.md) | `service-registry-daemon` (publisher) + on-chain `ServiceRegistry`/`AIServiceRegistry` + signed manifest |
| [Discover](../product-specs/discover.md) | `service-registry-daemon` (resolver) `Resolver.Select` |
| [SDKs](../product-specs/sdks.md) | `customer-portal` (TS shared library) |
| Protocol | `livepeer-network-protocol` + `proto-contracts` + `chain-commons` |
| [Gateways](../product-specs/gateways.md) | _Contracts only_ — gateway shell/adapters removed from this repo |

## Toolchain & layout

- Pins Node 24 + Go 1.25.7 (`.tool-versions`/`.nvmrc`), `pnpm@9.0.0` via Corepack.
- Root `pnpm` workspace; Go modules per component (auto-toolchain).
- Mainnet-only deployment target: **Arbitrum One**.
- Monorepo "for now" — components may be extracted to standalone repos once they
  stabilize and need independent release cadences.

## Source pointers

- Authoritative map: [`AGENTS.md`](../../modules/livepeer-network-modules/AGENTS.md)
- Architecture: [`docs/design-docs/architecture-overview.md`](../../modules/livepeer-network-modules/docs/design-docs/architecture-overview.md)
- Modes / trust / pricing: `docs/design-docs/{interaction-modes,trust-model,pricing-overview}.md`
- What's shipped: [`docs/exec-plans/completed/`](../../modules/livepeer-network-modules/docs/exec-plans/completed/)
- In flight: [`docs/exec-plans/active/`](../../modules/livepeer-network-modules/docs/exec-plans/active/) and [`PLANS.md`](../../modules/livepeer-network-modules/PLANS.md)
