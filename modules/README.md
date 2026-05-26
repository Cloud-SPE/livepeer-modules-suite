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
├── livepeer-network-modules/    # supply-side core (pinned 95c6415)
└── livepeer-open-clearinghouse/ # demand-side clearinghouse (pinned a529592)
```

- Docs: [`../docs/repos/livepeer-network-modules.md`](../docs/repos/livepeer-network-modules.md),
  [`../docs/repos/livepeer-open-clearinghouse.md`](../docs/repos/livepeer-open-clearinghouse.md)
- Canonical map: [`../AGENTS.md`](../AGENTS.md)

Expected next (per the suite taxonomy): a standalone gateway shell, runner backends, and
reference apps, as separate repos.
