# Reference Apps

**Status:** 🟡 Stub — awaiting repo
**Submodule(s):** _TBD — named gateway/reference families were removed from `livepeer-network-modules`_

## What this is

**Reference Apps** are example gateway applications that demonstrate how to build on the
suite end to end — submitting work, selecting orchestrators, paying, and consuming
results, typically through the [SDKs](sdks.md) and a [Gateway](gateways.md).

The first repo ([livepeer-network-modules](../repos/livepeer-network-modules.md)) once
contained named gateway/reference families (e.g. an OpenAI gateway reference, vtuber and
video flows) but they were **removed from its working tree** (2026-05-19/20); their
exec-plans remain as history. Expect reference apps to arrive as **separate repos**.

## Role in the suite

- **Built on:** [Gateways](gateways.md) and [SDKs](sdks.md).
- **Exercises:** the full job-flow in [`../../ARCHITECTURE.md`](../../ARCHITECTURE.md).

## To document when repo(s) are provided

- [ ] Which example apps exist and which workload each demonstrates
- [ ] Stack/framework and which SDKs + Gateway features they exercise
- [ ] How to run locally (setup, env, commands)
- [ ] What to copy vs treat as illustrative; pinned revision
