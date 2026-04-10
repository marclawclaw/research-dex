# Pattern: Batch Auction Settlement

**Also known as:** Fair Combinatorial Batch Auction (FCBA), batch auction DEX, uniform clearing price settlement
**Primary example:** [[projects/cow-protocol]]
**Status:** Production (CoW Protocol since 2021; FCBA upgrade in 2025)

## Summary

Batch auction settlement collects multiple user trade intents over a fixed time window, then runs a competitive auction among solvers to find the optimal joint settlement for all orders simultaneously. This architecture structurally prevents the majority of MEV attacks because transaction ordering within a batch becomes irrelevant.

## Core Mechanism

### 1. Intent Collection (Off-Chain Order Book)

Users sign trade intents off-chain. Each intent specifies:
- Tokens to sell and buy
- Minimum acceptable execution price (limit price)
- Expiry deadline

Signed intents are submitted to an off-chain order book (the Autopilot in CoW Protocol's case). Because intents are signed messages rather than on-chain transactions, they never appear in the public mempool and cannot be front-run.

### 2. Batch Formation

Every ~30 seconds, the Autopilot closes the collection window and bundles all pending valid intents into a batch. This batch is the unit of auction: solvers must consider all intents simultaneously when proposing a settlement.

### 3. Solver Competition

All whitelisted solvers receive the batch simultaneously. Each solver independently computes the best possible joint settlement across all orders, with the objective of maximising total user surplus (the aggregate positive difference between execution prices and signed minimum prices).

Solvers may propose:
- **Non-batched (individual) bids:** A solution for a single order.
- **Batched bids:** A joint solution covering multiple orders simultaneously, enabling peer-to-peer Coincidence of Wants (CoW) matches and complex multi-hop routes.

In the Fair Combinatorial Batch Auction (FCBA) model used by CoW Protocol, the protocol evaluates all combinations of submitted bids and selects the winning combination that maximises total surplus, subject to an individual fairness constraint: no order in a batched bid may receive worse execution than it would receive in a solo auction.

### 4. Uniform Directed Clearing Prices (UDCP)

All orders trading the same token pair in the same direction within a batch settle at the same price. This is a key MEV-resistance property: since every same-direction order in the batch gets identical pricing regardless of its position within the batch, there is no advantage to reordering transactions. Sandwich attacks, which rely on the ability to manipulate price before and after a target transaction, are structurally impossible.

### 5. On-Chain Settlement

The winning solver submits a single on-chain transaction that settles all orders in the batch simultaneously. The settlement contract:
- Verifies each user's cryptographic signature
- Confirms execution satisfies signed terms (price, quantity, deadline)
- Executes token transfers via vault relayer

**Source:** https://cow.fi/learn/understanding-batch-auctions, https://docs.cow.fi/cow-protocol/reference/core/auctions/competition-rules, https://docs.cow.fi/cow-protocol/concepts/introduction/fair-combinatorial-auction

## Why Batch Auctions Resist MEV

### Sandwich Attacks

A sandwich attack requires: (1) a transaction before the target that moves the price, and (2) a transaction after that profits from the shift. Under UDCP, every order in the same batch gets the same price; there is no "before" and "after" within a batch to exploit.

### Front-Running

Intents are off-chain signed messages. They are not broadcast to the public mempool. No bot can observe a pending intent and insert a transaction ahead of it in the mempool.

### Back-Running / Arbitrage Capture

In traditional block-sequential DEX models, the first transaction after a large trade can arbitrage the resulting price deviation. In a batch auction, solvers are incentivised to capture this arbitrage value and return it to users as surplus rather than leaking it externally.

## Coincidence of Wants (CoW)

When opposing intents exist in the same batch (e.g., Alice wants to sell ETH for USDC and Bob wants to sell USDC for ETH), a solver can match them peer-to-peer within the batch:
- No on-chain AMM liquidity is accessed
- No swap fees are paid to any pool
- Gas costs are minimal (only direct token transfers)
- MEV exposure is zero (no on-chain price movement)

The "CoW" in CoW Protocol's name refers to this Coincidence of Wants matching. It is the highest-priority settlement strategy because it produces the best possible execution quality.

**Source:** https://docs.cow.fi/cow-protocol/concepts/how-it-works/coincidence-of-wants

## Trade-offs and Limitations

| Property | Batch Auction | Traditional On-Chain DEX |
|---|---|---|
| MEV resistance | Strong (structural) | Weak (tool-dependent) |
| Execution latency | ~30 seconds per batch | Immediate (next block) |
| Price discovery | Cross-venue (solver-routed) | Single venue or aggregated |
| Coincidence matching | Yes (peer-to-peer batching) | No |
| Centralisation risk | Solver whitelist required | Fully permissionless |
| Off-chain dependency | High (order book, Autopilot) | None |
| Gas efficiency | High (amortised across batch) | Per-trade gas |

**Latency:** The ~30-second batch window creates inherent execution delay. This is acceptable for most retail and institutional trades but is incompatible with latency-sensitive strategies.

**Solver entry barrier:** Whitelisting and bonding requirements prevent permissionless solver participation. In practice, a small number of professional firms (e.g., Barter at ~28% of CoW Protocol batches on Ethereum as of September 2025) dominate.

**Source:** https://metalamp.io/magazine/article/cow-dao-and-cow-protocol-how-intent-based-trading-and-mev-protection-transform-defi

## Timing Parameters (CoW Protocol, as of 2025)

Solvers must submit and execute settlements within network-specific block windows after a batch is closed:

| Network | Deadline |
|---|---|
| Ethereum Mainnet | 3 blocks |
| Gnosis Chain | 5 blocks |
| Base, Avalanche, Polygon, Linea, Plasma | 20 blocks |
| Arbitrum, BNB | 40 blocks |

Exceeding the deadline triggers automatic solver denylisting pending manual review.

**Source:** https://docs.cow.fi/cow-protocol/reference/core/auctions/competition-rules

## Scoring and Slashing

**Scoring:** Solutions are scored by surplus generated (per CIP-38 and CIP-65 formulas). The solver with the highest score wins.

**Slashing conditions include:**
- EBBO violations: executing at worse prices than baseline on-chain protocols (Uniswap, Balancer, Sushiswap)
- Score inflation via fake tokens or wash trading
- Malicious use of internal buffers
- Systematic pennying/overbidding patterns
- Omitting eligible pre/post hooks

**Source:** https://docs.cow.fi/cow-protocol/reference/core/auctions/competition-rules

## Extensions to the Pattern

**Fair Combinatorial Batch Auction (FCBA):** Supports multi-asset atomic settlements and preserves individual order fairness guarantees even when orders are grouped into batched bids.

**AMM rebalancing via batch auction:** CoW AMM applies the same batch auction mechanism to pool rebalancing; see [[patterns/mev-capturing-amm]].

## Related Patterns

- [[patterns/intent-based-trading]] — the complementary order model
- [[patterns/mev-capturing-amm]] — CoW AMM applies batch auctions to LP positions

## Projects Using This Pattern

- [[projects/cow-protocol]] (CoW Protocol, production since 2021; FCBA since 2025)
