# Glossary

Shared vocabulary for the Livepeer Modules Suite. Definitions are grounded in what the
onboarded repositories actually say; terms that aren't yet confirmed against code are
marked _(draft)_. Keep this current as repos are onboarded — it is the fastest way for a
new reader (human or agent) to get oriented.

Sources so far: [`livepeer-network-modules`](repos/livepeer-network-modules.md).

## Suite & roles

- **Livepeer Modules (suite)** — this umbrella repository: the productized features that
  enable the Livepeer protocol, aggregated as submodules with a high-level overview.
- **Livepeer Network Modules** — the first onboarded repo; the workload-agnostic
  **supply-side** rearchitecture (the first abstraction layer over the Livepeer smart
  contracts). See [repos/livepeer-network-modules.md](repos/livepeer-network-modules.md).
- **Orchestrator (orch)** — a supply-side participant that advertises capabilities,
  accepts paid work, and is identified on-chain by an Ethereum address. In the new
  architecture an orchestrator is a *set of processes* (broker + daemons + trust spine),
  not a single worker binary.
- **Gateway** — the demand-side entry point that discovers orchestrators, selects a
  route, attaches payment, and forwards customer traffic. The gateway *shell* lives in
  other repos; this repo defines the broker/payment/discovery contracts it talks to.
- **Runner** — the **backend that actually executes a workload** (e.g. vLLM, FFmpeg, a
  third-party API, a session runtime). Runners are not daemons in this repo; the broker
  dispatches to them over standard wires (HTTP/WS/RTMP/subprocess) declared in config.
  See the dedicated note in [product-specs/runners.md](product-specs/runners.md).

## Capability broker & metering

- **Capability broker** — one workload-agnostic process per orchestrator host. Owns
  `GET /registry/offerings`, routes inbound paid requests by header to a declared
  **backend**, wraps the call in the declared **interaction mode**, and reports usage to
  the payment daemon. Carries zero per-capability code.
- **host-config.yaml** — the single declarative file a broker reads: identity,
  capabilities (id, mode, work unit, extractor, price, health probe), and backends.
  Adding a capability under an existing mode is a YAML edit, not a code change.
- **Capability** — a unit of service identified by an opaque id (e.g.
  `openai:chat-completions`). Carried on the wire in the `Livepeer-Capability` header.
- **Offering** — a priced/identified tier within a capability (opaque id, e.g.
  `default`). Carried in the `Livepeer-Offering` header.
- **Capability tuple** — the atomic registry/resolver unit:
  `(capability_id, offering_id, interaction_mode, work_unit_name, price_per_unit_wei, worker_url, eth_address, extra, constraints)`.
  **The host is not a registration unit** — tuples are.
- **Interaction mode** — a fixed wire contract a capability picks. Implemented once in
  the broker and once on the gateway side, then reused across all capabilities. Modes are
  **specifications, not a code library**. Current set: `http-reqresp`, `http-stream`,
  `http-multipart`, `ws-realtime`, `rtmp-ingress-hls-egress`,
  `session-control-plus-media`, `live-session-remote-runner`.
- **Work unit** — the opaque metering dimension for a capability (`tokens`, `frames`,
  `pixel-seconds`, `barks`, …). The payment daemon does pure arithmetic on it.
- **actualUnits** — the count the broker reports after a backend response; price =
  `price_per_unit_wei × actualUnits`.
- **Extractor** — a declarative recipe that turns a request/response into `actualUnits`.
  Fixed library: `openai-usage`, `response-jsonpath`, `request-formula`,
  `bytes-counted`, `seconds-elapsed`, `ffmpeg-progress`.
- **Probe recipe** — a declarative health check a capability selects; the broker runs it
  and normalizes the result to shared outward states: `ready`, `draining`, `degraded`,
  `unreachable`, `stale`.

## Wire headers

- **`Livepeer-Capability`** — opaque capability id on the request.
- **`Livepeer-Offering`** — opaque offering id on the request.
- **`Livepeer-Payment`** — the wire-format payment envelope (ticket + proof), kept
  backward-compatible with the existing protocol.

## Payment & chain

- **payment-daemon** — long-lived sidecar with a **sender** mode (gateway side: mints
  payment envelopes) and a **receiver** mode (worker side: validates tickets, tracks
  per-sender balances, redeems winning tickets on-chain). Talks to the broker over a unix
  socket; capability/work-unit names are opaque strings.
- **Ticket** — a probabilistic micropayment signed by the sender, with a face value and
  win probability. Only **winning** tickets settle on-chain; others are expected-value
  credit.
- **Expected value (EV)** — `face_value × win_prob`, credited to the off-chain balance
  per ticket before any on-chain redemption.
- **Per-request vs streaming payment** — request modes use one ticket per request with a
  usage true-up; streaming/session modes amortize via `OpenSession` → periodic `Debit` /
  top-up → `CloseSession`. **Worker meters, gateway ledgers.**
- **TicketBroker** — on-chain (Arbitrum One) contract that redeems winning tickets and
  pays the orchestrator's recipient (cold-key) address.
- **BondingManager / RoundsManager** — on-chain contracts driving stake/reward
  eligibility and the round cadence (~19h on Arbitrum One).
- **ServiceRegistry / AIServiceRegistry** — on-chain contracts mapping an orchestrator
  address → `serviceURI` (the off-chain manifest URL). Only a pointer lives on-chain.

## Discovery & trust

- **Manifest** — a cold-key-signed, off-chain JSON document listing an orchestrator's
  capability tuples, published at a well-known URL
  (`/.well-known/livepeer-registry.json`) that the on-chain `serviceURI` points to.
- **orch-coordinator** — public, key-less host process that scrapes broker offerings,
  builds a *candidate* manifest, and atomic-swap publishes the signed manifest. Does not
  hold keys.
- **secure-orch / secure-orch-console** — the firewalled host holding the **cold key**;
  the console renders a diff of candidate-vs-published manifest and signs it. Accepts
  **zero** inbound connections from outside the LAN.
- **Cold key** — the orchestrator's private key, HSM-backed, on the firewalled host. It
  signs canonical manifest bytes (and, via protocol-daemon, on-chain round/reward txs) —
  **never naked transactions or tickets**, and it never leaves the host.
- **protocol-daemon** — handles on-chain orchestrator duties: round init, reward calls,
  and `serviceURI` writes. Built on `chain-commons`.
- **service-registry-daemon** — decoupled discovery daemon with a **publisher** mode
  (operator side) and a **resolver** mode (gateway side). The resolver reads the on-chain
  pointer, fetches the manifest, **re-verifies the signature** (defense in depth), and
  caches flattened tuples.
- **Resolver / `Resolver.Select`** — the discovery API (a gRPC call in the resolver, not
  a separate service): given `(capability_id, offering_id?, tier?, min_weight?)` it
  returns a route tuple including `interaction_mode`. This is what "Discover" is.
- **Double verification** — the coordinator verifies the manifest signature on upload and
  every resolver verifies it again on fetch, so a compromised coordinator can't propagate
  tampered manifests.
- **chain-commons** — shared Go library for chain/RPC/transaction-intent plumbing
  (multi-RPC failover, durable tx state machine, reorg-aware confirmation, keystore
  signing) used by the daemons.

## Pools

- **Pool** — a control plane that aggregates multiple member backends behind one orch
  identity. From the gateway's view a Pool looks like one orchestrator (one manifest, one
  broker endpoint); the pooling machinery is internal. _(This is capacity + accounting
  pooling, distinct from classic on-chain stake delegation.)_
- **pool-controller** — owns persisted Pool state (members, backends, offers,
  assignments), renders broker config, ingests work receipts, and runs backend-selection
  scoring (cooldown, EMA, latency, warm-up).
- **pool-reconciler** — closes rounds using protocol-daemon timing, payment-daemon
  realized revenue, and pool-controller work receipts.
- **pool-payout-executor** — executes native-ETH member payouts on Arbitrum and writes
  back payout state.
- **Work receipt / round receipt / payout intent** — the accounting chain: per-request
  receipts → aggregated per-round receipts → per-member payable intents
  (`pending → exported → leased → submitted → paid` / `failed`).

## Protocol & platform

- **livepeer-network-protocol** — the spec repo (modes, extractors, manifest schema,
  payment/sessionrunner protos, conformance tooling). A specification surface, not a
  shared runtime dependency.
- **proto-contracts** — generated protobuf bindings shared by the daemon surfaces
  (`livepeer/payments/v1`, `livepeer/registry/v1`, `livepeer/protocol/v1`).
- **customer-portal** — a shared TS/SaaS library (API keys, customer ledger, Stripe
  top-ups, admin UI widgets) consumed via the pnpm workspace; relates to
  [SDKs](product-specs/sdks.md).
- **JCS (JSON Canonical Serialization)** — deterministic JSON encoding used so manifest
  and payment signatures verify regardless of field order.
