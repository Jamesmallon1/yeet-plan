# Step 5 — Devops (`yeet-devops`)

Decided 8 Sep: **Hetzner Cloud Ashburn (us-east)**, two CCX33 (8 dedicated vCPU / 32 GB / 240 GB NVMe, $166/mo each), Cloudflare for everything user-facing. All of it is code in `yeet-devops` and validated (`terraform validate` passes against hcloud 1.68 / cloudflare 5.24; cloud-init templates render to valid YAML under the 32 KiB limit).

## Topology

```
users ──▶ Cloudflare edge: DNS · TLS · WAF custom rules · rate limits · bot score · DDoS · 1 s API cache · Pages (frontend) · R2 (images)
                │  outbound-only QUIC tunnel from the monolith (no inbound ports anywhere)
                ▼
  Hetzner private network 10.10.1.0/24 (Ashburn)
  ├─ monolith  10.10.1.10  CCX33  yeet-backend (127.0.0.1:8080) · app Postgres (unix socket) · cloudflared · prometheus · grafana
  └─ timescale 10.10.1.20  CCX33  TimescaleDB (binds private IP only) · exporters · no public IPv4
```

## "Only through Cloudflare", enforced three times

| Layer | Monolith | Timescale |
|---|---|---|
| Hetzner Cloud firewall (public NIC) | inbound: ICMP only (optional break-glass SSH from `emergency_admin_cidrs`) | inbound: ICMP only; **no public IPv4 at all** (IPv6 for apt/docker egress) |
| nftables on the host | default drop; lo + established + 22 | default drop; 5432/9100/9187/22 **only from 10.10.1.10** |
| Application | backend listens on 127.0.0.1 only; reached solely via cloudflared; trusts `CF-Connecting-IP` | Postgres `pg_hba`: two roles, from 10.10.1.10/32, scram only; everything else `reject` |

SSH and Grafana are published through the same tunnel behind **Zero Trust Access** (email allow-list). `scripts/ssh.sh` wraps `cloudflared access ssh`; the Timescale box is reached by ProxyJump through the monolith over the private net.

## Cloudflare rules (Terraform, `cloudflare_edge.tf`)

- Zone: SSL strict, TLS ≥ 1.2, always-HTTPS, WebSockets on, Brotli.
- Custom WAF (all plans): api host accepts only `/api/v1/*` + `/healthz` and GET/POST/OPTIONS; ws host only `/ws`. Bot protection on `/tokens/launch` is **Turnstile** (frontend widget, backend verifies the token) because a WAF challenge cannot be solved by a `fetch()` call and bot score is Enterprise-only.
- Rate limits: launch 5/min/IP, api 300/min/IP, ws 30 new connections/min/IP. **Free plan allows 1 rate-limit rule, Pro 2, Business 5** → plan on Pro ($20/mo); it also unlocks managed WAF (`enable_managed_waf = true`: Cloudflare Managed + OWASP rulesets) and bot score.
- Cache rule: API GETs cached at the edge per origin `Cache-Control` (1 s default), except requests carrying `wallet_address=`.
- R2 bucket `yeet-images-prod` with custom domain `images.yeet.family`, TLS ≥ 1.2.
- Pages project `yeet-frontend` (next-on-pages build, `nodejs_compat`), domains `yeet.family` + `www`, env vars `NEXT_PUBLIC_YEET_CHAIN/API_URL/WS_URL/IMAGES_URL` for production and preview.

## Servers (cloud-init, runs once)

Both: Ubuntu 24.04, docker + compose v2, nftables, chrony, sshd key-only, docker log rotation, `live-restore`.

Monolith extras: sysctl for 100k+ sockets (`fs.nr_open` 2M, `somaxconn` 65535, conntrack 1M, tcp buffers), `nofile` 1M for root and containers, compose stack (backend, appdb, cloudflared, prometheus, grafana, node-exporter), `/opt/yeet/backend.env` generated from Terraform vars (chain, RPC URLs, DB URLs, Pinata, R2), nightly `pg_dump` of the app DB to R2.

Timescale extras: hugepages for 8 GB `shared_buffers`, `postgresql.conf` tuned for 8 vCPU / 32 GB / NVMe (`effective_cache_size` 24 GB, `random_page_cost` 1.1, zstd WAL compression, 16 Timescale background workers), roles `yeet_indexer` (write, `synchronous_commit=off`) and `yeet_api` (read-only, 2 s statement timeout) created on first boot, data on the local NVMe.

## Scripts

| Script | Does |
|---|---|
| `bootstrap.sh` | checks secrets, `terraform init/validate/plan/apply` |
| `ssh.sh monolith\|timescale [cmd]` | SSH via the Access-gated tunnel |
| `deploy.sh <tag>` | rolls the backend image, waits for `/healthz` |
| `switch-chain.sh arc-mainnet <tag>` | **the flip**: sets `YEET_CHAIN`, deploys, reminds to `terraform apply` for the Pages env |
| `scale-tunnel.sh N` | more cloudflared replicas = more WS capacity |

CI: PRs to `yeet-devops` run `terraform fmt -check` + `validate`. Apply stays manual (secrets stay on your machine).

## Costs

| Item | Monthly |
|---|---|
| 2 × CCX33 Ashburn | $332 |
| 2 public IPv4 (monolith v4, timescale none) | ~$1 |
| Cloudflare Pro (rate-limit rules, managed WAF, bot score) | $20 |
| R2 (images + backups + tf state) | < $1 |
| QuickNode | plan-dependent |
| **Total infra** | **~$355 + RPC** |

## Go-live runbook (16 Sep)

1. Circle publishes mainnet params → fill `chains/arc-mainnet.toml` in yeet-backend, verify `eth_getCode` on the official Uniswap v4 PoolManager.
2. `forge script Deploy --rpc-url arc-mainnet` → `deployments/5042.json` → commit → backend image tag `mainnet-1`.
3. `scripts/switch-chain.sh arc-mainnet mainnet-1` (fresh DBs: indexer replays from the deploy block, seconds).
4. `terraform apply -var yeet_chain=arc-mainnet` → Pages env → redeploy frontend.
5. Smoke: create a token with a 1% dev buy, buy, sell, claim, watch WS ticks, check Grafana.
6. Testnet stays reachable as a preview deployment pointing at a second backend only if we want it; otherwise testnet is retired.

## Open items before `bootstrap.sh`

- Cloudflare API token scopes, R2 state bucket + token, Hetzner project token.
- GitHub PAT (`read:packages`) so servers can pull the private backend image.
- Decide Cloudflare plan (Pro recommended). On Free, delete two of the three rate-limit rules.
- R2 API token for image uploads goes into `backend.env` after first boot (one manual step, documented).
