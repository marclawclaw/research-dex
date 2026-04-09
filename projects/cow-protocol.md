# CoW Protocol + CoW AMM

**Category:** Intent-based DEX aggregator / MEV-capturing AMM
**Ecosystem:** Ethereum (also Gnosis Chain, Arbitrum, Base, Avalanche, Polygon, Lens Chain, BNB Chain, Linea, Ink as of early 2026)
**Status:** Live, actively developed
**Website:** https://cow.fi
**Docs:** https://docs.cow.fi
**GitHub:** https://github.com/cowprotocol

## Summary

CoW Protocol is an intent-based trading protocol that uses off-chain batch auctions and a permissioned solver competition to match and settle user orders. Users sign trade intents rather than submitting on-chain transactions directly, which eliminates mempool exposure and enables MEV protection by construction.

CoW AMM is a companion product: a Function-Maximising AMM that captures arbitrage value (LVR) for liquidity providers instead of allowing MEV bots to extract it, by submitting pool rebalancing to the same solver auction mechanism.

## Origin and History

- CoW Protocol descends from Gnosis Protocol, first announced by the Gnosis team and the result of two years of R&D into better Ethereum trading mechanisms. Source: https://medium.com/gnosis-pm/announcing-gnosis-protocol-2fdc4e6d5c7b
- Gnosis Protocol v2 launched April 2021; CoW Swap was its first trading interface, initially as a proof-of-concept. Source: https://beincrypto.com/defi-deep-dive-cowswap-building-new-market-mechanisms-for-defi/
- Rebranded from Gnosis Protocol to CoW Protocol and spun out as an independent DAO (via GnosisDAO proposal GIP-13) in early 2022. Source: https://blog.cow.fi/gnosis-protocol-turns-cow-protocol-481c9fa90bb2
- Raised $23 million in a spin-out funding round in 2022. Source: https://www.theblock.co/linked/139692/cow-protocol-raises-23-million-and-spins-out-from-gnosis-dao
- Co-founders: Anna George (CEO) and Felix Leupold (Technical Co-Founder), formerly of Gnosis.
- 2024: Fee switch activated (January); CoW AMM launched on Balancer.
- 2025: $87B trading volume year; expanded to seven new chains.

## Adoption Metrics

| Metric | Value | Source | Date |
|---|---|---|---|
| 2025 trading volume | $87 billion | https://cow.fi/learn/cow-dao-2025-in-review | 2025 annual review |
| 2024 trading volume | $40.2 billion | https://cow.fi/learn/cow-dao-2025-in-review | 2025 annual review |
| Cumulative transactions | 73M+ | https://cow.fi/learn/cow-dao-2025-in-review | 2025 annual review |
| Surplus generated for users (since 2021) | $188M+ | https://www.dlnews.com/articles/defi/how-cow-swap-protocol-will-make-money-off-its-trading-volume/ | 2024 |
| Market position | Top 3 DEX aggregators across DeFi | https://cow.fi/learn/cow-dao-2025-in-review | 2025 annual review |
| New chain volume share | ~30% of total activity by Q4 2025 | https://cow.fi/learn/cow-dao-2025-in-review | Q4 2025 |
| Discord community | 12,000 members | https://cow.fi/learn/cow-dao-2025-in-review | 2025 annual review |
| Twitter/X followers | 66,800 | https://cow.fi/learn/cow-dao-2025-in-review | 2025 annual review |
| Barter solver market share | 28.2% (on Ethereum) | https://blockworks.co/news/barter-buys-rival-solver-codebase | September 2025 |
| Active solvers | 30+ independent teams | https://www.shoal.gg/p/cow-swap-intents-mev-and-batch-auctions | 2024-2025 |
| CoW AMM LP surplus captured | $100,000+ | https://cow.fi/cow-amm | 2024-2025 |
| CoW AMM TVL uplift vs reference pools | ~5% | https://cow.fi/cow-amm | 2024-2025 |

See also: [[metrics/cow-protocol-volume]]

## Architecture Overview

CoW Protocol separates intent expression from on-chain execution. The full flow:

1. **Intent signing:** A user signs an off-chain message specifying tokens to swap, minimum acceptable price, and a deadline. The signed intent never enters the public mempool.
2. **Off-chain order book:** Intents accumulate in an off-chain order book operated by the protocol's Autopilot service.
3. **Batch formation:** Every ~30 seconds, the Autopilot closes the collection window and bundles accumulated intents into a batch auction.
4. **Solver competition:** The batch is distributed to all whitelisted solvers simultaneously. Each solver submits a proposed settlement, optimising for maximum user surplus (price improvement above signed minimum).
5. **Winner selection:** The solver delivering the greatest total surplus across the batch wins and is rewarded in COW tokens.
6. **On-chain settlement:** The winning solver submits a single on-chain transaction. The settlement contract verifies each intent signature, checks that execution satisfies the user's terms (price, quantity, deadline), and executes transfers via a vault relayer holding ERC-20 approvals.

Source: https://cow.fi/learn/how-cow-protocol-actually-works, https://docs.cow.fi/cow-protocol

See also: [[patterns/batch-auction-settlement]], [[patterns/intent-based-trading]]

### Settlement Execution Hierarchy

Solvers attempt these approaches in preference order:

1. **Coincidence of Wants (CoW):** Directly match opposing intents in the same batch, peer-to-peer. No on-chain liquidity is touched; zero fees, zero slippage, zero MEV exposure.
2. **Private market-maker liquidity:** Access off-chain professional inventory for better quotes, particularly effective on large orders.
3. **On-chain DEX aggregation:** Route across Uniswap, Balancer, Curve, and other on-chain sources; gas costs are amortised across all bundled orders.

Source: https://cow.fi/learn/how-cow-protocol-actually-works

### Uniform Directed Clearing Prices

All orders trading the same token pair in the same direction within a batch settle at the same price (Uniform Directed Clearing Price, UDCP). This renders transaction ordering irrelevant: MEV bots gain nothing from reordering trades inside a batch.

Source: https://docs.cow.fi/cow-protocol/concepts/benefits/mev-protection, https://docs.cow.fi/cow-protocol/reference/core/auctions/competition-rules

### Fair Combinatorial Batch Auction (FCBA)

Each solver may submit multiple bids: bids on individual orders and "batched bids" covering groups of orders simultaneously (enabling CoW matches). The protocol filters batched bids that would give any individual order less than it would receive in a solo auction (individual fairness guarantee). The winning combination is the one maximising total surplus subject to this constraint.

Introduced in 2025 as a major architectural upgrade enabling safe, MEV-protected handling of multi-asset atomic trades.

Source: https://docs.cow.fi/cow-protocol/concepts/introduction/fair-combinatorial-auction

## Solver Ecosystem

Solvers are off-chain optimisation algorithms that compete to settle batches. They must be whitelisted by CoW DAO and post a bond (collateral that can be slashed for misconduct).

**Qualification:** Whitelisting requires posting a bond. Anyone with DeFi/optimisation knowledge can apply, but in practice the bonding requirement creates a significant financial barrier and most smaller participants cannot qualify without external vouching.

**Rewards:** The winning solver receives COW tokens. The reward model has evolved: CIP-38 introduced scoring based on executed trade amounts (surplus-based); CIP-67 aimed to increase reward amounts aligned with protocol revenue; CIP-74 (November 2025) introduced a dynamic cap tied to protocol fees and an unconditional 0.02% volume-based protocol fee.

**Execution deadlines (block-based):**
- Ethereum Mainnet: 3 blocks
- Gnosis Chain: 5 blocks
- Base, Avalanche, Polygon, Linea, Plasma: 20 blocks
- Arbitrum, BNB: 40 blocks

Missing the deadline results in immediate denylisting pending manual review.

**Slashing conditions:** CoW DAO may penalise solvers for EBBO violations (executing at worse prices than baseline protocols like Uniswap or Balancer), score inflation via fake tokens or wash trading, pre/post-hook omission, and systematic pennying/overbidding.

**Notable solvers (as of 2025):**
- Barter: ~28.2% Ethereum market share (September 2025); acquired Copium Capital's codebase in September 2025.
- Wintermute: early major solver.
- Velora (formerly Paraswap): active solver.
- 30+ independent teams active overall.

Source: https://docs.cow.fi/cow-protocol/concepts/introduction/solvers, https://docs.cow.fi/cow-protocol/reference/core/auctions/competition-rules, https://brrrdao.substack.com/p/cow-protocol-part-2-cow-solver-economy

## CoW AMM

CoW AMM is the first MEV-capturing AMM. It integrates with Balancer v3 as a pool type, so LPs deposit via the Balancer interface.

**Problem it solves:** Traditional CFAMs (constant-function AMMs) leak value to arbitrageurs via Loss-Versus-Rebalancing (LVR). LVR occurs whenever an AMM's price is stale relative to external markets; arbitrageurs extract that difference at LP expense. LVR costs LPs over $500M per year across DeFi. See [[patterns/mev-capturing-amm]], [[metrics/lvr-impact]].

**Mechanism:** When an arbitrage opportunity exists (the off-chain market price diverges from the pool's current price), solvers compete in a batch auction for the right to rebalance the pool. The winning solver is the one that moves the pool's invariant curve highest (maximising surplus returned to LPs). Rather than giving the arbitrage profit to a bot, the protocol captures it for the pool's LPs.

**Performance (backtesting 2023 data):** CoW AMM returns equalled or outperformed traditional CF-AMM returns for 10 of the 11 most liquid non-stablecoin pairs across a 6-month testing period.

**TVL uplift:** ~5% improvement in TVL compared to reference pools.

**Surplus captured for LPs:** $100,000+ since launch.

Source: https://cow.fi/cow-amm, https://docs.cow.fi/cow-amm, https://cow.fi/learn/cow-dao-launches-the-first-mev-capturing-amm

## MEV Blocker

A separate but related CoW DAO product: an RPC endpoint that routes Ethereum transactions through a network of searchers. Protects users from sandwich attacks. When backrunning occurs, 90% of the profit is returned to the user as a rebate in the same block.

Maintained jointly by CoW DAO, Agnostic Relay, and Beaver Build. Protected approximately 5% of all Ethereum transactions without major wallet integrations. In 2025, CoW DAO approved CIP-73 to sell its 50% stake in MEV Blocker.

Source: https://mevblocker.io, https://cow.fi/learn/what-does-an-mev-blocker-do

## Order Types

- **Market orders:** Executed in the next available batch at the best available price.
- **Limit orders:** User specifies a target price; executes when market price meets or beats it before expiry. CoW Protocol earns 50% of any surplus generated above the limit price.
- **TWAP orders:** Time-Weighted Average Price; splits a large order into equal-sized sub-orders executed at fixed intervals. Built on the Composable CoW framework.
- **Stop-loss orders:** Trigger a sell when price falls below a threshold. Implemented as conditional orders via Composable CoW.
- **Programmatic / conditional orders:** Smart-contract-driven orders using the Programmatic Order Framework; supports automated fee collection, DAO payrolls, and complex conditional logic.

Source: https://docs.cow.fi/category/order-types, https://cow.fi/learn/introducing-the-programmatic-order-framework-from-cow-protocol

## Governance and Token

- **Token:** COW (ERC-20 on Ethereum)
- **Total supply:** 1,000,000,000 COW
- **DAO treasury allocation:** 44.4% of total supply at TGE
- **Inflation cap:** 3% per annum maximum, minimum 365-day cadence between inflationary events
- **Governance platforms:** Snapshot (off-chain voting), on-chain execution via Safe
- **2025 budget:** CIP-79 earmarked 12.6M USDC from treasury to fund 2026 operations across four legal entities

Source: https://docs.cow.fi/governance/token, https://forum.cow.fi/t/cow-dao-treasury-2025-annual-review/3356

## Fee Model

- **Pre-2024:** Zero protocol fees; solver rewards funded entirely by COW token emissions.
- **January 2024:** Fee switch activated via DAO vote. Initial model charges 50% of surplus on out-of-market limit orders (quote improvement fee).
- **November 2025 (CIP-74):** Dynamic solver reward cap tied to protocol fees; unconditional 0.02% volume fee introduced.
- **February 2026:** Tiered fee structure; correlated asset pairs (stable-to-stable) reduced from 2 bps to 0.3 bps.
- **Partner fees:** CIP-75: CoW Protocol retains 25% of all partner integration fees as a service fee.

Source: https://www.dlnews.com/articles/defi/how-cow-swap-protocol-will-make-money-off-its-trading-volume/, https://forum.cow.fi/t/cip-74-retrospective-aligning-rewards-with-revenue/3358

## Chain Deployment (as of April 2026)

- Ethereum Mainnet (original, since 2021)
- Gnosis Chain (early deployment)
- Arbitrum
- Base
- Avalanche (June 2025)
- Polygon (July 2025)
- Lens Chain (September 2025)
- BNB Chain (October 2025)
- Linea (December 2025)
- Ink (Kraken L2, February 2026)

New chain volume: ~30% of total activity by Q4 2025. Cross-chain swaps (bridge-then-swap collapsed into a single gasless intent) launched in 2025.

Source: https://cow.fi/learn/cow-dao-2025-in-review

## Security and Audits

| Component | Auditor | Date | Critical | High | Medium | Notes |
|---|---|---|---|---|---|---|
| ComposableCoW + ExtensibleFallbackHandler | Ackee Blockchain | July 18-28, 2023 | 1 (fixed) | 0 | 1 (fixed) | 14 total findings; code quality rated professional |
| ComposableCoW + ExtensibleFallbackHandler | Gnosis (internal) | May/July 2023, August 2024 | — | — | — | Ongoing internal review |
| Safe fallback handler | G0 Group + Ackee Blockchain | 2023 | — | — | — | Dual external review |
| CoW Flash Loan Router | Ackee Blockchain | March 17-21, 2025 | 0 | 0 | 0 | 4 findings; 1 warning (missing events), 3 informational; all resolved |
| COW token, Core (GPv2), Native Token Sell Flow | Various | 2021-2022 | — | — | — | Reports in GitHub cowprotocol/contracts |

Source: https://ackee.xyz/blog/%D1%81ow-protocol-composablecow-extensiblefallbackhandler-audit-summary/, https://ackee.xyz/blog/cow-flash-loan-router-audit-summary/

## Key Differentiators

1. **MEV resistance by architecture:** Off-chain intents never enter the mempool; uniform clearing prices make reordering useless; solvers bear all execution risk.
2. **Coincidence of Wants:** Peer-to-peer order matching without any on-chain liquidity, achieving zero fees and zero slippage for matched pairs.
3. **Solver competition:** Open competition with COW token rewards produces genuine price discovery across all liquidity sources.
4. **CoW AMM:** First production MEV-capturing AMM; redirects LVR from MEV bots back to LPs.
5. **Composable / programmatic orders:** Rich conditional order types (TWAP, stop-loss, DAO payrolls) without giving up self-custody.
6. **Scale:** $87B volume in 2025, 73M+ cumulative transactions, top-3 DEX aggregator position.

## Limitations

- **Execution latency:** Batch formation every ~30 seconds creates inherent delay. Not suitable for latency-sensitive strategies.
- **Solver centralisation risk:** Bonding requirements create high barriers to entry; in practice, a small number of professional solvers (Barter at ~28% share) dominate. Permissionless solver participation is a stated future goal but not yet implemented.
- **Off-chain dependency:** The Autopilot order book and solver infrastructure are off-chain. Downtime or failure of these components blocks order creation.
- **Long-tail token limitations:** Thin order flow for rare tokens reduces the probability of CoW matches and limits solver incentives to compete aggressively.
- **Failed / expired orders:** Limit orders that do not find a matching price before expiry are simply cancelled with no on-chain cost to the user, but users must re-submit.
- **Solver MEV exposure:** While users are protected, the winning solver itself executes on-chain and may face MEV from block builders on the settlement transaction.

Source: https://metalamp.io/magazine/article/cow-dao-and-cow-protocol-how-intent-based-trading-and-mev-protection-transform-defi, https://docs.cow.fi/cow-protocol/concepts/order-types/limit-orders

## Notable Integrations (2025)

- **Aave:** Solver-powered, MEV-protected swaps integrated directly into Aave lending workflows; enables collateral swaps and debt rebalancing without leaving Aave. Source: https://www.theblock.co/post/381336/aave-cow-mev-protected-swaps-intent-based-flash-loans
- **Balancer v3:** CoW AMM deployed as a native pool type.
- **Safe Wallet:** TWAP orders via the Safe fallback handler (ComposableCoW).
- **Ink (Kraken L2):** February 2026 launch with gasless transactions and MEV protection.
- **Vitalik Buterin:** Used CoW Protocol for approximately 17,000 ETH (~$43M) in ETH sales in February 2026. Source: https://coinmarketcap.com/cmc-ai/cow-protocol/latest-updates/

## Related Patterns

- [[patterns/batch-auction-settlement]]
- [[patterns/intent-based-trading]]
- [[patterns/mev-capturing-amm]]

## Related Metrics

- [[metrics/cow-protocol-volume]]
- [[metrics/lvr-impact]]
