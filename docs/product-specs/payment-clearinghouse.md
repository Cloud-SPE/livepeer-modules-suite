# Payment Clearinghouse

**Status:** 🟠 Documented via [livepeer-open-clearinghouse](../repos/livepeer-open-clearinghouse.md) (`a529592`)
**Submodule:** `modules/livepeer-open-clearinghouse/`

> Open question resolved: there **is** a dedicated clearinghouse repo (this one), and it
> ships reference SDKs. The earlier "partial clearinghouse in network-modules" framing is
> superseded — see the boundary below.

## What this is

A **non-custodial-by-design payment clearinghouse** for Livepeer app developers. It
authenticates developers, holds their **wei-denominated credit balance**, and mints
signed Livepeer payment tickets **on their behalf** through a single operator-owned
**pooled wallet**. Customers integrate one HTTP API and never manage a wallet or signing
key.

It is a single Python/FastAPI service (the `livepeer-open-clearinghouse-gateway`
container) that runs alongside Postgres and, from
[livepeer-network-modules](../repos/livepeer-network-modules.md), the `payment-daemon`
(sender) and `service-registry-daemon` (resolver).

## Control plane, not data plane (handoff mode)

The clearinghouse mints a payment envelope and **hands it off** to the customer's SDK,
which then talks to the orchestrator broker **directly**. The clearinghouse is not on the
hot media/inference path. Charging is **expected-value at issuance** (encumbered from the
credit balance when the ticket is minted), reconciled to actual usage at settle time.

- **Jobs** — one-shot request/response work: mint → call broker → settle once.
- **Sessions** — long-lived, refillable work: open → refill on `Livepeer-Balance-Low` →
  close, with a background janitor reconciling against the daemon's authoritative
  `GetSessionDebits`.

See the repo doc for the full flow and the
[`docs/HANDOFF_MODE.md`](../../modules/livepeer-open-clearinghouse/docs/HANDOFF_MODE.md).

## What it owns

Auth/accounts, API keys, **billing** (credit ledger, top-ups, spend caps, auto-replenish),
discovery proxy, jobs/sessions issuance + settlement, usage reconciliation, telemetry,
notifications, and an operator admin console (incl. SDK approval registry + signed SDK
manifest).

## Boundary vs Payment and the network-modules clearing function

- [Payment](payment.md) is the **ticket primitive + on-chain settlement** (`payment-daemon`
  + `TicketBroker`), shared infrastructure.
- This clearinghouse is the **customer-facing credit + mint-on-behalf control plane** that
  sits in front of it.
- The supply-side **`pool-payout-executor`** (in network-modules) handles member payout
  distribution — a distinct clearing concern from this demand-side clearinghouse.

## Role in the suite

- **Fronts:** customer apps (via [SDKs](sdks.md)).
- **Consumes:** [Payment](payment.md) (`payment-daemon`) and [Discover](discover.md)
  (`service-registry-daemon`) from network-modules.
- **Hands off to:** the [Orchestrators](orchestrators.md) broker (SDK ↔ broker direct).

## Confirmed / open

- ✅ Dedicated, non-custodial clearinghouse; pooled wallet held only by `payment-daemon`.
- ✅ Handoff-mode jobs/sessions; EV-at-issuance with settle-time reconciliation.
- [ ] Repo is pre-alpha; horizontal scaling, blocked-SDK enforcement, and some operator
  tooling are deferred (v1.1/v2). Track maturity via the repo's `docs/QUALITY_SCORE.md`.
