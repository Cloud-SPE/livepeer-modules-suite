# livepeer-modules-transcode-runners

**Submodule:** `modules/livepeer-modules-transcode-runners/`
**Origin:** `git@github.com:Cloud-SPE/livepeer-modules-transcode-runners.git`
**Pinned revision:** `b33e32f` (documented 2026-05-26)
**Status:** 🟠 Onboarded — documented from code. transcode/abr runners mature; **live-runner
is active/design-phase** (Option B, with external broker + gateway blockers).

The **video [Runner](../product-specs/runners.md) backends** (the sibling runner repo to
[openai-runners](livepeer-modules-openai-runners.md)). Three Go HTTP runners plus shared
FFmpeg/GPU plumbing, with vendor-specific images for NVIDIA, Intel, and AMD.

Like all runners, it is **the execution layer only** — explicitly **not** the broker,
gateway, billing, end-user auth, or payment. It runs already-authorized jobs/sessions.

## The runners

| Runner | Endpoint | Capability | Offering / unit | Mode | Maturity |
| --- | --- | --- | --- | --- | --- |
| transcode-runner | `POST /v1/video/transcode` (+ `/status`, `/presets`) | `video-transcode` | `transcode-vod` / `video_seconds` | VOD submit→poll | Shipped |
| abr-runner | `POST /v1/video/transcode/abr` (+ `/status`, `/presets`) | `video-transcode-abr` | `transcode-abr` / `video_seconds` | VOD submit→poll (ABR ladder) | Shipped |
| live-runner | `POST /v1/video/live/sessions` (+ GET/DELETE) | `livepeer:transcode/live-rtmp-hls-abr` | usage unit `output_seconds` | `live-session-gateway-ingest@v0` | Active/design |

Each ships as `-nvidia` / `-intel` / `-amd` variants. Plus `transcode-tester` (Node smoke
harness). Shared logic in `transcode-core/`.

- **VOD (transcode/abr):** async submit-poll — `POST` returns `202` + `job_id`; status is
  polled via `POST .../status` (not GET). Caller supplies input/output URLs. Optional:
  subtitle burn-in, watermark overlay, thumbnail extraction, HDR→SDR tone mapping, and
  per-job `webhook_url` callbacks.
- **live-runner:** broker-facing session runtime — shared RTMP ingest on one port, FFmpeg
  live HLS packaging, HLS pushed to **caller-supplied S3-compatible storage** (it does not
  serve playback locally).

## live-runner ↔ broker contract ("Option B")

Topology: `client → transcode-gateway → broker/payment → live-runner`. The broker owns
session authority + payment; the runner owns the media runtime. Four explicit IDs are
tracked: `gateway_session_id`, `broker_session_id`, `runner_session_id`, `work_id`.

- **Session create** (`POST /v1/video/live/sessions`): broker sends `broker_session_id`,
  `work_id`, `capability_id`, `offering_id`, `session_params` (ladder, idle timeout),
  `broker_callbacks` (event_url + auth), **`output_credential`** (S3 endpoint/bucket/
  prefix/temp creds), and **`ingest_accept.stream_key`**. Runner returns
  `runner_session_id`, `state`, and a **`private_ingest_url`**
  (`rtmp://host:1935/live/{stream_key}`). The stream key is issued once and never
  returned again.
- **Runner → broker events** (POSTed to `broker_callbacks.event_url`, Bearer-auth):
  `session.ready` / `started` / `heartbeat` / `usage.tick` (cumulative `output_seconds`) /
  `failed` / `ended`, each with a monotonic `sequence` and unique `event_id` for
  idempotency. **This is how live work is metered.**
- **Terminate** (`DELETE …`) with an optional `reason` (e.g. `insufficient_balance`).

This connects directly to the [clearinghouse](livepeer-open-clearinghouse.md) `sessions`
domain, which already names `live-session-gateway-ingest@v0` as a supported mode. The full
live path is **clearinghouse (credit + session) → broker (authority + payment) →
live-runner (media)**. See
[`LIVE-OPTION-B-INTERFACE-SPEC.md`](../../modules/livepeer-modules-transcode-runners/LIVE-OPTION-B-INTERFACE-SPEC.md).

## transcode-core (shared)

GPU detection, FFmpeg/ffprobe command construction, preset parsing/validation
(`Preset` + `ABRPreset`), HLS generation, FFmpeg progress parsing, download/upload helpers,
and filter-graph construction for subtitle burn-in, watermark, thumbnail, and HDR→SDR tone
mapping.

## Hardware & strict GPU mode

- **NVIDIA** NVENC/NVDEC (CUDA 13), **Intel** QSV/VAAPI, **AMD** VAAPI — vendor-specific
  runtime images and hardware-filtered presets.
- **Strict GPU mode is the default**: jobs **fail closed** rather than silently using
  CPU. Presets are filtered at startup against the *actually usable* decode+encode path
  (visible hardware isn't enough). CPU-side features (subtitle/watermark/thumbnail) are
  rejected under strict mode.
- Build note: FFmpeg built without `libnpp` on CUDA 13.2.1 (upstream API incompatibility).

## Presets & offerings

- `infra/presets/` — operator-editable packs (`transcode.yaml`, `abr.yaml`, `live.yaml`,
  plus tuned packs like `nvidia-gtx1080-*`). Selected via `PRESETS_FILE`.
- `infra/offerings/` — fixed offering manifests (capability + offerings + `rate_card_hints`)
  the broker reads for discovery/pricing.

## Capability mapping (repo → suite capabilities)

| Suite capability | Where it lives in this repo |
| --- | --- |
| [Runners](../product-specs/runners.md) | All three runners — the video execution tier |
| [Orchestrators](../product-specs/orchestrators.md) | Backends the capability-broker dispatches to (VOD HTTP + live session contract) |
| [Gateways](../product-specs/gateways.md) | live-runner participates in the live "Option B" gateway→broker→runner path |

## Notes & open items

- **Naming variety:** capabilities here use hyphen form (`video-transcode`,
  `video-transcode-abr`) and the live capability uses colon+slash
  (`livepeer:transcode/live-rtmp-hls-abr`) — more inputs to the suite-wide capability-name
  reconciliation (TD-7).
- **live-runner is not done:** exec-plan `0001-live-option-b-remote-runner` is in design;
  external blockers are broker support for a remote live backend and gateway migration.
- **vtuber runners** remain a separate, not-yet-onboarded repo.

## Source pointers

- [`README.md`](../../modules/livepeer-modules-transcode-runners/README.md) ·
  [`DESIGN.md`](../../modules/livepeer-modules-transcode-runners/DESIGN.md) ·
  [`API.md`](../../modules/livepeer-modules-transcode-runners/API.md)
- [`OPERATIONS.md`](../../modules/livepeer-modules-transcode-runners/OPERATIONS.md) ·
  [`BUILD.md`](../../modules/livepeer-modules-transcode-runners/BUILD.md) ·
  [`LIVE-OPTION-B-INTERFACE-SPEC.md`](../../modules/livepeer-modules-transcode-runners/LIVE-OPTION-B-INTERFACE-SPEC.md)
