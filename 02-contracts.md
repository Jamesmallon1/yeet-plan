# Step 2 — Contracts (`yeet-launchpad-evm`)

Foundry, Solidity 0.8.26+, EVM target `cancun` (v4-core needs transient storage; Arc runs Osaka, a superset). Inspiration: pump.fun (curve shape and graduation split), four.meme `TokenManager` (single launchpad contract holding every curve), nad.fun (10k quote-token graduation on an EVM L1), Clanker (Uniswap v4 graduation with a permanently locked LP whose fees are claimable). Nothing is copied; the math and mechanisms are re-derived below.

## Product rules (fixed, 8 Sep)

| Rule | Value |
|---|---|
| Supply | 1,000,000,000, 18 decimals, minted once at creation, never again |
| Quote asset | native USDC (`msg.value`, 18-dec on chain), permanently |
| Graduation | when the curve has collected **10,000 USDC** (net of fees) |
| Protocol fee | **0.3%** of the USDC side of every trade, on the curve and after graduation |
| **Holder dividends** | creator picks **0%, 1% or 3%** at launch; immutable; charged in **USDC** on the USDC side of **every trade** (curve and Uniswap, any router); paid to holders pro-rata in **USDC**. Wallet-to-wallet transfers are free |
| Creator fee | none. Creator earns only as a holder |
| Dev buy | creator may buy up to 50% of supply in the creation transaction, paying USDC like anyone else |
| Metadata | name, symbol, image, website, X, Telegram. On-chain: `metadataURI` (IPFS JSON). Immutable |
| Creation cost | gas only (≈ $0.05 on Arc). No creation fee |

## Contracts

```
src/
  YeetLaunchpad.sol     singleton: createToken, buy, sell, graduate, fee withdrawal. Holds every curve's state
  YeetToken.sol         plain ERC-20 + USDC dividend ledger (accumulator, claim)
  YeetHook.sol          Uniswap v4 hook, one per chain: takes protocol fee + dividend in USDC on every swap of every graduated pool
  YeetGraduator.sol     v4 IUnlockCallback: init pool with the hook, add full-range liquidity, hold the position forever, claim LP fees
  YeetRouter.sol        post-graduation swaps for our UI: native USDC in/out, quote(), optional auto-claim
  libs/CurveMath.sol    pure functions, fuzzed
script/Deploy.s.sol     writes deployments/<chainId>.json (addresses + deploy block)
```

Indexer watches: launchpad, hook, every token's `Transfer` and `DividendClaimed`, PoolManager `Swap` filtered by our pool ids.

## 1. Bonding curve (`YeetLaunchpad`)

Constant-product with virtual reserves, pump.fun's shape re-parameterised for a 10,000 USDC raise.

```
V_U  = 3,500 USDC            initial virtual USDC reserve
R    = 10,000 USDC           real USDC collected at graduation (net of fees)
S    = 1e9 · (V_U+R)/(2V_U+R) = 794,117,647 tokens sold on the curve
L    = 1e9 − S               = 205,882,353 tokens reserved for the pool
V_T  = S · (V_U+R)/R         = 1,072,058,824 initial virtual token reserve
k    = V_U · V_T
```

`S` is derived, not round: it is the unique split for which the pool can be seeded with *all* of `R` and *all* of `L` at exactly the curve's final price, so there is no price gap at graduation (pump.fun's 793.1M / 206.9M is the same identity with 30 / 85 SOL).

| | Price (USDC) | Market cap |
|---|---|---|
| Launch | 3.26e-6 | $3,265 |
| Graduation | 4.86e-5 | $48,571 |
| Multiple | 14.9× | (pump.fun 14.7×) |

Dev buy cost at launch: 1% = 33 USDC, 5% = 171, 10% = 360, 20% = 803, 50% = 3,059 (plus fees).

State per token: `realUsdc`, `tokensSold`, `taxBps`, `creator`, `graduated`, `poolId`. Virtual reserves: `vU = V_U + realUsdc`, `vT = V_T − tokensSold`.

**buy(token, minTokensOut)** payable
1. `protocolFee = msg.value·30/1e4`, `dividend = msg.value·taxBps/1e4`, `in = msg.value − protocolFee − dividend`
2. `out = vT − k/(vU + in)`; if `out` exceeds the curve remainder, clamp, recompute `in` and both fees on the used amount, refund the rest of `msg.value`
3. transfer `out` tokens to buyer (plain transfer, no tax)
4. `token.notifyDividend{value: dividend}()` (after the transfer, so the buyer is already a holder; on the very first buy they simply get their own dividend back)
5. update state, emit `Trade(token, buyer, true, in, out, protocolFee, dividend, vU', vT', price)`
6. if `tokensSold == S` → mark complete, `try graduate(token)`; the buy succeeds even if graduation reverts; `graduate` stays permissionless

**sell(token, amountIn, minUsdcOut)**
1. `transferFrom(seller, launchpad, amountIn)` (plain)
2. `gross = vU − k/(vT + amountIn)`; `protocolFee`, `dividend` from `gross`; pay `gross − fees` to seller via native `call`
3. `token.notifyDividend{value: dividend}()` after the seller's balance is reduced (they don't earn on their own exit)
4. update state, emit `Trade(…, false, …)`

Native sends to a blocklisted address revert on Arc; the sell reverts, which is correct. All rounding is against the user; `k ≈ 3.75e48` fits uint256 with `mulDiv`.

**createToken(name, symbol, metadataURI, taxBps ∈ {0,100,300}, devBuyBps ≤ 5000)** payable
- deploys `YeetToken` via CREATE2 (salt = keccak(creator, nonce)), mints 1e9 to the launchpad
- registers dividend exclusions on the token: launchpad, graduator, PoolManager, hook, `0xdead`
- emits `TokenCreated(token, creator, name, symbol, metadataURI, taxBps)`
- if `devBuyBps > 0`, runs `buy` with `msg.value` in the same tx; must yield ≥ `devBuyBps` of supply or revert

## 2. Token with USDC dividend ledger (`YeetToken`)

The token itself is a **plain ERC-20** (no transfer tax, no fee-on-transfer, no rebasing), which keeps every wallet, explorer, DEX and indexer integration standard. Bolted on is a USDC dividend ledger, Synthetix-style, O(1) per operation:

- `notifyDividend()` payable, callable only by the launchpad and the hook: `dividendPerShare += msg.value·1e18 / eligibleSupply`. If `eligibleSupply == 0` the amount is parked and folded into the next notify.
- `eligibleSupply = totalSupply − Σ balanceOf(excluded)`; excluded = launchpad (unsold curve tokens), graduator, PoolManager (LP tokens), hook, dead. Exclusions are set once at creation and frozen; the token has no owner.
- `_update(from, to, amount)` checkpoints both parties first: `owed[a] += balance[a]·(dividendPerShare − paid[a])/1e18; paid[a] = dividendPerShare`. Then the plain balance move. Excluded addresses are skipped, and `eligibleSupply` is adjusted when tokens move to/from an excluded address.
- `claimable(a)` view; `claim()` sends the caller's accrued native USDC via `call`. **Pull only, by design (decided 8 Sep): users press a Claim button in the UI.** No pushes inside `_update`, no `claimFor`, no keeper, no router auto-claim. Simplest contract, no gas paid by the protocol, no blocklist edge cases.
- USDC held by the token contract is exactly Σ unclaimed. `totalSupply` never changes.
- Events: `DividendNotified(uint256 amount, uint256 dividendPerShare)`, `DividendClaimed(address indexed holder, uint256 amount)`.
- `taxBps == 0` tokens skip the ledger entirely and cost plain-ERC-20 gas.

Gas: a taxed *trade* pays ~40–60k extra for the ledger + notify. A plain transfer between wallets pays ~5k extra for the two checkpoints. On Arc that is well under a cent. The protocol never pays gas for distribution; there is no keeper and no loop over holders.

## 3. Uniswap v4: how it works and what we run

v4 is one contract, the **PoolManager**, holding every pool and every token balance for the whole chain. There are no per-pair contracts.

- A pool is identified by its `PoolKey = (currency0, currency1, fee, tickSpacing, hooks)`; `poolId = keccak(key)`. Anyone can `initialize` any key. Two keys that differ only in `hooks` are two different pools.
- Native currency is `address(0)`. Settling native means `settle{value}`; paying out native means `take`. No wrapper, which matches Arc where USDC *is* the native asset.
- **Flash accounting.** Every interaction happens inside `PoolManager.unlock(data)`, which calls back `unlockCallback(data)` on the caller. Inside, you `swap`, `modifyLiquidity`, `settle`, `take` as many times as you like; at the end every currency delta must be zero or the whole thing reverts. Deltas live in transient storage (EIP-1153), which is why v4 needs Cancun+.
- **Hooks.** A pool key can name a hook contract. PoolManager calls it before/after initialize, swap, liquidity changes, and a hook can *return a delta*, i.e. take part of a swap's input or output for itself. Which callbacks a hook receives is encoded in the **low 14 bits of its address**, so hooks are deployed with CREATE2 and a mined salt (`HookMiner` from v4-periphery). PoolManager checks the bits at `initialize`.
- Positions in core are keyed by `(owner, tickLower, tickUpper, salt)`; the LP NFT is a periphery (PositionManager) convenience we do not need.

### 3a. `YeetHook`, the venue-independent fee taker

One hook per chain, named in the key of every pool we graduate. Permissions: `beforeInitialize`, `beforeSwap` + `beforeSwapReturnDelta`, `afterSwap` + `afterSwapReturnDelta`.

- `beforeInitialize`: revert unless `sender == graduator` and `currency1` is a registered YeetToken. Nobody can attach our hook to a random pool and make it call `notifyDividend` on something else.
- Fee rule: `protocol 0.3% + dividend taxBps(token)` taken from the **USDC side** of every swap:
  - buy (exact-in USDC): USDC is the *specified* currency → `beforeSwap` returns a delta that skims the fee off the input before the pool sees it
  - sell (exact-in token): USDC is the *unspecified* output → `afterSwap` returns a delta that skims the fee off the output
  - exact-output swaps: mirrored (after / before). Same two code paths, chosen by which side USDC is on.
- Inside the callback the hook `take`s the skimmed USDC to itself, forwards `dividend` to `token.notifyDividend{value}` and keeps `protocol` in `accruedFees` for pull-withdrawal by the treasury.
- Because the hook is part of the pool key, **every router, aggregator or Uniswap's own UI pays it**. Dividends after graduation do not depend on users using our frontend.
- Pool `fee` field is set to 0: the LP fee is replaced by the hook's protocol fee, so the total cost to a trader is 0.3% + dividend, same as on the curve.

Gas: ~30–50k per swap on top of a v4 swap.

Trade-off to know: Uniswap's routing API and app are conservative about pools with unknown hooks and may not route through ours (unverified). Our own UI is the primary venue; the pool remains fully permissionless for anyone who talks to PoolManager directly.

### 3b. `YeetGraduator`

`graduate(token)` (anyone, once the curve is complete):
1. launchpad sends `R` USDC and `L` tokens to the graduator
2. `initialize(key{0, token, fee 0, spacing 60, hooks: YeetHook}, sqrtPriceX96(p_f))`, `p_f = (V_U+R)/(V_T−S)`; both sides 18-dec so no decimal shift
3. `unlock` → `modifyLiquidity` full range (`MIN_TICK..MAX_TICK` rounded to spacing) sized from `R` and `L`; `settle{value: R}` for USDC, `sync / transfer / settle` for the token
4. tick-rounding dust is burned
5. emit `Graduated(token, poolId, usdcIn, tokensIn, sqrtPriceX96)`; launchpad flips `graduated`, curve closes

The graduator owns the position inside PoolManager and has **no function to remove liquidity** → locked forever. With pool fee 0 there are no LP fees to claim; the protocol's cut comes from the hook. (If we ever want LPs to earn, set pool fee > 0 and the graduator's `claimFees` becomes meaningful; keep the function, it is 10 lines.)

Graduation gas is paid by the final buyer (≈1.2M gas ≈ $0.03), atomic, with the try/catch + permissionless fallback covering failure.

### 3c. What "keeping v4 going" means operationally

| Component | Testnet | Mainnet | Ongoing work |
|---|---|---|---|
| PoolManager | we deploy `v4-core` once (explicit gas limit; `eth_estimateGas` is unreliable on Arc) | official `0x8366…40951`, verify `eth_getCode` on launch day | none; it is a permissionless singleton nobody maintains |
| YeetHook | deploy once with mined salt | same, new salt (flags must match on both chains) | none; immutable. A new hook version only affects *future* graduations, old pools keep the old hook |
| YeetGraduator | deploy once | deploy once | none; positions are locked, no admin |
| Fees | `withdraw(to)` on launchpad and hook | same | backend cron: call `withdraw` when accrued > threshold, or do it by hand weekly |
| Liquidity | our locked full-range position seeds every pool | same | none required. Anyone can add more liquidity through PositionManager on mainnet |
| Indexing | PoolManager `Swap` events filtered by our pool ids; hook `FeesTaken` events | same | already in step 1 |

Nothing runs off-chain to keep pools alive. The only recurring task is sweeping protocol fees.

## 4. Post-graduation swaps (`YeetRouter`)

Uniswap's UniversalRouter is not on testnet and its mainnet address is unpublished, so we ship a thin router for our UI. The token is plain now, so no fee-on-transfer gymnastics:

- `buy(token, minOut)` payable: `unlock` → `swap` exact-in native → `settle{value}` → `take(token, user)`; hook has already skimmed the fee.
- `sell(token, amountIn, minOut)`: `transferFrom` user → PoolManager path via `sync / transfer / settle` → `swap` exact-in → `take(native, user)`.
- `quote(token, isBuy, amount)` view: closed-form constant-product from `slot0` + `liquidity` (single full-range position) minus hook fees. No Quoter contract.

## 5. Fees & admin

- Protocol fees accrue in the launchpad (curve trades) and the hook (pool trades). `withdraw(to)` by `owner`. Pull only.
- `owner` = hardware-wallet EOA with `Ownable2Step` (no Safe on Arc yet). Can withdraw fees, set treasury, pause **creation**. No upgradeability anywhere. Curve constants are `immutable`.

## 6. Token image & metadata

EVM convention (pump.fun, Zora, Clanker, four.meme): a JSON document on IPFS referenced from the contract.

```json
{ "name": "…", "symbol": "…", "description": "…",
  "image": "ipfs://<cid>",
  "website": "…", "twitter": "…", "telegram": "…" }
```
Frontend uploads image + JSON to the backend; backend pins both to IPFS via Pinata and mirrors the image to Cloudflare R2 (`images.yeet.family/<cid>`). The contract stores only `ipfs://<metadataCid>`. The indexer resolves it on `TokenCreated`. Immutable.

## 7. Events

```
TokenCreated(address indexed token, address indexed creator, string name, string symbol, string metadataURI, uint16 taxBps)
Trade(address indexed token, address indexed trader, bool isBuy, uint256 usdcAmount, uint256 tokenAmount,
      uint256 protocolFee, uint256 dividend, uint256 vUsdc, uint256 vToken, uint256 price)
Graduated(address indexed token, bytes32 indexed poolId, uint256 usdcIn, uint256 tokensIn, uint160 sqrtPriceX96)
FeesTaken(bytes32 indexed poolId, address indexed token, bool isBuy, uint256 protocolFee, uint256 dividend)   // hook
DividendNotified(uint256 amount, uint256 dividendPerShare)                                                    // token
DividendClaimed(address indexed holder, uint256 amount)                                                      // token
FeesWithdrawn(address to, uint256 amount)
```

## 8. Testing

- Unit + fuzz: `CurveMath` (monotonic, round-trip never profitable, `k` conserved within rounding); dividend ledger invariants (`Σ claimable + Σ claimed == Σ notified`, `eligibleSupply == totalSupply − Σ excluded`, no holder can claim more than their pro-rata share); full curve walk with random tax/dev-buy.
- Hook: fee taken equals the stated bps for exact-in/out buys and sells; pool price after a hooked swap matches the closed-form quote; `beforeInitialize` rejects foreign pools.
- Fork tests against **Arc testnet RPC**, not Anvil: native `settle{value}`, graduation end to end, router buy/sell, hook skim on native, blocklist-revert path (mocked).
- Static: `slither`, `forge coverage` ≥ 90%. `nonReentrant` on buy/sell/graduate/claim/router. CEI everywhere.

## 9. Deploy

`script/Deploy.s.sol` → (testnet: `PoolManager`) → `YeetHook` (mined salt) → `YeetGraduator` → `YeetLaunchpad` → `YeetRouter`; writes `deployments/<chainId>.json`: `{ chainId, launchpad, hook, graduator, router, poolManager, usdc: 0x3600…, deployBlock, gitSha }`. Verify on Arcscan.

## Order of work (contracts, ~3.5 days)

1. `CurveMath` + fuzz (½ day)
2. `YeetToken` dividend ledger + invariants (½ day)
3. `YeetLaunchpad` create/buy/sell/notify + tests (½ day)
4. `YeetHook` with return-delta fee skim + tests on self-deployed v4 (1 day)
5. `YeetGraduator` + fork test (½ day)
6. `YeetRouter` + quote, deploy script, testnet deploy, hand ABIs + `deployments/5042002.json` over (½ day)

## Defaults I chose that you may want to change

- Dev-buy hard cap 50% of supply.
- No graduation fee; all 10,000 USDC goes into the pool.
- Pool LP fee 0; protocol earns via the hook only. LPs other than us earn nothing (nobody else is expected to LP a meme pool anyway).
- Sellers do not earn dividends on their own sale; buyers do earn on their own buy (first buyer effectively pays no dividend).
- Metadata immutable after creation.

## Status (8 Sep, late)

Built, tested (31 tests: fuzz curve math, dividend ledger invariants, full curve → graduation, hook fees on all four swap shapes, foreign-pool rejection) and **deployed to Arc testnet** from `yeet-launchpad-evm`.

Deviation from the design above: the curve parameters (`vUsdc0`, `target`) are **constructor immutables** rather than compile-time constants, so testnet runs two stacks with identical code and curve shape:

| Stack | Curve | File | Launchpad |
|---|---|---|---|
| production params | 3,500 / 10,000 USDC | `deployments/5042002.json` | `0x80BC79DF1F17C94a8F0f3F88654a918ad05ec85b` |
| lite (E2E testing) | 3.5 / 10 USDC | `deployments/5042002-lite.json` | `0x1b1D1dcB4Ebae71A3395B83e8b154134732bF1b6` |

Shared self-deployed v4 PoolManager `0x6fa81c42cbD63791f5085D181b3fb82EC9EE504C`. On-chain smoke test on the lite stack passed end to end (create + 5% dev buy, sell, buys to completion, atomic graduation, router buy/sell through the hook, dividend claim of 0.3357 USDC). Arcscan verification pending (anonymous API is rate-limited; needs an Arcscan API key).

Owner on testnet = deployer key. Mainnet: `OWNER=<hardware wallet>` at deploy; owner then calls `launchpad.initialize` and `hook.acceptOwnership`.


## Update (9 Sep) — green candle split, protocol buybacks, snipe tax

Three mechanics added, all deployed on both testnet stacks (new addresses in `yeet-launchpad-evm/deployments/`; the previous stacks are retired):

1. **Dividend split ("green candle").** At launch the creator picks, next to the 0/1/3% rate, what share of every dividend buys the token back and burns it (0..100%); the rest is paid to holders in USDC. On the curve the buyback is executed inside the same trade (moves the curve like any buy, tokens to `0xdead`). After graduation the hook accumulates `pendingBurn[token]` and anyone can call `executeBuyback(token)` — the backend keeper does, every 15 s when ≥ 0.25 USDC is pending. A hook cannot re-enter its own pool from inside a swap callback, which is why the pool phase is two-step.
2. **Protocol fees → buyback + burn, immutable.** `YeetBuyback` receives 100% of protocol fees (launchpad and hook both `sweepFees()` into it; anyone can call). It has no withdraw. `setToken` works exactly once. `execute()` (permissionless; keeper) buys the protocol token from the curve while it is on the curve, from its Uniswap v4 pool after graduation, and burns everything. Protocol buybacks pay no protocol fee on themselves (fees on fees), but do pay the token's dividend. Mainnet order: deploy stack → launch the protocol token → `setToken` from the owner wallet.
3. **Snipe tax.** Buys within `SNIPE_TAX_SECONDS = 3` of creation pay `9900 bps >> (2 × elapsed seconds)`: 99% → 24.75% → 6.19% → 1.55% → 0. Dev buy in the creation tx is exempt. The tax is paid into the token's own split. `snipeTaxSeconds()` and `snipeTaxBps(token)` are public.

Testnet lite stack protocol token: YEET `0x12445cFF5bffc535325fDD7f7B80D59E76fD5ABE` (bound to the lite buyback contract). 41 tests pass. Keeper key = deployer on testnet.
