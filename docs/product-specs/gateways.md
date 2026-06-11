# Gateways

**Status:** 🟠 Documented — full in-path gateways for AI ([openai-gateway](../repos/livepeer-modules-openai-gateway.md)) and video ([transcode-gateway](../repos/livepeer-modules-transcode-gateway.md)), both now LOC-mediated; handoff control-plane path in [open-clearinghouse](../repos/livepeer-open-clearinghouse.md)
**Submodule(s):** `modules/livepeer-modules-openai-gateway/`, `modules/livepeer-modules-transcode-gateway/`

## What this is

A **Gateway** is the demand-side entry point: it accepts work from applications or end
users, **discovers** and selects an orchestrator, attaches **payment**, opens the right
transport, and forwards traffic. It turns the Livepeer network into a usable application
surface.

## Demand-side shapes

The suite currently has **three demand-side shapes**:

1. **Full in-path AI gateway** — [openai-gateway](../repos/livepeer-modules-openai-gateway.md):
   a TS/Fastify service exposing an OpenAI-compatible `/v1/*` API ("change `base_url`,
   keep your OpenAI client"). Fronts the [openai-runners](runners.md).
2. **Full in-path video gateway** — [transcode-gateway](../repos/livepeer-modules-transcode-gateway.md):
   a Go service exposing VOD ABR + live RTMP→HLS. Owns the public RTMP endpoint. Fronts
   the video [runners](runners.md).
3. **Non-custodial control plane + SDK** — [open-clearinghouse](payment-clearinghouse.md):
   handles auth/credit/discovery-proxy/mint and **hands off** a payment envelope; the
   customer's [SDK](sdks.md) is the data plane that talks to the broker directly.

(1) and (2) now share one LOC-mediated pattern: open a LOC job/session, receive the
selected broker URL plus `Livepeer-Payment`, forward to the broker, then settle usage back
to LOC. They remain **operator-funded in-path apps** (customers pay nothing in v1; thin
SaaS shell, no customer billing). (3) differs: the customer owns the LOC credit/API key and
the SDK is the data plane.

This means the gateway repos now read as **reference app candidates**: product surfaces
built on LOC without local chain keys or local payer/resolver daemons.

## How a gateway works (transcode-gateway, grounded)

- **Surface:** `/api/v1/abr*` (VOD ABR ladder), `/api/v1/live*` (RTMP→HLS),
  `/api/v1/capabilities` (LOC-backed catalog).
- **Wire translation:** request → `Livepeer-Capability` + interaction mode
  (`http-reqresp@v0` for VOD, `live-session-gateway-ingest@v0` for live).
- **Route selection + payment:** LOC selects the broker and mints the `Livepeer-Payment`
  envelope when the gateway opens a job/session.
- **Settlement:** VOD settles actual units; live refills/closes LOC sessions using duration
  estimates, with a settle janitor for retries.
- **Usage:** durable reservations (open → committed / refunded / pending-settle);
  MinIO/S3 for assets with per-session STS-scoped creds.

For live, the gateway is the one named in the "Option B" topology:
**transcode-gateway → broker → [live-runner](runners.md)**.

## Role in the suite

- **Consumes:** LOC ([Payment Clearinghouse](payment-clearinghouse.md)), which in turn
  consumes [Discover](discover.md), [Service Registry](service-registry.md), and
  [Payment](payment.md).
- **Forwards to:** [Orchestrators](orchestrators.md) / [Pools](pools.md) brokers and
  [Runners](runners.md).
- **Consumed by:** end users / [Reference Apps](reference-apps.md), and (clearinghouse
  pattern) via [SDKs](sdks.md).

## Confirmed / open

- ✅ Two full in-path gateways exist — **AI** (openai-gateway) and **video**
  (transcode-gateway). Both now use LOC for route/payment and pay through operator LOC
  credit; no customer billing in v1.
- ✅ Live = `live-session-gateway-ingest@v0`; the video gateway owns RTMP `:1935` and
  relays frames.
- [ ] A **daydream gateway** sibling remains a likely future repo.
- [ ] The pinned transcode-gateway `README.md` still has daemon-era wording; its
  `DESIGN.md`/`ARCHITECTURE.md` and code show the LOC path.
