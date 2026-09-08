# Arc — verified facts (checked 8 Sep 2026)

Everything below was checked against the live testnet RPC or primary docs unless marked *unverified*.

## Networks

| | Testnet | Mainnet |
|---|---|---|
| Chain ID | `5042002` | `5042` (from Uniswap sdk-core / interface source; Circle has not published yet) |
| HTTP RPC | `https://rpc.testnet.arc.io` (Circle), `https://rpc.quicknode.testnet.arc.io`, `https://rpc.drpc.testnet.arc.io`, `https://rpc.blockdaemon.testnet.arc.io` | `https://rpc.arc.io` *(unverified, from Uniswap interface config)* |
| WS RPC | `wss://rpc.testnet.arc.io` (+ same providers) | TBD |
| Explorer | `https://testnet.arcscan.app` | `https://explorer.arc.io` *(unverified)* |
| Faucet | `https://faucet.circle.com` | n/a |
| Status | live, block ~61.1M | public launch **16 Sep 2026** (currently private mainnet) |

## Chain behaviour that affects us

- **Gas / native token is USDC.** `msg.value` and `address(this).balance` are USDC in **18 decimals**. The ERC-20 view at `0x3600000000000000000000000000000000000000` is the *same balance* in **6 decimals**. There is **no wrapped-native contract** and Uniswap deliberately leaves `WRAPPED_NATIVE_CURRENCY` empty for Arc.
- Block time ~0.5s (measured: 10 heads in 5s). **Deterministic finality on inclusion, no reorgs.** Indexer can commit every block as final.
- `block.timestamp` is non-decreasing, not strictly increasing (sub-second blocks share a second). Never use timestamp as a unique key.
- `PREVRANDAO` = 0. `block.basefee` floor 20 gwei, base fee goes to validator (not burned). Tx cost ≈ $0.01.
- Transfers to blocklisted addresses revert (Circle compliance). Contracts must not assume a native send to an arbitrary address succeeds (use pull-payments for creator fees).
- Every native USDC movement also emits an ERC-20 `Transfer` log from the system address `0xffff…fffe` (EIP-7708). Indexer must filter by emitter to avoid double counting.
- `eth_estimateGas` reported unreliable by the ArcSwap team (V2 `createPair` needed ~5M gas). Set explicit gas limits in deploy scripts and the frontend.
- Anvil cannot emulate native-USDC semantics. Unit tests in Foundry are fine for curve math; integration tests must run against testnet RPC (`forge test --fork-url`).
- EVM = Osaka. Foundry / Hardhat / Viem / alloy all work. CREATE2 deployer, Multicall3, Permit2 at canonical addresses (verified code present).

## RPC limits (public Circle testnet endpoint, measured)

| Probe | Result |
|---|---|
| `eth_subscribe newHeads` | works |
| `eth_subscribe logs` (address filter) | works (88 USDC logs / 5s) |
| `eth_getLogs` 1,000-block span, single busy address | ok (11,888 logs) |
| `eth_getLogs` 10,000-block span | error `-32602` max **20,000 results**, returns suggested retry range |
| `eth_getLogs` 100,000-block span | error `-32012` "requested range too large" |
| `eth_getBlockReceipts` | supported |
| JSON-RPC batch | supported |
| `debug_traceBlockByNumber` | **not supported** (no tracing on public RPC) |
| 50 sequential requests | all 200, ~8 req/s from one client, no throttling seen |
| Client version | `arc/v1` (reth-based execution, Malachite consensus) |

## Running our own node (not for launch week)

`circlefin/arc-node` is open source (Apache 2.0), docker-compose, `--no-consensus` RPC mode, needs **64 GB+ RAM, 1 TB+ TLC NVMe, high clock CPU**, snapshot-only sync (~250 GB, minimal/full/archive profiles), testnet config only today. Hetzner AX42-1 (64 GB, 2×1.92 TB NVMe) is €97.30/mo + €49 setup ex VAT; AX102-1 (128 GB) is €257.30/mo + €129. Considered and rejected for launch (step 1); QuickNode instead.

## Hosted indexers with Arc testnet support

Envio HyperSync/HyperIndex, Goldsky, The Graph all list Arc **testnet**. None commit to mainnet on day 1. See step 1 for why we don't depend on them.

## Uniswap on Arc — actual status

Claim to check: "Uniswap is live on Arc testnet." Result: **partly**.

1. **Official Uniswap v3 + v4 are deployed on Arc *mainnet* (5042, private today)**, not on testnet. Addresses from `Uniswap/sdks` sdk-core `ARC_ADDRESSES` and the UniswapX playbook. Every one of these has **no code on testnet 5042002** (checked with `eth_getCode`):

   | Contract | Mainnet address (5042) |
   |---|---|
   | v4 PoolManager | `0x8366a39cc670b4001a1121b8f6a443a643e40951` |
   | v4 PositionManager | `0x6049c9a0e26405c0985f9e3685c87d0ae917f82b` |
   | v4 StateView | `0xf3334192d15450cdd385c8b70e03f9a6bd9e673b` |
   | v4 Quoter | `0x8dc178efb8111bb0973dd9d722ebeff267c98f94` |
   | v3 Factory | `0xf0db7b58379503491d857db50ac9ece64c653918` |
   | v3 SwapRouter02 | `0x53bf6b0684ec7ef91e1387da3d1a1769bc5a6f77` |
   | v3 NonfungiblePositionManager | `0x39654a85a4c05127f5fd6ed22caec077a0fb1377` |
   | v3 QuoterV2 | `0x7dfd4f31be6814d2906bde155c3e1b146eac1468` |
   | v2 Factory | `0x89e5db8b5aa49aa85ac63f691524311aeb649eba` |
   | v2 Router | `0x1f7d7550b1b028f7571e69a784071f0205fd2efa` |
   | Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
   | UniversalRouter | not published yet |

   Uniswap announced (15 Jun 2026) that v4 + routing + SDK go live *with* the public mainnet on 16 Sep. The Uniswap interface repo already has an Arc mainnet chain config (no testnet config).

2. **On testnet the only live AMM is "ArcSwap"**, a community Uniswap **V2** fork (circlefin/arc-node issue #160):
   - Factory `0x7483847d46db2920dd64efa676cf72dcf765814f` (code present, 83 pairs)
   - Router02 `0xe27d5d256b370604f1ff060fb489c6a8e3f8a6d9` (code present)
   - `router.WETH()` = `0x6be2c68117ca58086bd6a14e525835584d7f721e` which has **no code**. So any native-USDC path (`swapExactETHForTokens`, `addLiquidityETH`) is broken. Token/token only. Not something to graduate into.

**Consequence for the plan:** graduation targets Uniswap **v4 with native USDC as `currency0 = address(0)`** (v4 supports native currency without a wrapper, which is exactly Arc's model). On testnet we deploy v4-core ourselves (it's permissionless and MIT-licensed core); on mainnet we point the same config at the official PoolManager above. Details in step 2.
