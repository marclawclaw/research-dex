---
tags: [project, ethereum, dex, stableswap, amm]
ecosystem: Ethereum
category: AMM (StableSwap)
website: https://curve.finance/
docs: https://docs.curve.finance/stableswap-exchange/stableswap-ng/pools/overview/
repo: https://github.com/curvefi/stableswap-ng
launched: 2023
---

# Curve StableSwapNG

Curve StableSwapNG ("Next Generation") is the current production iteration of Curve Finance's stableswap AMM, designed for efficient low-slippage trading of pegged assets. It supersedes Curve V1/V2 pool contracts with native support for rebasing tokens, rate-oraclised tokens, and ERC4626 vault tokens, while introducing admin fee isolation from LP balances and pools containing up to eight coins. Written in Vyper, it is deployed across Ethereum mainnet and multiple L2s.

## Adoption metrics

| Metric | Value | Date | Source |
|--------|-------|------|--------|
| Curve total DEX TVL | ~$2.3B (Q3 2025) | Q3 2025 | [PR Newswire Q3 2025 report](https://www.prnewswire.com/news-releases/curve-finance-reports-strong-q3-2025-trading-volume-hits-29b-while-revenue-more-than-doubles-302611749.html) |
| Curve Q3 2025 trading volume | $29B | Q3 2025 | [PR Newswire Q3 2025 report](https://www.prnewswire.com/news-releases/curve-finance-reports-strong-q3-2025-trading-volume-hits-29b-while-revenue-more-than-doubles-302611749.html) |
| Curve full-year 2025 trading volume | ~$126B | 2025 | [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) |
| Average 2025 protocol TVL | ~$3.05B | 2025 | [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) |
| Q3 2025 revenue (veCRV holders) | $7.3M (doubled QoQ from $3.9M) | Q3 2025 | [PR Newswire Q3 2025 report](https://www.prnewswire.com/news-releases/curve-finance-reports-strong-q3-2025-trading-volume-hits-29b-while-revenue-more-than-doubles-302611749.html) |
| Full-year 2025 fees to veCRV | $13.6M | 2025 | [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) |
| New pools created in 2025 | 2,209 (vs 2,042 in 2024, +8%) | 2025 | [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) |
| Pool interactions 2025 | 25.2M transactions (vs 11.8M in 2024, +113%) | 2025 | [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) |
| Q4 2025 active users | 51.8K (vs 20.5K in Q4 2024, +152% YoY) | Q4 2025 | [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) |
| Ethereum DEX fee share | Rose from 1.6% to 44% over 2025 (27.5x increase) | 2025 | [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) |
| GitHub stars (stableswap-ng repo) | 54 stars, 31 forks | April 2026 | [curvefi/stableswap-ng](https://github.com/curvefi/stableswap-ng) |

See also: [[metrics/curve-stableswapng-tvl-volume]]

## How it works

### User perspective
1. User selects a StableSwapNG pool (e.g. USDC/USDT, stETH/ETH, sDAI/USDC) and inputs a token amount to swap.
2. The pool computes the output using the stableswap invariant and the applicable dynamic fee. See [[patterns/stableswap-invariant]].
3. For rebasing-token pools, the `exchange_received` function is disabled; the user calls `exchange` directly.
4. For rate-oraclised tokens (e.g. wstETH), the pool fetches the current exchange rate via the configured oracle and normalises balances before computing the swap.
5. Liquidity providers deposit tokens in proportion to current pool balances, receiving LP tokens in return. All rebases on deposited tokens accrue to LP holders, not to the admin fee.
6. Admin fees (a fraction of swap fees) accumulate in a separate `admin_balances` array and can be withdrawn by the fee receiver without disturbing LP accounting.

### Protocol perspective
- The pool contract is deployed via a factory (`CurveStableSwapFactoryNG.vy`) using Vyper's `create_from_blueprint()` with the CREATE opcode (non-deterministic addresses, unlike Uniswap V4's CREATE2).
- Each pool is an independent contract holding its own token balances; unlike Balancer, there is no shared singleton vault.
- Pool variants: plain pools (up to 8 coins), metapools (2 coins: one asset paired with a base-pool LP token).
- The amplification parameter `A` (stored as `A * n^(n-1)` on-chain for gas efficiency) controls the curve shape: a high `A` concentrates liquidity near peg; a low `A` degrades gracefully toward constant-product behaviour as the pool becomes imbalanced.
- Price oracles use an exponential moving average (EMA) with a configurable `ma_exp_time` window; the EMA updates once per block, preventing intra-block manipulation.
- Governance can upgrade pool logic by indexing new implementations in the factory without redeploying individual pools.

## Key behaviours

- [[patterns/rebase-safe-accounting]] — `_balance()` reads live token balances; admin fees are stored in `admin_balances[]` array (not inferred from pool state minus balances); LPs receive 100% of positive rebases.
- [[patterns/stableswap-invariant]] — The stableswap invariant `An^n∑x_i + D = An^nD + D^(n+1)/(n^n∏x_i)` interpolates between constant-sum and constant-product; D is solved via Newton's method each swap.
- **Token type dispatch:** Four asset types (0 = plain ERC20, 1 = rate-oraclised token, 2 = rebasing token, 3 = ERC4626 token) select different balance normalisation paths. Type 1 tokens fetch a rate via an external oracle; type 3 tokens call `convertToAssets()`.
- **`exchange_received` disabled for rebasing pools:** Because the function relies on the difference between the new balance and the stored balance to infer the amount received, a rebase arriving in the same block would corrupt that accounting. The function reverts if any coin has `asset_type == 2`.
- **Dynamic fees via `offpeg_fee_multiplier`:** The effective fee rises when the pool moves away from peg, making oracle-driven sandwich attacks more expensive when price divergence is highest.
- **EMA oracle (once per block):** The pool stores a last-seen price and updates the EMA at most once per block, preventing intra-block manipulation of the oracle used for internal normalisation.
- **No hooks:** Unlike Balancer V3 or Uniswap V4, StableSwapNG has no configurable swap or liquidity hook interface; all logic is fixed in the deployed contract and can only be changed by governance deploying a new blueprint implementation.

## Architecture decisions

- **Separate `admin_balances[]` array:** Fee amounts are tracked explicitly rather than derived from the difference between actual balance and the theoretical pool state. This ensures rebases (positive or negative) only affect the LP share, not admin fees. The trade-off: if a slashing event (negative rebase) occurs and admin fees are withdrawn before LPs can exit, LPs may bear a disproportionate share of the loss.
- **`_balance()` reads live on-chain balances:** On every interaction, the pool calls `_balance(i)` which issues a live `balanceOf` query (or ERC4626 `convertToAssets` equivalent) rather than relying on a cached reserve. This is safe but costs additional external calls per token per operation.
- **Rate normalisation for oracles and ERC4626:** Token balances are scaled to a common "price units" representation before the stableswap invariant is evaluated, using the oracle rate or `convertToAssets`. This allows the invariant to treat, for example, wstETH and stETH as near-equal in value without distorting the curve.
- **Vyper 0.3.10 (post-exploit):** StableSwapNG uses a Vyper version well above the 0.2.15/0.2.16/0.3.0 range that contained the reentrancy guard storage-slot bug exploited in July 2023. The earlier Curve V1 factory pools (not StableSwapNG) were the victims of that exploit.
- **Blueprint deployment:** Factory uses `create_from_blueprint()` so the factory can update the implementation reference for future pools without touching existing ones. Existing pool logic is immutable after deployment.
- **Oracle packing:** Oracle contract address (20 bytes) and method selector (4 bytes) are packed into a single 256-bit storage slot, reducing storage read costs on each rate fetch.

## Fee structure

- **Swap fee (`fee`):** Set at pool deployment; typically 0.01% to 0.04% for stable pairs. Applied to the output token amount.
- **Admin fee (`admin_fee`):** A fraction of the swap fee directed to the protocol fee receiver (Curve DAO). Tracked in `admin_balances[]`, not in pool reserves.
- **Dynamic fee (`offpeg_fee_multiplier`):** Scales the effective fee upward when pool balances deviate from parity. Prevents sandwich attacks from being profitable: when oracle prices jump and the pool momentarily misprices, the multiplier raises the round-trip cost so arb profit is eliminated.
- **Imbalance fee on add/remove:** Adding or removing liquidity in a way that imbalances the pool incurs a fee proportional to the imbalance. LP tokens are minted/burned at a slightly penalised rate.

## Security

### 2023 Vyper reentrancy exploit (not StableSwapNG)
On 30 July 2023, Curve Finance factory pools lost approximately $70M due to a reentrancy vulnerability. The root cause was a storage-slot allocation bug in Vyper compiler versions 0.2.15, 0.2.16, and 0.3.0, which caused the `@nonreentrant` decorator to silently allocate distinct storage slots per function rather than sharing one lock, defeating the guard. Affected pools included JPEGd's pETH/ETH pool ($11.4M), Metronome sETH/ETH ($1.6M), and Alchemix alETH/ETH. These were Curve V1 factory pools, not StableSwapNG. StableSwapNG uses Vyper 0.3.10, which contains the patched compiler.

Sources: [Halborn post-mortem](https://www.halborn.com/blog/post/explained-the-vyper-bug-hack-july-2023), [Vyper post-mortem](https://hackmd.io/@vyperlang/HJUgNMhs2), [CoinTelegraph](https://cointelegraph.com/news/curve-finance-pools-exploited-over-24-reentrancy-vulnerability)

### MixBytes audit (Sep–Oct 2023)
An audit of StableSwapNG by MixBytes (3 auditors, Sep 6 to Oct 26, 2023) identified:
- **3 Critical:** (1) rebasing rewards stuck in contract (stored_balances did not account for accrued rebases, fixed), (2) `get_virtual_price()` manipulation, (3) read-only reentrancy in metapool with old base pool.
- **5 High:** oracle address whitelisting, incorrect storage update, incorrect base-pool swap, incorrect rate update, incorrect rewards distribution. All fixed.
- **12 Medium:** including DoS of `exchange_received`, dynamic fee not applied, incorrect fee parameters.
- **21 Low:** various code quality and edge-case issues.
- All critical, high, and medium findings were fixed or acknowledged by the Curve team before deployment.

Sources: [MixBytes audit README](https://github.com/mixbytes/audits_public/blob/master/Curve%20Finance/StableSwapNG/README.md)

### Ongoing oracle risk
Pools using rate oracles with volatile or externally controlled oracle contracts face sandwich attack risk if `offpeg_fee_multiplier` is not set appropriately. A three-step attack (buy before oracle update, wait for keeper to post new rate, sell at updated rate) is profitable whenever the oracle jump exceeds twice the flat fee. The dynamic multiplier addresses this but requires careful parameterisation at pool creation.

Source: [MixBytes: Safe StableSwap-NG Deployment](https://mixbytes.io/blog/safe-stableswap-ng-deployment-how-to-avoid-risks-from-volatile-oracles)

## Differentiators

- Best-in-class rebasing token support in a standalone AMM: `_balance()` reads live balances and `admin_balances[]` stores fees independently, so neither positive nor negative rebases corrupt fee accounting.
- Up to eight tokens per pool, versus two in Uniswap V2/V4 and two in most stableswap implementations.
- Four native asset types (plain, rate-oraclised, rebasing, ERC4626) handled by a single unified contract.
- `exchange_received` for integrators: allows routers to pre-transfer tokens and batch the swap in one call for gas savings (except on rebasing-token pools where it is disabled for safety).
- Governance-upgradeable implementations via factory blueprints without migrating existing pool liquidity.
- EMA price oracle built in, usable by external protocols as an on-chain price feed.

## Limitations and criticisms

- **No hook extensibility:** Unlike Uniswap V4 or Balancer V3, StableSwapNG offers no mechanism for third-party developers to add custom logic to swaps or liquidity events. Behaviour is fixed per implementation version.
- **Complex parameterisation:** Deployers must correctly set `A`, `fee`, `offpeg_fee_multiplier`, `ma_exp_time`, and per-token oracle addresses. Misconfiguration can create oracle manipulation vectors or price inefficiencies.
- **Oracle dependency for Types 1 and 3:** Rate-oraclised and ERC4626 pools depend on the correctness and liveness of external oracle contracts. If an oracle is externally controlled by an EOA, a malicious or mistaken update can immediately disrupt pool pricing.
- **Admin fee slashing asymmetry:** If a slashing event (e.g. stETH slashing) reduces the rebasing token balance and admin fees are withdrawn before LP redemptions, LPs bear a disproportionate share of the loss. The `admin_balances[]` array is immune to the slash; LP balances are not.
- **Non-deterministic addresses:** CREATE opcode (not CREATE2) means pool addresses cannot be pre-computed, complicating front-end tooling and router path discovery compared to Uniswap V4's deterministic singleton.
- **Per-pool token custody:** Each pool holds its own balances; cross-pool multi-hop swaps require multiple token transfers, unlike Balancer's single-vault architecture.
- **MEV exposure:** Despite dynamic fees, pools remain subject to sandwich attacks on large trades. StableSwapNG has no batch-auction or intent layer like CoW Protocol.
- **`exchange_received` disabled for rebasing pools:** Integrators cannot use the gas-optimised single-call path when any pool token is a rebasing token.

## Recent developments (2025)

- USDC/USDT pools on StableSwapNG achieved nearly 50x volume-to-TVL ratios in early 2025, demonstrating capital efficiency of the new pool architecture.
- Dynamic fee StableSwapNG pools featuring `offpeg_fee_multiplier` drove significant fee growth; Curve's share of Ethereum DEX fees rose from 1.6% to 44% over 2025.
- Pool creation grew 8% YoY (2,209 in 2025 vs 2,042 in 2024), with significant growth in pools exceeding $1M TVL (+41% YoY), indicating stronger institutional LP adoption.

Sources: [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/), [PR Newswire Q3 2025](https://www.prnewswire.com/news-releases/curve-finance-reports-strong-q3-2025-trading-volume-hits-29b-while-revenue-more-than-doubles-302611749.html)

## Sources

- [Curve StableSwapNG docs: Overview](https://docs.curve.finance/stableswap-exchange/stableswap-ng/pools/overview/) — accessed 2026-04-09
- [Curve StableSwapNG GitHub repo](https://github.com/curvefi/stableswap-ng) — accessed 2026-04-09
- [Curve StableSwapNG factory deployer API](https://docs.curve.finance/factory/stableswap-ng/deployer-api/) — accessed 2026-04-09
- [MixBytes: Modern DEXes — Curve StableSwapNG](https://mixbytes.io/blog/modern-dex-es-how-they-re-made-curve-stable-swap-ng) — accessed 2026-04-09
- [MixBytes: Safe StableSwap-NG Deployment](https://mixbytes.io/blog/safe-stableswap-ng-deployment-how-to-avoid-risks-from-volatile-oracles) — accessed 2026-04-09
- [MixBytes StableSwapNG public audit](https://github.com/mixbytes/audits_public/blob/master/Curve%20Finance/StableSwapNG/README.md) — accessed 2026-04-09
- [Curve 2025 Year in Review](https://news.curve.finance/curve-2025-year-in-review/) — accessed 2026-04-09
- [Curve Finance Q3 2025 report (PR Newswire)](https://www.prnewswire.com/news-releases/curve-finance-reports-strong-q3-2025-trading-volume-hits-29b-while-revenue-more-than-updates-302611749.html) — accessed 2026-04-09
- [RareSkills: Curve get_D() and get_y()](https://rareskills.io/post/curve-get-d-get-y) — accessed 2026-04-09
- [Halborn: Explained the Vyper Bug Hack (July 2023)](https://www.halborn.com/blog/post/explained-the-vyper-bug-hack-july-2023) — accessed 2026-04-09
- [Vyper compiler post-mortem (HackMD)](https://hackmd.io/@vyperlang/HJUgNMhs2) — accessed 2026-04-09
- [CoinTelegraph: Curve pools exploited reentrancy](https://cointelegraph.com/news/curve-finance-pools-exploited-over-24-reentrancy-vulnerability) — accessed 2026-04-09
- [DeFiLlama Curve DEX](https://defillama.com/protocol/curve-dex) — accessed 2026-04-09
