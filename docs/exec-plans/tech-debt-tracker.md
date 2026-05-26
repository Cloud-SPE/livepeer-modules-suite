# Tech debt tracker

Known shortcuts, gaps, and follow-ups for the Livepeer Modules Suite. Pay these down
continuously rather than letting them compound (see
[`../design-docs/core-beliefs.md`](../design-docs/core-beliefs.md) #7).

| ID | Item | Why it's debt | Added | Status |
| --- | --- | --- | --- | --- |
| TD-1 | Product specs were stubs | Specs described intent, not confirmed behavior | 2026-05-26 | Mostly resolved — supply-side specs documented from `livepeer-network-modules`; Gateways/Reference Apps still stubs |
| TD-2 | `ARCHITECTURE.md` job-flow was provisional | Based on protocol convention, not code | 2026-05-26 | Resolved (supply side) — grounded against first repo; demand side still provisional |
| TD-3 | No submodules added | `modules/` was empty | 2026-05-26 | Resolved — `livepeer-network-modules` onboarded |
| TD-4 | Submodule pinned to a commit, not a release tag | `livepeer-network-modules` is pinned at `95c6415` (a branch tip), not a tagged network release | 2026-05-26 | Open — pin to a release tag when one is identified |
| TD-5 | Payment Clearinghouse scope unresolved | No standalone clearinghouse in first repo; function split across receiver + `pool-payout-executor`. May be a dedicated repo later | 2026-05-26 | Open — confirm with user / future repo |
| TD-6 | Gateway, Runner, and Reference-App repos not yet onboarded | These were removed from `livepeer-network-modules`; only contracts are known | 2026-05-26 | Open — awaiting repos |

## Conventions

- Add a row when you knowingly defer something.
- Close it (Status → Done, with the commit/PR) when resolved, or delete the row if it
  becomes irrelevant.
