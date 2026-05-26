# Runners

**Status:** 🟠 Documented (concept) via [livepeer-network-modules](../repos/livepeer-network-modules.md)
**Submodule:** backends are external; the protocol is in `modules/livepeer-network-modules/livepeer-network-protocol/`

## What this is

A **Runner** is the **backend that actually executes a workload** — the provided
capability that an orchestrator's broker manages and serves to gateways. Examples:
a vLLM/TGI server, an OpenAI-compatible API, an FFmpeg pipeline, or a long-lived
session runtime.

The key design point: **runners are not a fixed binary or a daemon in the supply-side
repo.** The [capability broker](orchestrators.md) is workload-agnostic — it dispatches
to runners over standard wires (HTTP, WebSocket, RTMP, or a managed subprocess/session)
declared in `host-config.yaml`. The broker knows the backend URL and the
**interaction mode** contract; the runner itself is opaque to it.

## How runners relate to the rest

- **Declared by:** an operator, in the broker's `host-config.yaml` (backend transport +
  URL + interaction mode + work unit + extractor + health probe).
- **Dispatched to by:** the [capability broker](orchestrators.md), after payment is
  validated.
- **Metered by:** the declared **extractor** → `actualUnits` → [Payment](payment.md).
- **Surfaced to:** [Gateways](gateways.md) only as capability tuples via
  [Discover](discover.md); gateways never address a runner directly.

## Runner shapes (by interaction mode)

| Mode | Runner shape |
| --- | --- |
| `http-reqresp` / `http-stream` / `http-multipart` | An HTTP backend; broker calls it and extracts units from the response |
| `ws-realtime` | A backend that speaks WebSocket; broker relays bidirectional frames |
| `rtmp-ingress-hls-egress` | An FFmpeg/RTMP backend; broker manages RTMP in / HLS out |
| `session-control-plus-media` | A local container/subprocess; broker owns the long-lived control + media plane |
| `live-session-remote-runner` | An **external** runtime; broker keeps payment/session authority while the runner owns its own RTMP/HLS production |

## Notes & open items

- The named runner product families (and gateway shells) were **removed from
  `livepeer-network-modules`'s working tree** (2026-05-20); the contracts and the
  `sessionrunner` protocol remain. Concrete runner implementations are expected to arrive
  as **separate repos** — document them here and in [repos/](../repos/index.md) as they
  land.
- [ ] Confirm which runner repos exist and how each maps to an interaction mode.
- [ ] Document the `sessionrunner` protocol surface once a remote-runner repo is onboarded.
