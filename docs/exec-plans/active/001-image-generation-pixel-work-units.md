# 001. Pixel-Based Work Units for Image Generation

**Status:** draft — decisions locked, ready for review
**Owner:** Mike Zupper (drafted with Claude)
**Opened:** 2026-08-02
**Closed:** —

## Intent

Re-denominate image-generation billing from "number of images" to
"output pixels, priced per megapixel," so pool pricing tracks how the
market actually charges for image generation and how much compute a
job really costs.

Today `openai:images-generations` meters one work unit per generated
image (`request-formula` on `$.n`). No major provider bills that way
anymore: OpenAI bills image-output tokens that scale with resolution
and quality (a 1024×1024 image spans $0.011 → $0.167 across quality
tiers — 15× at identical image count), fal.ai / Black Forest Labs
bill per megapixel, and Replicate bills GPU-seconds. All three reduce
to **pixels × compute intensity**. A flat per-image unit under-charges
large/high-step jobs and over-charges small drafts by up to an order
of magnitude, and misprices orchestrator payouts accordingly.

Bonus alignment: go-livepeer's historical AI pricing is wei-per-pixel,
and the clearinghouse proto's `PriceInfo.pixels_per_unit` field name
is literally this heritage. Pixels put image gen back on the same
footing as transcoding (`frame_megapixel`).

## Scope

- In: work-unit definition + extractor changes (protocol spec,
  capability-broker), runner work-unit reporting and steps clamping
  (openai-image-generation-runner), gateway estimate/commit path
  (livepeer-modules-openai-gateway), example configs, payment-daemon
  ticket-sizing scenario, price calibration, doc fixes.
- Out: quality tiers as a payment-bearing field (deferred — see D3),
  image-to-image / edits endpoints (same unit will apply when those
  land), video generation (separate plan; will likely want
  pixel-frames), migrating live pool configs (ops task once this
  ships).

## Current state (file references)

| What | Where |
|---|---|
| Per-image work unit (production example) | `livepeer-network-modules/capability-broker/examples/host-config.example.yaml:393-411` — `expression: "n"`, 4.6 Twei/image |
| Extractor implementation | `capability-broker/internal/extractors/requestformula/extractor.go` — `Extract()` floors to uint64; flat `$.foo` numeric fields only |
| Gateway estimate + commit | `livepeer-modules-openai-gateway/gateway/src/proxy/images.ts:55,68,81` — estimates `n`, commits the estimate (runner reports nothing) |
| Money math | `capability-broker/internal/server/middleware/settlement.go:83` — `price_per_unit × actualUnits` |
| Area-based unit, spec'd but unwired | `livepeer-network-protocol/extractors/request-formula.md:22-33` — `image_step_megapixel = (width × height × steps) / 1e6`, with conformance fixtures |
| Trailer-based runner reporting | `capability-broker/internal/extractors/responsetrailer/` — reads `X-Livepeer-Work-Units` from HTTP **trailers only** |
| Runner request/response models | `livepeer-modules-openai-runners/openai-image-generation-runner/src/openai_image_generation_runner/app.py:51-72` |
| Payment-daemon ticket sizing | `livepeer-network-modules/payment-daemon/scenarios/openai-multi-offering-ticket-sizing.yaml:110-127` — FLUX offering, 18.75 Twei/image |

Two latent problems the audit surfaced, both fixed by this plan:

1. **Unbilled compute variance.** The runner accepts a
   `num_inference_steps` override (`app.py:58,114`) and doubles steps
   on `quality: "hd"` (`app.py:121-122`). A client can buy 2–10× the
   compute for the same price today. Note `num_inference_steps` is a
   diffusers-style extension — the OpenAI Images API has **no** steps
   parameter (surface: `model, prompt, n, size, quality, background,
   output_format, style, user`), so no OpenAI-compatible SDK client
   ever sends it. Removing it breaks nobody.
2. **Doc drift.** `docs/repos/livepeer-modules-openai-runners.md:27`
   claims the image runner reports units via `usage.total_tokens`; the
   runner's response model is `{created, data}` with no usage object
   and no work-units header. Metering is entirely request-side today.

## Decisions

### D1 — Unit: pixels, priced per megapixel

`work_unit.name: "pixels"`, expression `n * width * height`, price
block `per_units: 1000000` with `amount_wei` set per megapixel.

Raw pixels — not megapixels — as the unit, because the extractor
floors to uint64: a megapixel-denominated expression would bill a
512×512 image (0.26 MP) as **0 units**. Pixels-as-units with a per-MP
price avoids truncation entirely and matches the wei-per-pixel
convention upstream.

```yaml
work_unit:
  name: "pixels"
  extractor:
    type: "request-formula"
    expression: "n * width * height"
    fields:
      n: "$.n"
      width: "$.size#width"     # syntax TBD in Phase 1 spec work
      height: "$.size#height"
    defaults: { n: 1, width: 1024, height: 1024 }
price:
  amount_wei: "<per-MP price, see Pricing>"
  per_units: 1000000
```

### D2 — Metering: request-side estimate, runner-reported settle

Same pattern LLM chat uses (`prompt_tokens + max_tokens` budget up
front, `usage.total_tokens` at commit):

- **Reserve:** gateway parses `size` (default 1024×1024) and computes
  `n × width × height` as `estimatedWorkUnits`.
- **Settle:** runner returns actual units (`n × actual_width ×
  actual_height`) in `X-Livepeer-Work-Units`; broker extracts it and
  the settlement record carries the actual.
- **Reconcile:** daemon-recorded actuals win (consistent with the
  long-running-sessions plan's reconciliation table).

For image gen the estimate and actual agree whenever the client sends
an explicit `size`; the runner-reported path resolves the case where
`size` is omitted and the runner's env defaults (unknown to the
gateway) apply. Because dimensions are fully request-determined, the
gateway SHOULD clamp the committed actual to the request-derived
maximum — a runner cannot inflate its payout past what the request
itself justifies.

### D3 — Quality and steps: deferred, not payment-bearing

Compute-per-pixel must be constant within an offering for D1 to be
honest, so the runner pins it:

- Drop the `quality: "hd"` → 2× steps behavior; log-and-ignore any
  `quality` value (OpenAI SDK clients send `low/medium/high/auto`;
  rejecting `auto` — the SDK default — would break them).
- Clamp `num_inference_steps` to the model's configured step count
  (env-configurable ceiling), or drop the field entirely.

Why not bill quality/steps: the concept is not universal. Lightning /
schnell-class models run a fixed 4–8 steps and ignore quality; future
non-diffusion models have no steps at all. Any protocol-level
quality→steps mapping would be fiction for half the catalog. Per-model
compute differences already have a home — the per-offering
`amount_wei`. When a genuine quality tier is wanted later, expose it
as a second offering (e.g. `realvisxl-hd`) with its own per-MP price.
No protocol change needed, ever.

### D4 — Price: benchmark to market, per megapixel

Anchor to the hosted-aggregator tier for comparable open-weight
models (fal/BFL FLUX-dev class: ~$0.025–0.075 per MP; hosted
open-weight images generally $0.008–0.04 each at ~1 MP), convert to
wei at current ETH spot, and set per-offering prices. For reference,
the old prices convert neutrally at the 1024×1024 (≈1.05 MP)
baseline: 4.6 Twei/image ≈ 4.4 Twei/MP (RealVisXL Lightning),
18.75 Twei/image ≈ 17.9 Twei/MP (FLUX). Final numbers are an open
question (Q#1) — pick during Phase 5.

### D5 — Settlement transport: extend `response-trailer` with a header fallback

The existing `response-trailer` extractor reads `X-Livepeer-Work-Units`
from HTTP **trailers**, which suits the streaming mode but not
`http-reqresp` — and emitting trailers from FastAPI is awkward. Extend
the extractor (spec + implementation) to fall back to a plain response
header of the same name when no trailer is present. This keeps the
extractor library at its deliberately small size instead of adding a
seventh recipe, and matches `BROKER-CONTRACT.md:26-33`, which already
promises "header or trailer."

## Work plan

Phases 1–2 are the critical path; 3 and 4 parallelize after 2.

### Phase 1 — Protocol spec (`livepeer-network-protocol`)

- [ ] `extractors/request-formula.md`: add derived-field syntax for
      parsing `"1024x1024"`-style size strings into numeric
      width/height with per-field defaults. Replace the
      `image_step_megapixel` example with the pixels unit (or keep it
      as a non-normative variant with a note on why steps were
      dropped).
- [ ] `extractors/response-trailer.md` (or equivalent): specify the
      header fallback (trailer wins if both present; `default` when
      neither parses).
- [ ] Conformance fixtures: pixel expression with explicit size,
      missing size (defaults), multi-image `n`, malformed size
      string; header-fallback settlement case.
- [ ] Document the reserve/settle/reconcile flow for atomic jobs with
      runner-reported actuals (shape *a/b* hybrid in the
      long-running-sessions taxonomy).

**Acceptance:** conformance suite green against test-broker-config;
spec review sign-off.

### Phase 2 — Broker (`capability-broker`)

- [ ] `requestformula`: implement size-string derived fields per
      Phase 1 spec (`lookupAndCoerce` today only handles flat numeric
      paths).
- [ ] `responsetrailer`: implement the header fallback.
- [ ] `examples/host-config.example.yaml`: rewrite the
      `openai:images-generations` capability — settlement extractor
      `response-trailer` (header fallback), `per_units: 1000000`,
      placeholder per-MP price with a comment pointing at Phase 5.
- [ ] Config validation: ensure `per_units > 1` paths are exercised
      in tests (settlement math divides… verify `per_units` handling
      in `settlement.go` — today's configs all use `per_units: 1`).

**Acceptance:** unit tests for both extractors; an end-to-end broker
test where a mock runner returns the header and `BilledUnits` equals
`n × w × h`; conformance suite green.

### Phase 3 — Runner (`openai-image-generation-runner`)

- [ ] Set `X-Livepeer-Work-Units: n × actual_width × actual_height`
      on the generation response.
- [ ] Clamp/remove `num_inference_steps`; log-and-ignore `quality`;
      delete the `hd` doubling.
- [ ] Update `BROKER-CONTRACT.md` if the header semantics need any
      image-specific note.

**Acceptance:** runner tests assert the header value for explicit
size, default size, and `n > 1`; steps cannot exceed the configured
ceiling regardless of request contents.

### Phase 4 — Gateway (`livepeer-modules-openai-gateway`)

- [ ] `gateway/src/proxy/images.ts`: parse `size` → estimate
      `n × width × height` (default 1024×1024); commit the
      runner-reported actual from the response header when present,
      clamped to the request-derived maximum; fall back to the
      estimate otherwise.
- [ ] Update the file-header comment (`images.ts:2`) describing the
      work unit.

**Acceptance:** gateway tests for estimate parsing, actual-wins
commit, clamp behavior, and fallback.

### Phase 5 — Pricing & configs

- [ ] Resolve Q#1 (market anchor + ETH conversion) and set real
      per-MP prices in `host-config.example.yaml` and
      `payment-daemon/scenarios/openai-multi-offering-ticket-sizing.yaml`
      (ticket sizing must be re-checked: units per request jump from
      ~1 to ~1,000,000 raw pixels — confirm no overflow/fee-floor
      surprises in ticket parameters).

### Phase 6 — Docs

- [ ] Fix `docs/repos/livepeer-modules-openai-runners.md:27`
      (`usage.total_tokens` claim is false today and stays false —
      the runner reports via header).
- [ ] `docs/glossary.md:50-57`: add `pixels` to the work-unit
      examples.
- [ ] `docs/product-specs/payment.md`: note the per-MP `per_units`
      pattern.

## Open questions

- **Q#1 (pricing):** exact market anchor and ETH conversion cadence.
  Options: one-time manual peg vs. periodic re-peg runbook. Owner
  decision at Phase 5.
- **Q#2 (broker-side verification):** should the broker *also* run
  the request-formula pixel expression as a sanity cap on the
  runner-reported value (defense in depth on the orchestrator side,
  mirroring the gateway clamp)? Cheap to add once Phase 2's derived
  fields exist; decide during Phase 2 review.
- **Q#3 (image-to-image / edits):** same unit should apply, but the
  request shape differs (multipart, input image dimensions). Out of
  scope here; note for the capability that adds those endpoints.

## Rejected alternatives

- **Per-image (status quo):** up to ~15× mispricing across
  resolutions; no provider prices this way anymore.
- **Megapixel-steps (`image_step_megapixel`):** most faithful to
  compute cost, but steps aren't universal across models (fixed-step
  distilled models, future non-diffusion), and pinning steps per
  offering (D3) makes the extra dimension redundant. The spec'd
  formula also floor-truncates sub-MP jobs to 0 units as written.
- **GPU-seconds (Replicate model):** unverifiable by the gateway; a
  slow or dishonest orchestrator inflates its own payout. Wrong trust
  model for a permissionless pool.
- **Quality multiplier in the gateway:** makes a quality→factor
  mapping payment-bearing logic that gateway and broker must keep in
  lock-step, for a field half the models ignore. Separate offerings
  achieve the same with zero protocol surface.
