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

> **Planned (TD-8):** the [openai-gateway](../repos/livepeer-modules-openai-gateway.md) and
> [transcode-gateway](../repos/livepeer-modules-transcode-gateway.md) are **meant to become
> the reference examples**. Once migrated off direct `service-registry-daemon`/`payment-daemon`
> usage and onto the [clearinghouse](payment-clearinghouse.md) + [SDKs](sdks.md), they
> demonstrate building on Livepeer without managing a wallet or on-chain complexity. See
> [Gateways](gateways.md) for the as-is vs intended posture.

## Role in the suite

- **Built on:** [Gateways](gateways.md) and [SDKs](sdks.md).
- **Exercises:** the full job-flow in [`../../ARCHITECTURE.md`](../../ARCHITECTURE.md).

## To document when repo(s) are provided

- [ ] Which example apps exist and which workload each demonstrates
- [ ] Stack/framework and which SDKs + Gateway features they exercise
- [ ] How to run locally (setup, env, commands)
- [ ] What to copy vs treat as illustrative; pinned revision
