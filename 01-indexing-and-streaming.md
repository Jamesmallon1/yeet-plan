# Step 1 — Indexing & streaming Arc into yeet

Goal: every buy/sell/launch/graduation on our contracts appears in the UI within ~1 s, with charts on 1m/5m/15m/1h/4h/1d and a recent-trades list, on testnet today and on mainnet on 16 Sep by changing one environment variable.

Fixed decisions (agreed 8 Sep):

- **USDC is the native quote token everywhere.** The curve is priced in native USDC (`msg.value`), the graduation pool is native USDC / token. No wrapper, no other quote asset.
- **Storage is TimescaleDB.** We store decoded trades only, never raw transactions or receipts. Charts come from continuous aggregates.
- **1m is the only materialised base candle.** 5m/15m/1h/4h/1d are hierarchical continuous aggregates built on the 1m one.
- **RPC is QuickNode on testnet and mainnet**, public Circle endpoint as fallback. No self-hosted node.

## Ingest: own pull-based indexer in the Rust backend

**Build a small indexer in `yeet-backend` (alloy + sqlx), driven by `newHeads` over WebSocket, reading logs with `eth_getLogs`. No third-party indexer in the critical path.**

| Option | Verdict |
|---|---|
| **Own indexer, head-driven `eth_getLogs`** | ~1 day of Rust. Same binary on mainnet. No mainnet-support risk. Arc has no reorgs so the hard part of indexers (rollback) does not exist. **Chosen.** |
| `eth_subscribe(logs)` push only | A dropped WS silently loses events and you need a backfill path anyway. We use the head subscription as a *tick*, never as the source of truth. |
| Hosted indexer (Envio / Goldsky / The Graph) | All support testnet, none guarantee mainnet on day 1. Second system to run. Envio HyperSync stays an *optional* backfill accelerator (Rust client exists). |
| Circle "Contracts API" webhooks | Testnet only, vendor-managed, no mainnet commitment. No. |
| Own Arc node | See cost section. Post-launch improvement to remove the RPC dependency, not week 1. |

### Loop

```
Arc RPC ──WS newHeads──▶ tick ─▶ eth_getLogs(cursor+1..head, address=[Factory, Curve, PoolManager, tokens…]) ─▶ decode ─▶ Timescale
        ◀─HTTP fallback poll eth_blockNumber every 500 ms if WS is down                                              │
                                                                                                                      ▼
                                                        tokio broadcast ──▶ actix-web /ws (launches, trades:<token>, token:<token>) ──▶ Next.js
                                                        actix-web /api/*  (candles from caggs, recent trades, token stats)
```

1. **Cursor.** `indexer_state(chain_id, last_block)`. On boot, backfill from `max(last_block, deploy block)` to head in 1,000-block windows. On the RPC's 20,000-result cap error, halve the window and retry (the error includes the safe range).
2. **Tick.** Each `newHeads` → one `eth_getLogs` for `(cursor+1 .. head)` with the address list. Heads arriving mid-fetch collapse into one call. WS down → poll `eth_blockNumber` over HTTP every 500 ms, reconnect WS in the background. Backfill and live are the same function.
3. **Decode.** alloy `sol!` bindings from the ABIs the contracts repo emits. Contracts emit everything the indexer needs (reserves, price, amounts, fee, trader) so there is never an `eth_call` per event.
4. **Write.** One transaction per block range: insert trades (unique on `(tx_hash, log_index)`, so re-processing is idempotent), upsert token stats and holders, advance cursor, commit. Finality is instant, so nothing is ever deleted or rewritten.
5. **Fan-out.** After commit, publish the batch on `tokio::sync::broadcast`; the WS server forwards by topic. The frontend applies incoming trades to the open 1m candle locally, so live chart updates never hit the DB. Single process for launch; swap the channel for Postgres `LISTEN/NOTIFY` if we ever run a second API instance.

### Events indexed

| Source | Event | Drives |
|---|---|---|
| Factory | `TokenCreated(token, creator, name, symbol, metadataUri, curveParams)` | token list, launches feed |
| Curve | `Trade(token, trader, isBuy, usdcAmount, tokenAmount, fee, reserveUsdc, reserveToken, price)` | trades hypertable → all candles, mcap, volume |
| Curve | `Graduated(token, poolId, usdcLiquidity, tokenLiquidity)` | UI switches from curve to Uniswap |
| Every token | ERC-20 `Transfer` | holders table (top holders, dev share) |
| Uniswap v4 PoolManager | `Swap(id, …)` filtered by our pool ids | post-graduation trades → same hypertable, `source = 'uniswap'` |

Token addresses are dynamic: the indexer keeps the address filter in memory and appends on every `TokenCreated`; batches of ~500 addresses per `eth_getLogs` call once the list is large.

## Storage: TimescaleDB, decoded trades only

Docker image `timescale/timescaledb-ha` (Postgres 17 + Timescale 2.x) on the Hetzner box. One database per environment (`yeet_testnet`, `yeet_mainnet`).

### Tables

```sql
-- plain tables
chains(chain_id pk, name)
indexer_state(chain_id pk, last_block, updated_at)
tokens(address pk, chain_id, creator, name, symbol, metadata_uri, created_at, created_block,
       graduated_at, pool_id, reserve_usdc numeric(78,0), reserve_token numeric(78,0),
       price numeric(78,0), mcap numeric(78,0), holders int, last_trade_at timestamptz)
holders(token, holder, pk(token,holder), balance numeric(78,0), updated_block)

-- hypertable: the only fact table. one row per decoded Trade/Swap. no raw tx, no receipts, no calldata.
trades(
  ts          timestamptz not null,       -- block timestamp
  token       bytea       not null,       -- 20 bytes
  trader      bytea       not null,
  is_buy      bool        not null,
  usdc_amount numeric(78,0) not null,     -- 18-dec native units
  token_amount numeric(78,0) not null,
  price       numeric(78,0) not null,     -- USDC per token, 18-dec, post-trade
  fee         numeric(78,0) not null,
  block       bigint not null,
  tx_hash     bytea not null,
  log_index   int  not null,
  source      smallint not null,          -- 0 curve, 1 uniswap
  unique (tx_hash, log_index)
);
select create_hypertable('trades','ts', chunk_time_interval => interval '1 day');
create index on trades (token, ts desc);   -- recent trades per token, and cagg source scan
```

Compression policy on `trades` after 3 days (`segmentby = token`, `orderby = ts desc`). No retention policy for launch; trades are small (≈120 B/row) and we may want lifetime volume. Revisit if the table passes ~50 GB.

### Continuous aggregates

```sql
-- base: 1m, the only cagg computed from raw trades
create materialized view candles_1m with (timescaledb.continuous) as
select time_bucket('1 minute', ts) as bucket, token,
       first(price, ts)  as open,   max(price) as high,
       min(price)        as low,    last(price, ts) as close,
       sum(usdc_amount)  as volume_usdc,
       count(*)          as trades
from trades group by bucket, token;

-- hierarchical: each built on the previous, never on raw trades
candles_5m  = time_bucket('5 minutes',  bucket) over candles_1m
candles_15m = time_bucket('15 minutes', bucket) over candles_5m
candles_1h  = time_bucket('1 hour',     bucket) over candles_15m
candles_4h  = time_bucket('4 hours',    bucket) over candles_1h
candles_1d  = time_bucket('1 day',      bucket) over candles_4h
-- open = first(open, bucket), close = last(close, bucket), high = max(high), low = min(low), volume = sum, trades = sum
```

- `materialized_only = false` (real-time aggregation) on every level, so the open bucket is computed on read and the chart is always current without waiting for a refresh.
- Refresh policies: 1m every 1 min (`start_offset 1h`, `end_offset 1m`); 5m/15m every 5 min; 1h/4h every 1 h; 1d every 1 h. Timescale orders hierarchical refreshes correctly.
- Chart API `GET /api/tokens/:addr/candles?i=5m&from=&to=` reads exactly one cagg with an index-only scan on `(token, bucket)`. Pagination by `bucket`.
- Recent trades `GET /api/tokens/:addr/trades?limit=50` = index scan on `(token, ts desc)`, latest chunk only.
- Token "24h volume / 24h change / txns" = one query over `candles_1h` for the last 24 buckets, cached in the API for 5 s. No per-trade updates to the tokens row for these.

Why this beats hand-rolled candle upserts: the write path becomes a single append per trade, candles are guaranteed consistent with trades, adding a timeframe is one `create materialized view`, and the caggs survive reindexing (truncate `trades`, replay, refresh).

## The testnet → mainnet switch

One knob: `YEET_CHAIN=arc-testnet | arc-mainnet`. It selects a profile:

```toml
# chains/arc-mainnet.toml   (arc-testnet.toml has identical keys)
chain_id     = 5042
http_rpcs    = ["${QUICKNODE_ARC_HTTP}", "https://rpc.arc.io"]      # testnet profile: QuickNode testnet URL, then https://rpc.testnet.arc.io
ws_rpcs      = ["${QUICKNODE_ARC_WS}",   "wss://rpc.arc.io"]
explorer     = "https://explorer.arc.io"
deployments  = "deployments/5042.json"   # written by `forge script` in yeet-launchpad-evm
```

- `deployments/<chainId>.json` (addresses, deploy block, ABI hashes) is the **single source of truth**, produced by the contracts repo's deploy script and consumed by backend and frontend. Frontend switch is `NEXT_PUBLIC_YEET_CHAIN`.
- Secrets (QuickNode URLs, DB URL) come from the environment, never the profile.
- No code branches on chain id. USDC address, Uniswap PoolManager, explorer, faucet all live in profile/deployments.
- Unknowns to fill in on 16 Sep: official RPC/WS URLs, explorer URL, Uniswap v4 addresses (expected ones are in chain-facts; verify `eth_getCode` first). Switch = fill those, run deploy script, set `YEET_CHAIN`, restart. Runbook in step 5.

## RPC

- **Testnet:** QuickNode Arc testnet endpoint (HTTP + WSS) as primary so we exercise the exact provider path we ship with. Circle public endpoint as fallback (measured: no throttling at ~8 req/s, `eth_getLogs` capped at 20k results / ~1k blocks, WS subscriptions fine). Our load is ~2 calls/s.
- **Mainnet:** QuickNode primary (they are a listed Arc provider and the Uniswap team's Arc playbook uses them), Circle public as fallback. alloy `RetryBackoffLayer` plus a small failover wrapper rotating on 429/5xx/timeout. ≈175k calls/day, trivial on any paid tier.

## Own Arc node: rejected (8 Sep)

Considered running an Arc node on a Hetzner AX42-1 (€97.30/mo). Rejected as too much to operate in the launch window: testnet-only software today, ~250 GB snapshot, half-day to 1.5-day setup, and a second box to monitor. **QuickNode is primary on both testnet and mainnet, Circle public RPC is the fallback.** Revisit only if QuickNode cost or limits become a problem after launch; the failover list in the chain profile means it can be added later without code changes.

## Crate layout (yeet-backend)

```
Cargo.toml (workspace)
crates/
  chain/     alloy provider builder, chain profiles, failover, sol! bindings from deployments/*.json
  indexer/   cursor, backfill, head loop, decoders, writers
  api/       actix-web REST + WS, broadcast subscriber, 5 s stats cache
  db/        sqlx migrations (hypertable, caggs, policies) + typed queries
bin/yeet     clap: --role indexer|api|all, reads YEET_CHAIN
chains/arc-testnet.toml, chains/arc-mainnet.toml
deployments/<chainId>.json (copied from yeet-launchpad-evm by CI)
```

Deps: `alloy` (provider, ws, sol-types), `tokio`, `actix-web`, `sqlx` (postgres, runtime-tokio, bigdecimal), `serde`, `tracing`, `clap`.

## Acceptance test (testnet)

1. Deploy contracts to testnet, note deploy block.
2. `YEET_CHAIN=arc-testnet yeet --role all` against an empty Timescale. It backfills to head and logs `caught up`.
3. Send a buy. Trade appears on `/ws` within 1 s; `/api/tokens/<addr>/candles?i=1m` and `?i=1d` both reflect it immediately (real-time cagg).
4. Block the WS host. Indexer switches to HTTP polling and keeps indexing. Restore; it reconnects.
5. Stop the binary 5 minutes, send trades, restart. Gap backfilled, no duplicates (row count = trades sent), caggs refresh and match.
6. `YEET_CHAIN=arc-mainnet` with a dummy deployments file fails fast with a clear error.
