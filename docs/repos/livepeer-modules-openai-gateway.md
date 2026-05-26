# livepeer-modules-openai-gateway

**Submodule:** `modules/livepeer-modules-openai-gateway/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-modules-openai-gateway.git`
**Pinned revision:** `5afaf96` (2 commits past tag `v1.3.1`; documented 2026-05-26)
**Status:** 🟠 Onboarded — documented from code. All domains grade C ("not yet exercised
against a real broker"), but better-tested than the video gateway (45 unit tests + a
`make smoke` e2e flow; SaaS shell verified against Postgres).

The **OpenAI-compatible AI [Gateway](../product-specs/gateways.md)** — the demand-side
front door for AI workloads, and the AI counterpart to the video
[transcode-gateway](livepeer-modules-transcode-gateway.md). The feature promise is
**"change `base_url`, keep your OpenAI client code."** A single TypeScript **Fastify**
service serves the `/v1/*` inference API plus the same SaaS shell (waitlist/portal/admin).

It is the natural front door for the
[openai-runners](livepeer-modules-openai-runners.md) backends.

## Same pattern as transcode-gateway

This repo is the AI twin of the video gateway — same six-layer shape, same
daemons-direct integration, same operator-funded model, same SaaS shell — differing in
language (TS/Fastify vs Go) and surface (OpenAI inference vs video):

| Aspect | This (openai-gateway) | transcode-gateway |
| --- | --- | --- |
| Surface | `/v1/chat`, `/embeddings`, `/images`, `/audio/*`, `/rerank`, `/models` | `/api/v1/abr*`, `/api/v1/live*` |
| Language | TypeScript / Fastify | Go |
| Modes | `http-reqresp@v0` / `http-stream@v0` / `http-multipart@v0` | `http-reqresp@v0` / `live-session-gateway-ingest@v0` |
| Funding | Operator pays; no customer billing in v1 | Same |
| Data path | In-path proxy of inference requests | In-path (RTMP relay + descriptors) |

Both are **distinct from the non-custodial [clearinghouse](livepeer-open-clearinghouse.md)**
(which uses handoff mode + customer credit). The suite now has **three demand-side front
doors**; a deployment picks one.

## Six layers (from DESIGN.md)

1. **OpenAI surface** — `/v1/chat/completions` (streaming), `/v1/embeddings`,
   `/v1/images/generations`, `/v1/audio/speech`, `/v1/audio/transcriptions`, `/v1/rerank`,
   `/v1/models`. (`/v1/realtime` is v2.)
2. **Wire translation** — OpenAI request → `Livepeer-Capability` + mode
   (`http-reqresp` / `http-stream` / `http-multipart`); in `gateway/src/proxy/livepeer/`.
3. **Route selection** — `service-registry-daemon` `SelectMany`; `routeSelector` ranks by
   constraints/extras/price; `routeHealth` cooldowns + failover.
4. **Payment** — `payment-daemon` mints `Livepeer-Payment` per request; gateway pays the
   network (customers pay nothing in v1).
5. **SaaS shell** — Postgres waitlist → email-verify → admin-approve → API-key; portal
   cookie sessions; `ADMIN_TOKEN` bootstrap.
6. **Usage tracking** — per-request reservations (open → committed / refunded) with
   route-aware settlement metadata (for visibility + future billing).

`/v1/models` reflects the on-chain registry live (dynamic discovery), not a hardcoded
catalog. Three zero-build Lit SPAs (`web/site`, `web/portal`, `web/admin`).

## Capability names (TD-7)

This gateway queries **colon-form** capability ids: `openai:chat-completions`,
`openai:embeddings`, `openai:images-generations`, `openai:audio-speech`,
`openai:audio-transcriptions` (+ `openai:realtime` reserved for v2), and `rerank`. The
[openai-runners](livepeer-modules-openai-runners.md) declare **hyphen-form** canonical
names (`openai-chat-completions`, …). Same colon-vs-hyphen gap as the video side — the
broker host-config / registry is the bridge; the end-to-end mapping still needs confirming.

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Gateways](../product-specs/gateways.md) | The whole service — full in-path OpenAI-compatible gateway |
| [SDKs](../product-specs/sdks.md) | OpenAI wire compatibility ("change `base_url`") + OpenAPI for the SaaS routes |
| [Discover](../product-specs/discover.md) | `routeSelector` over `service-registry-daemon` + dynamic `/v1/models` |
| [Payment](../product-specs/payment.md) | `payment-daemon` `CreatePayment`; gateway-funded, per request |

## Notes & open items

- **Intended as a reference example (TD-8):** like the video gateway, this currently embeds
  direct `service-registry-daemon` + `payment-daemon` usage and an operator wallet. The plan
  is to migrate it onto the [clearinghouse](livepeer-open-clearinghouse.md) +
  [SDKs](../product-specs/sdks.md) and remove that direct daemon/wallet code, demonstrating
  SDK-based building without on-chain/wallet complexity — folding it into the
  [Reference Apps](../product-specs/reference-apps.md).
- **Serves the AI runners:** its `/v1/*` capabilities line up with the
  [openai-runners](livepeer-modules-openai-runners.md) (chat, embeddings, images, audio,
  rerank).
- **Maturity:** v1.3.1+; grade C across the board but with real unit tests + smoke flow;
  still unverified against a live broker. No billing, no `/v1/realtime` in v1.
- **daydream gateway** (and vtuber runners, reference apps) remain separate, not-yet-
  onboarded repos.

## Source pointers

- [`README.md`](../../modules/livepeer-modules-openai-gateway/README.md) ·
  [`DESIGN.md`](../../modules/livepeer-modules-openai-gateway/DESIGN.md) ·
  [`ARCHITECTURE.md`](../../modules/livepeer-modules-openai-gateway/ARCHITECTURE.md)
- [`docs/product-specs/openai-surface.md`](../../modules/livepeer-modules-openai-gateway/docs/product-specs/openai-surface.md)
