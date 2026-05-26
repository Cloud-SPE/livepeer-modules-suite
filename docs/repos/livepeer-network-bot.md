# livepeer-network-bot

**Submodule:** `modules/livepeer-network-bot/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-network-bot.git`
**Pinned revision:** `0f62e37` (documented 2026-05-26)
**Status:** 🟠 Onboarded — documented from code. Crate `livepeer-payout-bot` v0.1.0;
opinionated about reliability over breadth.

> **Observability category** (see [protocol-explorer](livepeer-protocol-explorer.md)). The
> reporting companion to the explorer: a Livepeer-to-Discord notification bot.

A single Rust service that turns Livepeer activity into Discord messages. One binary, one
SQLite DB, one upstream (the explorer API), one Discord surface, with strict module
boundaries and snapshot-tested message formatting.

## Cross-repo dependency: it consumes the explorer API

The bot's **only** upstream is [livepeer-protocol-explorer](livepeer-protocol-explorer.md)'s
HTTP API — it does not touch chain nodes. A typed client (`progenitor`-generated from the
explorer's `openapi.json`) calls `/api/v1/events` (WinningTicketRedeemed, Reward, Bond,
Unbond, Rebond), `/api/v1/orchestrators/{addr}` (+ `/delegators`),
`/api/v1/gateways/{addr}/profile`, `/api/v1/payouts/summary/{period}/{date}`, and the
payout/reward leaderboards. Durable cursor + delivery state lives in SQLite (WAL), so
polling and posting resume safely across restarts (cursors advance only after a page is
persisted; rows are marked sent only after a 2xx delivery).

## Base mode (webhook-only, always on)

- **`event_poller`** — polls WinningTicketRedeemed into SQLite.
- **`digest_poster`** — on digest-window boundaries, posts public Discord webhook embeds:
  per-orchestrator **payout digests** grouped by job type (AI vs transcoding), single- or
  multi-ticket.
- **`summary_poster`** — scheduled **daily / weekly / monthly** payout summary embeds, with
  a `summary_watermarks` table preventing duplicates.
- Safety: `WEBHOOK_POST_ENABLED=false` keeps polling but suppresses posting (so dev can
  share a prod webhook URL without double-posting; backlog drains when re-enabled).

## Commands mode (`COMMANDS_ENABLED=true`)

Adds a Discord gateway bot (poise/serenity) with slash commands `/subscribe`,
`/unsubscribe`, `/subscriptions`, `/orchestrator {delegators,rewards,tickets}`; per-user
orchestrator **subscriptions** (capped, `MAX_SUBSCRIPTIONS_PER_USER`); **DM** alerts for
reward events; periodic **delegator-activity digest** DMs (bonds/unbonds/rebonds/stake
increases); and startup **seeding** of `delegator_history` so new-vs-existing delegators
are classified correctly from the start. DM failures increment a counter and
auto-unsubscribe at a threshold.

## Config / deployment modes

Strict env validation at startup (`src/config.rs`, `.env.example`):

- Always: `EXPLORER_BASE_URL`, `DISCORD_WEBHOOK_URL`, `DATABASE_URL`.
- Commands mode also: `DISCORD_BOT_TOKEN`, `DISCORD_APPLICATION_ID`, optional
  `DISCORD_GUILD_ID`, `MAX_SUBSCRIPTIONS_PER_USER`, `DM_FAILURE_AUTO_UNSUB`.
- Tunables: poll intervals, digest window, fetch limits, `WEBHOOK_POST_ENABLED`.

## Message contract

`docs/product-specs/messages.md` is the contract (embed envelopes, colors, fields for
single/multi-ticket digests, summaries, reward DMs, delegator digests). It is
**snapshot-tested** in `tests/embeds.rs` — output drift fails CI. Module boundaries are
enforced by `tests/architecture.rs`.

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Network Bot](../product-specs/network-bot.md) | The whole service — Discord reporting of protocol activity |
| (depends on) [Protocol Explorer](../product-specs/protocol-explorer.md) | Sole upstream data source (HTTP API) |

## Stack & status

Rust (Tokio, reqwest+rustls, sqlx/SQLite, poise/serenity, progenitor, chrono). v0.1.0,
MIT, actively maintained.

## Source pointers

- [`README.md`](../../modules/livepeer-network-bot/README.md) ·
  [`src/runtime.rs`](../../modules/livepeer-network-bot/src/runtime.rs) ·
  [`src/config.rs`](../../modules/livepeer-network-bot/src/config.rs)
- [`docs/design-docs/architecture.md`](../../modules/livepeer-network-bot/docs/design-docs/architecture.md) ·
  [`docs/product-specs/messages.md`](../../modules/livepeer-network-bot/docs/product-specs/messages.md)
