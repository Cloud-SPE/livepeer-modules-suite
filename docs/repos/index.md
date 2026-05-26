# Repositories (submodules)

The concrete repos onboarded into the suite, one doc each. This is the **repo axis**;
the **capability axis** (Gateways, Orchestrators, Pools, …) lives in
[`../product-specs/`](../product-specs/index.md). A single repo can implement several
capabilities — see each repo doc for the mapping.

| Repo | Path | Implements (capabilities) | Pinned | Status |
| --- | --- | --- | --- | --- |
| [livepeer-network-modules](livepeer-network-modules.md) | `modules/livepeer-network-modules/` | Orchestrators, Runners, Pools, Payment, Service Registry, Discover, Protocol, SDKs (customer-portal) | `95c6415` | 🟠 Onboarded — documented |
| [livepeer-open-clearinghouse](livepeer-open-clearinghouse.md) | `modules/livepeer-open-clearinghouse/` | Payment Clearinghouse, Payment (issuance), SDKs, Discover (proxy), Gateways (control-plane half) | `a529592` | 🟠 Onboarded — documented |
| [livepeer-modules-openai-runners](livepeer-modules-openai-runners.md) | `modules/livepeer-modules-openai-runners/` | Runners (OpenAI/Cohere-shaped AI backends) | `3ea3f17` | 🟠 Onboarded — documented (v1.3.0, grade C) |
| [livepeer-modules-transcode-runners](livepeer-modules-transcode-runners.md) | `modules/livepeer-modules-transcode-runners/` | Runners (video: VOD transcode, ABR ladder, live) | `b33e32f` | 🟠 Onboarded — documented (live-runner in design) |
| [livepeer-modules-transcode-gateway](livepeer-modules-transcode-gateway.md) | `modules/livepeer-modules-transcode-gateway/` | Gateways (full in-path video gateway), Discover, Payment | `4086880` | 🟠 Onboarded — documented (v1.3.0, grade C / tests F) |
| [livepeer-modules-openai-gateway](livepeer-modules-openai-gateway.md) | `modules/livepeer-modules-openai-gateway/` | Gateways (OpenAI-compatible AI gateway), SDKs (wire-compat), Discover, Payment | `5afaf96` | 🟠 Onboarded — documented (v1.3.1+, grade C, 45 tests) |

**Cross-repo dependencies:**
- **Three demand-side front doors** (pick one per deployment): non-custodial credit+handoff
  ([open-clearinghouse](livepeer-open-clearinghouse.md)) vs. operator-funded in-path
  gateways for video ([transcode-gateway](livepeer-modules-transcode-gateway.md)) and AI
  ([openai-gateway](livepeer-modules-openai-gateway.md)). All consume `payment-daemon` +
  `service-registry-daemon` from `livepeer-network-modules` over Unix-socket gRPC.
- `livepeer-modules-openai-runners` and `livepeer-modules-transcode-runners` sit **behind**
  the `capability-broker` in `livepeer-network-modules` (broker = client, runner = HTTP
  server). The **openai-gateway** fronts the openai-runners; the **transcode-gateway**
  fronts the transcode-runners.
- **Live path:** `transcode-gateway` → broker → `transcode-runners` **live-runner**
  (`live-session-gateway-ingest@v0`, "Option B").

## Status legend

- 🟡 **Listed** — known, not yet added as a submodule.
- 🟠 **Onboarded** — submodule added and a repo doc written from the code.
- ✅ **Tracked at release** — pinned to a tagged network release and kept current.

## Onboarding checklist

See [`../product-specs/index.md`](../product-specs/index.md#onboarding-a-module-checklist)
and [`../guides/git-submodules-primer.md`](../guides/git-submodules-primer.md). Record the
pinned revision in the table above and in the repo doc.
