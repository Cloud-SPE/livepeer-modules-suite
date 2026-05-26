# Gateways

**Status:** 🟠 Documented — full gateway in [livepeer-modules-transcode-gateway](../repos/livepeer-modules-transcode-gateway.md) (`4086880`); control-plane variant in [open-clearinghouse](../repos/livepeer-open-clearinghouse.md)
**Submodule:** `modules/livepeer-modules-transcode-gateway/`

## What this is

A **Gateway** is the demand-side entry point: it accepts work from applications or end
users, **discovers** and selects an orchestrator, attaches **payment**, opens the right
transport, and forwards traffic. It turns the Livepeer network into a usable application
surface.

## Two demand-side patterns

The suite currently has **two distinct, alternative front doors** (not layered):

1. **Full in-path gateway** — [transcode-gateway](../repos/livepeer-modules-transcode-gateway.md):
   one Go service that exposes a video API (VOD ABR + live RTMP→HLS), resolves routes via
   `service-registry-daemon`, mints `Livepeer-Payment` via `payment-daemon`, and forwards
   to the broker. The **gateway pays the network itself** (customers pay nothing in v1),
   owns the public RTMP endpoint, and ships a thin SaaS shell (waitlist + API keys, no
   billing) with three embedded UIs.
2. **Non-custodial control plane + SDK** — [open-clearinghouse](payment-clearinghouse.md):
   the clearinghouse handles auth/credit/discovery-proxy/mint and **hands off** a payment
   envelope; the customer's [SDK](sdks.md) is the data plane that talks to the broker
   directly.

A deployment chooses one. Both consume the same supply-side daemons.

## How a gateway works (transcode-gateway, grounded)

- **Surface:** `/api/v1/abr*` (VOD ABR ladder), `/api/v1/live*` (RTMP→HLS),
  `/api/v1/capabilities` (live registry catalog).
- **Wire translation:** request → `Livepeer-Capability` + interaction mode
  (`http-reqresp@v0` for VOD, `live-session-gateway-ingest@v0` for live).
- **Route selection:** `routeSelector` ranks `SelectMany` candidates by
  constraints/extras/price; `routeHealth` cools down failing routes (2 fails → 30s) and
  fails over. (This is [Discover](discover.md) on the gateway side.)
- **Payment:** per-request envelope for VOD; session-open + interim-debit for live.
- **Usage:** durable reservations (open → committed / refunded); MinIO/S3 for assets with
  per-session STS-scoped creds.

For live, the gateway is the one named in the "Option B" topology:
**transcode-gateway → broker → [live-runner](runners.md)**.

## Role in the suite

- **Consumes:** [Discover](discover.md), [Service Registry](service-registry.md),
  [Payment](payment.md).
- **Forwards to:** [Orchestrators](orchestrators.md) / [Pools](pools.md) brokers and
  [Runners](runners.md).
- **Consumed by:** end users / [Reference Apps](reference-apps.md), and (clearinghouse
  pattern) via [SDKs](sdks.md).

## Confirmed / open

- ✅ A full standalone gateway exists (video). It pays the network; no customer billing in v1.
- ✅ Live = `live-session-gateway-ingest@v0`; gateway owns RTMP `:1935` and relays frames.
- [ ] An **OpenAI/daydream gateway** sibling (transcode-gateway was ported from it) is a
  likely future repo.
- [ ] transcode-gateway is v1.3.0 / grade C, **tests grade F** — not yet run against a
  real broker.
- [ ] Capability-name mismatch with the runner manifests (TD-7).
