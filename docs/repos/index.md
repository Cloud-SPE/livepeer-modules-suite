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

**Cross-repo dependencies:**
- `livepeer-open-clearinghouse` consumes `payment-daemon` and `service-registry-daemon`
  from `livepeer-network-modules` over Unix-socket gRPC.
- `livepeer-modules-openai-runners` sit **behind** the `capability-broker` in
  `livepeer-network-modules` (broker = client, runner = HTTP server; see its
  `BROKER-CONTRACT.md`).
- `livepeer-modules-transcode-runners` sit behind the same broker; its **live-runner**
  implements `live-session-gateway-ingest@v0` — the live path runs clearinghouse `sessions`
  → broker → live-runner ("Option B").

## Status legend

- 🟡 **Listed** — known, not yet added as a submodule.
- 🟠 **Onboarded** — submodule added and a repo doc written from the code.
- ✅ **Tracked at release** — pinned to a tagged network release and kept current.

## Onboarding checklist

See [`../product-specs/index.md`](../product-specs/index.md#onboarding-a-module-checklist)
and [`../guides/git-submodules-primer.md`](../guides/git-submodules-primer.md). Record the
pinned revision in the table above and in the repo doc.
