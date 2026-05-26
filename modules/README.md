# modules/

Git submodules live here, **one per repository**, at `modules/<repo-name>/`. Repos are
onboarded one at a time and pinned to a revision. Add with
`git submodule add … modules/<repo-name>`; see
[`../docs/guides/git-submodules-primer.md`](../docs/guides/git-submodules-primer.md).

Note: submodules are named after the **repo**, not the conceptual capability — a single
repo can implement several capabilities (Gateways, Orchestrators, Pools, …). The
repo↔capability mapping lives in each [`docs/repos/<repo>.md`](../docs/repos/index.md).

## Onboarded

```text
modules/
└── livepeer-network-modules/   # supply-side core (pinned 95c6415)
```

- Doc: [`../docs/repos/livepeer-network-modules.md`](../docs/repos/livepeer-network-modules.md)
- Canonical map: [`../AGENTS.md`](../AGENTS.md)

Expected next (per the suite taxonomy): gateway shell(s), runner backends, and reference
apps, as separate repos.
