# Discover

**Status:** 🟠 Documented via [livepeer-network-modules](../repos/livepeer-network-modules.md) (`689b51a`)
**Submodule:** `modules/livepeer-network-modules/service-registry-daemon/` (resolver mode)

## What this is — open question resolved

"Discover" is the **resolver** half of `service-registry-daemon` — specifically the
`Resolver.Select` gRPC API. It is **not a separate service**, and it's more than a
library: it's a unix-socket gRPC daemon mode that gateways call to find an orchestrator
and resolve pricing.

Given `(capability_id, offering_id?, tier?, min_weight?)`, it returns a route tuple:
`{ worker_url, eth_address, interaction_mode, work_unit, price_per_unit_wei, extra }`.
The gateway picks its mode adapter from `interaction_mode` in this response — not from any
per-capability lookup table.

## Fetch flow

1. Per-round refresh (cron-driven, ~19h on Arbitrum One): enumerate orchestrators via
   `BondingManager`, read each `serviceURI` from the registry, fetch the signed manifest,
   **verify the signature**, flatten to tuples, cache.
2. Hot path: `Resolver.Select(...)` returns a cached route. Selection also applies live
   broker health (route admission) and, on the gateway, local recent-outcome cooldowns.

## Consumers

- **Gateway / SDK** may call the resolver directly in daemon-integrated deployments.
- The [Payment Clearinghouse](payment-clearinghouse.md) exposes an **auth-aware discovery
  proxy** (its `discovery` domain) over `service-registry-daemon` — `Resolve` / `Select` /
  `SelectMany` with an in-process TTL cache — so customers can list capabilities,
  orchestrators, and routes through the same authenticated HTTP API. `SelectedRoute.extra`
  carries the `interaction_mode`.
- The onboarded OpenAI and transcode gateways now consume discovery **through LOC**: the
  gateway opens a LOC job/session and receives the selected broker route in the response;
  local catalog endpoints are refreshed from LOC's capability catalog.

## Role in the suite

- **Reads:** [Service Registry](service-registry.md) (on-chain pointer + signed manifest).
- **Serves:** [Gateways](gateways.md), SDKs, and the [Payment Clearinghouse](payment-clearinghouse.md) proxy.
- **Selects among:** [Orchestrators](orchestrators.md) / [Pools](pools.md).

## Confirmed / open

- ✅ Resolver is a gRPC mode of the registry daemon; workload-agnostic (opaque filters).
- ✅ `interaction_mode` rides in the resolver response.
- [ ] Selection/scoring tunables (tier, min_weight semantics) — expand if needed.
