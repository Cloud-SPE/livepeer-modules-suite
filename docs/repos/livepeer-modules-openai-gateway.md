# livepeer-modules-openai-gateway

**Submodule:** `modules/livepeer-modules-openai-gateway/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-modules-openai-gateway.git`
**Pinned revision:** `819e059` (8 commits past tag `v1.3.1`; product version bumped to `v1.4.1`; documented 2026-06-15)
**Status:** 🟠 Onboarded — documented from code. The latest pin integrates LOC
([livepeer-open-clearinghouse](livepeer-open-clearinghouse.md)) for route selection,
payment minting, and settlement; no local daemon sidecars or chain keys live in the
gateway process.

The **OpenAI-compatible AI [Gateway](../product-specs/gateways.md)** — the demand-side
front door for AI workloads, and the AI counterpart to the video
[transcode-gateway](livepeer-modules-transcode-gateway.md). The feature promise is
**"change `base_url`, keep your OpenAI client code."** A single TypeScript **Fastify**
service serves the `/v1/*` inference API plus the same SaaS shell (waitlist/portal/admin).

It is the natural front door for the
[openai-runners](livepeer-modules-openai-runners.md) backends.

## Same pattern as transcode-gateway

This repo is the AI twin of the video gateway — same six-layer shape, same LOC-mediated
integration, same operator-funded model, same SaaS shell — differing in language
(TS/Fastify vs Go) and surface (OpenAI inference vs video):

| Aspect | This (openai-gateway) | transcode-gateway |
| --- | --- | --- |
| Surface | `/v1/chat`, `/embeddings`, `/images`, `/audio/*`, `/rerank`, `/models` | `/api/v1/abr*`, `/api/v1/live*` |
| Language | TypeScript / Fastify | Go |
| Modes | `http-reqresp@v0` / `http-stream@v0` / `http-multipart@v0` | `http-reqresp@v0` / `live-session-gateway-ingest@v0` |
| Funding | Operator pays through LOC credit; no customer billing in v1 | Same |
| Data path | In-path proxy of inference requests | In-path (RTMP relay + descriptors) |

Both now consume the [clearinghouse](livepeer-open-clearinghouse.md) over HTTPS with a
gateway-owned `pymth_` API key. They remain distinct from the customer's handoff-mode SDK
path: these gateways stay in the data path and pay from the operator's LOC credit balance,
while the clearinghouse SDK path hands the broker call to the customer app.

## Six layers (from DESIGN.md)

1. **OpenAI surface** — `/v1/chat/completions` (streaming), `/v1/embeddings`,
   `/v1/images/generations`, `/v1/audio/speech`, `/v1/audio/transcriptions`, `/v1/rerank`,
   `/v1/models`. (`/v1/realtime` is v2.)
2. **Wire translation** — OpenAI request → `Livepeer-Capability` + mode
   (`http-reqresp` / `http-stream` / `http-multipart`); in `gateway/src/proxy/`.
3. **Route selection + payment** — `gateway/src/loc/` opens a LOC job; LOC selects one
   broker route and returns the `Livepeer-Payment` envelope in the same response.
4. **Settlement** — a durable background settler reports actual units/outcome back to LOC
   (`LOC_SETTLE_INTERVAL_MS`, `LOC_SETTLE_MAX_ATTEMPTS`); LOC refunds unused estimate.
5. **SaaS shell** — Postgres waitlist → email-verify → admin-approve → API-key; portal
   cookie sessions; `ADMIN_TOKEN` bootstrap.
6. **Usage tracking** — per-request reservations (open → committed / refunded) with
   route-aware settlement metadata (for visibility + future billing).

`/v1/models` reflects a LOC-backed capability/offerings cache, not a hardcoded catalog.
Three zero-build Lit SPAs (`web/site`, `web/portal`, `web/admin`).

## Capability names (TD-7)

The model id is the LOC offering id. The gateway no longer maps user-facing model names to
locally hardcoded model metadata; `/v1/models` is built from LOC's `GET /v1/capabilities`
catalog plus operator overrides.

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Gateways](../product-specs/gateways.md) | The whole service — full in-path OpenAI-compatible gateway |
| [SDKs](../product-specs/sdks.md) | OpenAI wire compatibility ("change `base_url`") + OpenAPI for the SaaS routes |
| [Discover](../product-specs/discover.md) | LOC job open + LOC-backed catalog; no local route selector |
| [Payment](../product-specs/payment.md) | LOC-minted `Livepeer-Payment`; gateway-funded via operator LOC credit |

## Notes & open items

- **Reference-app posture:** this gateway now demonstrates building a product surface on
  LOC without local chain keys or daemon sidecars. It still stays in-path and pays from an
  operator LOC account, so it is not the same as the customer SDK handoff flow.
- **Serves the AI runners:** its `/v1/*` capabilities line up with the
  [openai-runners](livepeer-modules-openai-runners.md) (chat, embeddings, images, audio,
  rerank).
- **Maturity:** v1.4.1 with LOC integration and `make loc-smoke`; no customer billing,
  no `/v1/realtime` in v1.
- **daydream gateway** (and vtuber runners, reference apps) remain separate, not-yet-
  onboarded repos.

## Source pointers

- [`README.md`](../../modules/livepeer-modules-openai-gateway/README.md) ·
  [`DESIGN.md`](../../modules/livepeer-modules-openai-gateway/DESIGN.md) ·
  [`ARCHITECTURE.md`](../../modules/livepeer-modules-openai-gateway/ARCHITECTURE.md)
- [`docs/product-specs/openai-surface.md`](../../modules/livepeer-modules-openai-gateway/docs/product-specs/openai-surface.md)
