# Join a Livepeer Pool (member onboarding)

A guide for **suppliers**: contribute your GPU to a Livepeer Pool, run paid AI inference
jobs, and get paid in ETH — without running an orchestrator, a payment daemon, TLS, DNS,
or opening any inbound ports.

This is the member-facing companion to the operator's
[pool deployment runbook](./pool-deployment-runbook.md). Everything here is grounded in
the `pool-member-agent` and `pool-controller` components of
[`livepeer-network-modules`](../repos/livepeer-network-modules.md).

## What you're signing up for

You run a small stack on your machine — a **pool member agent** plus a **runner**
(the container that actually serves the model) — delivered to you as a ready-made bundle.
The agent opens a single **outbound** connection to the Pool's broker and waits for work.
When the broker sends you a paid job, your runner executes it; at the end of each
accounting round the Pool pays your wallet its share in native ETH on Arbitrum.

You never expose anything inbound. You never hold or handle payment tickets. Your **ETH
wallet address is your identity** and your payout destination.

```text
   your host                              the Pool (operator-run)
 ┌─────────────────────────┐   outbound   ┌──────────────────────────┐
 │ pool-member-agent  ─────┼──QUIC/WS────▶│ capability-broker         │
 │   └─ your GPU runner     │  (8443/udp   │  (sends you paid jobs)    │
 │      (serves the model)  │   or wss)    │ pool-controller           │
 └─────────────────────────┘              │  (admits you, pays you)   │
        ▲ paid work in, ETH out ──────────┘  pool-payout-executor      │
                                          └──────────────────────────┘
```

## Requirements

- A Linux host with **Docker + Docker Compose**.
- An **NVIDIA GPU** with working drivers + the NVIDIA container toolkit (the bundle runs
  containers with `gpus: all`). ML runners **fail-fast** if `DEVICE=cuda` and no GPU is
  visible — so a working GPU is mandatory for GPU capabilities.
- **Outbound** internet only. UDP `8443` to the Pool broker is preferred (QUIC); if your
  network blocks UDP, the agent automatically falls back to WebSocket over the broker URL.
- An **Ethereum wallet** (an address on Arbitrum One). This is your login and your payout
  address. You do **not** need it funded — the Pool pays *you*.
- The Pool's **controller URL**, given to you by the operator.

## The join flow

The operator's control plane (`pool-controller`) exposes a member API and a sign-in web
UI. The whole flow is: **sign in → enroll a host → download bundle → `docker compose up`.**

### 1. Sign in with your wallet

Prove control of your address with a signature challenge (no password, no account):

```text
POST /member/v1/auth/nonce    { "member_eth_address": "0xYOURADDRESS" }
POST /member/v1/auth/verify   { signed nonce }   → session
```

Most operators put a web page in front of this so you just click "Connect Wallet."

### 2. Enroll a host

Register the machine you'll contribute:

```text
POST /member/v1/enrollments   { "host_label": "rig-1" }
→ { enrollment_id, token, bundle_url }
```

You get an **enrollment id**, a bearer **token**, and a **bundle URL**.

### 3. Download your bundle

```text
GET /member/v1/enrollments/{enrollment_id}/bundle   (Authorization: Bearer <token>)
→ bundle.zip
```

The zip is self-contained and specific to your enrollment. It holds:

| File | What it is |
|---|---|
| `docker-compose.yaml` | the `pool_member_agent` service **plus** your assigned runner container(s) |
| `.env` | your enrollment's settings (see below) |
| `enrollment-token` | your bearer credential, mounted read-only into the agent |
| `pool-member-agent.yaml` | agent config |
| `README` | "run `docker compose up -d`" |
| update script | re-fetches a fresh bundle, `docker compose pull`, `up -d` |

The `.env` is pre-filled by the operator — you don't edit it. It carries:
`POOL_CONTROLLER_URL`, `POOL_BROKER_URL`, `POOL_BROKER_QUIC_ADDR`, `POOL_ENROLLMENT_ID`,
`POOL_MEMBER_ETH_ADDRESS`, `POOL_BROKER_SESSION_CREDENTIAL`, `POOL_WORKER_BACKENDS`, and
`POOL_ENROLLMENT_TOKEN_FILE`.

> **You don't pick or configure the runner.** The operator's *template* decides which
> runner image, command, and environment ship in your bundle, matched to the capability
> you're supplying and your hardware. The compose wires the agent to your local runner at
> `http://<runner-service>:8080` automatically.

### 4. Bring it up

```bash
unzip bundle.zip -d livepeer-pool && cd livepeer-pool
docker compose pull
docker compose up -d
```

That's the whole install. On startup the agent:

1. reports your **GPU inventory** to `POST /member/v1/enrollments/{id}/hardware`, and
2. opens **one outbound worker session** to the broker (QUIC if `POOL_BROKER_QUIC_ADDR`
   is reachable, else WebSocket).

### 5. Get admitted

The operator reviews your enrollment and verifies your backend. On Pools configured for
automated admission, this happens **without a manual step**: as soon as your backend
verifies against a matching offer you're approved, and the Pool's **synthetic probes**
start health-checking your runner. Once your backend is assigned to an offer and the
operator applies broker runtime, you start receiving paid work.

## How you get paid

- The broker meters each job you complete and emits a **work receipt** to the Pool
  controller.
- At each protocol round close (~19h on Arbitrum One), the `pool-reconciler` aggregates
  your receipts and derives a **payout intent** for your wallet — your share of realized
  revenue, after the operator's commission (`commission_bps`).
- The `pool-payout-executor` sends the payout to your address in **native ETH on
  Arbitrum**. Intents move `pending → exported → leased → submitted → paid` (with
  automatic retry on transient chain errors).

Payout share = your metered work × offer price − the operator's commission. Ask your
operator for their `commission_bps` and per-capability pricing.

## Staying healthy (and staying selected)

The Pool scores every backend and routes preferentially to the healthy, fast ones:

- **Keep the runner up and warm.** New backends go through a warm-up window before they
  score fully; restarts reset it.
- **Failures cost you routing.** Repeated backend failures (default: 5 within a rolling
  5-minute window) open a cooldown; sustained high failure rates can auto-**drain** your
  backend until it recovers.
- **Latency matters.** Scoring blends a recent-window latency signal with longer-lived
  memory (24h half-life), so a slow GPU gets less traffic than a fast one.
- **Probes must pass.** Synthetic probes call your runner directly; failing probes exclude
  you until they pass again.

## Updating

Re-run the bundled update script (or re-download the bundle) whenever the operator changes
your template/assignment or ships a new runner version:

```bash
./update.sh      # fetches a fresh bundle, docker compose pull, up -d
```

## Leaving

Stop contributing at any time:

```bash
docker compose down
```

Your last completed work still settles and pays out on the next round close. Tell the
operator if you're leaving permanently so they can retire your enrollment.

## FAQ

**Do I need to open ports / set up a domain / run a wallet daemon?** No. Outbound only;
no inbound, no TLS, no DNS. Your wallet is just an address for identity and payout.

**Do I need to fund my wallet?** No — the Pool pays you. You only need funds if you also
want to *use* the network as a customer, which is separate.

**Can I run multiple GPUs / hosts?** Yes — enroll each host separately (step 2). Each gets
its own bundle and reports its own hardware.

**What models will I serve?** Whatever capability the operator assigns you (Phase 1 is
OpenAI-shaped AI: chat, embeddings, audio, TTS, image, rerank). The runner image in your
bundle determines it; you don't configure the model yourself.

**Where does my payout come from?** The operator's `pool-payout-executor` wallet, in
native ETH on Arbitrum One, once per accounting round.
