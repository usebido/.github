# Bido

Bido turns AI agents into monetized decision agents. On every relevant user turn, Bido detects whether the message carries sponsorable intent (today: travel, with health and ecommerce verticals coming online), runs a first-price auction across eligible sponsors, and lets the agent inject the winning sponsor as internal context **before** the final answer. Settlement happens on-chain on Solana in USDC, with **95% of every winning bid going to the agent owner's wallet** and 5% to Bido.

This monorepo holds every moving piece of that pipeline.

---

## High-level architecture

```
                ┌──────────────────────────────┐
                │  AI agent (Claude Code,      │
                │  Codex, OpenClaw, etc.)      │
                │  with bido-sponsored-intent  │
                │  skill installed             │
                └──────────────┬───────────────┘
                               │  user turn
                               ▼
        ┌──────────────────────────────────────────┐
        │  detect-intent (Python / FastAPI / Groq) │
        │  POST /detect-intent                     │
        │  → sponsorable? + entities + vertical    │
        └──────────────┬───────────────────────────┘
                       │ if sponsorable
                       ▼
        ┌──────────────────────────────────────────┐
        │  backend (NestJS / Prisma / Postgres)    │
        │  POST /api/intent/match                  │
        │  → first-price auction across campaigns  │
        │  → returns selected_candidate            │
        └──────────────┬───────────────────────────┘
                       │ on settlement
                       ▼
        ┌──────────────────────────────────────────┐
        │  programs-sol/bido-campaign-program       │
        │  Solana program (native Rust)            │
        │  → settles USDC: 95% agent / 5% Bido     │
        │  ↑ gas paid by Kora paymaster            │
        └──────────────┬───────────────────────────┘
                       │ on confirmed payout
                       ▼
        ┌──────────────────────────────────────────┐
        │  backend/attestations (SAS)              │
        │  → AgentEvent → milestones / tier-change │
        │  → weekly AgentReputationSnapshot        │
        │  → on-chain reputation via Solana        │
        │    Attestation Service                   │
        └──────────────────────────────────────────┘

  frontend (Next.js)  ─►  sponsor dashboard, docs, marketing site
  skills/             ─►  the SKILL.md files agents install via npx skills
```

---

## Repository layout

| Path                                     | Role                                                    | Stack                                       |
| ---------------------------------------- | ------------------------------------------------------- | ------------------------------------------- |
| [`backend/`](./backend)                  | Sponsor app API, auction matcher, settlement orchestrator | NestJS 11, Prisma 6, PostgreSQL, Privy auth |
| [`backend/src/attestations/`](./backend/src/attestations) | Agent reputation pipeline anchored on Solana Attestation Service | NestJS, Borsh, Redis cache, SAS on-chain |
| [`detect-intent/`](./detect-intent)      | Sponsorable-intent classifier microservice              | Python 3.11+, FastAPI, Groq (llama-3.1-8b)  |
| [`programs-sol/bido-campaign-program/`](./programs-sol/bido-campaign-program) | On-chain campaign + settlement program | Solana native program (Rust), SPL Token, USDC |
| [`programs-sol/kora/`](./programs-sol/kora) | Kora paymaster configuration (gas sponsorship)       | TOML config for Kora                        |
| [`skills/`](./skills)                    | Agent skills, distributed via `npx skills add`          | Plain `SKILL.md` + `skill.json`             |
| [`sdk-ts/`](./sdk-ts)                    | Official TypeScript SDK (`@usebido/sdk`)                | TypeScript, ESM + CJS, Node 18+ / browser / edge |
| [`cli/`](./cli)                          | Local CLI to exercise the pipeline end-to-end           | TypeScript, consumes `@usebido/sdk`         |
| [`frontend/`](./frontend)                | Sponsor dashboard, docs, marketing                      | Next.js (App Router), Tailwind, shadcn/ui   |
| `CAMPAIGN_TRANSACTIONS_PLAN.md`          | Design doc — campaign transaction lifecycle             | —                                           |
| `SETTLEMENT_PLAN.md`                     | Design doc — settlement flow and edge cases             | —                                           |

---

## Components

### `backend/` — sponsor API & auction engine

The control plane. NestJS 11 service that owns:

- **Auth** — Privy server-side token verification (sponsors log in via Privy on the frontend, the backend validates the JWT).
- **Users + Campaigns** — sponsor accounts and their campaigns. Campaigns can be paused/resumed/soft-deleted and have CRUD plus public/private funding flows on Solana.
- **Intent matching** — exposes `POST /api/intent/match`, the matcher endpoint the skill calls every sponsorable turn. Picks eligible campaigns, runs the **first-price auction**, and returns the `selected_candidate`.
- **Campaign analytics** — overview totals + per-campaign chart series for the sponsor dashboard.
- **Settlements** — orchestrates `settle_winning_bid` calls into the on-chain program after a turn is monetized.
- **Attestations** — agent reputation pipeline anchored on the Solana Attestation Service (see below).
- **Cloak** — private campaign funding flows (on-chain confidential top-ups).
- **Swagger** — interactive API docs at `/api/docs`.

Key environment variables:

```
PRIVY_APP_ID, PRIVY_APP_SECRET
DATABASE_URL, DIRECT_URL                   # Postgres (Supabase pooler supported)
SOLANA_RPC_URL, SOLANA_USDC_MINT
SOLANA_CAMPAIGN_PROGRAM_ID                 # the deployed bido-campaign-program
SOLANA_BIDO_TREASURY_WALLET                # receives the 5% Bido share
KORA_RPC_URL                               # paymaster endpoint
```

Run locally:

```bash
cd backend
cp .env.example .env  # fill in the vars above
npm install
npm run prisma:generate
npm run prisma:migrate -- --name init
npm run start:dev
```

See [`backend/README.md`](./backend/README.md) for the full module map.

---

### `backend/src/attestations/` — agent reputation on Solana Attestation Service

The reputation layer for autonomous agents. Every confirmed payout becomes an event; once thresholds are crossed, Bido emits **on-chain attestations** through the [Solana Attestation Service (SAS)](https://attest.solana.com), giving any sponsor or protocol a trustless way to verify an agent's track record.

Module map:

| File                                | Role                                                                  |
| ----------------------------------- | --------------------------------------------------------------------- |
| `sas-protocol.ts`                   | Borsh codecs, PDA derivation, SAS instruction builders (framework-free) |
| `sas-gateway.service.ts`            | Submits SAS transactions via Kora (gasless)                           |
| `agent-reputation.service.ts`       | Scoring engine + milestone / tier-change emission                     |
| `tier-cache.service.ts`             | Redis cache (in-memory fallback) for fast tier lookups                |
| `snapshot-cron.service.ts`          | Weekly Sun 00:00 UTC dispatcher for reputation snapshots              |
| `attestations.controller.ts`        | Public REST: `GET /agents/:wallet/tier`, `GET /attestations`          |
| `bootstrap.cli.ts`                  | `npm run sas:bootstrap` — one-shot on-chain setup                     |
| `tiers.ts`                          | Tier thresholds + milestone enums                                     |

Lifecycle:

| Trigger                          | SAS schema                  | Notes                                              |
| -------------------------------- | --------------------------- | -------------------------------------------------- |
| Confirmed payout                 | `AgentEvent` (DB)           | `SettlementService` calls `recordConfirmedPayout`  |
| Payout milestone crossed         | `AgentMilestone`            | One-shot, salted nonce, permanent on-chain         |
| Tier transition                  | `AgentTierChange`           | One-shot, salted nonce, permanent on-chain         |
| Sun 00:00 UTC, agent had events  | `AgentReputationSnapshot`   | Same PDA (nonce = agent wallet), 8-day expiry      |

One-time setup:

```bash
cd backend
npx prisma migrate dev --name add_agent_reputation
npm run sas:bootstrap   # idempotent — skips PDAs that already exist
```

Verification from any sponsor or external protocol:

```ts
import { createBidoClient } from '@usebido/sdk';

const bido = createBidoClient();
const { tier, tierName, attestationPda } =
  await bido.getAgentTier('AgentPubkeyBase58');
```

For fully trustless reads, fetch the PDA bytes directly via any Solana RPC and decode them with `decodeSnapshotAttestation` from `@usebido/sdk`. Kora pays the gas on every SAS write, so agents and sponsors never need SOL to participate in the reputation layer.

See [`backend/src/attestations/README.md`](./backend/src/attestations/README.md) and [`backend/docs/agent-reputation-PRD.md`](./backend/docs/agent-reputation-PRD.md).

---

### `detect-intent/` — sponsorable-intent classifier

A small Python service that decides whether a single user message represents a sponsorable decision moment. Built on **FastAPI** + **Groq** (currently `llama-3.1-8b-instant`).

**Public endpoint:** `POST https://api-intent.usebido.com/detect-intent`

Request:

```json
{ "query": "I need a flight from São Paulo to Lisbon on Friday night." }
```

Response:

```json
{
  "sponsorable": true,
  "confidence": 0.87,
  "vertical": "travel",
  "intent_type": "voo",
  "purchase_stage": "ready_to_buy",
  "urgency": "high",
  "entities": {
    "destination": "Lisboa",
    "origin": "São Paulo",
    "checkin": null,
    "checkout": null,
    "travelers": null,
    "budget_signal": null,
    "flexibility": null
  }
}
```

The service is **stateless and unauthenticated**. It is the first step in the pipeline — if `sponsorable=false`, the skill stops immediately and the agent answers normally.

Run locally:

```bash
cd detect-intent
python -m venv .venv && source .venv/bin/activate
pip install -e .[dev]
export GROQ_API_KEY=...
detect-intent-api          # serve via uvicorn
# or
detect-intent "hotel pet friendly no Rio semana que vem"
```

See [`detect-intent/README.md`](./detect-intent/README.md).

---

### `programs-sol/bido-campaign-program/` — on-chain campaign & settlement

A **native Solana program** (no Anchor, raw `solana-program 2.1.16`) that holds campaign budgets in USDC and settles winning bids.

Instructions:

| Instruction                          | Purpose                                                                |
| ------------------------------------ | ---------------------------------------------------------------------- |
| `initialize_campaign`                | Sponsor opens a campaign account (PDA seeded by `b"campaign"` + id).   |
| `deposit_campaign_budget_public`     | Sponsor tops the campaign vault with USDC, publicly visible.           |
| `finalize_private_campaign_funding`  | Confidential funding flow (paired with the cloak module on backend).   |
| `settle_winning_bid`                 | Backend submits a settled auction → split USDC 95% agent / 5% Bido.    |

State lives in PDAs (`src/state/campaign.rs`) with a fixed discriminator (`BIDOCMP1`). The 5% Bido cut is hard-coded on-chain:

```rust
pub const BIDO_TREASURY_BPS: u64 = 500;        // 5%
pub const BPS_DENOMINATOR: u64  = 10_000;
```

This means **the split cannot be changed by the backend off-chain** — the program enforces it.

Build:

```bash
cd programs-sol/bido-campaign-program
cargo build-sbf
```

---

### `programs-sol/kora/` — gas paymaster configuration

[Kora](https://github.com/solana-foundation/kora) is an open-source Solana paymaster service. Bido runs Kora so that **agents and users never need SOL** to interact with the campaign program — Kora pays the gas, optionally rebated in SPL tokens.

This folder contains only the deployment config:

- `kora.toml` — rate limits, allowed programs (system, SPL token, ATA, ALT, memo, compute budget, and the Bido campaign program), allowed payment tokens (USDC).
- `signers.toml` — Kora signer pool (round-robin, key loaded from `KORA_PRIVATE_KEY`).

The Kora server itself is run as a separate process pointing at this config. The backend talks to it via `KORA_RPC_URL`.

---

### `skills/` — installable agent skills

The agent-side artifacts. Distributed via the public [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI, so any developer can install with one command:

```bash
npx skills add usebido/skills -a claude-code
npx skills add usebido/skills -a codex
npx skills add usebido/skills -a openclaw
```

Currently ships:

- **`bido-sponsored-intent`** — the runtime contract for the agent. On each turn it calls `detect-intent`, then (if sponsorable) the backend matcher, then injects `BIDO_SPONSOR_CONTEXT` before the final answer. Requires `SOLANA_AGENT_WALLET` (the agent owner's Solana address — receives 95% of winning bids).

The skill is just `SKILL.md` + `skill.json` — no executable code. The agent reads it.

See [`skills/README.md`](./skills/README.md).

---

### `sdk-ts/` — official TypeScript SDK (`@usebido/sdk`)

The TypeScript SDK any application (skill, agent runtime, sponsor backend, dashboard, third-party protocol) uses to talk to Bido. Published as [`@usebido/sdk`](https://www.npmjs.com/package/@usebido/sdk). Public mirror: https://github.com/usebido/sdk-ts.

Highlights:

- **One-call pipeline** — `bido.run(query)` runs `detect` and (if sponsorable) `match` in sequence, returning the selected sponsor + `decision_id` + `settlement_tx_hash`.
- **Granular API** — `bido.detect()`, `bido.match()`, plus reputation reads via `bido.getAgentTier(wallet)` and `decodeSnapshotAttestation` (for trustless on-chain verification).
- **Production defaults** — sane timeouts, exponential backoff with jitter, `AbortSignal` cancellation, typed `BidoError` discriminated by `code` (`timeout`, `network`, `http_4xx`, `http_5xx`, `invalid_response`, `invalid_input`, `aborted`).
- **Runtime agnostic** — Node.js 18+, modern browsers, Deno, Bun, edge runtimes. Ships ESM + CJS + `.d.ts`.

Quick start:

```ts
import { BidoClient } from '@usebido/sdk';

const bido = new BidoClient({
  agentTreasuryWallet: process.env.SOLANA_AGENT_WALLET,
});

const { intent, match } = await bido.run('quero viajar pra Lisboa em julho');
if (match?.selected_candidate) {
  console.log(`Sponsor: ${match.selected_candidate.advertiser}`);
}
```

See [`sdk-ts/README.md`](./sdk-ts/README.md).

---

### `cli/` — local CLI for end-to-end testing

A thin TypeScript CLI on top of `@usebido/sdk` for exercising the full pipeline from a terminal. **Not published** — it's a dev tool for the in-repo workflow. Public mirror: https://github.com/usebido/cli.

Commands:

| Command  | Calls                            | Needs wallet | Settles on-chain               |
| -------- | -------------------------------- | ------------ | ------------------------------ |
| `detect` | `bido.detect()`                  | no           | no                             |
| `match`  | `bido.detect()` + `bido.match()` | yes          | **yes**                        |
| `run`    | `bido.run()`                     | yes          | **yes** (when sponsorable)     |

Both `match` and `run` settle on-chain because the backend's `/intent/match` chains `settleWinningBid` in the same request — every successful invocation spends real campaign budget.

Setup:

```bash
cd cli
cp .env.example .env       # set SOLANA_AGENT_WALLET
npm install
npm run build
npm link                   # optional: exposes `bido` globally on this shell
```

One-shot usage:

```bash
bido detect "quero viajar pra Lisboa em julho"
bido match  "hotel pet friendly no Rio semana que vem"
bido run    "preciso de um voo SP -> Lisboa na sexta"
```

Or drop into the interactive REPL by running `bido` with no subcommand. Global flags include `--json`, `--wallet`, `--backend-url`, `--intent-url`, `--timeout`, `--verbose`, `--no-color`. Exit codes are stable (`0` ok, `2` missing env, `3` network, `4` HTTP, `5` invalid response) so the CLI plugs into CI and shell scripts.

See [`cli/README.md`](./cli/README.md) and [`cli/spec.md`](./cli/spec.md).

---

### `frontend/` — sponsor dashboard, docs & marketing

Next.js App Router app (`app/`) covering:

- **Marketing surfaces**: landing, sponsors page, devs page, customer cases.
- **Sponsor dashboard** (`app/app/`): authenticated UI for campaign CRUD, budget top-ups, analytics.
- **Docs** (`app/docs/`): full developer documentation with i18n (`pt-BR`, `en`), an install-tabs widget for the 3 agent targets, and a download-card linking to the skill files on GitHub.
- **i18n** in `lib/i18n/` — page-level message bundles per locale.

Run locally:

```bash
cd frontend
npm install
npm run dev    # http://localhost:3000
```

---

## End-to-end turn lifecycle

1. User sends a message to an agent that has `bido-sponsored-intent` installed.
2. The skill calls `detect-intent` with the raw query.
3. If `sponsorable=false`, the skill stops. Agent answers normally.
4. If `sponsorable=true`, the skill posts the detector JSON + the agent's wallet to `backend` `POST /api/intent/match`.
5. The backend selects eligible campaigns, runs the **first-price auction**, and returns `selected_candidate` (or `null` if no eligible campaign).
6. If a winner exists, the skill builds a `BIDO_SPONSOR_CONTEXT` block and injects it into the agent's system prompt.
7. The agent generates the final answer with the sponsor in mind, while keeping the response useful.
8. After the turn, the backend submits `settle_winning_bid` to the Solana program. USDC moves: 95% to `SOLANA_AGENT_WALLET`, 5% to the Bido treasury. Gas is paid by Kora.
9. The confirmed payout is recorded as an `AgentEvent` by the attestations service. If it crosses a milestone or tier threshold, Bido writes an `AgentMilestone` / `AgentTierChange` attestation on-chain via SAS. Every Sunday 00:00 UTC, active agents get a fresh `AgentReputationSnapshot` so sponsors can verify reputation trustlessly.

If any step in the pipeline fails (detector down, matcher down, wallet missing, no eligible campaign), the agent **continues normally** — sponsorship can never degrade the base UX.

---

## Money flow

```
sponsor wallet ──► campaign vault (PDA, USDC) ──► settle_winning_bid
                                                    │
                                          ┌─────────┴─────────┐
                                          │                   │
                                       95% agent           5% Bido
                                  (SOLANA_AGENT_WALLET)  (treasury wallet)
```

- All amounts are **USDC on Solana** (6 decimals).
- The split is **enforced on-chain** by the campaign program (`BIDO_TREASURY_BPS = 500`).
- Agents never need to hold SOL — Kora sponsors gas.
- Agents never sign anything — they only expose a public address via `SOLANA_AGENT_WALLET`.

---

## Local development quickstart

To run the full stack locally you'll need at least:

- Node.js 18+ (for backend, frontend, skills CLI)
- Python 3.11+ (for detect-intent)
- Rust + Solana CLI + `cargo build-sbf` (only if you want to rebuild the program)
- A Postgres instance (Supabase works)
- A Solana RPC (devnet is fine for development)
- A Groq API key (for detect-intent)

Order in which to bring services up:

1. `detect-intent` (FastAPI on `:8000` by default)
2. `backend` (NestJS on `:3001` or similar)
3. `frontend` (Next.js on `:3000`)
4. (optional) Local Kora server if testing settlement end-to-end

Each component has its own README with detailed steps. The design docs `CAMPAIGN_TRANSACTIONS_PLAN.md` and `SETTLEMENT_PLAN.md` at the repo root document the on-chain flows in depth.

---

## External services

| Service              | Why                                                               |
| -------------------- | ----------------------------------------------------------------- |
| **Privy**            | Sponsor authentication on the frontend + JWT verification on backend |
| **Supabase / Postgres** | Persistence layer for sponsors, campaigns, analytics, audit trail |
| **Groq**             | LLM provider behind the intent detector (`llama-3.1-8b-instant`)    |
| **Solana**           | Settlement layer (USDC, on-chain auction split)                   |
| **Solana Attestation Service (SAS)** | Trustless on-chain agent reputation (milestones, tier changes, weekly snapshots) |
| **Kora**             | Gas paymaster so agents/users don't need SOL (settlement + SAS writes) |
| **`vercel-labs/skills`** | CLI used by `npx skills add usebido/skills` to install the agent skill |

---

## Repos & links

- Skills repo (public): https://github.com/usebido/skills
- SDK repo (public): https://github.com/usebido/sdk-ts — published as [`@usebido/sdk`](https://www.npmjs.com/package/@usebido/sdk)
- CLI repo (public): https://github.com/usebido/cli
- Marketing site: https://usebido.com
- Detector API: https://api-intent.usebido.com
- Backend API: https://api.usebido.com
