---
name: Uniswap V4
slug: uniswap-v4
ecosystem: Ethereum
type: AMM (hook-based, flash accounting)
status: live
mainnet_launch: 2025-01-31
chains: 16 mainnets (as of April 2026)
tvl_usd: ">$1B (milestone reached 2025-07-27)"
cumulative_volume_usd: ">$190B (since January 2025 launch)"
auditors: [OpenZeppelin, Spearbit, Certora, "Trail of Bits", ABDK, "Pashov Audit Group"]
bug_bounty_usd: 15_500_000
tags: [amm, hooks, flash-accounting, singleton, eip-1153, concentrated-liquidity]
sources:
  - https://blog.uniswap.org/uniswap-v4-is-here
  - https://defillama.com/protocol/uniswap-v4
  - https://docs.uniswap.org/contracts/v4/overview
  - https://app.uniswap.org/whitepaper-v4.pdf
  - https://blog.uniswap.org/v4-bug-bounty
  - https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped
---

# Uniswap V4

## Summary

Uniswap V4 is the fourth major version of the Uniswap automated market maker (AMM), launched on Ethereum mainnet on 31 January 2025. It introduces three foundational architectural changes relative to V3: a singleton `PoolManager` contract that holds all pool state, flash accounting via EIP-1153 transient storage, and a permissionless hook system that lets developers attach custom logic to pool lifecycle events. Together these changes reduce gas costs substantially, enable novel AMM strategies, and transform Uniswap into a programmable liquidity infrastructure layer.

## Adoption metrics

| Metric | Value | Source | Date |
|--------|-------|--------|------|
| Mainnet launch | 31 January 2025 | [Uniswap blog](https://blog.uniswap.org/uniswap-v4-is-here) | 2025-01-31 |
| $1B TVL milestone | Reached within 177 days of launch | [DeFiLlama](https://defillama.com/protocol/uniswap-v4) | 2025-07-27 |
| Cumulative trading volume | >$190B since mainnet launch | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) | 2025 (year-end) |
| Hooks initialised on mainnet | 5,000+ | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) | 2025 (year-end) |
| Pools using Atrium alumni hooks | 313,533+ | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) | 2025 (year-end) |
| Volume via Atrium alumni hooks | $697M+ | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) | 2025 (year-end) |
| Unichain share of V4 volume | ~50% | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025/2026 |
| L2 share of V4 daily volume | 67.5% | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025/2026 |
| Chains deployed (mainnet) | 16 | [Uniswap docs](https://docs.uniswap.org/contracts/v4/deployments) | April 2026 |
| Overall Uniswap ecosystem TVL | ~$6.8B | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2026 |

### Deployment chains (mainnet, April 2026)

Ethereum (Chain ID 1), Unichain (130), Optimism (10), Base (8453), Arbitrum One (42161), Polygon (137), Blast (81457), Zora (7777777), Worldchain (480), Ink (57073), Soneium (1868), Avalanche (43114), BNB Smart Chain (56), Celo (42220), Monad (143), Tempo (4217).

Source: [Uniswap V4 Deployments](https://docs.uniswap.org/contracts/v4/deployments), April 2026.

`PoolManager` address on Ethereum mainnet: `0x000000000004444c5dc75cB358380D2e3dE08A90`

Note: Contract addresses differ per chain; integrators must verify each deployment independently.

## How it works

### Singleton PoolManager

All Uniswap V4 pools share a single deployed contract, `PoolManager.sol`. In V3, each pool was a separate deployed contract, incurring deployment gas on every new pair. In V4, creating a pool is a state update inside the singleton rather than a contract deployment, reducing pool creation costs by up to 99.99%.

The `PoolManager` manages pool state, handles swaps and liquidity modifications, enforces hook callbacks, and settles token balances. Developers create a pool by calling `initialize()` with a token pair, fee tier, tick spacing, and an optional hook contract address.

See also: [[patterns/singleton-pool-manager]]

### Flash accounting via EIP-1153

Instead of resolving token transfers at each step of a multi-hop swap, V4 tracks net balance changes (called `delta` values) in EIP-1153 transient storage and only requires token transfers at the end of the entire operation.

The unlock/callback pattern works as follows:

1. A periphery contract calls `unlock()` on the `PoolManager`.
2. The `PoolManager` invokes `unlockCallback()` on the caller.
3. Inside the callback, the caller performs swaps, liquidity modifications, and donations in any order.
4. Each operation updates the caller's transient `delta` values rather than moving tokens immediately.
5. Before the callback returns, the caller must settle all non-zero deltas via `settle()` and `take()`.
6. The `PoolManager` enforces that `NonzeroDeltaCount` is zero before relocking; any unsettled balance causes a revert.

Gas impact: A multi-hop swap through N pools requires exactly two token transfers (input and output) regardless of N, versus N+1 transfers in V3. Transient storage operations cost 100 gas each (TSTORE/TLOAD) versus ~20,000 gas for a cold SSTORE, a 98% reduction for state tracking.

See also: [[patterns/flash-accounting]]

### Hook architecture

Every V4 pool can optionally attach one hook contract. A hook is a Solidity contract whose address encodes its permissions as bit flags. The `PoolManager` reads these flags to know which lifecycle callbacks to invoke.

Available hook points:
- `beforeInitialize` / `afterInitialize`
- `beforeAddLiquidity` / `afterAddLiquidity`
- `beforeRemoveLiquidity` / `afterRemoveLiquidity`
- `beforeSwap` / `afterSwap`
- `beforeDonate` / `afterDonate`

Hooks can also return `delta` adjustments (custom accounting) to modify token amounts during swaps or liquidity operations, enabling alternative fee mechanisms, independent pricing curves, and redistribution strategies beyond standard concentrated liquidity.

See also: [[patterns/hook-architecture]]

### Vault accounting pattern

Uniswap V4 does not use per-pool token custody. All tokens are held by the singleton `PoolManager`. The accounting model:

- No `sync()`/`skim()` functions (contrast: [[projects/uniswap-v2]]).
- Net delta tracking via transient storage replaces per-pool reserve caching.
- Hooks can implement custom accounting by returning non-zero deltas from `beforeSwap`/`afterSwap`, allowing alternative liquidity sources or fee redistribution without touching pool reserves.
- Rebasing token support is hook-extensible but not natively handled by core contracts.

## Key behaviours

| Behaviour | Detail |
|-----------|--------|
| Pool creation | State update in singleton; no contract deployment |
| Multi-hop gas | Two token transfers regardless of hop count |
| Hook invocation | Per-pool; only callbacks whose permission bits are set in hook address |
| Custom accounting | Hooks return `BeforeSwapDelta` / `AfterSwapDelta` to adjust token amounts |
| Settlement enforcement | All deltas must resolve to zero before unlock completes; revert otherwise |
| Native ETH | Supported natively (V3 required WETH) |
| Fee tiers | Any fee tier is configurable; hooks can implement dynamic fees |

## Architecture decisions and rationale

| Decision | Rationale |
|----------|-----------|
| Singleton PoolManager | Eliminates per-pool deployment cost; enables cross-pool delta netting |
| EIP-1153 transient storage | ~98% gas reduction for state tracking vs. persistent storage |
| Permission bits in hook address | Prevents hooks claiming callbacks their code doesn't implement; no on-chain registry needed |
| Flash accounting over eager transfers | Scales to arbitrarily complex multi-pool operations with only two final transfers |
| No built-in protocol fee on V4 core | Fee logic delegated to hooks; protocol can enable fees via governance |

## Differentiators vs. V3

| Dimension | V3 | V4 |
|-----------|----|----|
| Pool architecture | One contract per pool | Singleton PoolManager |
| Pool creation cost | ~$20–100 in gas | ~99.99% cheaper |
| Multi-hop transfers | N+1 transfers for N hops | Always 2 transfers |
| Extensibility | Fixed swap curve | Arbitrary hook logic |
| Native ETH | Requires WETH | Supported natively |
| Transient storage | Not used | Core to flash accounting |
| Custom fee logic | Fixed fee tiers | Dynamic via hooks |

## Differentiators vs. Balancer V3

Both V4 and [[projects/balancer-v3]] use EIP-1153 transient accounting and hooks. Key differences:

- Uniswap V4 uses concentrated liquidity (tick-based) by default; Balancer V3 uses weighted and stable pools.
- Uniswap V4 hooks are permissionless and address-encoded; Balancer V3 hooks are registered.
- Balancer V3 has native yield-bearing token (ERC4626) integration; Uniswap V4 relies on hooks for this.
- Balancer V3 has a single external Vault; Uniswap V4 holds tokens inside the PoolManager singleton.

## Limitations

### MEV exposure

Hook logic based on external price data or dynamic fees is exploitable by MEV bots through front-running. In 2025 a single sandwich attack on Uniswap cost one trader $215,000 ([source](https://academy.exmon.pro/future-of-liquidity-uniswap-v4-hooks-vs-jit-mev-attacks)). Hooks can mitigate MEV (e.g., Angstrom by Sorella Labs implements an app-specific sequencer), but MEV protection is opt-in per pool.

### Hook security surface

The hook architecture dramatically expands the attack surface relative to V3. Documented vulnerability classes include:

- **Authorization bypass:** Hooks that do not verify `msg.sender == poolManager` can be called by any address.
- **Delta handling errors:** Incorrect `BeforeSwapDelta` sign or ordering leaves unsettled balances, causing reverts or fund loss.
- **Async hook exploitation:** Hooks taking full custody of swapped assets can redirect funds to an attacker.
- **Centralization via upgradeable hooks:** Owner-controlled UUPS hooks can change fees or pause withdrawals after deployment.
- **DoS via gas exhaustion:** Hooks with unbounded loops or expensive operations in `beforeSwap` can block all pool swaps.

The Uniswap Foundation does not audit or certify hook implementations. Users bear the full risk of interacting with pools attached to unaudited hooks.

### Complexity for integrators

The unlock/callback pattern requires integrators to restructure their code around callbacks rather than direct function calls. Multi-step operations must carefully manage delta accumulation and settlement order; incorrect ordering (e.g., `settle()` without a preceding `sync()`) causes reverts.

### Transient storage edge cases

EIP-1153 transient storage clears at transaction end but persists across internal calls within a transaction. In batched transactions through relayers or smart-contract wallets, stale transient values may persist across multiple logical operations if not explicitly cleared. Uniswap V4's codebase uses defensive explicit resets to mitigate this.

### Hook-level rebasing token risk

V4 core does not natively handle rebasing tokens. Hook implementations that take custody of rebasing tokens during multi-step operations may encounter balance discrepancies if a rebase occurs mid-transaction.

## Security

| Item | Detail | Source |
|------|--------|--------|
| Independent audits | 9 (OpenZeppelin, Spearbit, Certora, Trail of Bits, ABDK, Pashov Audit Group) | [Uniswap blog](https://blog.uniswap.org/uniswap-v4-is-here) |
| Security competition | $2.35M; 500+ researchers; no critical bugs found | [Uniswap blog](https://blog.uniswap.org/uniswap-v4-is-here) |
| Bug bounty | $15.5M (largest in DeFi history at launch); active via Cantina | [Uniswap blog](https://blog.uniswap.org/v4-bug-bounty) |
| Audit reports | [v4-core/docs/security/audits](https://github.com/Uniswap/v4-core/tree/main/docs/security/audits); [v4-periphery/audits](https://github.com/Uniswap/v4-periphery/tree/main/audits) | GitHub |
| Known critical bugs in core | None found pre-launch or post-launch | [Uniswap blog](https://blog.uniswap.org/v4-bug-bounty) |

## Notable hooks and ecosystem projects (2025)

| Project | Hook category | Description |
|---------|---------------|-------------|
| Angstrom (Sorella Labs) | MEV resistance | App-specific sequencer; auctions ordering rights to reward LPs |
| AdaptiveSwap | Dynamic fees | Adjusts fees based on oracle data to protect LPs during volatility |
| TWAMM hooks | Time-weighted execution | Time-weighted average market maker for large order execution |
| FrontrunThis | MEV capture | Dark pool-style orderflow via EigenLayer infrastructure |
| RampHook | Cross-chain | On-ramp hook delivering tokens cross-chain |

Source: [Atrium Academy Hook Incubator 2025 Wrapped](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped)

## Related notes

- [[patterns/flash-accounting]]
- [[patterns/singleton-pool-manager]]
- [[patterns/hook-architecture]]
- [[metrics/tvl-comparison]]
- [[metrics/vault-accounting-patterns]]
- [[projects/uniswap-v2]] (sync/skim predecessor)
- [[projects/balancer-v3]] (comparable transient accounting approach)
