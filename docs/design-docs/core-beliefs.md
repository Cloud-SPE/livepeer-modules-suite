# Core beliefs

The agent-first operating principles for the Livepeer Modules Suite. Adapted from the
harness-engineering approach in
[`../references/openai-harness-engineer.md`](../references/openai-harness-engineer.md)
and specialized for a documentation + submodules umbrella repo.

These are stable. Change them deliberately, in a PR, with a reason.

## 1. The repository is the system of record

If knowledge isn't in this repo as versioned markdown — or in a tracked submodule — it
effectively doesn't exist for an agent. Slack threads, calls, and tacit knowledge are
invisible. When something important is decided, **write it down here.**

## 2. AGENTS.md is a map, not a manual

[`AGENTS.md`](../../AGENTS.md) is a ~100-line table of contents. It points to deeper
sources of truth. We never grow it into an encyclopedia: a giant instruction file
crowds out the task, rots into stale rules, and becomes non-guidance ("when everything
is important, nothing is").

## 3. Progressive disclosure

Start from a small, stable entry point and follow links to detail. Readers (human or
agent) should be able to go from "what is this?" to "the specific fact I need" in a few
hops, without loading everything at once.

## 4. Don't fabricate module facts

This umbrella repo describes modules whose code lives elsewhere and arrives over time.
Until a module's repo is handed over and read, its spec is a **draft** and must be
marked as such. A correct "not yet confirmed" is more valuable than a confident guess
that later misleads. Document what the code actually shows, with a pointer to the
source under `modules/<name>/`.

## 5. The overview must match a real, pinned set of releases

This suite tracks specific submodule commits. The point is reproducibility: the
overview should always correspond to a known set of module releases, not "whatever is
on each module's main branch today." Update submodule pins deliberately, and note what
changed. See [`../guides/git-submodules-primer.md`](../guides/git-submodules-primer.md).

## 6. Enforce boundaries, allow local freedom

The valuable, durable thing about the suite is the **interfaces between modules**.
Protect those. Within a module's own docs, allow whatever structure communicates best.
Care deeply about the seams; be relaxed about the interiors.

## 7. Keep the map current; garbage-collect drift

When a module's status, interface, or release changes, update its spec *and* the
summary table in `AGENTS.md` in the same change. Stale docs are a liability — pay the
small cost continuously rather than letting drift compound.

## 8. Humans steer, the repo compounds

Human judgment (which module to onboard next, what the architecture should be, what's
correct) is captured here once and then reused by every future agent run. When an agent
struggles, the fix is usually a missing doc or guardrail — add it here.
