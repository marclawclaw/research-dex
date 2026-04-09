# Pattern: MEV-Capturing AMM

**Also known as:** Function-Maximising AMM, FM-AMM, LVR-free AMM, arbitrage-capturing AMM
**Primary example:** [[projects/cow-protocol]] (CoW AMM)
**Academic basis:** Columbia University research on LVR (Loss-Versus-Rebalancing)
**Status:** Production (CoW AMM on Balancer v3 since 2024)

## Summary

A MEV-capturing AMM replaces the traditional "always-on" constant-function AMM model with a batch auction mechanism for pool rebalancing. When an arbitrage opportunity exists (the pool's price is stale relative to external markets), solvers compete to rebalance the pool. The winning solver offers the most favourable terms for the pool, capturing the arbitrage profit for liquidity providers rather than allowing external MEV bots to extract it.

This pattern directly addresses Loss-Versus-Rebalancing (LVR), estimated at over $500M per year in LP losses across DeFi.

## The Problem: Loss-Versus-Rebalancing (LVR)

### What is LVR?

LVR (Loss-Versus-Rebalancing) is the value lost by liquidity providers due to arbitrage against their pool. It is distinct from impermanent loss.

**Mechanism:**
1. An AMM pool has a stale price (e.g., it prices ETH at $3,000 while external markets have moved to $3,050).
2. An arbitrageur notices this discrepancy and buys ETH from the pool at the stale price.
3. The arbitrageur immediately sells on external markets at $3,050, pocketing the $50 difference.
4. The LP's position rebalances to reflect the new price, but the value extracted by the arbitrageur came directly from the LP.

**Mathematical formulation:** Instantaneous LVR for a constant-function AMM equals σ²/8, where σ is the asset's price volatility. More volatile assets produce proportionally more LVR.

**Comparison with impermanent loss:** Unlike impermanent loss, LVR does not reverse when prices return to their original level. Every individual arbitrage event is a permanent, directional value transfer from LPs to arbitrageurs.

**Academic source:** Originally quantified by researchers at Columbia University. Academic paper: https://anthonyleezhang.github.io/pdfs/lvr.pdf

**Scale:** LPs lose more than $500M to LVR per year across DeFi. In an environment of 5% asset volatility, a Uniswap pool would need 10% daily volume turnover at 30 bps fees to offset LVR losses.

**Source:** https://cow.fi/learn/what-is-loss-versus-rebalancing-lvr, https://docs.cow.fi/cow-amm/concepts/the-problem-of-lvr

## The Solution: Batch Auction Rebalancing

### Core Mechanism

A MEV-capturing AMM replaces continuous open access to the pool with a competitive auction whenever a rebalancing opportunity exists:

1. **Monitoring:** Off-chain agents (using the same solver infrastructure as the order-matching protocol) continuously monitor the pool's current price versus external reference prices.
2. **Auction trigger:** When an arbitrage opportunity is detected (external price diverges from pool price beyond a threshold), an auction is opened.
3. **Solver competition:** All registered solvers bid for the right to rebalance the pool. The winning bid is the one that moves the pool's invariant curve highest (i.e., leaves the most value in the pool after rebalancing).
4. **Settlement:** The winning solver executes the rebalancing trade against the pool. Instead of an MEV bot taking 100% of the arbitrage profit, the solver takes a competitive margin and the remainder stays in the pool as LP surplus.

**Net effect:** The arbitrage still occurs (markets must be efficient), but the captured value flows to LPs rather than to extractive bots.

**Source:** https://cow.fi/learn/cow-dao-launches-the-first-mev-capturing-amm, https://docs.cow.fi/cow-amm

### Function Maximisation

The technical term for the selection criterion is "function-maximising." The pool's invariant function (e.g., x × y = k for a standard CFAM) is the objective. The winning solver is the one whose proposed post-rebalance pool state places the invariant at the highest possible value. This is equivalent to maximising the total value of the pool's reserves.

**Source:** https://docs.cow.fi/cow-amm

## CoW AMM Implementation

### Integration with Balancer v3

CoW AMM is implemented as a native pool type within Balancer v3's architecture. LPs:
1. Deposit tokens into a CoW AMM pool via the Balancer interface (same UX as standard Balancer pools)
2. Receive Balancer pool tokens representing their share
3. Participate automatically in rebalancing auctions via CoW Protocol's solver network

No additional UI or action is required from LPs beyond standard liquidity provision.

### Zero Swap Fees

CoW AMM pools charge zero swap fees to traders during rebalancing. The LP's revenue comes entirely from the surplus captured in the rebalancing auction rather than from per-trade fees. This is economically coherent because LVR extraction was previously the primary cost to LPs; replacing it with surplus capture removes the loss without needing to compensate via fees.

### Solver Role for CoW AMM

CoW Protocol solvers participate in CoW AMM rebalancing auctions using the same infrastructure they use to settle user trade batches. This creates synergies:
- Solvers can combine user order flow with CoW AMM rebalancing in a single batch
- A user's order to buy ETH might simultaneously rebalance a CoW AMM ETH/USDC pool that was holding a stale price
- Both the user and the LP benefit from the combined settlement

**Source:** https://docs.cow.fi/cow-amm/tutorials/cow-amm-for-solvers

## Performance Data

| Metric | Value | Source | Date |
|---|---|---|---|
| LP returns vs reference pools (backtesting) | Equal or better for 10 of 11 most liquid non-stablecoin pairs | https://cow.fi/cow-amm | 6-month 2023 backtesting |
| TVL uplift vs reference pools | ~4.75% (beta phase) | https://cow.fi/cow-amm | 2024-2025 |
| LP liquidity protected from LVR | $18M+ | https://cow.fi/cow-amm | 2024-2025 |
| LP surplus captured since launch | $1.2M+ (beta phase) | https://cow.fi/cow-amm | 2024-2025 |
| DeFi-wide LVR losses | $500M+ per year | https://cow.fi/learn/what-is-loss-versus-rebalancing-lvr | academic estimate |

See also: [[metrics/lvr-impact]]

## Trade-offs vs Traditional CFAMs

| Property | MEV-Capturing AMM | Traditional CFAM |
|---|---|---|
| LVR to LPs | Eliminated (captured as surplus) | Full exposure |
| Continuous trading | No (auction-based rebalancing) | Yes (always-on) |
| Swap fees | Zero (CoW AMM model) | Typically 5-30 bps |
| Latency to correct stale price | Auction delay | Immediate (any block) |
| Complexity | Higher (solver infrastructure required) | Lower |
| Applicable to | EVM chains with solver network | Any EVM chain |
| MEV resistance for LPs | Strong | None |
| Gas efficiency | High (batched with user trades) | Per-trade |

**Key trade-off:** The auction mechanism introduces a delay between price divergence and pool rebalancing. During this window, the pool operates with a temporarily stale price. In practice this is small (solver auctions complete in seconds) but non-zero.

## Relationship to Intent-Based Trading

The MEV-capturing AMM pattern is a direct extension of [[patterns/intent-based-trading]] applied to liquidity provision:

- Traditional LP: provides liquidity passively; all rebalancing occurs via immediate on-chain arbitrage (LP has no control)
- MEV-capturing LP: liquidity is rebalanced via competitive auction; LP effectively "intends" to rebalance at the best available terms

Both patterns share the insight that delegating execution to a competitive solver layer and settling via batch auction produces better economic outcomes for the counterparty to the trade (user in intent-based trading; LP in MEV-capturing AMM).

## Academic and Research Context

The LVR concept was formalised in the paper "Automated Market Making and Loss-Versus-Rebalancing" by Jason Milionis, Ciamac Moallemi, Tim Roughgarden, and Anthony Lee Zhang (Columbia University). The paper proves that LVR is an unavoidable structural property of constant-function AMMs and cannot be eliminated by fee choice alone.

The CoW AMM implements one of the theoretical remedies proposed in the research literature: replacing continuous access with periodic batch auctions.

**Sources:**
- Academic paper: https://anthonyleezhang.github.io/pdfs/lvr.pdf
- a16z analysis: https://a16zcrypto.com/posts/article/lvr-quantifying-the-cost-of-providing-liquidity-to-automated-market-makers/
- Fenbushi analysis: https://fenbushi.vc/2024/01/20/ending-lps-losing-game-exploring-the-loss-versus-rebalancing-lvr-problem-and-its-solutions/

## Limitations

- Requires an active solver network to function; bootstrapping is non-trivial.
- Auction-based rebalancing introduces a brief window of price staleness (sub-block latency on fast networks; seconds on Ethereum Mainnet).
- Not suitable for assets with very low solver interest (insufficient arbitrage volume to incentivise solver participation).
- Currently only production-deployed on Ethereum via Balancer v3; portability to other chains requires solver network availability.

## Related Patterns

- [[patterns/batch-auction-settlement]] — the auction mechanism used for rebalancing
- [[patterns/intent-based-trading]] — the complementary pattern for user trade execution

## Projects Using This Pattern

- [[projects/cow-protocol]] (CoW AMM, production on Balancer v3 since 2024)
