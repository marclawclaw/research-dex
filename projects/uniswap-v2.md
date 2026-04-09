---
tags: [dex, ethereum, amm, constant-product, vault-accounting, uniswap, reference-implementation]
ecosystem: Ethereum
status: live
launched: 2020-05-18
---

# Uniswap V2

## Overview

Uniswap V2 is the canonical constant-product automated market maker (AMM) deployed on Ethereum mainnet on 18 May 2020. It introduced direct ERC-20/ERC-20 trading pairs (eliminating the ETH-as-intermediary requirement of V1), a TWAP price oracle, flash swaps, and the `sync()`/`skim()` pattern for vault accounting. It remains the most-forked DEX codebase in DeFi history.

## Adoption metrics

| Metric | Value | Source | Date |
|--------|-------|---------|------|
| TVL (CoinMarketCap liquidity) | ~$945M | [CoinMarketCap](https://coinmarketcap.com/exchanges/uniswap-v2/) | April 2026 |
| 24h trading volume | ~$15.1M | [CoinMarketCap](https://coinmarketcap.com/exchanges/uniswap-v2/) | April 2026 |
| 30-day trading volume | ~$3.1B | [DeFiLlama](https://defillama.com/protocol/uniswap-v2) | Early 2026 |
| Cumulative all-time DEX volume | ~$604B | [DeFiLlama](https://defillama.com/protocol/uniswap-v2) | Early 2026 |
| Number of trading pairs (historical) | 260,544 | [DeFiLlama](https://defillama.com/protocol/uniswap-v2) | October 2023 |
| Protocol fee activation | 0.05% of swap fees (17% of LP fees) redirected to UNI burn | [Uniswap Blog](https://blog.uniswap.org/unification) | December 2025 |
| V2 share of Uniswap TVL on Ethereum | ~40% | [CoinSpeaker](https://www.coinspeaker.com/uniswap-hit-tvl-2-billion/) | 2025 estimate |
| TVL peak (all Uniswap versions combined) | >$10B | [CoinSpeaker](https://www.coinspeaker.com/uniswap-hit-tvl-2-billion/) | 2021 |
| Fork count (DeFiLlama tracked) | 100+ forks globally | [DeFiLlama Forks](https://defillama.com/forks/Uniswap%20V2) | 2025 |

**Notable forks:** PancakeSwap (BNB Chain, dominant), SushiSwap (multi-chain), Aerodrome V1 (Base), QuickSwap (Polygon), SpookySwap (Fantom). PancakeSwap and SushiSwap both launched 2020 as direct V2 forks.

## How it works: user perspective

1. A user selects two ERC-20 tokens and specifies an input amount.
2. The router computes the output amount using the constant product formula, deducting the 0.3% fee.
3. The user approves the router to spend their input token, then calls the swap function.
4. The router sends tokens to the pair contract, then calls `swap()` on the pair.
5. The pair verifies the new reserve product meets the invariant after fees, then transfers output tokens to the user.
6. Slippage protection is enforced by the router via a minimum output parameter.

As a liquidity provider:
1. Deposit equal value of both tokens; the pair mints LP tokens proportional to the geometric mean of deposited amounts.
2. The LP token represents a share of pool reserves plus accumulated fees.
3. Withdraw by returning LP tokens; the pair burns them and returns the proportional share of current reserves.

## How it works: protocol perspective

### Constant product invariant

The protocol enforces:

```
x · y = k
```

where `x` and `y` are the reserves of token0 and token1. After each swap the post-fee reserves must satisfy:

```
(x₁ · 0.997) · (y₁ · 0.997) ≥ x₀ · y₀
```

The 0.3% fee is collected by leaving it in the pool, growing `k` over time and thus growing LP token redemption value.

### Protocol fee (optional)

When the factory's `feeTo` address is set, the protocol collects 0.05% (one-sixth of the 0.3% fee). This is calculated lazily at liquidity events using the growth in the geometric mean of reserves:

```
fee = 1 − √(k₁) / √(k₂)
```

The fee is minted as new LP tokens to `feeTo`. After the December 2025 UNIfication governance vote, V2's protocol fee switch was activated on Ethereum mainnet from 28 December 2025, with fees flowing into the TokenJar contract for UNI token burns via the Firepit smart contract.

### Reserve update mechanism

The pair contract stores `reserve0` and `reserve1` as `uint112` values. After every swap, mint, or burn, the internal `_update()` function:
1. Reads the current ERC-20 balances of both tokens.
2. Computes the `price0CumulativeLast` and `price1CumulativeLast` accumulators (UQ112x112 fixed-point, time-weighted).
3. Writes the new balances into `reserve0` and `reserve1`.
4. Emits a `Sync` event.

The reserves serve as the authoritative cached state. They may diverge from actual balances when tokens are sent directly to the pair contract without triggering a swap (e.g., via direct transfer, positive rebase, or interest accrual).

### LP token minting

Initial deposit: `s_minted = √(x_deposited · y_deposited) − MINIMUM_LIQUIDITY` where `MINIMUM_LIQUIDITY = 10^-15` shares are burned to the zero address permanently, preventing pool poisoning attacks.

Subsequent deposits: `s_minted = min(x_deposited/x_reserve, y_deposited/y_reserve) · totalSupply`

### Flash swaps

A caller can receive tokens before paying by passing a non-zero `data` argument to `swap()`. The pair calls `IUniswapV2Callee.uniswapV2Call()` on the caller with the borrowed tokens. By the end of the callback, the caller must have returned the borrowed amount plus the 0.3% fee (or an equivalent value in the other token). The reentrancy `lock` modifier prevents nested flash swap exploitation.

### TWAP oracle

Each pair accumulates `price0CumulativeLast` and `price1CumulativeLast` as UQ112x112 fixed-point sums, incremented at the start of every block by `currentPrice × timeElapsed`. External contracts snapshot these at two points in time and divide the difference by elapsed seconds to compute a TWAP of any desired window. The oracle is manipulation-resistant because an attacker must hold a distorted price for the entire window duration while bearing arbitrage losses.

**Limitations of V2 TWAP:**
- External contracts must manage snapshots and elapsed time.
- Short windows (minutes) with thin liquidity remain manipulable.
- Multi-block MEV (post-Merge) reduces the cost of oracle manipulation if an attacker controls consecutive block proposals.
- The accumulator variable never decreases and eventually overflows (handled safely by modular arithmetic).

## Key behaviours

| Behaviour | Detail |
|-----------|--------|
| `sync()` | Resets `reserve0`, `reserve1` to match actual ERC-20 balances. Used to absorb a negative rebase or recover from a corrupted state without losing funds. |
| `skim(address to)` | Transfers any excess (balance minus reserve) to `to`. Handles positive rebase surplus and prevents `uint112` overflow when balance > 2^112 - 1. |
| Fee-on-transfer tokens | Core pair supports them; the router does not handle them correctly by default (use `swapExactTokensForTokensSupportingFeeOnTransferTokens`). |
| Rebasing tokens (positive) | Surplus accrues as publicly skimmable balance above reserves; anyone can call `skim()` to extract it. |
| Rebasing tokens (negative) | Reduces actual balance below cached reserve; the next swap reverts unless `sync()` is called first to write down the reserve. |
| Reentrancy | All mutating functions (`mint`, `burn`, `swap`, `skim`) use a `lock` modifier (uint `unlocked` flag) to prevent reentrancy. |
| Non-standard ERC-20 | Handles tokens with no return value on `transfer()` by assuming success. |

## Architecture decisions

### Core / periphery split

The protocol separates:
- **Core** (`UniswapV2Factory`, `UniswapV2Pair`): minimal, audited, immutable; holds all assets; executes AMM math.
- **Periphery** (`UniswapV2Router02`): stateless convenience layer; handles routing, multi-hop, slippage, fee-on-transfer helper functions, and deadline enforcement.

This design concentrates security surface in the immutable core while allowing the periphery to be upgraded without migrating liquidity.

### CREATE2 deterministic pair addresses

The factory deploys pairs using CREATE2 with a salt of `keccak256(abi.encodePacked(token0, token1))`. Any contract can compute a pair address off-chain without an on-chain lookup, saving gas on every router hop.

### `uint112` reserves with `uint224` product

Reserves are stored as `uint112` (max ~5.19 × 10^33 with 18 decimals: ~5.19 × 10^15 tokens). The product `reserve0 × reserve1` fits in `uint224`. The `skim()` function removes balance surplus before it triggers overflow reversion.

### Optimistic transfer model

`swap()` sends output tokens first, then calls the optional callee callback, then checks the invariant. This is what enables flash swaps. The reentrancy lock is essential to this design.

### Factory fee governance

`feeTo` and `feeToSetter` are set in the factory. Changing `feeToSetter` is a single-step operation (identified as a medium security risk: if set to an invalid address, fee governance is lost permanently).

## Differentiators vs V1

| Feature | V1 | V2 |
|---------|----|----|
| Token pairs | ERC-20/ETH only | ERC-20/ERC-20 direct |
| Price oracle | None | TWAP (on-chain cumulative) |
| Flash swaps | None | Yes |
| Protocol fee | None | Optional 0.05% (activated Dec 2025) |
| Fee-on-transfer support | No | Partial (periphery helper) |

## Limitations

1. **Impermanent loss:** LPs suffer value loss relative to holding when token prices diverge. Loss is proportional to the square of the price ratio change: IL = 2√r/(1+r) − 1 where r is the price ratio change.
2. **No MEV protection:** Swaps are mempool-visible; sandwich attacks (front-run buy, back-run sell) extract value from users. A 2025 Trail of Bits report found >12% of Ethereum AMM transactions were sandwich attacks.
3. **Rebasing token vulnerability:** Positive rebases create publicly extractable surpluses via `skim()`. Negative rebases cause swap reversions until `sync()` is called.
4. **Capital inefficiency:** Full-range liquidity provision (x·y=k) means most capital is deployed at prices far from the current price. Concentrated liquidity (V3+) addresses this.
5. **TWAP oracle manipulation:** Short windows and thin liquidity reduce manipulation cost. Multi-block MEV makes manipulation cheaper post-Merge.
6. **`uint112` reserve cap:** Tokens with very high total supply or price can overflow the reserve, requiring `skim()` to prevent permanent pool lock.
7. **Single-step `feeToSetter`:** No two-step transfer for fee governance admin, risking permanent loss of control.
8. **Fee-on-transfer router incompatibility:** Default router reverts for deflationary tokens; dedicated helper functions required.

## Security

- **Audit:** Formal verification and code review by a team of six engineers (dapp.org), January to April 2020. No critical or high-severity issues found. Two medium-severity issues: (1) router incompatibility with fee-on-transfer tokens; (2) a race condition in the deflation-fix mechanism.
- **Reentrancy guard:** `lock` modifier on all mutating pair functions.
- **Flash swap safety:** Invariant check enforced at end of `swap()` after callback; cannot be bypassed without reverting.
- **Minimum liquidity:** 10^-15 LP shares burned at first deposit, making LP share price manipulation attacks prohibitively expensive (~$100,000+ cost).
- **Chain fork replay:** ERC-20 Permit's `DOMAIN_SEPARATOR` is computed once at deployment; signatures can be replayed on forked chains (known limitation).
- **Bug bounty:** Ongoing program on Cantina (covers V4 and other versions; V2 is considered mature).

## References

- Uniswap V2 whitepaper: https://app.uniswap.org/whitepaper.pdf (Hayden Adams, Noah Zinsmeister; 2020)
- V2 pair contract docs: https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair
- V2 oracle docs: https://docs.uniswap.org/contracts/v2/concepts/core-concepts/oracles
- V2 security docs: https://docs.uniswap.org/contracts/v2/concepts/advanced-topics/security
- DeFiLlama V2 page: https://defillama.com/protocol/uniswap-v2
- DeFiLlama V2 forks: https://defillama.com/forks/Uniswap%20V2
- CoinMarketCap V2 exchange: https://coinmarketcap.com/exchanges/uniswap-v2/
- Rebasing token support article: https://support.uniswap.org/hc/en-us/articles/34258911235469-Rebasing-reflection-and-debasing-tokens
- Hacken V2 security analysis: https://hacken.io/discover/uniswap-v2-core-contracts-security/
- Zealynx V2 fork security guide: https://www.zealynx.io/blogs/uniswap-v2
- UNIfication governance blog post: https://blog.uniswap.org/unification
- RareSkills TWAP oracle deep dive: https://rareskills.io/post/twap-uniswap-v2
- Uniswap V2 GitHub repo: https://github.com/Uniswap/v2-core

## Related notes

- [[patterns/sync-skim]] — The `sync()`/`skim()` vault accounting pattern
- [[patterns/constant-product-amm]] — x·y=k invariant and constant product pricing
- [[metrics/tvl-comparison]] — TVL comparison across DEXs
- [[metrics/volume-comparison]] — Volume comparison across DEXs
