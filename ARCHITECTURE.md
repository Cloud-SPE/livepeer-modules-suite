# ARCHITECTURE.md

Top-level map of the Livepeer Modules Suite: what the domains are, how a unit of work
flows through them, and where the boundaries sit. This is a bird's-eye view — each
module's detail lives in [`docs/product-specs/`](docs/product-specs/index.md).

> **Status.** Confirmed against three onboarded repos: the supply side
> ([livepeer-network-modules](docs/repos/livepeer-network-modules.md)), the demand-side
> control plane ([livepeer-open-clearinghouse](docs/repos/livepeer-open-clearinghouse.md)),
> and concrete AI Runner backends
> ([livepeer-modules-openai-runners](docs/repos/livepeer-modules-openai-runners.md)). A
> standalone full **Gateway shell**, **video/vtuber runners**, and **reference apps** are
> not yet onboarded, so those parts remain provisional.

## The big picture

The Livepeer Modules are the productized features that let participants use the
Livepeer protocol to get work done (video transcoding, AI inference, and related
media compute). Conceptually they divide into four concerns:

1. **Demand side** — *Gateways* and the *Reference Apps* built on top of them submit
   work and pay for it.
2. **Supply side** — *Orchestrators*, optionally fronted by *Pools*, perform the work
   and earn payment.
3. **Coordination** — *Service Registry* and *Discover* let the demand side find and
   select the right supply.
4. **Settlement** — *Payment* moves value per unit of work; the *Payment
   Clearinghouse* settles and accounts for it. *SDKs* cut across all of the above.

## How a unit of work flows

```text
          ┌──────────────────────────────────────────────────────────────┐
          │                        Reference Apps                         │
          │            (example apps built on a Gateway / SDKs)           │
          └───────────────────────────────┬──────────────────────────────┘
                                           │ uses
                                           ▼
   ┌──────────────┐   discover/select   ┌───────────┐   register/advertise   ┌──────────────────┐
   │   Discover    │◀───────────────────│  Gateway   │───────────────────────▶│ Service Registry │
   │ (selection)   │                    │ (demand)   │◀──────────────────────  │  (capabilities)  │
   └──────────────┘                     └─────┬─────┘    looks up endpoints    └────────┬─────────┘
                                              │ submit work + payment                    │ advertise
                                              ▼                                          ▼
                                       ┌──────────────┐    fronts capacity      ┌──────────────────┐
                                       │ Orchestrator │◀───────────────────────│       Pools       │
                                       │  (supply)    │                         │ (capacity aggreg.)│
                                       └──────┬───────┘                         └──────────────────┘
                                              │ work done → payment tickets
                                              ▼
                                       ┌──────────────┐    settle/account       ┌──────────────────┐
                                       │   Payment    │───────────────────────▶│ Payment           │
                                       │ (micropay)   │                         │ Clearinghouse     │
                                       └──────────────┘                         │   (+ SDKs)        │
                                                                                └──────────────────┘
```

Narrative (provisional):

1. **Service Registry** — Orchestrators advertise where they are and what they can do
   (endpoints, supported capabilities, capacity).
2. **Discover** — A Gateway queries discovery to find candidate orchestrators and
   selects among them (by capability, price, latency, reliability).
3. **Gateway** — The demand-side entry point. It accepts work from apps/users, opens
   sessions with selected orchestrators, streams work to them, and sends payment.
4. **Pools** — Optionally aggregate many orchestrators/workers behind one logical
   endpoint, so a Gateway sees pooled capacity rather than individual nodes.
5. **Orchestrator** — Performs the work and returns results, exchanging it for
   **Payment**.
6. **Payment** — Off-chain, per-unit micropayments flow from Gateway to Orchestrator
   as work is performed.
7. **Payment Clearinghouse** — Settles, reconciles, and accounts for those payments,
   exposing **SDKs** for integration.
8. **Reference Apps** — Demonstrate the whole loop end to end on top of a Gateway.

## Supply side, grounded

The first repo confirms the supply-side shape. The orchestrator is **not a worker
binary** — it's a **workload-agnostic capability broker** (one per host) plus payment,
publishing, and trust daemons. The broker:

- reads one declarative `host-config.yaml`,
- routes inbound paid requests by `Livepeer-Capability` header to a declared **Runner**
  (the backend) wrapped in a fixed **interaction mode** (`http-reqresp`, `http-stream`,
  `ws-realtime`, `rtmp-ingress-hls-egress`, `session-control-plus-media`, …),
- and reports usage to the payment daemon. **New capability under an existing mode = a
  YAML edit, no code.**

Discovery is **hybrid**: an on-chain pointer (`ServiceRegistry`/`AIServiceRegistry`
`getServiceURI`) → an off-chain, **cold-key-signed manifest** of capability tuples.
A firewalled `secure-orch` holds the cold key and signs manifests; the public
`orch-coordinator` only scrapes and publishes; every resolver re-verifies the signature.
Payment is probabilistic micropayment **tickets** settled via the on-chain `TicketBroker`.

Concrete runners now exist: the
[openai-runners](docs/repos/livepeer-modules-openai-runners.md) repo ships
OpenAI/Cohere-shaped AI backends (chat, embeddings, audio, TTS, image, rerank) that
implement the broker↔runner HTTP contract and report work units back to the broker.

See [`docs/repos/livepeer-network-modules.md`](docs/repos/livepeer-network-modules.md)
for the component map and the [glossary](docs/glossary.md) for terms.

## Demand side, grounded

The [Payment Clearinghouse](docs/repos/livepeer-open-clearinghouse.md) is how app
developers reach the network without managing wallets or keys. It is **non-custodial at
the user boundary**: developers hold a wei credit balance; an operator-owned **pooled
wallet** (held only by `payment-daemon`) signs every ticket.

It operates in **handoff mode** — it is the control plane, not the data plane:

1. The customer's **SDK** calls the clearinghouse to open a **job** (one-shot) or
   **session** (long-lived, refillable).
2. The clearinghouse resolves a route via `service-registry-daemon`, checks the credit
   balance, mints a ticket via `payment-daemon.CreatePayment`, and **encumbers expected
   value at issuance**.
3. It returns the signed envelope; the **SDK talks to the orchestrator broker directly**
   and reports actual usage back for settle-time reconciliation.

So the gateway role is split: **control plane** (auth, credit, discovery proxy, mint) in
the clearinghouse; **data plane** (interaction-mode transport, `Livepeer-Payment`) in the
SDK. The clearinghouse **consumes the supply-side daemons** (`payment-daemon`,
`service-registry-daemon`) over Unix-socket gRPC — the first concrete cross-repo
dependency in the suite.

## Domains & boundaries

Each module is its own repository (its own deploy/release unit) mounted under
`modules/<name>/`. The suite favors **explicit, narrow interfaces between modules**:

- The Gateway depends on Discover and the Registry for *coordination*, and on Payment
  for *settlement* — not on orchestrator internals.
- Orchestrators and Pools are interchangeable from the Gateway's perspective: a Pool
  presents an orchestrator-shaped interface.
- SDKs are the *only* sanctioned client surface for the Clearinghouse (and likely
  Gateway) APIs; apps should consume SDKs rather than reimplement protocol details.

These boundaries are the thing to protect as the suite evolves. Where they are
enforced mechanically (lints, contract tests, schema validation) will be documented
per module as repos are onboarded.

## Where things live

| Concern | Where to look |
| --- | --- |
| Per-module overview & status | [`docs/product-specs/`](docs/product-specs/index.md) |
| Operating principles | [`docs/design-docs/core-beliefs.md`](docs/design-docs/core-beliefs.md) |
| Submodule workflow / release tracking | [`docs/guides/git-submodules-primer.md`](docs/guides/git-submodules-primer.md) |
| The actual code | `modules/<name>/` (git submodules) |
