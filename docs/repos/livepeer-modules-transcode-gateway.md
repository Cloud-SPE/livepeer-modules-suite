# livepeer-modules-transcode-gateway

**Submodule:** `modules/livepeer-modules-transcode-gateway/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-modules-transcode-gateway.git`
**Pinned revision:** `f970fab` (10 commits past tag `v1.3.1`; product/image version bumped to `v1.4.1`; documented 2026-06-15)
**Status:** 🟠 Onboarded — documented from code. Latest architecture/design/deployment
docs move routing and payment to LOC ([livepeer-open-clearinghouse](livepeer-open-clearinghouse.md)):
the gateway no longer needs local payer/resolver daemon sidecars or an operator keystore.

The first concrete **[Gateway](../product-specs/gateways.md)** — the standalone demand-side
shell tracked as outstanding in earlier rounds. One Go service that exposes a **video API**
(VOD ABR + live RTMP→HLS), serves three product UIs, handles access control, and **forwards
work to the Livepeer network** rather than processing media locally.

## How it relates to the clearinghouse

This gateway now consumes [livepeer-open-clearinghouse](livepeer-open-clearinghouse.md)
(LOC) over HTTPS with an operator-issued `pymth_` API key. LOC selects routes, mints
payment envelopes, and owns the credit ledger; the gateway remains the in-path video app
surface and RTMP relay.

| | LOC handoff SDK path | transcode-gateway (this repo) |
| --- | --- | --- |
| Custody/account | Customer app holds its own LOC credit/API key | Gateway operator holds a LOC API key and funded LOC credit balance |
| Data path | **Handoff** — SDK talks to the broker directly | Gateway is **in-path** (relays RTMP; returns descriptors for VOD) |
| Billing | LOC customer credit ledger | **No customer billing** in v1; operator pays via LOC |
| Focus | Generic app integration | Video (ABR + live) |

So they are related but still different application shapes: LOC is the shared
route/payment control plane; this gateway is a video product surface built on it.

## Six layers (from DESIGN.md)

1. **Transcode surface** — `/api/v1/abr*` (VOD), `/api/v1/live*` (RTMP→HLS), `/api/v1/capabilities`.
2. **Wire translation** — request → `Livepeer-Capability` + interaction mode
   (`http-reqresp@v0` for VOD, `live-session-gateway-ingest@v0` for live).
3. **Route selection + payment** — `gateway/internal/proxy/loc/` calls LOC. `POST /v1/jobs`
   or `POST /v1/sessions` selects a broker and returns a `Livepeer-Payment` envelope in one
   response.
4. **Settlement** — VOD settles actual units back to LOC; live closes/refills LOC sessions
   with duration/runway estimates. A settle janitor re-drives pending settlements.
5. **SaaS shell** — Postgres waitlist → email-verify → admin-approve → API-key issuance;
   portal cookie sessions; `ADMIN_TOKEN` bootstraps admin.
6. **Usage + asset tracking** — per-request **usage reservations** (open → committed /
   refunded); MinIO (S3) for VOD ingest and live HLS output (per-session STS-scoped creds).

## API surface

- **VOD ABR:** `POST /api/v1/abr/upload-url` (MinIO presign) → `POST /api/v1/abr` (submit;
  returns job + `master_playlist_url` + renditions) → `GET /api/v1/abr/:id` (poll) →
  `DELETE /api/v1/abr/objects`. Capability `video:transcode.abr`, offering `default`.
  ABR ladder only (no single-rendition VOD).
- **Live:** `POST /api/v1/live` (allocate → returns `rtmp://gateway:1935/live/<stream_key>`
  + HLS playback URL) → `GET /api/v1/live/:id` (poll) → `DELETE /api/v1/live/:id` (close).
  Capability `video:transcode.live`, offering `gateway-ingest`.
- **Catalog:** `GET /api/v1/capabilities` — reflects the LOC catalog snapshot (cached,
  60s refresh), not a hardcoded list.
- **Public/portal/admin:** waitlist signup + email verify; portal login/account/keys/
  usage/history; admin waitlist approval, users, usage, live-streams, ABR jobs, registry
  health. Plus `/health`, `/metrics`, `/openapi.json`, `/docs`.

## Live path (the "Option B" gateway role)

This gateway is the **gateway** named in the
[transcode-runners](livepeer-modules-transcode-runners.md) live "Option B" topology:

> **transcode-gateway** → broker / payment → **live-runner**

It owns the public RTMP endpoint (`:1935`), validates the stream key, **relays FLV frames
verbatim** to the broker's `private_ingest_url`, opens/refills/closes a LOC session, and
tracks gateway/broker/runner/LOC session identifiers. The live-runner writes HLS to
gateway-owned MinIO via per-session STS credentials.

## Web apps

Three zero-build Lit SPAs embedded into the Go binary via `//go:embed`: `web/site`
(marketing + waitlist), `web/portal` (user dashboard + playground), `web/admin` (operator
console). In production it's one Go process on one port serving `/`, `/portal/`, `/admin/`.

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Gateways](../product-specs/gateways.md) | The whole service — a full, in-path video gateway |
| [Discover](../product-specs/discover.md) | LOC job/session open + `/api/v1/capabilities` LOC catalog snapshot |
| [Payment](../product-specs/payment.md) | LOC-minted `Livepeer-Payment`; gateway-funded via operator LOC credit |
| [SDKs](../product-specs/sdks.md) | OpenAPI 3.1 (`/openapi.json`) + embedded portal/admin UIs (no separate client SDK here) |

## Notes & open items

- **Reference-app posture:** this gateway now demonstrates video product code on top of LOC
  without local daemon sidecars or wallet management. It still pays from an operator LOC
  account and stays in the data path, so it is not identical to the customer SDK handoff
  path.
- **README drift:** the pinned submodule's `README.md` still contains daemon-era wording,
  but `DESIGN.md`, `ARCHITECTURE.md`, `DEPLOYMENT.md`, and the code under
  `gateway/internal/proxy/loc/` describe the current LOC path.
- **Ported from an "openai gateway":** the code references a sibling OpenAI/daydream
  gateway it was ported from — a likely future repo to onboard.
- **Maturity:** v1.4.1 with LOC integration. No customer billing, no playback proxy,
  poll-only (no SSE/webhooks to clients) in v1.

## Source pointers

- [`README.md`](../../modules/livepeer-modules-transcode-gateway/README.md) ·
  [`DESIGN.md`](../../modules/livepeer-modules-transcode-gateway/DESIGN.md) ·
  [`ARCHITECTURE.md`](../../modules/livepeer-modules-transcode-gateway/ARCHITECTURE.md)
- [`docs/product-specs/transcode-surface.md`](../../modules/livepeer-modules-transcode-gateway/docs/product-specs/transcode-surface.md) ·
  [`docs/design-docs/live-stream-pipeline.md`](../../modules/livepeer-modules-transcode-gateway/docs/design-docs/live-stream-pipeline.md)
