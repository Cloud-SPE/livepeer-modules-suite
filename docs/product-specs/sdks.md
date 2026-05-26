# SDKs

**Status:** 🟠 Partial via [livepeer-network-modules](../repos/livepeer-network-modules.md) (`95c6415`)
**Submodule(s):** `modules/livepeer-network-modules/customer-portal/` (more TBD)

## What this is

**SDKs** are the client libraries that let applications and modules integrate with the
suite without reimplementing protocol details. So far the only library-shaped surface
onboarded is **`customer-portal`**, a shared **TypeScript** SaaS library (not a deployed
service) consumed across the workspace via `workspace:*`.

`customer-portal` exposes subpaths for `auth` (API-key auth), `billing` (customer ledger,
Stripe top-ups), `payment`, `middleware` (Fastify pre-handlers), `admin` (operator admin
engine), `db`, and `registry`, plus a Lit + RxJS frontend widget catalog.

## Role in the suite

- **Wraps / supports:** customer accounts, billing, and admin surfaces used by
  [Gateways](gateways.md) and the [Payment Clearinghouse](payment-clearinghouse.md)
  function.
- **Consumed by:** gateway shells and (future) [Reference Apps](reference-apps.md).

## SDK catalog

| SDK | Language(s) | Wraps | Package / install | Repo | Status |
| --- | --- | --- | --- | --- | --- |
| customer-portal | TypeScript | API keys, ledger, Stripe billing, admin UI | `workspace:*` (pnpm) | livepeer-network-modules | 🟠 Shipped (library) |

## To document as more SDKs arrive

- [ ] Are there client SDKs for the Gateway/Payment/Clearinghouse wire APIs (beyond the
  SaaS-shell library)?
- [ ] Languages and public package registries (npm, PyPI, Go module, …)
- [ ] Compatibility matrix vs the modules they wrap
- [ ] Quickstart per SDK
