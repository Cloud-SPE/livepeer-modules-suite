# Payment Clearinghouse

**Status:** 🟠 Partial via [livepeer-network-modules](../repos/livepeer-network-modules.md) (`95c6415`)
**Submodule:** `modules/livepeer-network-modules/` (`payment-daemon` receiver, `pool-payout-executor`)

> Open question (Payment vs Clearinghouse boundary) is **partially resolved**. There is no
> standalone "clearinghouse" service in the first repo; the clearing/settlement function
> is split across components below. A dedicated clearinghouse repo + SDKs may still be
> onboarded later — confirm and update.

## What this is — current understanding

Where [Payment](payment.md) is the per-unit instrument exchanged during work, the
**clearinghouse function** is the settlement, redemption, and distribution of that value.
In `livepeer-network-modules` this is realized by:

- **`payment-daemon` (receiver):** off-chain balance ledger + redemption of winning
  tickets on-chain.
- **On-chain `TicketBroker` (Arbitrum One):** validates redeemed tickets and pays the
  orchestrator's recipient (cold-key) address.
- **`pool-payout-executor`:** for pooled orchestrators, distributes realized round
  revenue to members as native-ETH payouts on Arbitrum.

So: payment-daemon = sender/receiver session + settlement; pool-payout-executor =
inter-member distribution. Together they cover clearing for the supply side.

## SDKs

The first repo ships `customer-portal` (a TS shared library with billing/ledger/Stripe
surfaces) — see [SDKs](sdks.md). Whether a separate clearinghouse SDK exists is TBD.

## Role in the suite

- **Settles:** payments from [Payment](payment.md) between [Gateways](gateways.md) and
  [Orchestrators](orchestrators.md).
- **Distributes:** pooled revenue to [Pools](pools.md) members.

## Confirmed / open

- ✅ Clearing is realized via receiver + on-chain `TicketBroker` + `pool-payout-executor`.
- [ ] Is there a dedicated Payment Clearinghouse repo (with its own SDKs) coming? Confirm
  and, if so, re-scope this spec to it.
- [ ] Exact Payment↔Clearinghouse responsibility split once any dedicated repo lands.
