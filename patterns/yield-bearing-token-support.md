# Pattern: Yield-Bearing Token Support

**Canonical implementation:** [[balancer-v3]] (100% boosted pools, December 2024)
**Earlier version:** Balancer V2 boosted pools (2022); Curve StableSwapNG (ERC-4626 asset type)
**Related:** [[single-vault-architecture]]

---

## Problem

Yield-bearing tokens (e.g. Aave's waDAI, Compound's cDAI, Morpho vaults) continuously accrue value relative to their underlying asset. This creates three interrelated challenges for AMMs:

1. **Invariant drift:** If a pool stores raw token balances, the increasing exchange rate between a yield-bearing token and its underlying causes the invariant to drift over time. The pool math becomes stale.
2. **Yield leakage to arbitrageurs:** Without rate-aware pricing, the growing value of yield-bearing tokens can be extracted through swap arbitrage rather than accruing to LPs.
3. **Gas cost of wrapping/unwrapping:** Users typically hold underlying assets (DAI, USDC), not yield-bearing wrappers (waDAI, waUSDC). An AMM that requires yield-bearing tokens must either (a) require users to wrap before trading, or (b) handle wrapping/unwrapping internally, which adds gas cost.
4. **Idle capital:** Early boosted pool designs (Balancer V2 "linear pools") required a buffer of unwrapped tokens to handle small swaps gas-efficiently, leaving 10–25% of LP capital undeployed to yield protocols.

---

## Approaches by protocol

### Curve StableSwapNG: rate-oraclised tokens

Curve StableSwapNG supports three token types:

- **Standard ERC-20:** No special handling.
- **Rebasing ERC-20 (e.g. stETH):** Balances read directly from the token; LPs receive full rebase; admin fees stored separately to avoid corruption.
- **ERC-4626 tokens:** The pool calls `convertToAssets(1e18)` to obtain a live exchange rate, normalises the token's balance by this rate before computing the StableSwap invariant, and adjusts swap math accordingly.

Curve's approach keeps rate logic inside the pool contract and does not require wrapping/unwrapping: pools hold yield-bearing tokens natively, and users trade between yield-bearing tokens directly. The exchange rate is embedded in the invariant calculation.

Source: https://docs.curve.finance/stableswap-exchange/stableswap-ng/pools/overview/

### Balancer V2: linear pools and boosted pools (older design)

V2 introduced "linear pools" as a wrapping layer: a two-token pool holding DAI and waDAI, plus a virtual BPT. The linear pool maintained a target reserve of raw DAI for efficient small swaps, and routed excess to Aave. A "boosted pool" (e.g. the Aave USD pool) was then a pool of linear pool BPTs. This created a nested BPT structure.

Limitation: the nested structure left approximately 10–25% of capital as unwrapped reserves in linear pools, reducing effective yield. The BPT-nesting also created the composable pool complexity that enabled the November 2025 V2 exploit.

Source: https://docs-v2.balancer.fi/concepts/pools/boosted.html

### Balancer V3: Rate Providers + ERC-4626 liquidity buffers (current best practice)

V3 replaces nested BPTs with two complementary mechanisms:

**Rate Providers:**
Each yield-bearing token registered in a V3 pool has an associated Rate Provider, a lightweight contract with a single function `getRate()` returning the current exchange rate (e.g. 1 waDAI = 1.08 DAI). The Vault applies this rate during scaling before pool math executes:

```
liveAmount = rawAmount × rate
```

Pool math operates entirely in "live" (underlying-equivalent) amounts. Swap output is descaled back to raw amounts before token transfer. This ensures:
- Arbitrageurs cannot profit from yield accrual (the rate is reflected in prices at every swap).
- LPs capture all yield.
- Pools work correctly with any ERC-4626 token regardless of its exchange rate mechanics.

**ERC-4626 liquidity buffers:**
A liquidity buffer is a Vault-internal mechanism (not a pool) that holds reserves of both a yield-bearing token (e.g. waDAI) and its underlying (e.g. DAI). Buffers facilitate gas-efficient wrapping/unwrapping during swaps without requiring a separate pool contract.

Buffer mechanics:
1. Buffers execute wraps and unwraps at the `previewDeposit` / `previewMint` rate of the ERC-4626 contract (no external oracle required).
2. When a buffer has sufficient reserves, wrapping/unwrapping is serviced directly from buffer reserves (gas-efficient).
3. When a buffer lacks reserves, it calls the ERC-4626 contract to wrap/unwrap as needed (higher gas, but correct).
4. Buffer liquidity providers receive internal non-transferable shares (not BPT).
5. Buffers are initialised with `initializeBuffer()` and liquidity is added/removed proportionally only.

**Combined result: 100% boosted pools**

A 100% boosted pool holds only yield-bearing tokens (e.g. waDAI and waUSDC). All LP capital is deployed to the underlying yield protocol (e.g. Aave) at all times. A user swapping DAI→USDC experiences:

1. Router calls `unlock()` on Vault.
2. DAI buffer wraps DAI → waDAI (or serves from buffer reserves).
3. Vault swaps waDAI → waUSDC via the boosted pool math (Rate Provider rates applied; StableSwap invariant computed in underlying-equivalent amounts).
4. USDC buffer unwraps waUSDC → USDC.
5. Net: user delivers DAI; user receives USDC. All wrapped token handling is internal.

Capital efficiency: 100% of LP deposits earn Aave yield continuously, plus swap fees on top.

Source: https://docs.balancer.fi/concepts/vault/buffer.html; https://docs.balancer.fi/concepts/explore-available-balancer-pools/boosted-pool.html

---

## Comparison of approaches

| Dimension | Curve StableSwapNG | Balancer V2 (linear pools) | Balancer V3 (buffers) |
|-----------|-------------------|--------------------------|----------------------|
| Rate application | Inside pool (per-invariant call) | Inside linear pool contract | Vault-level (Rate Provider) |
| Unwrapped reserve for swaps | Not applicable (holds yield-bearing tokens only; users trade yield-bearing to yield-bearing) | Yes (~10–25% of capital) | Minimal (buffer reserves; not capital idle) |
| Capital deployed to yield | 100% (of yield-bearing token fraction) | 75–90% | 100% |
| User experience | Users receive yield-bearing tokens | Users can interact in underlying; BPT nesting hidden | Users interact in underlying; wrapping/unwrapping hidden |
| Rebasing tokens | Yes (native) | No (not natively supported) | No (explicitly incompatible with rebasing tokens) |
| ERC-4626 tokens | Yes (native) | Partial (via bespoke linear pool adapters) | Native (any ERC-4626-compliant token) |
| External oracle needed | No (uses `convertToAssets`) | Rate provider per token (V2 complexity) | No (ERC-4626 `previewDeposit`/`previewMint` used) |

---

## Risks of yield-bearing token integration

1. **Rate Provider manipulation:** If a Rate Provider's `getRate()` can be manipulated (e.g. via a flash loan to skew an on-chain price), the pool can be drained. Rate Providers must be immutable or use time-weighted rates.
2. **Underlying protocol risk:** If Aave suffers an exploit or a liquidity crunch, all 100% boosted pools backed by Aave assets are affected. This is a direct dependency risk for LPs.
3. **ERC-4626 implementation correctness:** The buffer relies on `previewDeposit` and `previewMint` being correct. A non-compliant or buggy ERC-4626 implementation could cause incorrect pricing.
4. **Buffer liquidity shortage:** If a buffer runs dry and the underlying ERC-4626 wrapping is expensive or reverts, swaps through that buffer will fail or become very costly.

---

## Portability assessment for a new chain

Yield-bearing token support is highly valuable but optional for an MVP DEX. The key design decisions:

1. **Rate scaling at Vault level** (Balancer V3 approach) is superior to per-pool scaling because it is applied consistently across all pool types and reduces per-pool audit complexity.
2. **ERC-4626 as the standard interface** is the most portable choice: any yield protocol implementing ERC-4626 integrates without bespoke code.
3. **Liquidity buffers** can be a Vault-internal mechanism without pool contracts; they require careful initialisation and proportional-only liquidity management to avoid edge cases.
4. **Rebasing tokens** (non-ERC-4626, like stETH) require separate handling. Wrapping them into ERC-4626 equivalents (e.g. wstETH) before they enter the Vault is the recommended approach.

---

## See also

- [[balancer-v3]] — main project note with full architecture
- [[single-vault-architecture]] — how the vault enables vault-level rate scaling
