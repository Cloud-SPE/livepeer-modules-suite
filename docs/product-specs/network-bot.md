# Network Bot (Discord Reporting)

**Group:** Observability & Reporting (off-network)
**Status:** 🟠 Documented via [livepeer-network-bot](../repos/livepeer-network-bot.md) (`0f62e37`)
**Submodule:** `modules/livepeer-network-bot/`

## What this is

The **Network Bot** turns Livepeer protocol activity into **Discord** messages. It is the
reporting companion to the [Protocol Explorer](protocol-explorer.md): it polls the
explorer's HTTP API, keeps durable state in SQLite, and posts payout digests and summaries
to a public channel — and, in commands mode, runs an interactive Discord bot with
per-user orchestrator subscriptions and DM alerts.

## What it provides

- **Base (webhook) mode:** per-orchestrator **payout digests** for `WinningTicketRedeemed`
  (grouped AI vs transcoding) + scheduled **daily/weekly/monthly** summary embeds, with
  durable cursor + delivery tracking so it resumes safely across restarts.
- **Commands mode** (`COMMANDS_ENABLED=true`): slash commands (`/subscribe`,
  `/unsubscribe`, `/subscriptions`, `/orchestrator …`), capped per-user subscriptions, DM
  reward alerts, periodic delegator-activity digests, and startup seeding.

Message formatting is a snapshot-tested contract (`messages.md`).

## Role in the suite

- **Depends on:** the [Protocol Explorer](protocol-explorer.md) API (its sole upstream).
- **Reports to:** Discord (public webhook + per-user DMs).

## Confirmed / open

- ✅ Single binary / single SQLite / single explorer API / single Discord surface.
- ✅ Typed explorer client (no raw JSON); deterministic delivery semantics.
- [ ] v0.1.0; reliability-first scope (intentionally narrow feature set).
