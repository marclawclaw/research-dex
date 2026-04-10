---
tags: [pattern, amm, constant-product, uniswap-v2, pricing, invariant]
origin: Uniswap V1 / V2
---

# Pattern: Constant Product AMM (x·y = k)

## Overview

The constant product AMM is the foundational pricing mechanism behind Uniswap V1, Uniswap V2, and hundreds of forks. It allows any two tokens to be traded permissionlessly without an order book by maintaining a simple mathematical invariant: the product of the two token reserves must not decrease after a trade (net of fees).

## The invariant

```
x · y = k
```

Where:
- `x` = reserve of token0
- `y` = reserve of token1
- `k` = a non-decreasing constant (it increases when fees are collected)

A swap that takes in `Δx` of token0 and gives out `Δy` of token1 must satisfy:

```
(x + Δx) · (y − Δy) ≥ x · y
```

Rearranging to find the output:

```
Δy = y · Δx / (x + Δx)
```

## Fee incorporation

In Uniswap V2, the 0.3% fee is applied to the input amount before computing the output. The effective invariant check becomes:

```
(x + 0.997 · Δx) · (y − Δy) ≥ x · y
```

The fee (0.3% of `Δx`) remains in the pool, increasing `k` by a small amount with each trade. This is how LP returns accrue over time.

With the optional protocol fee (0.05%, activated on Ethereum mainnet December 2025): 5/6 of the 0.3% stays with LPs (0.25%), and 1/6 goes to the protocol (0.05%). The invariant formula is unchanged; the protocol fee is collected lazily at liquidity events by measuring `√k` growth.

## Marginal price and price impact

The marginal price (spot price) of token0 in terms of token1 is:

```
P = y / x
```

Because the function is a hyperbola, the marginal price moves continuously as reserves shift. Large trades relative to pool size cause significant price impact (slippage). Price impact grows quadratically with trade size relative to pool depth.

## Liquidity provision

When a user deposits tokens into a constant product pool, they receive LP tokens representing a proportional share of the pool. The LP token balance grows as fees accumulate (because `k` grows while total LP token supply stays constant until mints or burns).

LP token value at time t:

```
LPvalue(t) = (x(t) · y(t))^(1/2) / totalSupply(t)
```

## Impermanent loss

When token prices diverge from the price ratio at the time of deposit, LPs hold a different ratio of tokens than they deposited. If they had simply held the original tokens instead of providing liquidity, they would have more value. This difference is impermanent loss (IL).

For a price ratio change of factor `r` (e.g., r=2 means token0 doubled vs token1):

```
IL = 2√r / (1 + r) − 1
```

| Price ratio change | Impermanent loss |
|-------------------|-----------------|
| 1.25× | ~0.6% |
| 1.50× | ~2.0% |
| 2× | ~5.7% |
| 4× | ~20.0% |
| 10× | ~42.5% |

IL is "impermanent" because it disappears if the price returns to the original ratio. It becomes permanent upon withdrawal at a diverged price.

## Invariant properties

| Property | Detail |
|----------|--------|
| Price discovery | Automatic: price is always `y/x` |
| Liquidity range | Infinite price range (0 to ∞), but capital inefficient away from current price |
| Slippage | Deterministic and computable off-chain from reserves |
| Composability | Any contract can compute pool price and simulate swaps using public reserve state |
| Manipulation resistance | Price can only be moved by trading, which costs arbitrageurs money |

## Capital efficiency vs concentrated liquidity

A key limitation of x·y=k is that liquidity is spread across the entire price range (0, ∞). At the current price, only a fraction of the deposited capital is actively used for trading. Uniswap V3 solved this with concentrated liquidity (CLMM), allowing LPs to specify a price range. A concentrated position earns higher fees per unit of capital but suffers full IL when price exits the range.

For a new DEX, constant product is the simplest and most battle-tested starting point: no tick math, no concentrated position management, straightforward LP token accounting. The tradeoff is lower capital efficiency and higher IL for volatile pairs.

## Who uses this pattern

| Protocol | Variant | Notes |
|----------|---------|-------|
| [[projects/uniswap-v2]] | Pure x·y=k with 0.3% fee | Reference implementation; most forked |
| PancakeSwap V2 | Fork of Uniswap V2 | BNB Chain dominant DEX |
| SushiSwap (V1 pools) | Fork of Uniswap V2 | Multi-chain |
| Aerodrome V1 (volatile pools) | Fork, ve(3,3) incentives | Base chain |
| QuickSwap | Fork | Polygon |
| SpookySwap | Fork | Fantom |

Curve (StableSwap) and Balancer (weighted product) are generalisations/variations of the AMM invariant approach, trading different tradeoffs for stability or multi-token support.

## Portability to a new chain

The constant product formula itself is chain-agnostic pure arithmetic. Key implementation decisions for a new chain:

1. **Fixed-point arithmetic:** Uniswap V2 uses UQ112x112 (112 bits integer, 112 bits fractional). A new chain may use native floats or different fixed-point formats.
2. **Reserve storage:** Uniswap V2 stores two `uint112` values in one `uint256` storage slot (bit-packing for gas efficiency). On a new chain with different storage costs, this optimisation may or may not apply.
3. **Vault accounting:** Does the pair trust cached reserves (V2 approach) or read live balances on every operation (more expensive but more rebase-friendly)? See [[patterns/sync-skim]].
4. **Fee precision:** 0.3% expressed as 997/1000 in Uniswap V2 integer arithmetic. On a new chain, consider whether integer division introduces meaningful rounding error.
5. **Reentrancy:** The `lock` modifier is essential when output tokens are sent before input is verified (optimistic transfer model enabling flash swaps).

## References

- Uniswap V2 whitepaper: https://app.uniswap.org/whitepaper.pdf
- RareSkills Uniswap V2 architecture tutorial: https://rareskills.io/post/uniswap-v2-tutorial
- Xord Uniswap V2 deep dive: https://xord.com/research/uniswap-v2-protocol-lets-dive-in/
- Impermanent loss explanation: https://support.uniswap.org/hc/en-us/articles/20904453751693-What-is-Impermanent-Loss
- ZeroLuck mathematical review: https://mirror.xyz/zer0luck.eth/rnu5BTT8f45lq3YUlTBoADGOhI0zpfa710SinXaGoZ8

## Related notes

- [[projects/uniswap-v2]]
- [[patterns/sync-skim]]
- [[metrics/tvl-comparison]]
