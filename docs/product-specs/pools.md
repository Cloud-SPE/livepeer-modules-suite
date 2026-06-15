# Pools

**Status:** 🟠 Documented via [livepeer-network-modules](../repos/livepeer-network-modules.md) (`689b51a`)
**Submodule:** `modules/livepeer-network-modules/` (`pool-controller`, `pool-member-agent`, `pool-reconciler`, `pool-payout-executor`)

## What this is — open question resolved

In this suite, **"Pools" means a control plane that aggregates multiple member backends
behind one orchestrator identity, with round-based accounting and member payouts.** From
a Gateway's perspective a Pool looks like one orchestrator (one manifest, one broker
endpoint); the pooling machinery stays inside the Pool operator's control plane.

This is **capacity + accounting pooling**, distinct from classic on-chain stake
delegation. Pools are optional — a single-host orchestrator runs the broker +
payment-daemon without any pool component.

## The three pool components

- **`pool-controller`** — owns persisted Pool state (members, backends, offers,
  backend↔offer assignments), renders the broker's runtime config from that state,
  ingests work receipts, and runs **backend-selection scoring** (cooldown, EMA, latency,
  warm-up) to decide routing.
- **`pool-member-agent`** — runs on connected member hosts. It reports hardware inventory,
  keeps an outbound worker session open to the broker, prefers QUIC when available, and
  falls back to WebSocket for UDP-blocked networks. Members do not need inbound broker,
  payment-daemon, TLS, DNS, or wallet infrastructure.
- **`pool-reconciler`** — closes rounds using `protocol-daemon` round timing,
  `payment-daemon` realized revenue, and `pool-controller` work receipts; emits the
  round-close payload.
- **`pool-payout-executor`** — executes **native-ETH** member payouts on Arbitrum and
  writes back payout state.

## Accounting chain

Work receipt (per request, emitted by broker) → round receipt (aggregated at round close)
→ payout intent per member (`pending → exported → leased → submitted → paid` / `failed`,
with auto-requeue on transient failure).

## Role in the suite

- **Fronts:** member backends ([Runners](runners.md) / [Orchestrators](orchestrators.md)).
- **Presents to:** [Gateways](gateways.md) a single orchestrator identity.
- **Settles via:** [Payment](payment.md) (realized revenue) and the
  [Payment Clearinghouse](payment-clearinghouse.md) function (member distribution).

## Confirmed / open

- ✅ Pool = capacity aggregation + round accounting + ETH member payouts (not staking).
- ✅ Gateway sees one orch identity; pool internals are invisible externally.
- ✅ Member admission/join-request UX, connected-pool session state, broker runtime
  rendering, and member hardware reporting are implemented in `pool-controller` plus
  `pool-member-agent`.
