# Pool deployment runbook (public supply onboarding)

How to stand up a **public Pool orchestrator** — a supply-side deployment that onboards
third-party ("public") compute behind a single orchestrator identity, with round-based
accounting and automatic native-ETH member payouts.

This runbook is grounded in
[`livepeer-network-modules`](../repos/livepeer-network-modules.md) (`689b51a`, images
`v1.4.1`) and the `infra/scenarios/pool-orchestrator/` scenario. It assumes **Phase 1 =
AI inference** (OpenAI-shaped runners); the video path is additive (see the last section).

For the vocabulary (Pool, offer, capability tuple, work receipt, payout intent) see the
[glossary](../glossary.md#pools) and the [Pools spec](../product-specs/pools.md). The
member-facing companion to this operator guide is
[`pool-member-onboarding.md`](./pool-member-onboarding.md).

> **What "Pool" means here:** capacity + accounting aggregation, **not** on-chain stake
> delegation. From a gateway's view the Pool is one orchestrator (one on-chain address,
> one signed manifest, one broker endpoint). Who actually ran the work and who gets paid
> stays inside your control plane.

## Topology: two hosts, two trust zones

```text
┌─ SECURE-ORCH / PROTOCOL HOST (reuse your existing orchestrator's) ─┐
│  protocol-daemon         round init, reward, serviceURI writes
│  service-registry-daemon manifest publisher
│  secure-orch-console     holds the COLD ORCH KEY; signs manifests
│  → zero inbound from outside the LAN
└────────────────────────────────────────────────────────────────────┘
             │ exposes /var/run/livepeer/protocol.sock to ▼
┌─ PUBLIC / DATA-PLANE HOST (the net-new stack) ─────────────────────┐
│  pool-controller         control plane: members, offers, payouts
│  capability-broker       routes paid work to member backends
│  payment-daemon (recv)   validates tickets, redeems on-chain
│  orch-coordinator        scrapes broker, publishes signed manifest
│  pool-reconciler         closes rounds → round receipts
│  pool-payout-executor    pays members in native ETH on Arbitrum
└────────────────────────────────────────────────────────────────────┘
             ▲ outbound QUIC / WebSocket worker sessions
┌─ MEMBER HOSTS (the public supply — as many as you want) ───────────┐
│  pool-member-agent + runner (delivered as a generated bundle)
└────────────────────────────────────────────────────────────────────┘
```

If you already operate an orchestrator, **you already have the secure-orch side**. The
Pool work is the public data-plane stack plus one cross-host wire: `protocol-daemon` must
expose `/var/run/livepeer/protocol.sock` to the public host (shared bind, or a
socket-forwarding tunnel over your private link). `pool-reconciler` reads it to time round
closes.

## 1. Provision the public host

Docker + Compose, a public IP, and a DNS name (e.g. `pool-broker.example.com`).

| Port (host) | Proto | Purpose |
|---|---|---|
| 443 → broker `:8082` (container `:8080`) | TCP/TLS | gateways submit paid work |
| coordinator public `:8081` (front with TLS) | TCP/TLS | serves the signed manifest |
| **`:8443`** | **UDP** | **member worker QUIC sessions (critical for supply)** |

- Terminate **TLS** in front of the broker and coordinator public port before live
  traffic. QUIC on `8443/udp` is terminated by the broker itself.
- Keep `pool-controller` admin (`:8080`) and all `:909x` metrics ports **private** (bind
  `127.0.0.1` or a VPN). The scenario `.env` already binds broker metrics to `127.0.0.1`.

## 2. Wallets & keys (two funded hot wallets on the public host)

| Keystore env | Role | Funding |
|---|---|---|
| `PAYMENT_KEYSTORE` (+ `..._PASSWORD_FILE`) | payment-daemon **receiver** — redeems winning tickets on Arbitrum One to your orch recipient address. **Revenue in.** | ETH for redemption gas |
| `POOL_PAYOUT_EXECUTOR_KEYSTORE` (+ `..._PASSWORD_FILE`) | pays **members** their share in native ETH on Arbitrum. **Payouts out.** | ETH for member payouts + gas |

Both are V3 keystore JSON + a password file, stored outside the repo, mounted read-only.
The **cold orch key stays on the secure host** and is never present here.

## 3. Build the images

From the repo root:

```bash
./infra/scripts/build-images.sh \
  livepeer-pool-controller livepeer-pool-member-agent \
  livepeer-pool-reconciler livepeer-pool-payout-executor \
  livepeer-capability-broker livepeer-payment-daemon \
  livepeer-orch-coordinator
```

The scenario `.env` defaults to `REGISTRY=tztcloud`, `TAG=v1.4.1`.

## 4. Configuration files

Copy the scenario examples and fill them in:

```bash
cd infra/scenarios/pool-orchestrator
cp .env.example .env
cp coordinator-config.example.yaml coordinator-config.yaml
```

### `.env` — the values that matter

```bash
REGISTRY=<your-registry>          ; TAG=v1.4.1
POOL_CONTROLLER_ADMIN_TOKEN=<strong secret>
ORCH_COORDINATOR_ADMIN_TOKENS=<strong secret>
CHAIN_RPC=https://arb1.arbitrum.io/rpc
ORCH_ADDRESS=<your orch eth address>
PAYMENT_KEYSTORE=/opt/livepeer/payment-keystore.json
PAYMENT_KEYSTORE_PASSWORD_FILE=/opt/livepeer/payment-keystore-password
POOL_PAYOUT_EXECUTOR_KEYSTORE=/opt/livepeer/payout-keystore.json
POOL_PAYOUT_EXECUTOR_KEYSTORE_PASSWORD_FILE=/opt/livepeer/payout-keystore-password
LIVEPEER_SOCKET_DIR=/var/run/livepeer
# host ports: broker 8082, broker QUIC 8443/udp, coordinator public 8081, controller 8080
```

### `pool-controller.yaml` — identity, automation, member-bundle URLs

```yaml
identity:
  orch_eth_address: "<your orch address>"
  label: <pool-label>

admin_auth:
  bearer_token_ref: env://POOL_CONTROLLER_ADMIN_TOKEN

listen: { paid: ":8080", metrics: ":9090" }

# --- Mostly-automated admission ---
policy:
  auto_approve_join_requests: true      # auto-approve any join whose admission preview is Approvable
  auto_drain_backends: true             # drain backends whose recent failure rate is too high
  backend_failure_rate_threshold: 0.25
  backend_min_samples: 20               # don't drain new/quiet backends on one bad sample
  evaluation_interval_ms: 15000         # policy-worker tick

synthetic_probes: { enabled: true, interval_ms: 30000, timeout_ms: 3000 }

scoring:                                # defaults are sane; tune later
  cooldown_duration_ms: 300000
  cooldown_failure_trigger: 5
  ema_half_life_ms: 86400000
  latency_target_ms: 1200

bootstrap:
  broker_admin_url: http://capability-broker:8080
  broker_admin_auth: { method: bearer, secret_ref: env://POOL_CONTROLLER_ADMIN_TOKEN }
  broker_apply_command: [ /usr/local/bin/livepeer-broker-apply ]  # stages rendered YAML for the broker
  broker_apply_timeout_ms: 30000
  # These three flow into every MEMBER BUNDLE:
  public_controller_url:   https://pool.example.com
  public_broker_url:       https://pool-broker.example.com
  public_broker_quic_addr: pool-broker.example.com:8443

payment_daemon: { socket: /var/run/livepeer/payment-daemon.sock }
```

> **`auto_approve_join_requests` does not weaken any check.** It only removes the manual
> click — a join still auto-approves *only if* backend verification, capability claims, and
> offer matching all pass (`admissionreview` preview says `Approvable`). Paired with
> synthetic probes and `auto_drain_backends`, flaky supply is cooled/drained automatically.
> Keep `GET /admin/v1/audit-events` (kind `join_request_auto_approved`) under review early.

### `coordinator-config.yaml`

```yaml
identity: { orch_eth_address: "<your orch address>" }
brokers:
  - name: pool-edge
    base_url: https://pool-broker.example.com   # public TLS broker URL
publish: { manifest_ttl: 24h }
```

### `pool-reconciler.yaml` — your commission is here

```yaml
pool_controller: { url: http://pool-controller:8080, bearer_token_ref: env://POOL_CONTROLLER_ADMIN_TOKEN }
payment_daemon:  { socket: /var/run/livepeer/payment-daemon.sock }
pool: { commission_bps: 1000 }          # 1000 bps = 10% orch cut; members get the remainder
reconcile:
  state_path: /var/lib/livepeer/pool-reconciler-state.db
  backfill_limit: 32
  retry_interval_ms: 5000
round_source: { protocol_daemon_socket: /var/run/livepeer/protocol.sock }
```

### `pool-payout-executor.yaml`

```yaml
pool_controller: { url: http://pool-controller:8080, bearer_token_ref: env://POOL_CONTROLLER_ADMIN_TOKEN }
executor:
  batch_size: 25
  executor_id: executor-a
  rpc_url: https://arb1.arbitrum.io/rpc
  chain_id: 42161                       # Arbitrum One
  keystore_path: /etc/livepeer/keystore.json
  keystore_password_path: /etc/livepeer/keystore-password
  confirmation_blocks: 1
  auto_requeue_failed: true             # example ships false; enable for hands-off retries
  max_retries: 3
  requeue_cooldown_seconds: 3600
```

## 5. Bring up the stack

```bash
docker compose \
  -f infra/scenarios/pool-orchestrator/docker-compose.yml \
  --env-file infra/scenarios/pool-orchestrator/.env \
  up -d

curl -s localhost:8080/healthz   # pool-controller
curl -s localhost:8080/readyz
```

Admin web UI: `/admin/login` → `/admin/offers`, `/admin/members`, `/admin/assignments`,
`/admin/broker-runtime`, `/admin/audit`.

## 6. Define what you sell — offers

Create **orch-owned offers** (one per AI capability). An offer is a priced capability
tuple:

```text
Offer = { capability_id, offering_id, interaction_mode, work_unit, price, constraints }
```

Phase-1 AI examples (canonical capability names are hyphenated):

| capability_id | interaction_mode | work_unit |
|---|---|---|
| `openai-chat-completions` | `http-stream` (streaming) / `http-reqresp` | tokens |
| `openai-text-embeddings` | `http-reqresp` | tokens |
| `openai-audio-transcriptions` | `http-multipart` | audio_seconds |
| `openai-audio-speech` | `http-reqresp` | input_chars |
| `image-generation` | `http-reqresp` | images |

Create via `POST /admin/v1/offers` (or the `/admin/offers` UI). The `price` (wei per work
unit) and `commission_bps` are the two economic dials that are yours to set.

## 7. Onboard public supply

The member experience (a single `docker compose up -d`) and what the downloadable bundle
contains are documented in [`pool-member-onboarding.md`](./pool-member-onboarding.md). On
the operator side you must first configure a **runner template** so the bundle knows which
runner container to ship with the agent (image, command, environment). With
`auto_approve_join_requests` on, an enrolled member whose backend verifies is admitted
automatically; synthetic probes begin scoring it.

Useful reads: `GET /admin/v1/join-requests`, `/members`, `/member-backends`.

## 8. Assign → apply → converge

1. Assign approved member backends to active offers: `GET /admin/v1/assignment-candidates`
   → create assignments.
2. Render + push broker runtime: `POST /admin/v1/broker-runtime/apply`.
3. **Confirm convergence — never trust the shell exit alone:**
   ```bash
   curl -s localhost:8080/admin/v1/broker-runtime          # desired == loaded revision?
   curl -s localhost:8080/admin/v1/broker-runtime/history
   curl -s localhost:8082/admin/v1/runtime                 # broker's own loaded_revision + attempt_id
   ```
   Proceed only when `loaded_revision == desired_revision`.

## 9. Sign & publish (existing secure cycle)

Unchanged from a normal orchestrator:

1. `orch-coordinator` scrapes the broker's new offerings/health → builds the candidate.
2. `secure-orch-console` (cold key) signs — or agent mode auto-signs within your
   sign-policy envelope (plan 0042).
3. `orch-coordinator` publishes the signed manifest at the `serviceURI` gateways resolve.

**Do not publish until step 8 convergence is confirmed.**

## 10. Smoke test & go live

Run one low-risk paid request through a gateway (or LOC job) at an offered capability.
Confirm: broker routed to a member backend → work receipt in `pool-controller`
(`GET /admin/v1/state`) → a round closes → a payout intent appears
(`GET /admin/v1/payout-intents`).

## 11. Ongoing operations

- **Payout alarm:** alert on `livepeer_pool_payout_intent_failed_age_seconds_max > 3600`.
  Watch `..._with_retries_total` and `..._retry_count_max` as leading signals. See the
  triage table in [`pool-controller/RUNBOOK.md`](../../modules/livepeer-network-modules/pool-controller/RUNBOOK.md).
- **Supply quality:** scoring/probe metrics on the controller metrics port (selection
  counts by state, cooldown/warm-up, per-capability probe status).
- **Audit:** `GET /admin/v1/audit-events` — your window into what the automation admitted.
- **Backups:** back up the entire `pool-controller --data-dir` (BoltDB) — canonical
  receipt / payout / control-plane store. Restart is safe if the data-dir is persisted.

## 12. Phase 2: video (additive, no re-architecture)

Stand up [`livepeer-modules-transcode-runners`](../repos/livepeer-modules-transcode-runners.md)
on members, add a video runner template, add video offers (`interaction_mode:
rtmp-ingress-hls-egress` / `http-multipart`; `work_unit: pixel-seconds` etc.), then re-run
assign → apply → sign. Same pool, same member model.

## Source pointers

- Scenario: `infra/scenarios/pool-orchestrator/{docker-compose.yml,README.md,.env.example}`
- Operator control plane: [`pool-controller/RUNBOOK.md`](../../modules/livepeer-network-modules/pool-controller/RUNBOOK.md)
- Production rollout notes: `docs/design-docs/pool-orchestrator-production-rollout.md`
  (in the network-modules submodule)
