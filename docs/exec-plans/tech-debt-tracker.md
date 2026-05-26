# Tech debt tracker

Known shortcuts, gaps, and follow-ups for the Livepeer Modules Suite. Pay these down
continuously rather than letting them compound (see
[`../design-docs/core-beliefs.md`](../design-docs/core-beliefs.md) #7).

| ID | Item | Why it's debt | Added | Status |
| --- | --- | --- | --- | --- |
| TD-1 | Product specs were stubs | Specs described intent, not confirmed behavior | 2026-05-26 | Mostly resolved — supply-side specs documented from `livepeer-network-modules`; Gateways/Reference Apps still stubs |
| TD-2 | `ARCHITECTURE.md` job-flow was provisional | Based on protocol convention, not code | 2026-05-26 | Resolved (supply side) — grounded against first repo; demand side still provisional |
| TD-3 | No submodules added | `modules/` was empty | 2026-05-26 | Resolved — `livepeer-network-modules` onboarded |
| TD-4 | Submodules pinned to commits, not release tags | All six pinned at non-tag commits; the gateways are just past tag **`v1.3.1`** (`transcode-gateway` `4086880` = +1, `openai-gateway` `5afaf96` = +2). Gateways demonstrably publish release tags; check the others and re-pin to tags | 2026-05-26 | Open — pin to release tags (start with the gateways' `v1.3.1`) |
| TD-5 | Payment Clearinghouse scope | Was unclear in repo 1 | 2026-05-26 | Resolved — dedicated repo `livepeer-open-clearinghouse` onboarded; demand-side clearinghouse vs supply-side `pool-payout-executor` boundary documented |
| TD-6 | daydream gateway, vtuber runners, Reference-App repos not onboarded | AI + video gateways and AI + video runner tiers are now onboarded; a daydream gateway, vtuber runners, and reference apps are still missing | 2026-05-26 | Partial — both gateways + both runner tiers onboarded; daydream gateway / vtuber runners / reference apps still awaited |
| TD-7 | Capability-name form varies across repos | Confirmed on both sides: gateways query **colon** form (`openai:chat-completions`, `video:transcode.abr`); runner manifests declare **hyphen/slash** form (`openai-chat-completions`, `video-transcode-abr`, `livepeer:transcode/live-rtmp-hls-abr`) | 2026-05-26 | Open — registry/host-config is the bridge; confirm end-to-end mapping carried by `Livepeer-Capability` |

## Conventions

- Add a row when you knowingly defer something.
- Close it (Status → Done, with the commit/PR) when resolved, or delete the row if it
  becomes irrelevant.
