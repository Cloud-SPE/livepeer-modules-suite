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
├── livepeer-network-modules/           # supply-side core (pinned 6406a6d)
├── livepeer-open-clearinghouse/        # demand-side clearinghouse (pinned 98b23ab)
├── livepeer-modules-openai-runners/    # AI runner backends (pinned 3ea3f17)
├── livepeer-modules-transcode-runners/ # video runner backends (pinned b33e32f)
├── livepeer-modules-transcode-gateway/ # full video gateway (pinned 2a3cbfa)
├── livepeer-modules-openai-gateway/    # full OpenAI-compatible AI gateway (pinned baf214a)
├── livepeer-protocol-explorer/         # observability: on-chain data platform (pinned 88f0ff8)
└── livepeer-network-bot/               # observability: Discord reporting bot (pinned 1c81f00)
```

- Docs: [`../docs/repos/`](../docs/repos/index.md) — one per repo
- Canonical map: [`../AGENTS.md`](../AGENTS.md)

Expected next (per the suite taxonomy): a daydream gateway, vtuber runner backends, and
reference apps, as separate repos.
