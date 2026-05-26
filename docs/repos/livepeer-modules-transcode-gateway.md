# livepeer-modules-transcode-gateway

**Submodule:** `modules/livepeer-modules-transcode-gateway/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-modules-transcode-gateway.git`
**Pinned revision:** `4086880` (documented 2026-05-26)
**Status:** 🟠 Onboarded — documented from code. Published as
`tztcloud/livepeer-video-gateway:v1.3.0`, but QUALITY_SCORE is all **grade C** and
**tests are grade F** (not yet exercised against a real broker).

The first concrete **[Gateway](../product-specs/gateways.md)** — the standalone demand-side
shell tracked as outstanding in earlier rounds. One Go service that exposes a **video API**
(VOD ABR + live RTMP→HLS), serves three product UIs, handles access control, and **forwards
work to the Livepeer network** rather than processing media locally.

## How it relates to the clearinghouse — two distinct demand-side surfaces

This gateway is **independent of [livepeer-open-clearinghouse](livepeer-open-clearinghouse.md)**
(no reference to it anywhere in the code). The suite now has two different demand-side
patterns, both talking to the same supply-side daemons:

| | open-clearinghouse | transcode-gateway (this repo) |
| --- | --- | --- |
| Custody | **Non-custodial** — customers hold a wei credit balance | Gateway holds the keystore; **gateway pays the network**, customers pay nothing in v1 |
| Data path | **Handoff** — SDK talks to the broker directly | Gateway is **in-path** (relays RTMP; returns descriptors for VOD) |
| Billing | Credit ledger, Stripe top-ups | **None** in v1 (waitlist + API keys only) |
| Focus | Generic / OpenAI-shaped | Video (ABR + live) |

So they are **alternative front doors**, not layered. A given deployment chooses one.

## Six layers (from DESIGN.md)

1. **Transcode surface** — `/api/v1/abr*` (VOD), `/api/v1/live*` (RTMP→HLS), `/api/v1/capabilities`.
2. **Wire translation** — request → `Livepeer-Capability` + interaction mode
   (`http-reqresp@v0` for VOD, `live-session-gateway-ingest@v0` for live).
3. **Route selection** — `service-registry-daemon` `SelectMany` gives candidates;
   `routeSelector` ranks by constraints/extras/price; `routeHealth` applies per-candidate
   failure cooldowns (2 failures → 30s) and fails over.
4. **Payment** — `payment-daemon` `CreatePayment` mints `Livepeer-Payment` envelopes; the
   gateway pays on behalf of each request (per-request for VOD; session-open + interim
   debit for live).
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
- **Catalog:** `GET /api/v1/capabilities` — reflects the on-chain registry (cached, 60s
  refresh), not a hardcoded list.
- **Public/portal/admin:** waitlist signup + email verify; portal login/account/keys/
  usage/history; admin waitlist approval, users, usage, live-streams, ABR jobs, registry
  health. Plus `/health`, `/metrics`, `/openapi.json`, `/docs`.

## Live path (the "Option B" gateway role)

This gateway is the **gateway** named in the
[transcode-runners](livepeer-modules-transcode-runners.md) live "Option B" topology:

> **transcode-gateway** → broker / payment → **live-runner**

It owns the public RTMP endpoint (`:1935`), validates the stream key, **relays FLV frames
verbatim** to the broker's `private_ingest_url`, mints a session-open payment (TopUp
allowed), and tracks the three IDs `gateway_session_id` / `broker_session_id` /
`runner_session_id`. The live-runner writes HLS to gateway-owned MinIO via per-session STS
credentials.

## Web apps

Three zero-build Lit SPAs embedded into the Go binary via `//go:embed`: `web/site`
(marketing + waitlist), `web/portal` (user dashboard + playground), `web/admin` (operator
console). In production it's one Go process on one port serving `/`, `/portal/`, `/admin/`.

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Gateways](../product-specs/gateways.md) | The whole service — a full, in-path video gateway |
| [Discover](../product-specs/discover.md) | `routeSelector` over `service-registry-daemon` `SelectMany` + `/api/v1/capabilities` catalog |
| [Payment](../product-specs/payment.md) | `payment-daemon` `CreatePayment`; gateway-funded, per-request + live session |
| [SDKs](../product-specs/sdks.md) | OpenAPI 3.1 (`/openapi.json`) + embedded portal/admin UIs (no separate client SDK here) |

## Notes & open items

- **Intended as a reference example (TD-8):** this gateway currently embeds direct
  `service-registry-daemon` + `payment-daemon` usage and an operator wallet. The plan is to
  migrate it onto the [clearinghouse](livepeer-open-clearinghouse.md) + [SDKs](../product-specs/sdks.md)
  and drop that direct daemon/wallet code, so it demonstrates SDK-based building without
  on-chain/wallet complexity. It then becomes a [Reference App](../product-specs/reference-apps.md).
- **Capability-name mismatch (TD-7):** this gateway queries `video:transcode.abr` /
  `video:transcode.live`, while the [transcode-runners](livepeer-modules-transcode-runners.md)
  offering manifests declare `video-transcode-abr` and
  `livepeer:transcode/live-rtmp-hls-abr`. The broker host-config / registry is the bridge;
  the exact end-to-end mapping still needs confirming.
- **Ported from an "openai gateway":** the code references a sibling OpenAI/daydream
  gateway it was ported from — a likely future repo to onboard.
- **Maturity:** v1.3.0 image exists but all domains grade C and **tests are F**; not yet
  run against a real broker. No customer billing, no playback proxy, poll-only (no
  SSE/webhooks to clients) in v1.

## Source pointers

- [`README.md`](../../modules/livepeer-modules-transcode-gateway/README.md) ·
  [`DESIGN.md`](../../modules/livepeer-modules-transcode-gateway/DESIGN.md) ·
  [`ARCHITECTURE.md`](../../modules/livepeer-modules-transcode-gateway/ARCHITECTURE.md)
- [`docs/product-specs/transcode-surface.md`](../../modules/livepeer-modules-transcode-gateway/docs/product-specs/transcode-surface.md) ·
  [`docs/design-docs/live-stream-pipeline.md`](../../modules/livepeer-modules-transcode-gateway/docs/design-docs/live-stream-pipeline.md)
