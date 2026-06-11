# Protocol Explorer

**Group:** Observability & Reporting (off-network)
**Status:** 🟠 Documented via [livepeer-protocol-explorer](../repos/livepeer-protocol-explorer.md) (`88f0ff8`)
**Submodule:** `modules/livepeer-protocol-explorer/`

## What this is

The **Protocol Explorer** is the suite's on-chain **data platform**: it indexes every
Livepeer event on Arbitrum One, prices monetary activity per block from on-chain oracles,
derives stake/profile/analytics state, and serves it through a versioned HTTP API and a web
explorer SPA. It is a *read-only observer* of the protocol — it neither provides network
supply nor accepts inbound demand.

## Where it sits in the suite

This is a different axis from the network capabilities. The supply side
([Orchestrators](orchestrators.md), [Pools](pools.md)), the [Payment](payment.md) layer,
and the [Gateways](gateways.md) all produce **on-chain economic activity** (rewards,
bonds, tickets, payouts) on the Livepeer contracts. The Protocol Explorer **observes and
accounts for that activity** — independently, by reading the chain directly (not via the
suite's daemons). It is the analytical mirror of the network.

## What it provides

- **Deterministic indexing & replay** — immutable raw events; cached RPC; same inputs
  reproduce the same database.
- **Historical valuation** — rewards/bonds/tickets/payouts priced per block via Uniswap V3
  TWAP × Chainlink, as versioned immutable records.
- **Derived analytics** — stake balances, gateway state, orchestrator/broadcaster
  profiles, current delegator/orchestrator state, daily rollups
  (payouts/rewards/tickets/event metrics), leaderboards.
- **Broad read API + explorer UI** — events/valuations, prices, summaries, governance,
  gateway/orchestrator/delegator/round views, CSV exports; a Lit SPA (Dashboard,
  Orchestrators, Gateways, Reports, Rewards, Governance, Delegators, Rounds).

## Role in the suite

- **Observes:** the Livepeer Arbitrum One contracts (`BondingManager`, `TicketBroker`,
  `RoundsManager`, `LivepeerToken`, Governor).
- **Consumed by:** the [Network Bot](network-bot.md) (Discord reporting), dashboards, and
  any analytics client.

## Confirmed / open

- ✅ Full data platform (indexer + valuation + analytics + API + SPA), deterministic replay.
- ✅ Reads chain directly; independent of the suite's `service-registry-daemon`/`payment-daemon`.
- ✅ Latest pin includes delegation-state correctness fixes and safer null USD amount
  decoding in payout/report APIs.
- [ ] A few backfill/throughput items remain in the repo's own tech-debt tracker.
