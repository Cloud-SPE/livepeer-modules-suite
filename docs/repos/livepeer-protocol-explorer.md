# livepeer-protocol-explorer

**Submodule:** `modules/livepeer-protocol-explorer/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-protocol-explorer.git`
**Pinned revision:** `88f0ff8` (documented 2026-06-11)
**Status:** 🟠 Onboarded — documented from code. Spec v1.9; v1 complete and operationally
deployed (a few backfill/throughput items open in its own tech-debt tracker).

> **New category — Observability, not network.** This repo and
> [livepeer-network-bot](livepeer-network-bot.md) **track on-chain activity**; they do not
> provide network supply or accept inbound demand. They are read-only observers of the
> same Arbitrum One contracts the network side writes to.

A full **Livepeer protocol data platform** for Arbitrum One (Rust + Postgres): it ingests
every Livepeer on-chain event, prices monetary activity per block from on-chain oracles,
derives stake/profile/analytics state, builds rollups, and serves it all through a
versioned HTTP API and a bundled web explorer SPA. Its load-bearing guarantee is
**byte-deterministic replay** — same cached inputs reproduce a bit-identical database.

## Worker pipeline

A shared `core` lib plus ~12 worker binaries, run live under `livepeer-daemon` or one-shot
via `livepeer-orchestrator`:

- **Indexing:** `livepeer-indexer` (decode logs → immutable `raw_protocol_events`),
  `livepeer-reorg-watcher` (parent-hash continuity, mark non-canonical),
  `livepeer-finality-watcher` (advance finality as L1 batches post).
- **Pricing & derivation:** `livepeer-valuator` (price finalized events → versioned
  `event_valuations`), `livepeer-staker` (stake balances, gateway state, profile views).
- **Aggregation & enrichment:** `livepeer-rollups` (daily aggregates), `livepeer-enricher`
  (ENS names/avatars).
- **Serving & ops:** `livepeer-api` (Axum HTTP + SPA), `livepeer-seed-migrator`
  (SQLite→Postgres historical seed), `livepeer-alert-bot` (Telegram/Discord ops alerts).

## Valuation

Every monetary event is priced **at its own block** using on-chain sources only:
`LPT/USD = TWAP_30min(LPT/WETH, Uniswap V3) × Chainlink(ETH/USD)`; `ETH/USD` direct from
Chainlink. A Chainlink L2 Sequencer Uptime feed gates pricing during sequencer downtime.
Records are **immutable + versioned** (e.g. `v1_lpt_weth_twap_30min_x_chainlink_eth`, with
a degraded fallback for pre-cardinality blocks). Every RPC call is cached
(`rpc_call_cache`) with a primary/secondary bytes-equal cross-check, which is what makes
replay deterministic. Priced events: LPT-valued (Bond/Unbond/Rebond/Reward/Transfer/…) and
ETH-valued (WinningTicketRedeemed, Deposit/Reserve funded, Withdrawal, EarningsClaimed fee).

## Data model & API

- **Raw + valuation:** `raw_protocol_events`, `event_valuations`, `token_prices_by_block`,
  reorg/decode audit tables.
- **Derived:** `stake_balances_by_block`, `orch_stake_by_round`, gateway balances/flows,
  `orchestrator_profile` / `broadcaster_profile` (materialized views), `delegator_registry`.
- **Rollups:** `orch_payouts_daily`, `orch_rewards_daily`, `tickets_daily`,
  `event_metrics_daily`, and current staker/delegation state.
- **API (`/api/v1/*`):** events + valuations, price lookups, payout/reward summaries +
  leaderboards, ticket timeseries, governance proposals/votes, gateway analytics,
  orchestrator profiles/economics, delegator/stake history, round views, network stats,
  CSV exports; plus ops endpoints (`/health`, `/metrics`, `/config.json`, `/openapi.json`,
  `/docs`, `/backfills/status`). Route inventory: `crates/livepeer-api/src/lib.rs`.

## Frontend

A Lit 3 + Vite + RxJS SPA (ECharts, modern CSS) bundled into the Axum process. Sections:
**Dashboard, Orchestrators, Gateways, Reports, Rewards, Governance, Delegators, Rounds**
(`frontend-ui/src/components/side-nav.ts`).

## Contracts tracked (the link to the rest of the suite)

It indexes the **same Arbitrum One contracts the network side interacts with** —
`BondingManager` (Bond/Unbond/Rebond/Reward/EarningsClaimed…), `TicketBroker`
(WinningTicketRedeemed, Deposit/Reserve funded…), `RoundsManager` (NewRound),
`LivepeerToken`, and the Governor — resolved via the Livepeer `Controller`. In other
words, it observes the **economic output** that [Payment](../product-specs/payment.md),
[Orchestrators](../product-specs/orchestrators.md), and [Pools](../product-specs/pools.md)
produce on-chain. (Note: it reads the protocol directly from chain, **not** from the
suite's `service-registry-daemon`/`payment-daemon`.)

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Protocol Explorer](../product-specs/protocol-explorer.md) | The whole platform — indexing, valuation, analytics, API, SPA |
| (consumed by) [Network Bot](../product-specs/network-bot.md) | The bot polls this repo's `/api/v1/*` |

## Stack & status

Rust 1.94 workspace (Tokio, Alloy, sqlx/Postgres 17, Axum, utoipa); deterministic-replay
CI. v1 deployed; latest pin includes delegation-state correctness fixes and safer null
USD amount decoding in report/payout APIs; open items in its own
[`docs/exec-plans/tech-debt-tracker.md`](../../modules/livepeer-protocol-explorer/docs/exec-plans/tech-debt-tracker.md)
(e.g. LPT on-chain backfill throughput TD-011, finality heuristic TD-008).

## Source pointers

- [`README.md`](../../modules/livepeer-protocol-explorer/README.md) ·
  [`docs/ARCHITECTURE.md`](../../modules/livepeer-protocol-explorer/docs/ARCHITECTURE.md) ·
  [`crates/livepeer-api/src/lib.rs`](../../modules/livepeer-protocol-explorer/crates/livepeer-api/src/lib.rs)
- spec: [`docs/product-specs/v1-livepeer-indexer.md`](../../modules/livepeer-protocol-explorer/docs/product-specs/v1-livepeer-indexer.md)
