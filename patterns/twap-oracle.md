---
tags: [pattern, oracle, twap, uniswap-v2, price-manipulation, on-chain]
origin: Uniswap V2
---

# Pattern: TWAP Oracle (Time-Weighted Average Price)

## Overview

Uniswap V2 introduced the first widely adopted on-chain price oracle for decentralised finance. Rather than providing a spot price (which is trivially manipulable within a single block), it accumulates prices over time so that external protocols can compute a Time-Weighted Average Price (TWAP) over any historical window.

## Mechanism

Each `UniswapV2Pair` contract stores two cumulative price accumulators:

```solidity
uint256 public price0CumulativeLast;
uint256 public price1CumulativeLast;
uint32  public blockTimestampLast;
```

At the start of every block in which the pair state changes (any call to `_update()`), the accumulators are updated:

```
price0CumulativeLast += price0 · timeElapsed
price1CumulativeLast += price1 · timeElapsed
```

Where `price0 = reserve1 / reserve0` (token0's price in token1 units) expressed as a UQ112x112 fixed-point number, and `timeElapsed` is the number of seconds since the last update.

Because the update happens once per block (at the first state-changing call), the accumulator captures the time-weighted average of spot prices across blocks, not across individual transactions within a block.

## Computing TWAP externally

An external oracle consumer must:
1. Record `price0CumulativeLast` and `blockTimestampLast` at time T₁.
2. Wait for the desired averaging window to elapse.
3. Record `price0CumulativeLast` at time T₂.
4. Compute: `TWAP = (price0Cumulative_T2 − price0Cumulative_T1) / (T2 − T1)`

The result is the average of reserve1/reserve0 (in UQ112x112 fixed-point) across the window, weighted by how long the price was at each level.

The pair contract does not store or expose this computation directly; external contracts must manage snapshots.

## Security properties

| Property | Detail |
|----------|--------|
| Manipulation cost | An attacker must hold the price at a distorted level for the entire window duration, continuously incurring arbitrage losses |
| Window length | Longer windows cost more to manipulate but are less responsive to genuine price changes |
| Liquidity depth | Deeper liquidity makes each unit of price movement more expensive to achieve |
| Linear cost | Both manipulation cost and liquidity-pool resistance scale approximately linearly with window length and TVL |

## Limitations

1. **No built-in snapshot storage:** The pair does not store historical values; consumers must implement their own snapshot logic.
2. **Short-window vulnerability:** Windows of a few minutes in low-liquidity pools can be manipulated cheaply.
3. **Multi-block MEV (post-Merge):** An Ethereum validator controlling consecutive block proposals can distort prices across multiple blocks with lower cost. A validator with 0.15% of staked ETH would expect to propose two consecutive blocks roughly every 62 days (per ChainSecurity analysis, 2022), enabling TWAP manipulation with reduced opportunity cost.
4. **Fixed-point precision:** UQ112x112 format limits precision for extreme price ratios (e.g., tokens priced millions of times apart).
5. **Overflow:** Accumulators grow monotonically and eventually overflow `uint256`. Safe by modular arithmetic: subtraction of an overflowed newer value from an older value still yields the correct delta.
6. **One-sided prices:** `price0CumulativeLast` tracks token0 in token1 terms; `price1CumulativeLast` tracks the inverse. Both must be tracked because `1/TWAP(p) ≠ TWAP(1/p)`.

## Improvements in later versions

Uniswap V3 introduced geometric mean TWAPs stored in a ring buffer of 65,535 observations per pool, making it significantly easier to compute TWAPs without external snapshot management. V4 continues this approach within hook-enabled pools.

For a new DEX design, V2's accumulator pattern is the minimal viable on-chain oracle. More sophisticated alternatives include:
- Chainlink price feeds (off-chain aggregated, not subject to on-chain manipulation)
- Pyth Network (push-based off-chain oracle)
- V3-style tick accumulator with ring buffer

## References

- Uniswap V2 oracle docs: https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles
- Uniswap V2 whitepaper (section 2.2): https://app.uniswap.org/whitepaper.pdf
- RareSkills TWAP deep dive: https://rareskills.io/post/twap-uniswap-v2
- ChainSecurity multi-block MEV analysis: https://www.chainsecurity.com/blog/oracle-manipulation-after-merge
- Uniswap V3 TWAP blog post: https://blog.uniswap.org/uniswap-v3-oracles

## Related notes

- [[projects/uniswap-v2]]
- [[patterns/constant-product-amm]]
