# Orchestrators

**Status:** 🟠 Documented via [livepeer-network-modules](../repos/livepeer-network-modules.md) (`689b51a`)
**Submodule:** `modules/livepeer-network-modules/`

## What this is

An **Orchestrator** is the supply-side participant that advertises capabilities, accepts
paid work, and is identified on-chain by an Ethereum address. In the new architecture an
orchestrator is **not a single worker binary** — it's a set of cooperating processes
fronted by one **workload-agnostic capability broker** per host.

The orchestrator's day-to-day surface is three no-code steps: **define** capabilities +
price in `host-config.yaml`, **identify** the backend ([Runners](runners.md)), **serve**.

## The processes that make an orchestrator

- **`capability-broker`** — one per host. Owns `GET /registry/offerings` and
  `/registry/health`, routes inbound paid requests by `Livepeer-Capability` header to the
  declared backend wrapped in the declared interaction mode, and reports `actualUnits` to
  the payment daemon. Carries zero per-capability code. Current code also includes broker
  metrics, registry/backend health gauges, and outbound worker sessions over QUIC.
- **`payment-daemon` (receiver)** — validates tickets, tracks balances, redeems winning
  tickets on-chain. See [Payment](payment.md).
- **`orch-coordinator`** — public, key-less; scrapes broker offerings, builds a candidate
  manifest, and publishes the signed manifest. See [Service Registry](service-registry.md).
- **`secure-orch-console` + cold key** — the firewalled trust spine that signs manifests.
  An opt-in agent mode (plan 0042) auto-signs within an operator-authored sign-policy
  envelope; anything outside it is held for operator review.
- **`protocol-daemon`** — on-chain round init, reward, and `serviceURI` writes. Current
  code also includes orchestrator admin actions, reward/round-init lock handling,
  treasury/op-config support, and gRPC action endpoints used by operator consoles.

## Role in the suite

- **Advertises via:** [Service Registry](service-registry.md).
- **Selected by:** [Gateways](gateways.md) through [Discover](discover.md).
- **Dispatches to:** [Runners](runners.md) (backends).
- **Paid through:** [Payment](payment.md); may be operated as a [Pool](pools.md).

## Request lifecycle (broker)

Resolve `(capability_id, offering_id)` from headers → validate payment **before** the
backend call → forward to backend → run extractor → report `actualUnits` → return
response. Mismatched routing fails closed; the broker knows nothing about money beyond
"did the payment daemon say yes."

## Confirmed / open

- ✅ Workload-agnostic broker, declarative config, opaque capability/work-unit names.
- ✅ Cold-key trust spine; broker never holds the orch key.
- [ ] Standalone gateway/runner repos that pair with this orchestrator (separate repos).
