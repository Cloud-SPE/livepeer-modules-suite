# Tech debt tracker

Known shortcuts, gaps, and follow-ups for the Livepeer Modules Suite. Pay these down
continuously rather than letting them compound (see
[`../design-docs/core-beliefs.md`](../design-docs/core-beliefs.md) #7).

| ID | Item | Why it's debt | Added | Status |
| --- | --- | --- | --- | --- |
| TD-1 | Product specs were stubs | Specs described intent, not confirmed behavior | 2026-05-26 | Mostly resolved — supply-side specs documented from `livepeer-network-modules`; Gateways/Reference Apps still stubs |
| TD-2 | `ARCHITECTURE.md` job-flow was provisional | Based on protocol convention, not code | 2026-05-26 | Resolved (supply side) — grounded against first repo; demand side still provisional |
| TD-3 | No submodules added | `modules/` was empty | 2026-05-26 | Resolved — `livepeer-network-modules` onboarded |
| TD-4 | Submodules pinned to commits, not release tags | All five pinned at non-tag commits (`network-modules` `95c6415`, `open-clearinghouse` `a529592`, `openai-runners` `3ea3f17`, `transcode-runners` `b33e32f`, `transcode-gateway` `4086880` = 1 commit past tag **`v1.3.1`**). `transcode-gateway` demonstrably has release tags; check the others and re-pin to tags | 2026-05-26 | Open — pin to release tags (start with transcode-gateway `v1.3.1`) |
| TD-5 | Payment Clearinghouse scope | Was unclear in repo 1 | 2026-05-26 | Resolved — dedicated repo `livepeer-open-clearinghouse` onboarded; demand-side clearinghouse vs supply-side `pool-payout-executor` boundary documented |
| TD-6 | OpenAI/daydream gateway, vtuber runners, Reference-App repos not onboarded | A full video gateway + AI + video runners are now onboarded; transcode-gateway was ported from an OpenAI/daydream gateway sibling; vtuber runners + reference apps still missing | 2026-05-26 | Partial — gateway + both runner tiers onboarded; OpenAI/daydream gateway / vtuber runners / reference apps still awaited |
| TD-7 | Capability-name form varies across repos | Concrete mismatch: transcode-gateway queries `video:transcode.abr` / `video:transcode.live`, transcode-runners manifests declare `video-transcode-abr` / `livepeer:transcode/live-rtmp-hls-abr`; openai-runners hyphen (`openai-chat-completions`) vs broker host-config colon (`openai:chat-completions`) | 2026-05-26 | Open — registry/host-config is the bridge; confirm end-to-end mapping carried by `Livepeer-Capability` |

## Conventions

- Add a row when you knowingly defer something.
- Close it (Status → Done, with the commit/PR) when resolved, or delete the row if it
  becomes irrelevant.
