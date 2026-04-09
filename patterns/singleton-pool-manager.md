---
pattern: singleton-pool-manager
aka: ["singleton AMM", "monolithic pool contract"]
used_by: ["Uniswap V4"]
tags: [architecture, singleton, gas-optimisation, pool-management]
sources:
  - https://docs.uniswap.org/contracts/v4/concepts/PoolManager
  - https://docs.uniswap.org/contracts/v4/overview
  - https://app.uniswap.org/whitepaper-v4.pdf
  - https://blog.uniswap.org/uniswap-v4-is-here
---

# Pattern: Singleton Pool Manager

## Problem

In Uniswap V2 and V3, each liquidity pool is a separate deployed smart contract. Creating a new pair requires deploying a new contract, which costs significant gas (tens of thousands of dollars at peak Ethereum gas prices). Cross-pool operations (multi-hop swaps, flash loans across pools) require cross-contract calls and intermediate token transfers, each incurring storage and call overhead.

As the number of supported token pairs grows, the deployment surface expands and on-chain state is fragmented across thousands of individual contracts.

## Solution

Deploy a single contract, the `PoolManager`, that manages state for all pools. Creating a new pool is a state update (a mapping write) rather than a contract deployment. All swaps, liquidity modifications, and hook invocations pass through a single entry point.

## Uniswap V4 implementation

### Contract: `PoolManager.sol`

The `PoolManager` is the sole authoritative contract for Uniswap V4. It:

- Stores state for every pool in a mapping keyed by `PoolId`.
- Executes swaps and liquidity modifications.
- Invokes hook callbacks at defined lifecycle points.
- Holds all token balances for all pools.
- Enforces flash accounting settlement via `NonzeroDeltaCount`.

Ethereum mainnet address: `0x000000000004444c5dc75cB358380D2e3dE08A90`

Source: [Uniswap V4 Deployments](https://docs.uniswap.org/contracts/v4/deployments), April 2026.

### Pool identity: `PoolKey` and `PoolId`

Each pool is identified by a `PoolKey` struct containing:

```
currency0      address (token, lower sort order)
currency1      address (token, higher sort order)
fee            uint24  (fee tier in pips)
tickSpacing    int24   (minimum tick width for LP positions)
hooks          address (hook contract; address(0) for no hook)
```

The `PoolId` is the `keccak256` hash of the `PoolKey`. Two pools with identical token pairs but different fee tiers, tick spacings, or hook addresses are distinct pools.

### Pool creation

Developers call `PoolManager.initialize(PoolKey calldata key, uint160 sqrtPriceX96)`. This:

1. Validates that `currency0 < currency1` (enforced sort order).
2. Checks the hook address encodes valid permission flags.
3. Invokes `hook.beforeInitialize()` if the hook's permission bit is set.
4. Writes initial pool state to storage (slot0, liquidity, fee accumulators).
5. Invokes `hook.afterInitialize()` if set.

No contract is deployed. Pool creation gas cost is reduced by up to 99.99% vs. V3.

Source: [Uniswap blog](https://blog.uniswap.org/uniswap-v4-is-here), 2025-01-31.

### Operations routed through PoolManager

| Operation | Entry point | Notes |
|-----------|-------------|-------|
| Swap | `unlock()` + `swap()` inside callback | Flash accounting; delta resolved at callback end |
| Add liquidity | `unlock()` + `modifyLiquidity()` | Same callback pattern |
| Remove liquidity | `unlock()` + `modifyLiquidity()` | Negative liquidity delta |
| Donate | `unlock()` + `donate()` | Adds tokens to fee accumulators without a swap |
| Flash loan | `unlock()` + `take()` + `settle()` | Standard flash accounting within one callback |

### Token custody model

All tokens across all pools reside in the `PoolManager` contract. There are no per-pool token contracts or wallets. The `PoolManager` tracks each pool's share of the pooled tokens via internal accounting; the flash accounting delta system ensures correct settlement.

This contrasts with:

- **Uniswap V2/V3:** Each pool contract holds its own tokens.
- **Balancer V2:** A single external `Vault` holds tokens; pools are separate math contracts.

## Gas impact

| Operation | V3 (per pool contract) | V4 (singleton) |
|-----------|----------------------|----------------|
| Pool creation | ~$20–100+ in gas (contract deploy) | ~99.99% cheaper (state write only) |
| Multi-hop swap (N hops) | N cross-contract calls + N+1 transfers | 1 contract + 2 transfers (all hops) |
| Liquidity modification | Cross-contract call to pool | Direct state write in PoolManager |

## Deployment considerations

Pool addresses are derived differently on each chain; Uniswap V4 explicitly warns integrators not to assume the same `PoolManager` address across chains. As of April 2026, V4 is deployed on 16 mainnet chains with distinct addresses per chain.

Source: [Uniswap V4 Deployments](https://docs.uniswap.org/contracts/v4/deployments).

## Trade-offs

| Advantage | Disadvantage |
|-----------|-------------|
| Massive gas savings on pool creation | Single contract is a larger attack surface than isolated pools |
| Efficient cross-pool delta netting | Bugs in PoolManager affect all pools simultaneously |
| Simpler router design (one contract to call) | Upgradeability is constrained; any upgrade affects all pools |
| Native ETH support (no WETH wrapping) | Integrators must update all hard-coded pool addresses per chain |

## Related notes

- [[patterns/flash-accounting]] (how the singleton enables efficient multi-hop accounting)
- [[patterns/hook-architecture]] (hook callbacks routed through PoolManager)
- [[projects/uniswap-v4]] (full protocol note)
- [[projects/balancer-v3]] (comparable single-vault pattern with separate pool math contracts)
