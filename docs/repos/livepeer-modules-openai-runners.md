# livepeer-modules-openai-runners

**Submodule:** `modules/livepeer-modules-openai-runners/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-modules-openai-runners.git`
**Pinned revision:** `3ea3f17` (documented 2026-05-26)
**Status:** 🟠 Onboarded — documented from code. Repo's own status: **v1.3.0 initial
release; all images graded "C"** (build, not yet exercised end-to-end against a broker).

The first concrete set of **[Runner](../product-specs/runners.md) backends**: containerized
AI services that sit **behind the capability broker** and expose model APIs in
(mostly) OpenAI-compatible shapes, plus one Cohere-compatible rerank API. One Docker image
per capability, plus model-downloader and smoke-test images.

This is the supply-side "workload-binary tier." It is **deliberately not responsible** for
auth, billing, payment validation, or tenant identity — those are handled upstream by the
[capability broker](livepeer-network-modules.md) and the
[clearinghouse](livepeer-open-clearinghouse.md).

## The runners

| Runner | Lang | Endpoint(s) | Backend | Canonical capability | Work units | Device |
| --- | --- | --- | --- | --- | --- | --- |
| openai-chat-runner | Go | `POST /v1/chat/completions` | vLLM / Ollama | `openai-chat-completions` | `X-Livepeer-Work-Units` (streaming **trailer**) | CPU proxy |
| openai-embeddings-runner | Go | `POST /v1/embeddings` | vLLM / Ollama | `openai-text-embeddings` | `X-Livepeer-Work-Units` header | CPU proxy |
| openai-audio-runner | Python | `POST /v1/audio/{transcriptions,translations}` | Whisper | `openai-audio-transcriptions` / `-translations` | `usage.total_tokens` body | GPU (CUDA) |
| openai-tts-runner | Python | `POST /v1/audio/speech` | Kokoro | `openai-audio-speech` | `usage.total_tokens` body | GPU |
| openai-image-generation-runner | Python | `POST /v1/images/generations` | diffusers (SDXL/RealVisXL/FLUX) | `image-generation` | `usage.total_tokens` body | GPU required |
| rerank-runner | Python | `POST /v1/rerank` | CrossEncoder (`zeroentropy/zerank-2`) | `rerank` | `usage.total_tokens` body | GPU (CPU fallback) |

Plus: **`image-model-downloader`** / **`rerank-model-downloader`** (one-shot images that
prefetch Hugging Face weights into a shared volume) and **`openai-tester`** (Node smoke
tests exercising every runner via the OpenAI SDK).

**Interaction modes:** chat is streaming (`http-stream@v0`, work units in the trailer);
the rest are single request/response. The broker assigns the actual mode per its
host-config (audio uploads are multipart on the broker side); the runner just answers HTTP.

## The broker contract (how it plugs into repo 1)

Broker = client, runner = HTTP server. The broker forwards the gateway's
method+path+body **after payment validation**, plus informational `Livepeer-Capability`
/ `Livepeer-Offering` headers (the runner may ignore them — **they are not auth tokens**).
The broker sends the runner **no** customer identity, API keys, payment envelopes, or
Authorization headers.

Every runner must implement (see `RUNNER-INVARIANTS.md`):

- `POST <capability-path>` — capability-shaped JSON; report work units (body field or
  `X-Livepeer-Work-Units` header/trailer).
- `GET /healthz` — 200 once warm, 503 while loading/faulting (broker liveness probe).
- `GET /<capability>/options` — structured options the **orch-coordinator** scrapes for
  discovery; the broker merges the returned `extra` into host-config (operator wins).
- `GET /metrics` — Prometheus, opt-in via `METRICS_ENABLED`.
- Error contract: 400 / 429 (`Retry-After`) / 503 / 500 (error shape, never a stack trace).
- **GPU fail-fast:** ML runners exit non-zero if `DEVICE=cuda` and no GPU is present.
- `EXPOSE 8080`, a `HEALTHCHECK`, structured logs (`structlog`/`slog`).

## Offering manifest

Each image bakes `/etc/runner/offering.yaml` (from `infra/offerings/`) declaring the
capability, offerings (methods, response formats, **rate-card hints** like
`unit: audio_seconds | input_chars | images | documents`), and model metadata
(HF id, defaults, voice aliases, etc.). This feeds the broker's discovery + pricing.

## Build & images

`build-images.sh` is the single build entrypoint. Two shared bases
(`SHARED-BASE-IMAGES.md`): `python-base` (CPU) and `cuda13-python-base` (GPU, CUDA 13 +
Python 3.13). Go proxies are standalone multi-arch (amd64+arm64); ML runners are
amd64-only. Build context is the repo root so images can `COPY` from `infra/offerings/`.

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Runners](../product-specs/runners.md) | All six runners + downloaders + tester — the concrete backends |
| [Orchestrators](../product-specs/orchestrators.md) | These are what the broker dispatches to; the repo implements the broker-contract server side |

## Notes & open items

- **Naming nuance:** this repo's canonical capability names are **hyphenated**
  (`openai-chat-completions`); the broker's `host-config.yaml` examples in network-modules
  used **colon** form (`openai:chat-completions`). Confirm the exact mapping the
  `Livepeer-Capability` header carries end-to-end. (Logged as a suite open item.)
- **Maturity:** all images grade **C** — built, not yet validated end-to-end against a
  running broker or on GPU hardware (CI has no GPUs). Track via the repo's
  `QUALITY_SCORE.md` / `PLANS.md`.
- **Sibling repos:** video / vtuber runners are separate runner repos (not yet onboarded).

## Source pointers

- [`README.md`](../../modules/livepeer-modules-openai-runners/README.md) ·
  [`ARCHITECTURE.md`](../../modules/livepeer-modules-openai-runners/ARCHITECTURE.md) ·
  [`RUNNERS.md`](../../modules/livepeer-modules-openai-runners/RUNNERS.md)
- [`BROKER-CONTRACT.md`](../../modules/livepeer-modules-openai-runners/BROKER-CONTRACT.md) ·
  [`RUNNER-INVARIANTS.md`](../../modules/livepeer-modules-openai-runners/RUNNER-INVARIANTS.md) ·
  [`CANONICAL-CAPABILITIES.md`](../../modules/livepeer-modules-openai-runners/CANONICAL-CAPABILITIES.md)
