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
├── livepeer-network-modules/           # supply-side core (pinned 95c6415)
├── livepeer-open-clearinghouse/        # demand-side clearinghouse (pinned a529592)
├── livepeer-modules-openai-runners/    # AI runner backends (pinned 3ea3f17)
├── livepeer-modules-transcode-runners/ # video runner backends (pinned b33e32f)
├── livepeer-modules-transcode-gateway/ # full video gateway (pinned 4086880)
└── livepeer-modules-openai-gateway/    # full OpenAI-compatible AI gateway (pinned 5afaf96)
```

- Docs: [`../docs/repos/`](../docs/repos/index.md) — one per repo
- Canonical map: [`../AGENTS.md`](../AGENTS.md)

Expected next (per the suite taxonomy): a daydream gateway, vtuber runner backends, and
reference apps, as separate repos.
