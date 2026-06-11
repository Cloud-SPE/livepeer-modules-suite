# Design docs index

Catalog of design documentation for the Livepeer Modules Suite, with verification
status. "Verified" means a human or agent has confirmed the document matches reality
(the code in submodules, or an explicit decision).

| Doc | Purpose | Status |
| --- | --- | --- |
| [core-beliefs.md](core-beliefs.md) | Agent-first operating principles for this repo | ✅ Verified |
| [../../ARCHITECTURE.md](../../ARCHITECTURE.md) | Top-level domain map + job-flow across capabilities | 🟠 Documented across onboarded repos; Reference Apps still stubbed |
| [../glossary.md](../glossary.md) | Shared vocabulary for the suite | 🟠 Verified across onboarded repos |
| [../product-specs/index.md](../product-specs/index.md) | Capability overviews + onboarding status | 🟠 Network + observability documented |
| [../repos/index.md](../repos/index.md) | Per-submodule docs + pinned revisions | 🟠 Eight repos onboarded |

## Status legend

- ✅ **Verified** — reviewed against reality; safe to rely on.
- 🟡 **Draft** — provisional; describes intent, not yet confirmed against code.
- 🔴 **Stale** — known to be out of date; fix or delete.

## How to use this index

- Adding a design doc? Add a row here with an honest status.
- Onboarding a module? Move its spec from 🟡 toward ✅ as you confirm facts against the
  submodule, and update its row in [../product-specs/index.md](../product-specs/index.md).
