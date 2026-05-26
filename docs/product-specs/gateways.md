# Gateways

**Status:** 🟠 Documented — full gateways for AI ([openai-gateway](../repos/livepeer-modules-openai-gateway.md)) and video ([transcode-gateway](../repos/livepeer-modules-transcode-gateway.md)); control-plane variant in [open-clearinghouse](../repos/livepeer-open-clearinghouse.md)
**Submodule(s):** `modules/livepeer-modules-openai-gateway/`, `modules/livepeer-modules-transcode-gateway/`

## What this is

A **Gateway** is the demand-side entry point: it accepts work from applications or end
users, **discovers** and selects an orchestrator, attaches **payment**, opens the right
transport, and forwards traffic. It turns the Livepeer network into a usable application
surface.

## Two demand-side patterns

The suite currently has **three distinct, alternative front doors** (not layered):

1. **Full in-path AI gateway** — [openai-gateway](../repos/livepeer-modules-openai-gateway.md):
   a TS/Fastify service exposing an OpenAI-compatible `/v1/*` API ("change `base_url`,
   keep your OpenAI client"). Fronts the [openai-runners](runners.md).
2. **Full in-path video gateway** — [transcode-gateway](../repos/livepeer-modules-transcode-gateway.md):
   a Go service exposing VOD ABR + live RTMP→HLS. Owns the public RTMP endpoint. Fronts
   the video [runners](runners.md).
3. **Non-custodial control plane + SDK** — [open-clearinghouse](payment-clearinghouse.md):
   handles auth/credit/discovery-proxy/mint and **hands off** a payment envelope; the
   customer's [SDK](sdks.md) is the data plane that talks to the broker directly.

(1) and (2) share one pattern: resolve via `service-registry-daemon`, mint
`Livepeer-Payment` via `payment-daemon`, forward to the broker, and **pay the network
themselves** (customers pay nothing in v1; thin SaaS shell, no billing). (3) differs:
customers hold credit and the SDK is the data plane. A deployment chooses one; all consume
the same supply-side daemons.

> **Direction (planned — TD-8).** The two full gateways are intended to be **reference
> examples** of building on Livepeer, *not* standalone production stacks. The plan is to
> migrate them to consume the [clearinghouse](payment-clearinghouse.md) + [SDKs](sdks.md)
> and **remove the direct `service-registry-daemon`/`payment-daemon` and operator-wallet
> code** — so they demonstrate how simple it is to build on the network via SDKs without
> handling wallet or on-chain protocol complexity. After that migration they fold into
> [Reference Apps](reference-apps.md), and pattern (3) becomes the recommended path with
> (1)/(2) as its showcases.

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

- ✅ Two full standalone gateways exist — **AI** (openai-gateway) and **video**
  (transcode-gateway). Both pay the network; no customer billing in v1.
- ✅ Live = `live-session-gateway-ingest@v0`; the video gateway owns RTMP `:1935` and
  relays frames.
- [ ] A **daydream gateway** sibling remains a likely future repo.
- [ ] Both gateways are grade C, not yet run against a real broker (openai-gateway has 45
  tests + a smoke flow; transcode-gateway tests grade F).
- [ ] Capability-name mismatch with the runner manifests (TD-7), on both AI and video.
