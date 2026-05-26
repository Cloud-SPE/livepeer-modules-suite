# Tech debt tracker

Known shortcuts, gaps, and follow-ups for the Livepeer Modules Suite. Pay these down
continuously rather than letting them compound (see
[`../design-docs/core-beliefs.md`](../design-docs/core-beliefs.md) #7).

| ID | Item | Why it's debt | Added | Status |
| --- | --- | --- | --- | --- |
| TD-1 | Product specs were stubs | Specs described intent, not confirmed behavior | 2026-05-26 | Mostly resolved — supply-side specs documented from `livepeer-network-modules`; Gateways/Reference Apps still stubs |
| TD-2 | `ARCHITECTURE.md` job-flow was provisional | Based on protocol convention, not code | 2026-05-26 | Resolved (supply side) — grounded against first repo; demand side still provisional |
| TD-3 | No submodules added | `modules/` was empty | 2026-05-26 | Resolved — `livepeer-network-modules` onboarded |
| TD-4 | Submodules pinned to commits, not release tags | `network-modules` `95c6415`, `open-clearinghouse` `a529592`, `openai-runners` `3ea3f17`, `transcode-runners` `b33e32f` are branch tips, not tagged releases | 2026-05-26 | Open — pin to release tags when identified |
| TD-5 | Payment Clearinghouse scope | Was unclear in repo 1 | 2026-05-26 | Resolved — dedicated repo `livepeer-open-clearinghouse` onboarded; demand-side clearinghouse vs supply-side `pool-payout-executor` boundary documented |
| TD-6 | Standalone Gateway shell, vtuber runners, Reference-App repos not onboarded | Gateway role split (clearinghouse + SDK); AI + video runner tiers onboarded; vtuber is a sibling repo; reference-apps removed from network-modules | 2026-05-26 | Partial — AI + video runners onboarded; gateway shell / vtuber runners / reference apps still awaited |
| TD-7 | Capability-name form varies across repos | openai-runners hyphen (`openai-chat-completions`); transcode-runners hyphen (`video-transcode`) + colon/slash (`livepeer:transcode/live-rtmp-hls-abr`); broker host-config examples colon (`openai:chat-completions`) | 2026-05-26 | Open — confirm end-to-end mapping carried by `Livepeer-Capability` |

## Conventions

- Add a row when you knowingly defer something.
- Close it (Status → Done, with the commit/PR) when resolved, or delete the row if it
  becomes irrelevant.
