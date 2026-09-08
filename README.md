# yeet.family — launch plan

Meme / token launchpad on Arc (Circle's USDC-native L1). Arc public mainnet goes live **16 Sep 2026**.
Target: ship on testnet first, flip one switch to mainnet on launch day.

Repos:

| Repo | Purpose |
|---|---|
| `yeet-plan` | this plan |
| `yeet-launchpad-evm` | Foundry contracts (token factory, bonding curve, graduation to Uniswap v4) |
| `yeet-backend` | Rust: chain indexer + REST/WebSocket API |
| `yeet-frontend` | Next.js on Cloudflare |
| `yeet-devops` | Hetzner (backend, Postgres) locked to Cloudflare ingress |

Plan steps:

1. [Indexing & streaming](01-indexing-and-streaming.md) — how we ingest Arc blocks/logs and the testnet→mainnet switch
2. [Contracts](02-contracts.md) — curve, holder-rewards token, v4 graduation, router
3. [Backend](03-backend.md) — actix-web monolith, db layer rules, Hetzner Ashburn sizing, network isolation
4. [API definition](04-api.md) — REST /api/v1, image pipeline, WebSocket protocol, WS capacity
5. Frontend — (next)
6. [Devops](05-devops.md) — Hetzner Ashburn + Cloudflare in Terraform, isolation, runbook, costs

Reference: [chain-facts.md](chain-facts.md) — everything verified about Arc testnet, RPC limits, Uniswap status.
