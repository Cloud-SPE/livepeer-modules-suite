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
| Gateways | [gateways.md](gateways.md) | LOC-mediated full gateways [openai-gateway](../repos/livepeer-modules-openai-gateway.md) (AI) + [transcode-gateway](../repos/livepeer-modules-transcode-gateway.md) (video); handoff path in [open-clearinghouse](../repos/livepeer-open-clearinghouse.md) | 🟠 Documented |
| Orchestrators | [orchestrators.md](orchestrators.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Runners | [runners.md](runners.md) | [livepeer-modules-openai-runners](../repos/livepeer-modules-openai-runners.md) (backends) + contracts in [network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Pools | [pools.md](pools.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Payment | [payment.md](payment.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) (+ issuance via [open-clearinghouse](../repos/livepeer-open-clearinghouse.md)) | 🟠 Documented |
| Service Registry | [service-registry.md](service-registry.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Discover | [discover.md](discover.md) | [livepeer-network-modules](../repos/livepeer-network-modules.md) (+ proxy in [open-clearinghouse](../repos/livepeer-open-clearinghouse.md)) | 🟠 Documented |
| Payment Clearinghouse | [payment-clearinghouse.md](payment-clearinghouse.md) | [livepeer-open-clearinghouse](../repos/livepeer-open-clearinghouse.md) | 🟠 Documented |
| SDKs | [sdks.md](sdks.md) | [open-clearinghouse](../repos/livepeer-open-clearinghouse.md) `sdks/` + runnable `examples/`; `customer-portal` in [network-modules](../repos/livepeer-network-modules.md) | 🟠 Documented |
| Reference Apps | [reference-apps.md](reference-apps.md) | TBD (removed from network-modules) | 🟡 Stub — awaiting repo |

### Observability & Reporting (off-network)

These **track on-chain activity** — they don't provide network supply or accept demand.
They observe the economic output the network capabilities above produce on-chain.

| Capability | Spec | Implemented in | Status |
| --- | --- | --- | --- |
| Protocol Explorer | [protocol-explorer.md](protocol-explorer.md) | [livepeer-protocol-explorer](../repos/livepeer-protocol-explorer.md) | 🟠 Documented |
| Network Bot | [network-bot.md](network-bot.md) | [livepeer-network-bot](../repos/livepeer-network-bot.md) | 🟠 Documented |

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
