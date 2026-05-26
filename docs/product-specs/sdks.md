# SDKs

**Status:** 🟠 Documented via [livepeer-open-clearinghouse](../repos/livepeer-open-clearinghouse.md) (`a529592`) + customer-portal in [livepeer-network-modules](../repos/livepeer-network-modules.md)
**Submodule(s):** `modules/livepeer-open-clearinghouse/`, `modules/livepeer-network-modules/customer-portal/`

## What this is

**SDKs** are the client libraries that let applications integrate with the suite without
reimplementing protocol details. In the handoff-mode model, the **SDK is the data-plane
client**: it gets a minted payment envelope from the [Payment
Clearinghouse](payment-clearinghouse.md), talks to the orchestrator broker directly, and
reports usage back for settlement.

## What exists today

- **Reference SDKs** — in the clearinghouse repo's
  [`examples/`](../../modules/livepeer-open-clearinghouse/examples/): `python`,
  `typescript`, `go`, `rust`, plus a published `openapi.json`.
- **SDK governance** — the clearinghouse admin domain runs an **SDK approval registry**
  (keyed on `(lang, version, git_sha7)`) and publishes a **signed SDK manifest**
  (`GET /v1/sdk/manifest`, EdDSA) that SDKs check at startup (advisory in v1). SDKs
  identify via the `Livepeer-Open-Clearinghouse-SDK: <lang>/<semver>/<git_sha7>` header.
- **Telemetry ingest** — SDKs can post events to `POST /v1/telemetry`.
- **Conformance harness** — [`conformance/`](../../modules/livepeer-open-clearinghouse/conformance/)
  (mock broker + mock clearinghouse + scenario runners).
- **customer-portal** (network-modules) — a TS SaaS-shell library (API keys, ledger,
  Stripe billing, admin UI widgets), distinct from the client SDKs above.

## SDK catalog

| SDK | Language(s) | Wraps | Where | Status |
| --- | --- | --- | --- | --- |
| Reference SDKs | Python, TypeScript, Go, Rust | Clearinghouse HTTP API + broker handoff | `livepeer-open-clearinghouse/examples/` | 🟠 Reference (lint + coverage gated) |
| customer-portal | TypeScript | SaaS shell: API keys, ledger, Stripe, admin UI | `livepeer-network-modules/customer-portal/` | 🟠 Shipped (library) |

## Role in the suite

- **Wraps:** the [Payment Clearinghouse](payment-clearinghouse.md) HTTP API and the
  [Orchestrators](orchestrators.md) broker (data-plane handoff).
- **Consumed by:** [Reference Apps](reference-apps.md) and external applications.

## To document as SDKs mature

- [ ] Are the `examples/` SDKs published to package registries (PyPI/npm/Go/crates) or
  example-only? Confirm and add install coords.
- [ ] Per-SDK quickstart and compatibility matrix vs the clearinghouse API version.
