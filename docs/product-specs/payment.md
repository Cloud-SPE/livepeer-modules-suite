# Payment

**Status:** 🟠 Documented via [livepeer-network-modules](../repos/livepeer-network-modules.md) (`95c6415`)
**Submodule:** `modules/livepeer-network-modules/payment-daemon/`

## What this is — open question resolved

Payment uses **probabilistic micropayment tickets**, exchanged between a gateway-side
**sender** and a worker-side **receiver** (`payment-daemon`), settling on-chain through
the `TicketBroker` contract on Arbitrum One. The big architectural change vs. the old
suite: the daemon **stops enforcing a closed enum of capability/work-unit names** — both
are opaque strings, and the daemon just does arithmetic: `price = price_per_unit_wei ×
actualUnits`. Custom work units (`barks`, `pixel-seconds`, …) work with no trunk change.

## How it works

- **Sender (gateway side):** `CreatePayment(...)` mints a signed ticket carried in the
  `Livepeer-Payment` header.
- **Receiver (worker side):** validates the ticket **before** the backend call. Winning
  tickets are redeemed on-chain via `TicketBroker`; non-winning tickets are credited as
  **expected value** (`face_value × win_prob`) to an off-chain balance.
- **True-up:** after the backend responds, the broker reports `actualUnits`, so over- and
  under-spend are corrected rather than gambled.

### Per-request vs streaming

- **Request modes** (`http-reqresp` / `-stream` / `-multipart`): one ticket per request +
  usage true-up.
- **Streaming/session modes** (`ws-realtime` / `session-control-plus-media` / `rtmp-…`):
  amortized — `OpenSession` → periodic `Debit` + top-ups → `CloseSession`.
  **Worker meters, gateway ledgers;** usage ticks are idempotent so retries never
  double-charge.

### Who mints, in practice

In the [Payment Clearinghouse](payment-clearinghouse.md) model, the customer does **not**
run a sender. The clearinghouse calls `payment-daemon.CreatePayment` with a single
operator-owned **pooled wallet** and hands the signed envelope to the customer's SDK
(handoff mode); the customer is charged **expected value at issuance** against a wei
credit balance and trued up on settle. So `payment-daemon` (this module) is the shared
ticket engine; the clearinghouse is the credit/accounting layer in front of it.

## Role in the suite

- **Sent by:** the [Payment Clearinghouse](payment-clearinghouse.md) (pooled wallet) on
  behalf of customers; conceptually the [Gateways](gateways.md) / [Pools](pools.md) role.
- **Received by:** [Orchestrators](orchestrators.md) (broker reports usage to the receiver).
- **Settled/distributed by:** on-chain `TicketBroker` + supply-side `pool-payout-executor`.

## Confirmed / open

- ✅ Probabilistic tickets; opaque capability/work-unit; EV credit + on-chain redemption.
- ✅ `Livepeer-Payment` header stays wire-compatible; routed tuple in sibling headers.
- [ ] Warm/cold key handling details for redemption (see repo plan 0017) — expand if needed.
