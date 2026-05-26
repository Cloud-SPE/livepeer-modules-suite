# Product specs index

One overview per Livepeer Module. Each starts as a **stub** and is filled in when the
module's repository is handed over and read. Keep this table in sync with
[`../../AGENTS.md`](../../AGENTS.md).

**Two axes.** These are conceptual **capabilities** (the "what"). The concrete repos that
implement them (the "where") are in [`../repos/`](../repos/index.md). One repo can
implement several capabilities.

## Modules

| Module | Spec | Implemented in | Status |
| --- | --- | --- | --- |
| Gateways | [gateways.md](gateways.md) | _contracts only in_ [livepeer-network-modules](../repos/livepeer-network-modules.md); shell TBD | 🟡 Stub — awaiting gateway repo |
| Orchestrators | [orchestrators.md](orchestrators.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Runners | [runners.md](runners.md) | contracts in [livepeer-network-modules](../repos/livepeer-network-modules.md); backends TBD | 🟠 Documented (concept) |
| Pools | [pools.md](pools.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Payment | [payment.md](payment.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Service Registry | [service-registry.md](service-registry.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Discover | [discover.md](discover.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Payment Clearinghouse | [payment-clearinghouse.md](payment-clearinghouse.md) | _partial in_ [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Partial |
| SDKs | [sdks.md](sdks.md) | `customer-portal` in [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Partial |
| Reference Apps | [reference-apps.md](reference-apps.md) | TBD (removed from network-modules) | 🟡 Stub — awaiting repo |

## Status legend

- 🟡 **Stub** — placeholder; describes intent only, no repo onboarded yet.
- 🟠 **Documented / Partial** — at least one repo onboarded; spec filled from the code
  (Partial = only part of the capability is covered by onboarded repos so far).
- ✅ **Documented** — spec confirmed against the submodule(s) at a pinned revision.

## Onboarding a module (checklist)

When the user hands over a module repository:

1. Add it as a submodule under `modules/<name>/` (see
   [`../guides/git-submodules-primer.md`](../guides/git-submodules-primer.md)).
2. Read the repo: README, architecture, public APIs/interfaces, config, deploy.
3. Fill in `<module>.md` — replace hedged language with confirmed facts; link to
   specific files/dirs in the submodule.
4. Record the pinned revision (commit/tag) you documented against.
5. Update this table's **Submodule** and **Status** columns, and the table in
   `AGENTS.md`.
6. If you cut any corners, log them in
   [`../exec-plans/tech-debt-tracker.md`](../exec-plans/tech-debt-tracker.md).
