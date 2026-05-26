# Runners

**Status:** 🟠 Documented — concrete backends in [openai-runners](../repos/livepeer-modules-openai-runners.md) (AI) and [transcode-runners](../repos/livepeer-modules-transcode-runners.md) (video); contracts in [network-modules](../repos/livepeer-network-modules.md)
**Submodule(s):** `modules/livepeer-modules-openai-runners/`, `modules/livepeer-modules-transcode-runners/` (+ broker contract in network-modules)

## What this is

A **Runner** is the **backend that actually executes a workload** — the provided
capability that an orchestrator's broker manages and serves to gateways. Examples now
exist as concrete containers: chat, embeddings, audio (transcribe/translate), TTS, image
generation, and rerank.

The key design point holds: **runners are not hard-coded into the broker.** The
[capability broker](orchestrators.md) is workload-agnostic — it dispatches to runners over
standard HTTP, declared in `host-config.yaml`. A runner receives a **fully-authenticated**
request (the broker already handled payment/auth) and just does the work; it is **not**
responsible for auth, billing, payment, or tenant identity.

## The broker ↔ runner contract

Broker is the client, runner is the HTTP server. Every runner implements:

- `POST <capability-path>` returning capability-shaped JSON and reporting **work units**
  (a body field like `usage.total_tokens`, or an `X-Livepeer-Work-Units` header/trailer).
- `GET /healthz` (broker liveness), `GET /<capability>/options` (orch-coordinator
  discovery; broker merges the `extra` block into host-config), `GET /metrics` (opt-in).
- A standard error contract (400/429/503/500) and, for ML runners, **GPU fail-fast**.

Each image bakes an `offering.yaml` (capability, offerings, rate-card hints, model
metadata) that drives broker discovery and pricing. See
[`BROKER-CONTRACT.md`](../../modules/livepeer-modules-openai-runners/BROKER-CONTRACT.md)
and [`RUNNER-INVARIANTS.md`](../../modules/livepeer-modules-openai-runners/RUNNER-INVARIANTS.md).

## Concrete runners (openai-runners repo)

| Capability | Runner | Backend | Mode |
| --- | --- | --- | --- |
| `openai-chat-completions` | openai-chat-runner (Go) | vLLM / Ollama | `http-stream@v0` |
| `openai-text-embeddings` | openai-embeddings-runner (Go) | vLLM / Ollama | `http-reqresp@v0` |
| `openai-audio-transcriptions` / `-translations` | openai-audio-runner (Py) | Whisper | request/response (multipart upload) |
| `openai-audio-speech` | openai-tts-runner (Py) | Kokoro | `http-reqresp@v0` |
| `image-generation` | openai-image-generation-runner (Py) | diffusers (SDXL/FLUX) | `http-reqresp@v0` |
| `rerank` | rerank-runner (Py) | CrossEncoder | `http-reqresp@v0` |

See the repo doc for the full table, model downloaders, and the smoke-test image.

## Concrete runners (transcode-runners repo)

Video execution tier (Go + FFmpeg; NVIDIA/Intel/AMD images; strict GPU mode by default):

| Capability | Runner | Shape | Mode |
| --- | --- | --- | --- |
| `video-transcode` | transcode-runner | single-rendition VOD (submit→poll) | async HTTP job |
| `video-transcode-abr` | abr-runner | multi-rendition ABR/HLS ladder (submit→poll) | async HTTP job |
| `livepeer:transcode/live-rtmp-hls-abr` | live-runner | live RTMP-in → HLS-out session | `live-session-gateway-ingest@v0` |

VOD runners report `video_seconds`; the **live-runner** emits `output_seconds` usage
events to broker callbacks. The live path runs
[clearinghouse](payment-clearinghouse.md) `sessions` → broker → live-runner ("Option B").
See [the repo doc](../repos/livepeer-modules-transcode-runners.md) for the contract.

## Runner shapes (by interaction mode)

| Mode | Runner shape |
| --- | --- |
| `http-reqresp` / `http-stream` / `http-multipart` | An HTTP backend; broker calls it and extracts units from the response |
| `ws-realtime` | A backend that speaks WebSocket; broker relays bidirectional frames |
| `rtmp-ingress-hls-egress` | An FFmpeg/RTMP backend; broker manages RTMP in / HLS out |
| `session-control-plus-media` | A local container/subprocess; broker owns the long-lived control + media plane |
| `live-session-remote-runner` | An **external** runtime; broker keeps payment/session authority while the runner owns its own RTMP/HLS production |

## Role in the suite

- **Declared by:** an operator in the broker's `host-config.yaml`.
- **Dispatched to by:** the [capability broker](orchestrators.md), after payment validation.
- **Metered by:** the work-unit field/header → [Payment](payment.md).
- **Surfaced to:** [Gateways](gateways.md)/SDKs only as capability tuples via [Discover](discover.md).

## Open items

- [ ] **Capability naming:** runners use hyphen form (`openai-chat-completions`); broker
  host-config examples used colon form (`openai:chat-completions`). Confirm the exact
  end-to-end mapping carried by `Livepeer-Capability`.
- [ ] **vtuber runners** remain a separate, not-yet-onboarded repo. (AI + video tiers done.)
- [ ] openai-runners is v1.3.0 / grade C; transcode-runners' **live-runner is design-phase**
  (Option B) — neither runner tier is validated end-to-end on GPU yet.
