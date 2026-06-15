# Tech debt tracker

Known shortcuts, gaps, and follow-ups for the Livepeer Modules Suite. Pay these down
continuously rather than letting them compound (see
[`../design-docs/core-beliefs.md`](../design-docs/core-beliefs.md) #7).

| ID | Item | Why it's debt | Added | Status |
| --- | --- | --- | --- | --- |
| TD-1 | Product specs were stubs | Specs described intent, not confirmed behavior | 2026-05-26 | Mostly resolved — specs documented from eight onboarded repos; Reference Apps still stubbed |
| TD-2 | `ARCHITECTURE.md` job-flow was provisional | Based on protocol convention, not code | 2026-05-26 | Resolved for onboarded repos — supply, demand, gateway, runner, and observability paths grounded against code |
| TD-3 | No submodules added | `modules/` was empty | 2026-05-26 | Resolved — `livepeer-network-modules` onboarded |
| TD-4 | Submodules pinned to commits, not release tags | All eight are pinned at exact commits. Gateways' product/image version is now **`v1.4.1`** but the latest git tag is still `v1.3.1` (`transcode-gateway` `f970fab` = +10, `openai-gateway` `819e059` = +8); `transcode-runners` (`0b32b67`) and `network-modules` (`689b51a`) also ship `v1.4.1` images while untagged; `open-clearinghouse` is past `v1.3.3` (`98b23ab` = +15); observability repos are untagged at `88f0ff8` / `1c81f00`. Re-pin to release tags where available | 2026-05-26 | Open — upstream ships v1.4.1 in package/image versions without cutting matching git tags; pin to release tags once they exist |
| TD-5 | Payment Clearinghouse scope | Was unclear in repo 1 | 2026-05-26 | Resolved — dedicated repo `livepeer-open-clearinghouse` onboarded; demand-side clearinghouse vs supply-side `pool-payout-executor` boundary documented |
| TD-6 | daydream gateway, vtuber runners, Reference-App repos not onboarded | AI + video gateways and AI + video runner tiers are now onboarded; a daydream gateway, vtuber runners, and reference apps are still missing | 2026-05-26 | Partial — both gateways + both runner tiers onboarded; daydream gateway / vtuber runners / reference apps still awaited |
| TD-7 | Capability-name/catalog mapping now lives behind LOC | Gateways no longer own local route selection; model/capability catalogs are LOC-backed. Keep checking runner manifest names against LOC offerings as new workloads are added | 2026-05-26 | Reduced — no local gateway selector bridge; verify LOC catalog mappings in integration tests |
| TD-8 | Gateway migration to LOC completed, but reference-app taxonomy still open | `openai-gateway` and `transcode-gateway` now use LOC for route selection, payment minting, and settlement, removing local daemon/wallet requirements. They are still operator-funded in-path apps rather than customer SDK handoff examples | 2026-05-26 | Partially resolved — LOC migration done; decide whether/when these move under Reference Apps |
| TD-9 | `transcode-gateway` README has daemon-era wording | The pinned submodule's `README.md` still mentions local resolver/payer daemons and vendored proto, while its `DESIGN.md`, `ARCHITECTURE.md`, `DEPLOYMENT.md`, and code show the LOC path | 2026-06-11 | Open — fix upstream README or carry this caveat in suite docs |

## Conventions

- Add a row when you knowingly defer something.
- Close it (Status → Done, with the commit/PR) when resolved, or delete the row if it
  becomes irrelevant.
