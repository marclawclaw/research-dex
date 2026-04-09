---
tags: [pattern, mathematics, invariant, amm, stableswap]
applies-to: [curve-stableswapng]
---

# Pattern: StableSwap Invariant

## Overview

The stableswap invariant is the pricing formula at the heart of Curve Finance pools. It interpolates between a constant-sum AMM (`x + y = D`, zero price impact, infinite slippage at pool boundaries) and a constant-product AMM (`xy = k`, infinite liquidity range, significant price impact near peg). This interpolation is controlled by the amplification coefficient `A`, allowing the curve to concentrate liquidity near the pegged price while gracefully degrading to constant-product behaviour when the pool becomes severely imbalanced.

Used by: [[projects/curve-stableswapng]]

## The invariant formula

For a pool with `n` tokens and balances `x_1, x_2, ..., x_n`:

```
A * n^n * sum(x_i) + D = A * n^n * D + D^(n+1) / (n^n * prod(x_i))
```

Where:
- `A` = amplification coefficient (governance-set parameter)
- `n` = number of tokens in the pool (up to 8 in StableSwapNG)
- `x_i` = balance of token `i` (after rate normalisation for oracle/ERC4626 tokens)
- `D` = the invariant: the total amount of tokens when all have equal price (analogous to `k` in Uniswap V2, but with different units)
- `sum(x_i)` = sum of all token balances
- `prod(x_i)` = product of all token balances

When all tokens have equal balances, `x_i = D/n` for all `i`, and the equation reduces to a check that `D = n * (D/n)`, which is always true. This confirms that D equals the total pool value when prices are equal.

### Limiting cases

- **When `A = 0`:** The formula reduces to the constant-product invariant. Slippage is high near peg; liquidity is distributed across all price ranges.
- **When `A` is very large:** The formula approaches constant-sum. The pool offers near-zero slippage across a wide range around peg, but all liquidity is consumed if prices diverge significantly.
- **In practice:** `A` values for stablecoin pools are typically in the range 100 to 2000. For pegged-asset pairs like stETH/ETH, values around 50 to 200 are common.

## The amplification coefficient A

`A` determines how "flat" the bonding curve is around equilibrium:

- A **higher** `A` concentrates more liquidity near parity, reducing price impact for balanced swaps. It behaves like virtual leverage.
- A **lower** `A` provides more liquidity spread across a wider price range, which is safer when the peg relationship is uncertain.
- `A` can be changed by governance via a time-locked ramp process (`ramp_A()`), which linearly interpolates the coefficient over a specified number of blocks to prevent abrupt price jumps.

On-chain optimisation: StableSwapNG stores `A` as `A * n^(n-1)` in a single storage slot. This precomputed value avoids computing `n^n` on every swap, saving gas.

## Solving for D: `get_D()` and Newton's method

Given a set of token balances `{x_i}` and an amplification value `A`, the invariant must be solved for `D`. There is no closed-form algebraic solution, so Curve uses Newton's iteration.

Rearranging the invariant into an iterative form:

```
D_next = (A * n^n * S + n * D_p) * D / ((A * n^n - 1) * D + (n + 1) * D_p)
```

Where:
- `S = sum(x_i)` (sum of all balances)
- `D_p = D^(n+1) / (n^n * prod(x_i))`

The loop converges when `|D_next - D| < 1` (in token base units). Convergence typically takes 4 to 6 iterations for standard pool configurations.

`get_D()` is called on every swap, add-liquidity, and remove-liquidity operation because `D` must remain constant during a swap (ignoring fees) and must update correctly when liquidity is deposited or withdrawn.

## Solving for the swap output: `get_y()`

During a swap of token `i` for token `j`, the pool knows the new balance `x_i_new` and must solve for `x_j_new` that satisfies the invariant (holding `D` constant). After algebraic manipulation:

```
y_next = (y^2 + c) / (2 * y + b - D)
```

Where:
- `c = D^(n+1) / (n^n * prod_of_other_balances * A * n^n)`
- `b = sum_of_other_balances + D / (A * n^n)`

The output amount for the user is `x_j_old - y_next`, minus the swap fee applied to the output token.

## Rate normalisation for oracle and ERC4626 tokens

For pools containing rate-oraclised tokens (asset type 1, e.g. wstETH) or ERC4626 tokens (asset type 3, e.g. sDAI), raw balances are not fed directly into the invariant. Instead, each balance is scaled by the token's current exchange rate:

```
x_i_normalised = x_i_raw * rate_i / PRECISION
```

Where `rate_i` is fetched from the configured oracle contract (for type 1) or from `convertToAssets(1e18)` (for type 3). This normalisation ensures the invariant treats, for example, wstETH and stETH as economically equivalent near parity, even though their nominal balances differ.

For rebasing tokens (asset type 2), no rate normalisation is applied; the raw balance from `_balance(i)` is used directly. See [[patterns/rebase-safe-accounting]] for how rebases are accounted for without corrupting fee isolation.

## Price oracle (EMA)

After each swap, StableSwapNG updates a stored price using an exponential moving average:

```
price_ema = price_last * (1 - alpha) + price_spot * alpha
```

Where `alpha` is derived from the time elapsed since the last update and the `ma_exp_time` parameter. The EMA is updated at most once per block (the check `block.timestamp != last_timestamp` gates the update), preventing intra-block oracle manipulation.

The pool exposes `price_oracle()` and `ema_price_oracle()` functions that external protocols (e.g. crvUSD lending, other Curve components) use as on-chain price feeds.

## MEV and sandwich attack economics

The stableswap curve's flatness near peg is a double-edged property:

- **Low slippage for users:** large swaps cause minimal price impact when the pool is balanced.
- **Oracle sandwich risk for rate-oraclised pools:** because the pool's swap price is tightly coupled to the oracle rate (the normalisation factor), a sudden oracle update relocates the equilibrium curve. An actor who positions before the update and reverses after can capture the price step as profit. The round-trip cost is approximately `2 * fee`; any oracle jump larger than this is exploitable.

The `offpeg_fee_multiplier` addresses this by raising the fee when pool balances diverge from parity, increasing the break-even threshold for oracle sandwich attacks.

Flat-curve pools are also attractive for direct large-volume arbitrage (not sandwiches): because price impact is minimal, an arbitrageur can push a very large trade to bring two pegged assets back to parity with very little loss to slippage, making these pools efficient self-correcting price anchors.

## Sources

- [RareSkills: Curve get_D() and get_y()](https://rareskills.io/post/curve-get-d-get-y) — accessed 2026-04-09
- [Atul Agarwal: Understanding the Curve AMM, Part 1](https://atulagarwal.dev/posts/curveamm/stableswap/) — accessed 2026-04-09
- [Miguel Mota: Understanding StableSwap (Curve)](https://miguelmota.com/blog/understanding-stableswap-curve/) — accessed 2026-04-09
- [Curve StableSwap whitepaper](https://docs.curve.finance/references/whitepaper/) — accessed 2026-04-09
- [Curve docs: StableSwapNG oracles](https://docs.curve.finance/stableswap-exchange/stableswap-ng/pools/oracles/) — accessed 2026-04-09
- [MixBytes: Safe StableSwap-NG Deployment](https://mixbytes.io/blog/safe-stableswap-ng-deployment-how-to-avoid-risks-from-volatile-oracles) — accessed 2026-04-09
