# Balancer V3

**Ecosystem:** Ethereum (also deployed on Arbitrum, Base, Gnosis Chain, Optimism, Avalanche, Sonic, Plasma, HyperEVM)
**Category:** Vault-based AMM with hooks framework and native yield-bearing token support
**Status:** Live (launched 11 December 2024); Balancer Labs corporate entity wound down March 2026; protocol continues under DAO governance
**Research date:** 2026-04-09

---

## Summary

Balancer V3 is a major architectural overhaul of Balancer V2 that centralises token accounting in a single Vault and shifts core design patterns out of individual pool contracts. The three headline changes are: (1) transient accounting via EIP-1153 enabling a secure "Till" pattern for re-entrant vault operations; (2) a formalised hooks framework allowing third-party contracts to intercept and modify every stage of the pool lifecycle; and (3) native ERC-4626 integration enabling 100% boosted pools where all LP capital accrues yield from external protocols between swaps. V3 launched on Ethereum and Gnosis Chain and subsequently expanded to Arbitrum, Base, Optimism, Avalanche, Plasma, and HyperEVM through 2025.

Related patterns: [[single-vault-architecture]], [[yield-bearing-token-support]]

---

## Adoption metrics

| Metric | Value | Source | Date |
|--------|-------|--------|------|
| V3 TVL (DeFiLlama estimate, pre-exploit peak on Plasma) | ~$200M on Plasma chain alone in one week | https://outposts.io/article/balancer-v3-achieves-dollar200m-tvl-on-plasma-in-one-week-2947d575-e829-435b-9971-c0ff783d19f6 | 2025-10-16 |
| V3 TVL (DeFiLlama, post-Labs shutdown period) | ~$104–151M across all chains | https://defillama.com/protocol/balancer-v3 | 2026-03 |
| All-Balancer TVL (peak, 2021) | ~$3.5B | https://www.coindesk.com/tech/2026/03/24/balancer-labs-will-shut-down-as-corporate-entity-became-a-liability-after-usd110-million-exploit | 2021 peak |
| All-Balancer TVL (post-exploit, 2026) | ~$157M (approx. 95% below peak) | https://defillama.com/protocol/balancer | 2026-03 |
| V3 30-day fees | ~$207K | https://mantapex.com/defi/fees/balancer-v3 | 2026-04 |
| V3 7-day fees | ~$51K | https://mantapex.com/defi/fees/balancer-v3 | 2026-04 |
| V3 chains deployed | 9 (Ethereum, Arbitrum, Base, Gnosis, Optimism, Avalanche, Sonic, Plasma, HyperEVM) | https://mantapex.com/defi/fees/balancer-v3 | 2026-04 |
| V3 launch date | 11 December 2024 | https://medium.com/balancer-protocol/balancer-v3-is-live-2e5e8462aa4c | 2024-12-11 |
| Avalanche TVL (early deployment) | ~$10M in first week | https://www.theblock.co/post/345510/balancer-v3-expands-to-avalanche-following-governance-vote | 2025-03 |
| Avalanche TVL (April 2025) | ~$20M | https://forum.balancer.fi/t/balancer-maxis-monthly-update-for-april-2025/6507 | 2025-04 |
| Internal bot initiative incremental volume | ~$2.3M additional volume since early April 2025 | https://forum.balancer.fi/t/balancer-maxis-monthly-update-for-april-2025/6507 | 2025-04-30 |
| Oracle/StableSurge test pools (7-day) | ~$5M volume; ~$28K fees to LPs and DAO | https://forum.balancer.fi/t/balancer-maxis-monthly-update-for-april-2025/6507 | 2025-04-30 |
| DAO revenue target (monthly) | $250K+ from V3 products | https://medium.com/@balancer.ballers/the-balancer-report-b2c610f15fa8 | 2025-04 |

---

## Architecture

### 1. Vault as accounting layer

The Balancer V3 Vault is the single point of custody for all tokens across all pools. Pools do not hold tokens; they are pure mathematics contracts that receive token amounts, compute invariants, and return output amounts. Token accounting, decimal scaling, rate scaling, fee collection, and BPT (Balancer Pool Token) minting are all handled by the Vault.

Key vault responsibilities in V3 (vs. V2, where pools handled more logic):

- **Decimal scaling:** The vault normalises all token amounts to 18-decimal precision before passing them to pool math, preventing per-pool scaling bugs.
- **Rate scaling:** For yield-bearing tokens, the vault applies live rates from registered Rate Providers, converting between raw token amounts and "live" (yield-accrued) amounts.
- **ERC20MultiToken:** The vault atomically updates both pool token balances and BPT supply in a single operation, closing the read-only reentrancy window that existed in V2.
- **Liquidity approximations:** Unbalanced add/remove operations are computed inside the vault, not the pool, making these operations available across all pool types without per-pool implementation.

Source: https://medium.com/balancer-protocol/balancer-v3-the-future-of-amm-innovation-f8f856040122

### 2. Transient accounting (EIP-1153 "Till" pattern)

Balancer V3's vault uses EIP-1153 transient storage (TLOAD/TSTORE opcodes, available since Ethereum's Dencun upgrade) to implement a deferred settlement pattern called the "Till".

**How it works:**

1. A caller (Router) invokes `unlock()` on the Vault, which sets a transient "unlocked" flag and calls back into the Router via a callback.
2. Inside the callback, the Router can execute multiple operations (swaps, adds, removes) across multiple pools. Each operation calls `_takeDebt()` or `_supplyCredit()` on the Vault, which increments a transient delta mapping per token.
3. A counter `_nonZeroDeltaCount` tracks how many tokens have outstanding non-zero deltas.
4. At the end of the callback, the Vault checks that `_nonZeroDeltaCount == 0`. If any token delta is non-zero, the entire transaction reverts.
5. Actual token transfers are settled via `settle()` (tokens transferred in) and `sendTo()` (tokens transferred out) at the net amounts only.

**Benefits:**

- Multiple pool hops settle net amounts: a multi-hop swap that routes through three pools moves only the input and output tokens, not intermediate tokens, across the Vault boundary.
- Gas savings: transient storage is much cheaper than persistent storage (SLOAD/SSTORE). Values are automatically cleared after the transaction completes.
- Secure re-entrancy: pools can call back into the Vault during hooks execution because the transient lock is designed to permit this safely; all outstanding debts must still net to zero.
- `settle()` includes an `amountHint` parameter to mitigate "donation" attacks where tokens are sent directly to the Vault to inflate credits.

Source: https://docs.balancer.fi/concepts/vault/transient-accounting.html; https://mixbytes.io/blog/modern-dex-es-how-they-re-made-balancer-v3

### 3. Hooks framework

Hooks are standalone contracts that can be registered against a pool at deployment time. The registration is immutable: a pool's hook contract cannot be changed after creation. A single hook contract may serve multiple pools of different types.

**Hook lifecycle points (10 total):**

| Hook | Trigger |
|------|---------|
| `onRegister` | When a pool is registered with the Vault |
| `onBeforeInitialize` | Before the pool's first liquidity provision |
| `onAfterInitialize` | After the pool's first liquidity provision |
| `onBeforeAddLiquidity` | Before any add-liquidity operation |
| `onAfterAddLiquidity` | After any add-liquidity operation |
| `onBeforeRemoveLiquidity` | Before any remove-liquidity operation |
| `onAfterRemoveLiquidity` | After any remove-liquidity operation |
| `onBeforeSwap` | Before swap math executes |
| `onAfterSwap` | After swap math executes |
| `onComputeDynamicSwapFeePercentage` | Fee computation point (replaces static fee) |

**Return deltas (hook-adjusted amounts):** When `enableHookAdjustedAmounts` is configured, after-hooks can modify the `amountCalculated` output of a swap or proportional liquidity operation. This allows a hook to divert a portion of swap output as a fee or redirect tokens to a different destination, all within the transient accounting settlement.

Source: https://docs.balancer.fi/concepts/core-concepts/hooks.html

### 4. Boosted pools and ERC-4626 liquidity buffers

A "boosted pool" in V3 is a pool whose registered tokens are ERC-4626 yield-bearing tokens (e.g. Aave's waDAI, waUSDC, waUSDT). Because all LP capital sits in yield-generating tokens, LPs earn the underlying yield (e.g. Aave lending rates) continuously between swaps, on top of swap fees.

**The 100% boosted pool:** V2 boosted pools required a "linear pool" wrapper that left a reserve of unwrapped tokens (e.g. raw DAI) for gas-efficient swaps. This reduced the yield-bearing fraction to typically 75–90%. V3 eliminates the need for BPT-nesting by introducing Vault-native liquidity buffers.

**Liquidity buffers:**

- Buffers are internal Vault mechanisms, not pools. They hold reserves of both a yield-bearing token (e.g. waDAI) and its underlying asset (e.g. DAI) to facilitate gas-efficient wrapping/unwrapping during swaps.
- Buffers operate at the `previewDeposit`/`previewMint` exchange rate of the ERC-4626 contract, so no external price oracle is required.
- Buffers are not transferable (LPs receive internal shares, not BPT).
- When a buffer lacks sufficient reserves for a swap, it automatically wraps or unwraps via the ERC-4626 contract.
- Liquidity buffers are initialised with `initializeBuffer()` and liquidity is added/removed proportionally only.

**Result:** 100% of LP capital can reside in yield-bearing tokens. A swap of DAI→USDC in a 100% boosted pool: (1) wraps DAI to waDAI via buffer; (2) swaps waDAI to waUSDC in the pool; (3) unwraps waUSDC to USDC via buffer. All three steps occur within a single `unlock()` callback, settled at net token deltas.

Source: https://docs.balancer.fi/concepts/vault/buffer.html; https://docs.balancer.fi/concepts/explore-available-balancer-pools/boosted-pool.html

### 5. Rate providers and yield capture

For each yield-bearing token registered in a pool, pool creators supply a Rate Provider: a contract implementing a `getRate()` function that returns the current exchange rate between the wrapped token and its underlying asset. The Vault applies this rate during scaling to convert raw amounts into "live" (yield-inclusive) amounts before pool math.

This means the pool's invariant is computed in terms of the underlying assets' value, preventing arbitrageurs from extracting yield through the pool. Yield accrual benefits LPs, not traders. A configurable yield fee (default 10% in V3, reduced from 50% in V2) is charged by the protocol on yield captured through boosted pools.

Source: https://medium.com/balancer-protocol/balancer-v3-is-live-2e5e8462aa4c

### 6. Pool types

| Pool type | Description |
|-----------|-------------|
| Weighted Pool | Up to 8 tokens with arbitrary weights (e.g. 80/20 BAL/WETH); classic Balancer weighted math |
| Stable Pool | StableSwap invariant optimised for correlated assets (stablecoins, LSTs) |
| Boosted Pool | ERC-4626 token pool with 100% yield-bearing composition; uses liquidity buffers |
| reCLAMM Pool | Readjusting Concentrated Liquidity AMM; self-adjusting virtual balances shift the active price range automatically (launched July 2025) |
| E-CLP | Elliptic Concentrated Liquidity Pool by Gyroscope |
| Blockchain Traded Funds | QuantAMM dynamic pool rebalancing product |

Source: https://docs.balancer.fi/concepts/explore-available-balancer-pools/; https://cryptoadventure.com/balancer-suspends-reclamm-pools-after-bug-bounty-report-says-funds-remain-safe/

### 7. Pool creator fee and ERC-5313 ownership

V3 introduces a `poolCreatorFeePercentage` that allows the address that deployed a pool to earn a share of swap fees and yield fees. The fee is set during pool registration and cannot be changed. Pool creators claim accrued fees via `withdrawPoolCreatorFees()`. The maximum creator fee is 100% of the protocol's share (not 100% of all fees), meaning LPs always receive their base LP share.

The `poolCreator` address is established at registration and implements ERC-5313 (lightweight contract ownership interface) to identify the fee recipient.

Source: https://docs.balancer.fi/concepts/core-concepts/pool-creator-fee.html

### 8. Fee model

| Fee component | V2 | V3 |
|--------------|----|----|
| Protocol share of swap fees | 50% | ~50% (split with creator) |
| Protocol yield fee | 50% | 10% |
| veBAL staker share | 75% of protocol fees | 82.5% of non-core pool fees; 12.5% of core pool fees |
| Pool creator fee | Not available | Up to 100% of protocol share |

The reduced yield fee in V3 (50% → 10%) was designed to make boosted pools more competitive by directing more yield to LPs.

Source: https://medium.com/balancer-protocol/balancer-v3-is-live-2e5e8462aa4c

### 9. Swap flow (step-by-step)

Using the MEV hook as an illustrative example:

1. User calls `swapSingleTokenExactIn` on the Router.
2. Router calls `unlock()` on the Vault; Vault calls back into the Router's callback.
3. Vault prepares `poolData` (balances, rates, scaling factors), `swapState` (token indices, scaled input), and `poolSwapParams`.
4. If the pool has a dynamic fee hook: `onComputeDynamicSwapFeePercentage` fires. For the MEV hook: checks priority gas price; if above threshold, computes `fee += (priorityGasPrice - threshold) × multiplier`, capped at maximum MEV fee.
5. Fee is applied to the input amount (EXACT_IN). Pool's `onSwap` is called; returns output in live-scaled units.
6. Vault descales the output, checks it meets the user's minimum (slippage limit).
7. `_takeDebt` / `_supplyCredit` record token deltas in transient storage. Protocol and creator fee amounts are deducted.
8. Pool's updated token balances are written to persistent storage.
9. Router calls `settle()` to transfer input tokens in; `sendTo()` to transfer output tokens out. Delta counter reaches zero.
10. Vault's `transient` modifier confirms `_nonZeroDeltaCount == 0`; state clears.

Source: https://medium.com/balancer-protocol/inside-a-balancer-v3-swap-a-step-by-step-walkthrough-with-the-mev-hook-f4a694928594

---

## Key behaviours

### Transient accounting ensures atomic multi-hop settlement
Multi-hop swaps accumulate credits and debts across all intermediate pools within a single `unlock()` callback. Only the net input and net output tokens are physically transferred. This reduces gas costs and eliminates intermediate token custody.

### Hooks enable MEV capture for LPs (MEV-Cap hook)
The MEV-Cap hook dynamically increases the pool swap fee based on the transaction's priority gas price. Transactions with high priority fees (characteristic of MEV arbitrage bots on L2 chains with priority ordering, such as Base and Optimism) pay a higher swap fee; this excess fee grows the pool invariant, benefiting all LPs. Retail users paying base-level priority fees are unaffected. The hook works on any pool type without external routing.

Source: https://medium.com/balancer-protocol/unlocking-mev-for-lps-introducing-balancer-v3-mev-capture-hooks-c81da5a7c022

### StableSurge hook protects stable asset pegs
The StableSurge hook implements directional dynamic fees for stable pools. When a pool's composition deviates beyond a configurable threshold (e.g. shifts from 50/50 to 60/40), the hook charges a higher fee on swaps that further worsen the imbalance, while swaps that restore balance pay only the base fee. The fee increases linearly beyond the threshold, parameterised by: threshold (Δ), surge coefficient (μ), base fee, and token count (n).

Source: https://medium.com/balancer-protocol/balancers-stablesurge-hook-09d2eb20f219

### reCLAMM pools provide maintenance-free concentrated liquidity
reCLAMM (Readjusting Concentrated Liquidity AMM) pools automatically shift their virtual balance parameters to keep the pool's active price range centred on the current market price. Admins and the pool itself can adjust the range, providing a "fire-and-forget" concentrated liquidity solution.

Source: https://cryptoadventure.com/balancer-suspends-reclamm-pools-after-bug-bounty-report-says-funds-remain-safe/

### 100% of LP capital earns yield
In 100% boosted pools, no "idle" unwrapped token reserve is held for swaps. The ERC-4626 liquidity buffer mechanism handles wrapping/unwrapping on demand, so all deposited capital earns yield from the underlying protocol (e.g. Aave) at all times.

---

## Security

### Pre-launch audits (V3)
- **Trail of Bits:** Manual code review (pre-launch 2024)
- **Spearbit:** Manual code review; report dated 2024-10-04
- **Certora:** Formal verification, August–September 2024; reported no vulnerabilities in V3 contracts
- Code-review competitions (open external review)

Sources: https://github.com/balancer/balancer-v3-monorepo/blob/main/audits/spearbit/2024-10-04.pdf; https://bitcoinethereumnews.com/tech/balancer-v3-security-overhaul-new-guardrails-and-audits/

### V2 exploit (November 2025) and V3 isolation

On 3 November 2025, Balancer V2's Composable Stable Pools were exploited for approximately $128M across Ethereum, Base, Polygon, and Arbitrum. The root cause was a rounding error in V2's `_upscaleArray` function that compounded across 65 micro-swap operations within a single `batchSwap`, enabling invariant manipulation and BPT price distortion. The attacker used a two-stage approach: compute the exploit in one transaction, extract profits in a separate transaction, and funded the operation via 100 ETH from Tornado Cash.

V3 was not affected. Architectural reasons:
1. **Unified decimal precision:** All scaling is handled in the Vault (not per pool), eliminating the localised rounding that could be exploited.
2. **Eliminated composable pools:** V3 replaced BPT-nesting composable pools with ERC-4626 buffers, removing the recursive structure that created exploitable edge cases.
3. **Explicit rounding directions:** Every calculation in V3 has a specified rounding direction, enforced by design.
4. **Formal verification of the roundtrip property:** Certora verified that "swapping some amount from one token to another and back again does not result in a gain of funds."

V3 security guardrails added post-exploit (for reference):
- Minimum token balance limits
- Maximum imbalance ratio (10,000:1) in stable pools
- Flash swap restrictions preventing impossible scenarios

Sources: https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025; https://www.certora.com/blog/breaking-down-the-balancer-hack; https://bitcoinethereumnews.com/tech/balancer-v3-security-overhaul-new-guardrails-and-audits/

### reCLAMM bug bounty pause (2026)
In 2026, Balancer paused reCLAMM-linked pools after receiving a security report through Immunefi. User funds were accessible throughout. The investigation was ongoing at research date.

Source: https://cryptoadventure.com/balancer-suspends-reclamm-pools-after-bug-bounty-report-says-funds-remain-safe/

---

## Limitations and risks

| Limitation | Detail |
|-----------|--------|
| Single Vault as systemic risk | All tokens across all V3 pools reside in one contract. A critical Vault vulnerability would affect all pools simultaneously (demonstrated by V2's single-vault exploit affecting all V2 pools). |
| Hook security surface | Hook contracts are third-party code. Malicious or buggy hooks can modify swap amounts via return deltas, potentially extracting LP value. The Vault does not audit hook logic. |
| Rate provider trust | Boosted pools depend on Rate Providers. A compromised or manipulable `getRate()` function could allow arbitrageurs to extract yield or cause incorrect pricing. |
| ERC-4626 dependency | 100% boosted pools inherit the security and liquidity risk of their underlying yield protocol (e.g. Aave). An Aave exploit or liquidity crunch propagates directly to Balancer boosted pool LPs. |
| reCLAMM operational risk | Self-adjusting pools shift parameters autonomously; the July 2025 bug bounty report indicates unexpected edge cases remain possible. |
| Rebasing token incompatibility | The V3 Vault documentation explicitly notes incompatibility with "rebasing or double entry point tokens." Positive or negative rebases in non-ERC-4626 rebasing tokens are not natively handled. |
| Governance and organisational risk | Balancer Labs dissolved as a corporate entity in March 2026. A lean OpCo (new corporate entity with essential staff) continues under DAO governance, but with BAL emissions ending and veBAL winding down, long-term LP incentive structures are uncertain. |
| Slippage parameter misconfiguration | V3 swap parameters require correct slippage limits to protect against MEV frontrunning; misconfigured limits can lead to user losses. |

---

## Differentiators (vs. V2)

| Feature | V2 | V3 |
|---------|----|----|
| Transient accounting | No (persistent storage throughout) | Yes (EIP-1153 Till pattern) |
| Hooks | No | Yes (10 lifecycle points; return deltas) |
| 100% boosted pools | No (linear pool wrappers; ~75–90% boosted) | Yes (ERC-4626 buffers; 100% boosted) |
| Decimal scaling | Per-pool | Vault-level (uniform 18-decimal) |
| Rate scaling | Via asset managers (complex) | Vault-level Rate Providers (standardised) |
| Pool creator fee | No | Yes (ERC-5313 ownership; up to 100% of protocol share) |
| Composable pool BPT nesting | Yes (source of V2 exploit) | Replaced by ERC-4626 buffers |
| Yield fee | 50% | 10% |
| Developer complexity | High (pools implement many responsibilities) | Reduced (~10x improvement claimed) |

Source: https://medium.com/balancer-protocol/balancer-v3-the-future-of-amm-innovation-f8f856040122

---

## Deployments

| Chain | Status | Notes |
|-------|--------|-------|
| Ethereum | Live (Dec 2024) | Primary deployment |
| Gnosis Chain | Live (Dec 2024) | Launch alongside Ethereum |
| Arbitrum | Live | GHO boosted pools; $2.3M+ additional volume from internal bots (Apr 2025) |
| Base | Live | MEV-Cap hook deployed; GHO/USDC pool >$13M TVL |
| Optimism | Live | MEV tax hook applicable |
| Avalanche | Live (Mar 2025) | ~$20M TVL as of Apr 2025; wAVAX/sAVAX/ggAVAX pools |
| Sonic | Live | [NOT FOUND: specific TVL] |
| Plasma | Live (Oct 2025) | $200M TVL in first week; USDe/USDT pool $28M |
| HyperEVM | Live (Jul 2025) | First week: $2.5M TVL in test pools; HyperCore oracle integration planned |

Sources: https://medium.com/balancer-protocol/balancer-v3-is-live-2e5e8462aa4c; https://medium.com/balancer-protocol/why-balancer-is-launching-on-hyperevm-7a45d5e61353; https://outposts.io/article/balancer-v3-achieves-dollar200m-tvl-on-plasma-in-one-week-2947d575-e829-435b-9971-c0ff783d19f6

---

## Organisational status (2026)

In March 2026, Balancer Labs (the Swiss corporate entity) announced it would wind down, citing legal and financial strain from the November 2025 V2 exploit. The protocol itself remains live. Key changes:

- A new "Balancer OpCo" (lean corporate entity with essential staff) continues operations.
- BAL token emissions end; veBAL governance model winds down.
- DAO treasury captures 100% of protocol fees (up from 17.5% previously).
- V3 protocol share drops to 25% to attract organic liquidity.
- Product scope narrows to: reCLAMM pools, liquidity bootstrapping pools, stablecoin/LST pools, weighted pools, and non-EVM chain expansion.
- A BAL buyback is planned as a fair exit for token holders who do not support the restructured protocol.

Source: https://unchainedcrypto.com/balancer-labs-shuts-down-its-corporate-entity-after-128-million-exploit-unchained/

---

## Data sources

- Balancer V3 launch announcement: https://medium.com/balancer-protocol/balancer-v3-is-live-2e5e8462aa4c
- Balancer V3 future vision: https://medium.com/balancer-protocol/balancer-v3-the-future-of-amm-innovation-f8f856040122
- Transient accounting docs: https://docs.balancer.fi/concepts/vault/transient-accounting.html
- Liquidity buffer docs: https://docs.balancer.fi/concepts/vault/buffer.html
- Boosted pool docs: https://docs.balancer.fi/concepts/explore-available-balancer-pools/boosted-pool.html
- Hooks docs: https://docs.balancer.fi/concepts/core-concepts/hooks.html
- Pool creator fee docs: https://docs.balancer.fi/concepts/core-concepts/pool-creator-fee.html
- MixBytes technical deep-dive: https://mixbytes.io/blog/modern-dex-es-how-they-re-made-balancer-v3
- Spearbit audit: https://github.com/balancer/balancer-v3-monorepo/blob/main/audits/spearbit/2024-10-04.pdf
- Certora exploit analysis: https://www.certora.com/blog/breaking-down-the-balancer-hack
- Halborn exploit analysis: https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025
- V3 security overhaul: https://bitcoinethereumnews.com/tech/balancer-v3-security-overhaul-new-guardrails-and-audits/
- MEV-Cap hook: https://medium.com/balancer-protocol/unlocking-mev-for-lps-introducing-balancer-v3-mev-capture-hooks-c81da5a7c022
- MEV swap walkthrough: https://medium.com/balancer-protocol/inside-a-balancer-v3-swap-a-step-by-step-walkthrough-with-the-mev-hook-f4a694928594
- StableSurge hook: https://medium.com/balancer-protocol/balancers-stablesurge-hook-09d2eb20f219
- Plasma $200M TVL: https://outposts.io/article/balancer-v3-achieves-dollar200m-tvl-on-plasma-in-one-week-2947d575-e829-435b-9971-c0ff783d19f6
- HyperEVM launch: https://medium.com/balancer-protocol/why-balancer-is-launching-on-hyperevm-7a45d5e61353
- Balancer Labs shutdown: https://unchainedcrypto.com/balancer-labs-shuts-down-its-corporate-entity-after-128-million-exploit-unchained/
- DeFiLlama V3: https://defillama.com/protocol/balancer-v3
- Maxis April 2025 update: https://forum.balancer.fi/t/balancer-maxis-monthly-update-for-april-2025/6507
