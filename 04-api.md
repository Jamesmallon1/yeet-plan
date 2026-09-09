# Step 4 — API definition (`yeet-backend`)

Unauthenticated. Rate limiting and bot filtering at Cloudflare (per-IP rules on `/api/v1/*`, stricter on `/tokens/launch`, WS connection rate per IP). Prefix `/api/v1`. JSON, `snake_case`, all USDC amounts as decimal strings in whole USDC (6 dp), token amounts as decimal strings, addresses lowercase hex, timestamps as unix seconds.

## Two databases

| | Where | Holds | Access |
|---|---|---|---|
| **app Postgres** (small, Postgres 17) | on the monolith box, unix socket | `tokens`, `token_stats`, `holders`, `dividends`, `metadata_uploads`, `indexer_state` | ~0.1 ms, serves every list/detail endpoint |
| **TimescaleDB** | second AX42-1 over vSwitch | `trades` hypertable + caggs `candles_1m…1d` | ~0.3 ms, serves charts, trades, and the stats refresher |

A **stats refresher** task runs every 5 s: one query over `candles_1h` (last 24 buckets, grouped by token) + `candles_1m` (last bucket) on Timescale → upserts `token_stats(token, price, mcap, vol_24h, change_24h, txns_24h, updated_at)` into app Postgres. `/tokens/all` therefore never touches Timescale and sorts on indexed columns.

## REST

### `GET /api/v1/tokens/all`

| Query | Type | Default | Notes |
|---|---|---|---|
| `sort` | `mcap` \| `newest` \| `vol_24h` | `mcap` | desc |
| `q` | string ≤ 64 | – | matches `name` or `symbol`, case-insensitive, prefix + substring (`pg_trgm` GIN) |
| `page` | int ≥ 1 | 1 | |
| `limit` | 1–100 | 50 | |
| `wallet_address` | address | – | if present, `your_pending_rewards` is filled |
| `status` | `all` \| `curve` \| `graduated` | `all` | optional filter |

Response `200`:
```json
{
  "page": 1, "limit": 50, "total": 1234,
  "items": [{
    "token_address": "0x…",
    "name": "Yeet Cat", "symbol": "YCAT",
    "image_url": "https://images.yeet.family/bafy…/512.webp",
    "market_cap": "48571.23",
    "price": "0.0000486",
    "vol_24h": "12034.50",
    "change_24h_pct": 37.2,
    "status": "curve" | "graduated",
    "curve_progress_pct": 63.1,
    "pair_address": "0x…",            // graduated: Uniswap v4 poolId (bytes32); curve: launchpad contract address
    "pair_type": "uniswap_v4" | "launchpad",
    "tax_bps": 300,
    "dollar_rewards_distributed": "1520.44",   // Σ DividendNotified, USDC
    "your_pending_rewards": null,              // deprecated: the client reads claimable() itself via Multicall3 (no server RPC per user)
    "reserves": { "venue": "curve"|"uniswap_v4", "fee_bps": 330, "v_usdc": "…", "v_token": "…", "curve_remaining": "…",
                  "sqrt_price_x96": "…", "liquidity": "…", "launchpad": "0x…", "router": "0x…" },   // raw 18-dec ints; client quotes locally (src/lib/quote.ts), updated on every tick
    "created_at": 1788885479,
    "creator": "0x…",
    "links": { "website": "…", "twitter": "…", "telegram": "…" }
  }]
}
```
Note on `pair_address`: Uniswap v4 has no pair contract; a pool is a `bytes32` id inside PoolManager. We return the id in that field so the frontend can deep-link (`explorer/pool/<id>`) and the field name stays as you asked.

`your_pending_rewards`: one `Multicall3.aggregate3` `eth_call` for the page's tokens (`claimable(wallet)` on each), cached 5 s per `(wallet, page)`. Everything else is one indexed query on app Postgres. Whole response cached 2 s in-process and `Cache-Control: public, max-age=1` at Cloudflare (with `wallet_address` present: `private`).

### `GET /api/v1/tokens/{address}`
Same shape as one list item plus `description`, `metadata_uri`, `reserve_usdc`, `tokens_sold`, `holders_count`, `top_holders[10]{address, balance, pct}`, `graduated_at`, `pool_id`, `sqrt_price_x96`.

### `GET /api/v1/tokens/{address}/trades?cursor=&limit=50`
Cursor = `ts:log_index` of the last item. `{ items:[{ts, tx_hash, trader, side:"buy"|"sell", usdc_amount, token_amount, price, source:"curve"|"uniswap"}], next_cursor }`. Timescale, latest chunk, `(token, ts desc)`.

### `GET /api/v1/tokens/{address}/candles?tf=1m|5m|15m|1h|4h|1d&before=&limit=300`
History paging for the chart when the user scrolls left. `{ items:[{t, o, h, l, c, v, n}], next_before }`. One cagg, index-only.

### `GET /api/v1/tokens/{address}/holders?page=&limit=`
### `GET /api/v1/tokens/{address}/dividends/{wallet}`
`{ claimable, claimed_total, claims:[{ts, tx_hash, amount}] }`. `claimable` via `eth_call`, rest from app Postgres.

### `GET /api/v1/tokens/{address}/quote?side=buy|sell&amount=`
Curve: closed-form in the service. Graduated: `router.quote` via `eth_call`. `{ amount_out, price_impact_pct, protocol_fee, dividend, min_out_default }`.

### `POST /api/v1/tokens/launch`  (multipart/form-data)

| Field | Rule |
|---|---|
| `name` | 1–32 chars, printable |
| `symbol` | 1–10 chars, `[A-Z0-9]` |
| `description` | ≤ 500 chars |
| `website`, `twitter`, `telegram` | optional, `https://` URLs, host allow-list for twitter (`x.com`, `twitter.com`) and telegram (`t.me`) |
| `tax_bps` | `0 \| 100 \| 300` |
| `dev_buy_bps` | `0–5000` |
| `creator` | address |
| `image` | ≤ 2 MB, png/jpeg/gif/webp |

Returns `200` `{ "metadata_uri": "ipfs://bafy…", "image_url": "https://images.yeet.family/bafy…/512.webp", "calldata_hint": { "tax_bps": 300, "dev_buy_bps": 500 } }`.
The backend does **not** send a transaction. The frontend then calls `launchpad.createToken(name, symbol, metadata_uri, tax_bps, dev_buy_bps)` from the user's wallet; the indexer picks up `TokenCreated` and the token appears in `/tokens/all` on the next block. An upload not followed by a `TokenCreated` within 1 h is unpinned and deleted (`metadata_uploads` table).

Errors: `400 {code:"invalid_image"|"image_too_large"|"invalid_symbol"|…, message}`, `429` from Cloudflare.

#### Image upload safety

The monolith never stores or serves user-supplied bytes. Pipeline, all in-process (Rust `image` crate is memory-safe, no ImageMagick):

1. Cloudflare: 2 MB body limit on this route, WAF, rate limit 5/min/IP.
2. Read into memory, **sniff magic bytes** (ignore the client's `Content-Type` and filename); reject anything but PNG/JPEG/GIF/WebP.
3. Decode with hard limits: max 4096×4096, max 64 MB decoded, first frame only for GIF (animated GIFs are flattened; animated output is a v2 nicety).
4. **Re-encode** to fresh WebP at 512×512 (cover-crop) and 64×64. Re-encoding from raw pixels discards EXIF, ICC, comments, appended data, polyglot payloads and any exploit that lives in the container. Only these two generated files exist from here on.
5. Upload the two WebPs + the metadata JSON to **Pinata** (IPFS pinning; that is the service you were thinking of) and mirror the WebPs to **Cloudflare R2** behind `images.yeet.family` with forced `Content-Type: image/webp`, `X-Content-Type-Options: nosniff`, `Content-Disposition: inline`, immutable cache. Separate origin from the app, so even a hypothetically bad file cannot run in the app's origin.
6. Original bytes are dropped. Nothing is written to the Hetzner disk.
Alternative if you want zero image bytes on our box: frontend uploads straight to Pinata with a scoped one-time key we mint (`POST /api/v1/uploads/key`); we still re-encode server-side from the CID before publishing the metadata, so the safety step stays the same.

## WebSocket  `wss://ws.yeet.family/ws`

One connection per browser tab; JSON text frames; server `ping` every 20 s, close after 2 missed. Client rules: subscribe on entering a token page, unsubscribe on leaving. A connection may hold ≤ 10 subscriptions.

Client → server
```json
{ "op": "subscribe",   "token": "0x…", "tf": "1m", "bars": 300, "trades": 50 }
{ "op": "unsubscribe", "token": "0x…" }
{ "op": "set_tf",      "token": "0x…", "tf": "5m", "bars": 300 }      // switch timeframe, gets a new snapshot
{ "op": "subscribe_list" } / { "op": "unsubscribe_list" }                // home page: launches + stats ticks
```

Server → client. **Candles travel as binary frames (f32); everything else is JSON text frames.**

Binary frame layout (little-endian, no padding):

```
header (28 bytes)
  u8   op          0x01 = snapshot candles, 0x02 = tick candles
  u8   tf          1=1m 2=5m 3=15m 4=1h 5=4h 6=1d
  u16  reserved    0
  [20] token       address bytes
  u32  count       number of candle records that follow
record (28 bytes) × count, oldest first
  u32  t           bucket start, unix seconds
  f32  o, h, l, c  USDC per token
  f32  v           USDC volume in the bucket
  u32  n           trade count
```
300 bars = 8,428 bytes vs ~30 KB as JSON. Frontend: `new DataView(buf)` / `Float32Array` straight into the chart series, zero parsing. f32 keeps ~7 significant digits, plenty for display at any price magnitude; the DB stays exact and all percentage math is done server-side.

JSON frames:
```json
{ "op": "snapshot", "token": "0x…", "tf": "1m",
  "token_info": { …same as list item… },
  "trades":  { "items": [ … ], "next_cursor": "…" } }   // newest first; older pages via REST
// immediately followed by one binary 0x01 frame with the last `bars` candles

{ "op": "tick", "token": "0x…", "tf": "1m", "ts": 1788885485,
  "trades":  [ … ],                        // trades since the last tick (may be empty), newest first
  "stats":   { "price": "…", "market_cap": "…", "vol_24h": "…", "change_24h_pct": 12.3, "curve_progress_pct": 64.0,
               "dollar_rewards_distributed": "…" } }
// immediately followed by one binary 0x02 frame: the open bucket plus any bucket that closed since the last tick

{ "op": "event", "token": "0x…", "type": "graduated" | "created", … }   // rare, immediate
{ "op": "list_tick", "ts": …, "new_tokens": [ … ], "stats": [ {token_address, price, market_cap, vol_24h, change_24h_pct}, … ] }  // every 5 s, only changed tokens
{ "op": "error", "code": "bad_token" | "too_many_subs" | "bad_tf", "message": "…" }
```

### Server side

- Subscriptions are keyed by `(token, tf)`. For each key with ≥ 1 subscriber a **ticker task** fires every 5 s, aligned to the wall clock (`:00, :05, :10`). It reads the open bucket (and any newly closed bucket) from the cagg once, drains that token's trade ring buffer (fed in-memory by the indexer bus, no DB), and builds **one** `tick` frame that is serialised once and sent to every subscriber of that key. Cost is per `(token, tf)`, not per client: 10 000 viewers of one token = 1 query and 1 serialisation per 5 s.
- `snapshot` is served from the cagg + trades table on subscribe (~2 queries, < 5 ms). Candle rows are read as `f32` straight from the query into a preallocated `Vec<u8>`; one allocation per frame.
- Trades arriving between ticks are not pushed individually; they ride the next tick. This is the "candles move every 5 s" behaviour you asked for and caps outbound to one frame per subscription per 5 s.
- `graduated`/`created` events are pushed immediately because the UI must swap the trading panel.
- Backpressure: per-connection outbound queue of 64 frames; a client that cannot keep up is disconnected, not buffered.

## Concurrent WebSockets on one AX42-1

| Constraint | Number | Basis |
|---|---|---|
| Memory | ~20–40 KB per idle connection in actix-ws/tokio (task + buffers + TLS-free, TLS terminates at Cloudflare) → **100 k connections ≈ 2–4 GB** | 64 GB box |
| File descriptors | raise `nofile` to 1 048 576, `somaxconn`/`tcp_max_syn_backlog` 65 535, `net.ipv4.ip_local_port_range` irrelevant server-side | kernel |
| Outbound bandwidth | tick ≈ 1.5 KB; 100 k subscriptions × 1 frame / 5 s = 30 MB/s ≈ **240 Mbit/s** | 1 Gbit unmetered |
| CPU | 20 k frames/s of pre-serialised bytes is a few percent of 16 threads | |
| cloudflared | the tunnel proxies every socket; run 2–4 `cloudflared` replicas on the box and it comfortably carries 50 k+ each in practice | the least-documented limit; measure |

Realistic answer: **50 000 concurrent viewers per box is comfortable, 100 000+ is achievable with the sysctl tuning above and a couple of tunnel replicas, and the first thing to run out is Cloudflare tunnel throughput, not the machine.** We set a hard cap of 100 000 accepted sockets and expose the count as a metric. A second AX42-1 behind the same tunnel hostname doubles it; the WS layer is stateless except for the per-key tickers, which each box runs independently.

For scale: pump.fun's biggest days were on the order of tens of thousands of concurrent users site-wide.
