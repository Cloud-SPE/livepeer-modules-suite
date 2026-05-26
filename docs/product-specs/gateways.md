# Gateways

**Status:** 🟡 Stub — contracts known, gateway repo awaited
**Submodule:** _TBD — gateway shell/adapters were removed from `livepeer-network-modules`_

## What this is

A **Gateway** is the demand-side entry point: it accepts work from applications or end
users, **discovers** and selects an orchestrator, attaches **payment**, opens the right
transport, and forwards traffic. Historically this role was the "Broadcaster."

The first repo ([livepeer-network-modules](../repos/livepeer-network-modules.md)) defines
the **contracts a gateway talks to** but no longer contains a gateway shell — the named
gateway products and `gateway-adapters` were removed from its working tree (2026-05-19).

**The gateway role is split in practice.** The
[Payment Clearinghouse](../repos/livepeer-open-clearinghouse.md) (the
`livepeer-open-clearinghouse-gateway` service) owns the **control-plane half** — auth,
credit, discovery proxy, and minting the payment envelope — while the **data-plane half**
(talking the broker's interaction modes, sending `Livepeer-Payment`, reading work units)
lives in the customer's [SDK](sdks.md). A standalone, full data-plane gateway shell may
still arrive as its own repo.

## What the contracts tell us a gateway does

1. **Resolve** a route via [Discover](discover.md) → gets a tuple including
   `interaction_mode`.
2. **Pick the mode adapter** (reqresp / stream / multipart / ws / rtmp / session) — code
   is **per-mode, not per-capability**.
3. **Mint payment** via the [Payment](payment.md) sender daemon.
4. **Wrap headers** — `Authorization` (customer bearer), `Livepeer-Payment` (ticket),
   `Livepeer-Capability`, `Livepeer-Offering` — open transport, forward to the broker.
5. Apply **local route-health cooldowns** on top of manifest + live health.

The only per-workload code is the customer-facing surface (e.g. OpenAI-shaped routes);
everything beneath is capability-agnostic.

## Role in the suite

- **Consumes:** [Discover](discover.md), [Service Registry](service-registry.md),
  [Payment](payment.md) (sender).
- **Talks to:** [Orchestrators](orchestrators.md) / [Pools](pools.md) brokers.
- **Consumed by:** [Reference Apps](reference-apps.md) and external apps, via [SDKs](sdks.md).

## To document when a gateway repo is provided

- [ ] Which gateway shells exist and their customer-facing surfaces
- [ ] How adapters per interaction mode are implemented
- [ ] Auth/customer model and how it ties to [customer-portal](sdks.md)
- [ ] Deployment, config, pinned revision
