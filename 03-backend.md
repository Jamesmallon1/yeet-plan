# Step 3 — Backend (`yeet-backend`)

Rust, Tokio, **actix-web**. One n-tier monolith binary running indexer + REST + WebSocket, with a small **app Postgres** on the same box (unix socket) for tokens/holders/dividends/state, and one **TimescaleDB** instance on its own box for trades + candles (split defined in step 4). Both in Hetzner Cloud **Ashburn (us-east)**, on a private network, reachable only through Cloudflare.

## Architecture

```
                    ┌───────────────────────── yeet-backend (1 × CCX33) ─────────────────────────┐
Cloudflare ─tunnel─▶│ api/            actix-web handlers, one file per path prefix, WS endpoint      │
                    │   │ injects                                                                   │
                    │ services/       business logic. Generic over the traits below (static dispatch)│
                    │   │ injects               │ injects                                           │
                    │ networking/     RPC (QuickNode→Circle failover), IPFS/Pinata, R2, Cloudflare  │
                    │ db/             sqlx repositories against Timescale. Hot path, hand-tuned     │
                    │ common/         config, errors, types, telemetry, DI container                │
                    └─────────────────────────────────┬──────────────────────────────────────────────┘
                                          private net │ 10.0.0.0/24, ~0.3 ms RTT
                                     ┌────────────────▼───────────────┐
                                     │ TimescaleDB (1 × CCX33, NVMe)  │
                                     └────────────────────────────────┘
```

### Dependency injection, zero-overhead

Services are generic over traits, not `Box<dyn>`. Monomorphised at compile time, no vtable on the hot path, mocks in tests via `mockall`.

```rust
// services/trade_service.rs
pub struct TradeService<R: TradeRepo, C: CandleRepo> { trades: R, candles: C }
impl<R: TradeRepo, C: CandleRepo> TradeService<R, C> { pub async fn recent(&self, token: Address, n: u32) -> Result<Vec<Trade>> { self.trades.recent(token, n).await } }

// common/container.rs — the single place concrete types are named
pub type Trades = TradeService<PgTradeRepo, PgCandleRepo>;
pub struct Container { pub trades: Trades, pub tokens: Tokens, pub indexer: Indexer, pub bus: Bus, pub cfg: Arc<Config> }
pub fn build(cfg: Config) -> Container { … }        // constructed once in main, handed to actix as web::Data<Container>
```

Handlers receive `web::Data<Container>` and call one service method. Handlers contain no logic beyond parsing and status codes. Services contain no SQL and no HTTP. Repos contain no business rules.

### Folder layout

```
yeet-backend/
  Cargo.toml                 single crate, modules below (a workspace is not worth it for one binary)
  chains/arc-testnet.toml, chains/arc-mainnet.toml
  deployments/<chainId>.json  (copied from yeet-launchpad-evm)
  migrations/                sqlx migrations: hypertable, caggs, policies, indexes
  src/
    main.rs                  load config → build container → spawn indexer task → start actix
    api/
      mod.rs                 route registration, middleware (cors, request-id, rate limit)
      tokens.rs              GET /api/tokens, /api/tokens/{addr}, /api/tokens/{addr}/holders
      trades.rs              GET /api/tokens/{addr}/trades
      candles.rs             GET /api/tokens/{addr}/candles?i=1m|5m|15m|1h|4h|1d&from&to
      dividends.rs           GET /api/tokens/{addr}/dividends/{holder}   (claimable via eth_call, claimed history from events); claiming itself is a wallet tx from the UI
      metadata.rs            POST /api/metadata (image + json → IPFS + R2), SIWE-gated
      quotes.rs              GET /api/tokens/{addr}/quote?side&amount    (curve math or router.quote via eth_call)
      health.rs              GET /healthz (db, rpc head lag, indexer lag)
      ws.rs                  GET /ws — topics: launches, trades:{addr}, token:{addr}, dividends:{addr}
    services/
      indexer/               head loop, backfill, decoders, writers (from step 1)
      token_service.rs, trade_service.rs, candle_service.rs, dividend_service.rs,
      quote_service.rs, metadata_service.rs, stats_service.rs (24h volume/change cache)
    networking/
      rpc/                   alloy provider, failover list, retry, WS head subscription
      pinata.rs              IPFS pinning
      r2.rs                  Cloudflare R2 (S3 API) image mirror
      contracts/             alloy sol! bindings generated from deployments ABIs
    db/
      pool.rs                PgPool construction + tuning
      trade_repo.rs, candle_repo.rs, token_repo.rs, holder_repo.rs, dividend_repo.rs, indexer_state_repo.rs
      traits.rs              the repo traits services depend on
    common/
      config.rs (YEET_CHAIN → profile + env), error.rs, types.rs (Address, U256 newtypes, 18-dec math),
      telemetry.rs (tracing + Prometheus), bus.rs (tokio broadcast), container.rs
```

## db/ — the low-latency contract

The db layer is the only code that knows SQL. Rules:

1. **sqlx with compile-time checked queries** (`query!`/`query_as!`, offline `.sqlx` metadata committed). No ORM, no query builder on the hot path.
2. **One PgPool**, `max_connections = 24` (DB box has 8 vCPU; Postgres likes ~2–3× cores), `min_connections = 8` kept warm, `acquire_timeout = 1s`, `statement_timeout = 2s` on the API role, TCP keepalives on. Private network only, no TLS (saves a round trip; the link never leaves the Hetzner zone).
3. **Writes** (indexer): one transaction per block range; trades inserted with a single `INSERT … SELECT FROM UNNEST($1::…[], …)` (one round trip per batch, not per row); `ON CONFLICT (tx_hash, log_index) DO NOTHING`. Indexer role runs with `synchronous_commit = off`: every row is re-derivable from chain, so losing the last few ms on a crash is free and write latency halves.
4. **Reads**: every API query hits either a cagg (index-only scan on `(token, bucket)`) or the latest chunk of `trades` via `(token, ts desc)`. Any query plan that touches more than one chunk for a "recent" request is a bug; `EXPLAIN (ANALYZE, BUFFERS)` is part of the PR checklist for db/.
5. **In-process cache** (`moka`) in services, not in db: token list and token stats 2 s TTL, candles for the open bucket 1 s, everything else uncached. A hype launch is thousands of readers of the same few tokens; the DB should see tens of QPS, not thousands.
6. **Holders**: table maintained from `Transfer` events; `balanceOf` is exact now that the token is plain. Top-holders query is `(token, balance desc)` index, limit 50.
7. Separate Postgres roles: `yeet_indexer` (write), `yeet_api` (read-only, statement timeout). Same pool type, two pools.

Target numbers on the private network: candles p50 < 3 ms, recent trades p50 < 2 ms, indexer block-range commit < 10 ms.

## networking/

- `rpc`: alloy `RootProvider` over a failover transport: `[QuickNode HTTP, Circle public HTTP]`, `[QuickNode WSS, Circle public WSS]`; rotate on 429/5xx/timeout, exponential backoff, per-endpoint health. Exposes `head_stream()`, `get_logs(range, addrs)`, `call(...)`.
- `pinata`: pin image + metadata JSON, returns CIDs. `r2`: mirror image to `images.yeet.family/<cid>`.
- `contracts`: `sol!` bindings for launchpad, token, hook, graduator, router, PoolManager events. ABIs come from the deployments file so a contract change is a file copy + rebuild.

## api/

- actix-web 4, `web::Data<Container>`, JSON via `serde`, errors mapped once in `common/error.rs` to `{code, message}`.
- Middleware: request-id, `tracing-actix-web`, CORS locked to `https://yeet.family`, per-IP rate limit (governor) with Cloudflare's `CF-Connecting-IP` as the key.
- WebSocket: `actix-ws`; each connection subscribes to topics; server pushes from the broadcast bus; heartbeat 20 s; max 5 000 connections per instance (CCX33 handles far more, the cap is a safety valve).
- `POST /api/metadata` is the only mutating endpoint. It requires a SIWE-signed nonce (the creator's wallet) and a 2 MB image cap, so nobody can use us as free IPFS hosting.
- Everything else is GET, cacheable at Cloudflare for 1 s (`Cache-Control: public, max-age=1`) which absorbs launch-day bursts before they reach the box.

## Hetzner machines

### Option A — Ashburn (us-east), cloud only

Ashburn has **no dedicated line**; CCX = dedicated vCPU, local NVMe, post-June-2026 US rates.

| Role | Model | vCPU / RAM / NVMe | $/mo |
|---|---|---|---|
| Monolith | CCX33 | 8 / 32 GB / 240 GB | $166 |
| TimescaleDB | CCX33 | 8 / 32 GB / 240 GB | $166 |
| Total | | | ~$332/mo, 2 TB traffic each |

### Option B — Falkenstein, bare metal (recommended on price/perf)

| Role | Model | Spec | €/mo | Setup |
|---|---|---|---|---|
| Monolith | **AX42-1** | Ryzen 7 PRO 8700GE 8c/16t, 64 GB DDR5 ECC, 2×1.92 TB NVMe (RAID-1), 1 Gbit unmetered | 97.30 | 49 |
| TimescaleDB | **AX42-1** | same | 97.30 | 49 |
| Total | | | **€194.60/mo** + €98 once (+ VAT if applicable) | |

Same money as one US cloud box buys two bare-metal boxes with 2× the cores, 2× the RAM and 8× the NVMe each. Timescale on real NVMe with 64 GB RAM will not be the bottleneck for years. Provisioning is minutes for the AX line in FSN. Cheaper variants: one AX42-1 running both (€97.30, fine for launch, single point of failure and DB contention under load) or Falkenstein cloud CCX33 at €138.49 if you want hourly billing.

Private link: Hetzner **vSwitch** (free VLAN between dedicated servers in the same DC) carries Postgres traffic; Timescale binds only to the vSwitch interface.

Trade-off vs us-east: ~90–100 ms RTT from the US east coast instead of ~10 ms. Cloudflare caches every GET at the edge for 1 s, so page loads are unaffected; only WS push and uncached calls pay the extra ~45 ms one way. For a launchpad this is invisible. Arc/QuickNode RPC location is unknown, so neither region has a known RPC-latency advantage.

Timescale config (`timescaledb-tune`, 8c/64 GB): `shared_buffers 16GB`, `effective_cache_size 48GB`, `work_mem 128MB`, `max_connections 100`, `wal_compression on`, checkpoint 15 min, data + WAL on the NVMe RAID-1.

### Network isolation ("only Cloudflare")

- **Monolith:** no public inbound ports at all. `cloudflared` runs on the box and makes an outbound tunnel; Cloudflare routes `api.yeet.family` and `ws.yeet.family` into it. The Hetzner Cloud firewall on the monolith allows inbound only from the private network. SSH via Cloudflare Access (or Tailscale) over the same tunnel, no port 22 exposed.
- **Timescale:** listens only on the vSwitch/private interface, nftables allows 5432 only from the monolith's private IP; public interface drops everything except outbound. (Cloud variant: Hetzner firewall + no public IP.)
- Frontend on Cloudflare Pages calls `api.yeet.family`; the browser's WS goes to `ws.yeet.family` through the same tunnel (Cloudflare supports WS on tunnels).

### Ops

- Docker on the monolith: `yeet-backend` image from GitHub Actions, `cloudflared`, `node-exporter`. Timescale box: `timescale/timescaledb-ha:pg17` container with data on the local NVMe, `postgres-exporter`.
- Metrics: Prometheus scrape over the private net into a tiny Grafana on the monolith (or Grafana Cloud free tier). Alerts: indexer lag > 20 blocks, RPC failover triggered, p99 candles > 50 ms, disk > 70 %.
- Backups: nightly `pg_dump` of `tokens/holders/dividends` to R2. `trades` and caggs are fully re-derivable by replaying the indexer from the deploy block, so a full restore is "restore small tables + replay", tested once before launch.
- Deploy: push tag → GH Actions builds image → SSH over tunnel → `docker compose pull && up -d`. Indexer resumes from its cursor; downtime is the container restart (~2 s).
- All of this is scripted in `yeet-devops` (Terraform for Hetzner + Cloudflare, compose files, Timescale config).

## Order of work (~3 days, parallel with contracts)

1. Skeleton: config, container, actix boot, healthz, migrations, Timescale up locally in Docker (½ day)
2. db/ repos + EXPLAIN-verified queries + cagg migrations (½ day)
3. indexer service against testnet deployment (1 day, from step 1)
4. REST + WS + stats cache + metadata upload (½ day)
5. Provision boxes (Terraform hcloud or Robot API for dedicated), vSwitch, nftables, tunnel; deploy; load test WS fan-out (½ day)


## Status (8 Sep, late)

Built in `yeet-backend` and verified against the Arc testnet lite stack: indexer backfills from the deploy block, decodes launchpad/graduator/router/PoolManager/token events, writes trades to Timescale and everything else to the app Postgres, and every figure matched on-chain state (dividends total, holders, graduation, pool price). Stats refresher, metadata resolver, Multicall3 pending-rewards, quotes via launchpad/router views, and the binary-candle WebSocket all run. Image builds in GitHub Actions to `ghcr.io/jamesmallon1/yeet-backend`.

Two indexer lessons worth keeping: (1) a token created inside a backfill window is not in that window's address filter, so the indexer does a second `eth_getLogs` for newly discovered tokens over the same range; (2) v4 `Swap` events name the router as sender, so the router's own `Swapped` event in the same tx supplies the end user.
