# livepeer-open-clearinghouse

**Submodule:** `modules/livepeer-open-clearinghouse/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-open-clearinghouse.git`
**Pinned revision:** `a529592` (documented 2026-05-26)
**Status:** 🟠 Onboarded — documented from code. Repo's own status: **pre-alpha**, but the
headline jobs/sessions path is implemented and end-to-end tested.

The **demand-side control plane**: a **non-custodial-by-design payment clearinghouse**
for Livepeer app developers. It authenticates developers, holds their wei-denominated
credit balance, and mints signed Livepeer payment tickets **on their behalf** — so a
customer integrates one HTTP API and never manages a wallet or signing key.

This is the suite's [Payment Clearinghouse](../product-specs/payment-clearinghouse.md),
and it **consumes the first repo**: its Docker Compose runs `payment-daemon` and
`service-registry-daemon` images from
[livepeer-network-modules](livepeer-network-modules.md). It is the first concrete
cross-repo dependency in the suite.

## Stack

Single Python 3.13 / FastAPI service. SQLAlchemy 2.0 async on Postgres 16, Alembic
migrations, in-process APScheduler for background jobs, `uv` for packaging. Two zero-build
Lit SPAs (`web/portal`, `web/admin`) served as static files. structlog JSON + Prometheus
`/metrics`. Runs as **one container** (`livepeer-open-clearinghouse-gateway`) — not
horizontally scalable in MVP (single pooled wallet + in-process scheduler).

## Handoff mode — the key idea

The clearinghouse is **control plane only; it is not in the data plane.** The customer's
SDK talks to the orchestrator broker directly. This is "handoff mode" (replaced the older
direct mint path):

1. SDK calls the clearinghouse `POST /v1/jobs` or `POST /v1/sessions` with
   `{capability, offering, estimated_units, max_total_units}`.
2. Clearinghouse `registry.Select(...)` → route + price; checks balance ≥
   `max_total_units × price` (fail closed → HTTP 402).
3. Clearinghouse `payment_daemon.CreatePayment(...)` → signed envelope; **encumbers the
   expected value from the user's credit balance at issuance** (not on-chain redemption).
4. Returns `{payment_envelope, broker_url, work_id, settle_endpoint}`.
5. **SDK → broker directly**, sending the envelope as `Livepeer-Payment`; reads
   `Livepeer-Work-Units` from the response.
6. SDK calls the settle endpoint with `actual_units`; clearinghouse refunds the unused
   encumbrance.

Blast radius is decoupled: if the clearinghouse is down, in-flight work continues; only
new mints pause. A background **reconciliation janitor** cross-checks SDK self-reports
against the daemon's authoritative `GetSessionDebits`.

- **Jobs** — one-shot (`http-reqresp@v0`, `http-stream@v0`, `http-multipart@v0`): mint →
  call broker → settle once.
- **Sessions** — long-lived, refillable (`ws-realtime@v0`,
  `session-control-plus-media@v0`, `live-session-remote-runner@v0`,
  `live-session-gateway-ingest@v0`): open → `refill` on `Livepeer-Balance-Low` → close.

See [`docs/HANDOFF_MODE.md`](../../modules/livepeer-open-clearinghouse/docs/HANDOFF_MODE.md).

## Domains (layered: `types → config → repo → service → runtime → ui`)

| Domain | Owns / does |
| --- | --- |
| `accounts` | Signup/login, email verification, password reset, web sessions, Google/GitHub OAuth, operator approval |
| `api_keys` | Per-user API keys — shown once, `sha256(pepper‖key)` at rest, display prefix |
| `billing` | Wei credit balance + append-only `credit_ledger`, top-ups, spend-window caps, auto-replenish |
| `discovery` | Auth-aware proxy to `service-registry-daemon` (`Resolve`/`Select`/`SelectMany`) with TTL cache; no tables |
| `jobs` | One-shot handoff-mode jobs: mint + settle (encumber worst-case, refund delta) |
| `sessions` | Long-lived payment sessions: open/refill/close, state machine `open → draining → closed` |
| `payments` | Read-only history; deposit-snapshot poller of the pooled wallet |
| `usage` | Idempotent usage reconciliation on `(api_key_id, request_id)` |
| `telemetry` | SDK event ingest, customer query/export, GeoIP enrichment, rate limiting, retention |
| `notifications` | Resend email events, webhook config/test (HMAC), notification preferences |
| `admin` | Operator console: approve users, billing config, audit log, deposit snapshots, SDK approval registry + signed SDK manifest |

Layering is enforced mechanically by `scripts/check_layering.py`; cross-cutting concerns
live in `providers/` (db, auth, email, oauth, `payment_daemon`, `registry_daemon`, clock,
telemetry).

## Integration with livepeer-network-modules

Unix-socket gRPC over a shared Docker volume (`livepeer-run`, matching uid/gid `65532`,
no token auth — filesystem-mediated trust):

- **payment-daemon (sender):** `CreatePayment`, `GetSessionDebits` (authoritative billed
  amount), `GetDepositInfo` (pooled-wallet deposit/reserve, polled every 5 min).
- **service-registry-daemon (resolver):** `Select` / `SelectMany` / `Resolve`. The
  `SelectedRoute.extra` carries the `interaction_mode`.

**Key custody:** the **pooled wallet** (V3 keystore) is held only by `payment-daemon`;
the clearinghouse never sees keystore material. Users hold no on-chain key — only a wei
credit balance in Postgres. **Non-custodial at the user boundary, custodial at the
network boundary** (operator owns the signing wallet). See
[`docs/SECURITY.md`](../../modules/livepeer-open-clearinghouse/docs/SECURITY.md).

## SDK surface

- **Reference SDKs / examples** in [`examples/`](../../modules/livepeer-open-clearinghouse/examples/):
  `python`, `typescript`, `go`, `rust`, plus `openapi.json`.
- **SDK approval registry** (admin domain, `sdk_approval` keyed on
  `(lang, version, git_sha7)`) and a **public signed SDK manifest** (`GET /v1/sdk/manifest`,
  EdDSA-signed) that SDKs check at startup (advisory in v1).
- SDKs identify themselves via the `Livepeer-Open-Clearinghouse-SDK: <lang>/<semver>/<git_sha7>`
  header and may ingest telemetry to `POST /v1/telemetry`.
- **Conformance harness** in [`conformance/`](../../modules/livepeer-open-clearinghouse/conformance/)
  (mock broker + mock clearinghouse + scenario runners).

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Payment Clearinghouse](../product-specs/payment-clearinghouse.md) | The whole service: credit ledger, mint-on-behalf, jobs/sessions settlement |
| [Payment](../product-specs/payment.md) | `jobs`/`sessions` issuance via `payment-daemon.CreatePayment`; EV-at-issuance charging |
| [SDKs](../product-specs/sdks.md) | `examples/` reference SDKs, SDK approval registry + signed manifest, telemetry ingest |
| [Discover](../product-specs/discover.md) | `discovery` domain — auth-aware proxy over `service-registry-daemon` |
| [Gateways](../product-specs/gateways.md) | Control-plane half of the gateway role (auth + credit + mint); data plane is the SDK ↔ broker |

## Source pointers

- Map: [`AGENTS.md`](../../modules/livepeer-open-clearinghouse/AGENTS.md) · Architecture:
  [`ARCHITECTURE.md`](../../modules/livepeer-open-clearinghouse/ARCHITECTURE.md)
- Handoff model: [`docs/HANDOFF_MODE.md`](../../modules/livepeer-open-clearinghouse/docs/HANDOFF_MODE.md)
- Product/scope: [`docs/PRODUCT_SENSE.md`](../../modules/livepeer-open-clearinghouse/docs/PRODUCT_SENSE.md) ·
  Security: [`docs/SECURITY.md`](../../modules/livepeer-open-clearinghouse/docs/SECURITY.md) ·
  Maturity: [`docs/QUALITY_SCORE.md`](../../modules/livepeer-open-clearinghouse/docs/QUALITY_SCORE.md)
