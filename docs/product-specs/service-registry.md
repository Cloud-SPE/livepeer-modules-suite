# Service Registry

**Status:** 🟠 Documented via [livepeer-network-modules](../repos/livepeer-network-modules.md) (`95c6415`)
**Submodule:** `modules/livepeer-network-modules/service-registry-daemon/` (+ `orch-coordinator`)

## What this is — open question resolved

The Service Registry is **hybrid: an on-chain pointer plus an off-chain signed
manifest.** Only a URL lives on-chain; all capability/price/backend metadata lives in a
cold-key-signed JSON manifest at that URL.

- **On-chain (Arbitrum One):** `ServiceRegistry.getServiceURI(orch_addr)` (and the newer
  `AIServiceRegistry`) return the orchestrator's manifest URL. That's the only on-chain
  storage — no prices, no capabilities.
- **Off-chain manifest:** published at `/.well-known/livepeer-registry.json`, it's a flat
  list of [capability tuples](../glossary.md#capability-broker--metering). One orch
  publishes **one** signed manifest mixing transcoding and AI tuples, and registers the
  same URL with whichever contract(s) it participates in.

## How it's written and read

- **Publisher (`orch-coordinator` + `service-registry-daemon` publisher mode):** scrapes
  broker `/registry/offerings`, builds a candidate manifest, and (after cold-key signing
  via the [trust spine](orchestrators.md)) atomic-swap publishes it.
- **Resolver (consumer side):** reads the on-chain pointer, fetches the manifest, and
  **re-verifies the signature** against on-chain orch identity (defense in depth — the
  coordinator host is not trusted). See [Discover](discover.md).

**Two verifications, intentionally:** coordinator verifies on upload; every resolver
verifies again on fetch.

## Role in the suite

- **Written by:** [Orchestrators](orchestrators.md) / [Pools](pools.md).
- **Read by:** [Discover](discover.md) → [Gateways](gateways.md).

## Confirmed / open

- ✅ On-chain pointer + off-chain signed manifest; capabilities are opaque tuples.
- ✅ Host is **not** a registration unit — capability tuples are.
- [ ] Manifest schema specifics live in `livepeer-network-protocol/manifest/` — expand if needed.
