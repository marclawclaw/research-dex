# Pattern: Intent-Based Trading

**Also known as:** Order intents, signed intents, declarative trading, meta-DEX
**Primary example:** [[projects/cow-protocol]]
**Status:** Production (CoW Protocol since 2021; broadly adopted across DeFi from ~2023)

## Summary

Intent-based trading separates what a user wants to achieve (the intent) from how that outcome is produced (the execution). Users sign an off-chain message expressing a desired trade outcome; third-party solvers or searchers then compete to find the optimal execution path and settle on-chain. This separation provides MEV protection by keeping orders out of the public mempool and enables cross-venue optimisation by giving executors freedom to route across all available liquidity.

## Core Concept

### Traditional DEX Model

In a traditional DEX swap, the user:
1. Constructs a transaction specifying exact routing (e.g., swap X USDC for ETH via Uniswap v3 pool Y)
2. Signs and broadcasts the transaction to the public mempool
3. Waits for a miner/validator to include it in a block

This exposes the user to:
- **Front-running:** Bots observe the pending transaction and insert a buy ahead of it
- **Sandwich attacks:** Bots place a buy before and a sell after the user's trade
- **Suboptimal routing:** Users cannot efficiently access all liquidity sources in a single transaction

### Intent-Based Model

The user instead:
1. Signs an off-chain message expressing a desired outcome: "I want to trade 5,000 USDC for WETH; I require at least 1.85 WETH; this intent expires in 1 hour."
2. Submits the signed intent to an off-chain order book
3. Waits for a solver to settle the trade on-chain

The signed message never enters the public mempool. The solver handles all on-chain execution risk.

**Source:** https://cow.fi/learn/understanding-crypto-intents-the-future-of-your-de-fi-trades, https://cow.fi/learn/how-cow-protocol-actually-works

## What an Intent Contains

In CoW Protocol's implementation, a signed intent specifies:
- **Sell token:** The token to give
- **Buy token:** The token to receive
- **Sell amount / buy amount:** The maximum to sell and minimum to receive (or vice versa for buy-side orders)
- **Limit price:** The worst acceptable execution price
- **Expiry:** Deadline after which the intent is void
- **Partial fill flag:** Whether the order may be partially executed

The intent is a typed structured data message (EIP-712) signed with the user's Ethereum key. No gas is spent until execution.

**Source:** https://docs.cow.fi/cow-protocol

## Role of Solvers

Solvers are the execution layer. They receive batches of intents and compete to find optimal settlements. In CoW Protocol:

- Solvers may match intents peer-to-peer (Coincidence of Wants), use private market-maker liquidity, or route through on-chain DEXes
- The solver bearing the greatest execution responsibility also bears MEV risk on the settlement transaction
- Users are guaranteed their signed minimum price regardless of what the solver does on-chain; the settlement contract enforces this

**Source:** https://docs.cow.fi/cow-protocol/concepts/introduction/solvers

## MEV Protection Properties

Intent-based trading provides MEV protection through several mechanisms:

1. **No mempool exposure:** Signed intents are transmitted off-chain, invisible to monitoring bots. Front-running requires observing a pending transaction; intents provide nothing to observe.

2. **Delegated execution:** The solver executes on-chain, not the user. If the solver's settlement transaction is sandwiched, the solver absorbs the cost. The user is guaranteed their signed limit price.

3. **Uniform clearing prices (in batch implementations):** When multiple intents are settled in the same batch, all same-direction pairs receive identical prices, eliminating the price differential that sandwich attacks exploit.

**Source:** https://docs.cow.fi/cow-protocol/concepts/benefits/mev-protection

## Order Types Enabled by Intent Architecture

Because intents are off-chain signed messages rather than on-chain calls, the system can support sophisticated order types without requiring complex on-chain logic at order submission:

- **Limit orders:** Execute only if market price meets the specified condition
- **TWAP (Time-Weighted Average Price):** Split a large order over time intervals, implemented as a sequence of conditional intents via the Composable CoW framework
- **Stop-loss:** Trigger a sell if price drops below a threshold
- **Programmatic / conditional orders:** Smart-contract-driven intents for automated DAO treasury operations, payroll, and complex conditional logic

These are implemented via CoW Protocol's Programmatic Order Framework and ComposableCoW conditional order system (audited by Ackee Blockchain, July 2023).

**Source:** https://cow.fi/learn/navigating-the-evolution-of-cow-swap-from-market-orders-to-programmatic-orders, https://docs.cow.fi/cow-protocol/concepts/order-types/programmatic-orders

## Gasless Trading

Because the user signs a message rather than submitting an on-chain transaction, they pay no gas at intent submission time. Gas is paid by the solver on settlement. In practice, gas costs are deducted from the user's surplus (the positive difference between execution price and signed minimum) or charged as a small fee.

This enables genuinely gasless trading for the user, reducing friction and eliminating the need to hold the network's native gas token.

## Trade-offs

| Property | Intent-Based | Direct On-Chain |
|---|---|---|
| MEV resistance | Strong | Weak |
| Execution latency | Batch-window dependent (~30s in CoW Protocol) | Next block |
| User gas cost | Zero at submission | Paid per trade |
| Routing quality | Solver-optimised, cross-venue | User-specified |
| Execution guarantee | Limit-price guarantee; no fill guarantee | Fill guarantee (if price met) |
| Counterparty risk | Solver must honour limit price (enforced on-chain) | None (direct execution) |
| Order book transparency | Off-chain (not mempool-visible) | On-chain (transparent) |
| Complexity | Higher (requires solver infrastructure) | Lower |

**Execution risk:** An intent may expire unfilled if no solver finds a profitable settlement path (e.g., for illiquid or exotic token pairs during low-activity periods). Users receive no fill in this case, rather than a bad fill.

**Off-chain infrastructure dependency:** If the off-chain order book or Autopilot service is unavailable, users cannot submit or have their intents processed.

**Source:** https://metalamp.io/magazine/article/cow-dao-and-cow-protocol-how-intent-based-trading-and-mev-protection-transform-defi, https://docs.cow.fi/cow-protocol/concepts/order-types/limit-orders

## Cross-Chain Intents (2025)

CoW Protocol's 2025 expansion included cross-chain intents: a user signs a single intent expressing a desired outcome across two chains (e.g., sell USDC on Ethereum, receive ETH on Arbitrum). The solver handles all bridging and execution. This collapses the traditional bridge-then-swap flow into a single gasless user action.

**Source:** https://cow.fi/learn/cow-dao-2025-in-review

## Ecosystem Adoption

Intent-based trading emerged broadly across Ethereum DeFi from approximately 2023:
- CoW Protocol: the original production implementation (2021); $87B volume in 2025
- UniswapX: launched July 2023 and gained 6.7% market share within months [BlockCrunch, November 2023](https://blockcrunch.substack.com/p/cow-1inch-uniswap-intents-and-the)
- 1inch Fusion: intent model on top of 1inch routing
- Various solver-based cross-chain protocols

The pattern is now considered a standard approach to MEV-resistant DEX design.

**Source:** [BlockCrunch (2023-11-28)](https://blockcrunch.substack.com/p/cow-1inch-uniswap-intents-and-the)

## Related Patterns

- [[patterns/batch-auction-settlement]] — the settlement mechanism used by CoW Protocol
- [[patterns/mev-capturing-amm]] — extends intent principles to liquidity provision

## Projects Using This Pattern

- [[projects/cow-protocol]] (CoW Protocol; most established, $87B 2025 volume)
