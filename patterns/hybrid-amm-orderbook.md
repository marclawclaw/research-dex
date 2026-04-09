# Pattern: Hybrid AMM / Order Book Integration

**Applies to:** Raydium AMM v4 (Solana + OpenBook)
**Key example:** [[raydium]]
**Related patterns:** [[solana-account-model-pools]]

---

## Overview

A hybrid AMM/order-book system bridges two historically separate liquidity models:

- **AMM (Automated Market Maker):** Passive, always-on liquidity governed by a mathematical invariant (e.g. `x * y = k`). No counterparty needed; price is a function of the ratio of reserves.
- **Central Limit Order Book (CLOB):** Active, discrete bids and asks placed by market makers and traders. Price discovery is granular; spreads reflect actual market maker behaviour.

Raydium's AMM v4 combines both: the constant-product pool acts as the primary liquidity source, while the AMM simultaneously places limit orders on OpenBook (formerly Serum) derived from its own reserves. This means Raydium's LP capital participates in two markets at once.

---

## Why Combine Them?

Pure AMMs have two structural inefficiencies:

1. **Capital inefficiency:** Liquidity is spread across the full price range (`0` to `∞`), but nearly all trading occurs near the current price. The majority of LP capital never earns fees.
2. **Price isolation:** Pure AMM swaps affect only the pool's internal price, with no direct connection to the broader market order flow. This means the AMM can lag behind centralised exchange prices, creating arbitrage opportunities that drain LP value.

A pure order book requires active market makers who must quote continuously, post margin, and manage inventory. This creates high operational complexity and means bootstrapping liquidity is harder.

The hybrid approach aims to use passive AMM capital to provide order-book liquidity automatically, improving capital efficiency while keeping the constant-product invariant as a backstop.

---

## Raydium's Implementation

### Core Accounts

| Account | Purpose |
|---------|---------|
| `AmmInfo` | Central pool state; references the OpenBook market, stores `order_num` (max orders), `depth` (price level range) |
| `OpenOrders` | Raydium's active orders on the OpenBook DEX; tracks unsettled balances from fills |
| `TargetOrders` | Raydium-specific account defining the target bid/ask grid; tracks PnL via `calc_pnl_x` and `calc_pnl_y` |

Source: https://deepwiki.com/raydium-io/raydium-amm/3.3-orderbook-integration

### State Machine

The AMM maintains two nested state machines:

**`AmmStatus` (high-level operational mode):**
- `Uninitialized`, `Initialized`, `Disabled`, `WithdrawOnly`, `LiquidityOnly`, `OrderBookOnly`, `SwapOnly`, `WaitingTrade`
- Transitions are time-dependent (`pool_open_time`, `orderbook_to_init_time`) and admin-controlled.

**`AmmState` (order management cycle):**
```
IdleState → PlanOrdersState → PlaceOrdersState → PurgeOrderState → IdleState
                                           ↑
                                  CancelAllOrdersState (reset path)
```

The `MonitorStep` instruction drives the cycle, called periodically by crankers (permissionless off-chain bots). Each cycle:
1. Loads the current OpenBook market state.
2. Reads active orders from the `OpenOrders` account.
3. Cancels stale orders.
4. Plans new orders based on current pool reserves.
5. Places new limit orders at Fibonacci-derived price levels.
6. Settles any filled orders (moving tokens from the DEX market vaults back to AMM vaults).

### Fibonacci Liquidity Placement

The AMM distributes its limit orders at Fibonacci-derived price deviations from the current market price: 23.6%, 38.2%, and 61.8%. This provides:
- Tight spread for small trades near the current price.
- Progressively deeper liquidity for larger price movements.
- A shape that approximates the liquidity distribution a rational market maker would place, rather than uniform distribution.

Source: https://extremelysunnyyk.medium.com/solana-amm-under-the-hood-raydium-insights-for-solana-builders-218ac339fde1

### PDA Signatures for DEX Interaction

All order placement on OpenBook requires PDA signatures. The AMM generates a PDA using the seed `"AUTHORITY_AMM"` plus a nonce value stored in `AmmInfo`. This ensures only the AMM program can manage its own order book positions.

### PnL Extraction via `withdrawPNL`

Because the AMM holds liquidity in two places simultaneously (pool vaults and open orders on the DEX), there is a continuous arbitrage between the AMM's constant-product price and the executed orderbook price. The `withdrawPNL` function collects this accumulated profit without disrupting the core liquidity.

The `TargetOrders` account stores baseline values `calc_pnl_x` and `calc_pnl_y`; the function `calc_take_pnl` compares current balances against these baselines to determine extractable profit.

**Security note:** The December 2022 exploit targeted this mechanism. The attacker used `SetParams` with `AmmParams::SyncNeedTake` to inflate the `need_take_pc` and `need_take_coin` fields (fee parameters used by `withdrawPNL`) without actual trading. This caused `withdrawPNL` to drain pool reserves as if they were protocol fees. After the exploit, the `SyncNeedTake` and related admin parameters were removed entirely. See [[raydium]] for full post-mortem.

---

## Interaction Flows

### Swap
1. Load OpenBook state.
2. Calculate total liquidity including funds in open orders (settled + unsettled).
3. Execute swap against the constant-product formula.
4. Potentially rebalance orderbook positions in the same transaction.

### Deposit
1. Calculate total position including funds in open orders.
2. Update PnL baseline values in `TargetOrders`.
3. Mint LP tokens proportional to share of total pool.
4. Adjust orderbook positions if necessary.

### Withdrawal
1. Cancel all open orders on OpenBook.
2. Settle DEX funds back to AMM vaults.
3. Calculate total position (now entirely in vaults, no orders outstanding).
4. Burn LP tokens; transfer proportional token amounts to LP.
5. Update PnL baselines.

The withdrawal flow is more expensive than a pure AMM because it requires cancelling orders and settling DEX funds before the pool balance is known. This creates higher transaction costs for withdrawals versus deposits.

---

## Trade-offs

| Dimension | Hybrid (Raydium v4) | Pure AMM (Uniswap v2 style) |
|-----------|---------------------|------------------------------|
| Capital efficiency | Higher: idle capital earns on CLOB | Lower: uniform distribution |
| Price discovery | Participates in CLOB order flow | Isolated; depends on arbitrageurs |
| Withdrawal complexity | High: must cancel+settle orders | Low: burn LP tokens directly |
| External dependency | OpenBook market must exist and have volume | None |
| Admin surface | Larger: AMM + DEX interaction parameters | Smaller |
| Cranker requirement | Yes: `MonitorStep` requires periodic off-chain calls | No |

**Critical dependency:** Standard AMM v4 pools require an OpenBook market to exist for the trading pair. If no market exists, or if OpenBook ceases to function, the integration breaks. This is why Raydium introduced CPMM (which has no OpenBook dependency) as an alternative for new token launches.

---

## Current Status (2025)

With the rise of CPMM and CLMM pools, the OpenBook-integrated AMM v4 is increasingly treated as legacy infrastructure. Most new pools use CPMM or CLMM, which do not require an OpenBook market. AMM v4 pools retain the largest share of TVL (83% of Raydium TVL in Q3 2025) because older, established trading pairs have deep liquidity there. Source: https://altsignals.io/post/raydium-q3-2025-defi-growth-analysis

The hybrid model has not been widely replicated. Orca (the second-largest Solana DEX) uses pure CLMM pools without order-book integration, choosing to focus on concentrated liquidity mechanics instead.

---

## Sources

- https://deepwiki.com/raydium-io/raydium-amm/3.3-orderbook-integration
- https://deepwiki.com/raydium-io/raydium-amm/1-overview
- https://github.com/raydium-io/raydium-amm
- https://extremelysunnyyk.medium.com/solana-amm-under-the-hood-raydium-insights-for-solana-builders-218ac339fde1
- https://raydium.medium.com/detailed-post-mortem-and-next-steps-d6d6dd461c3e
- https://altsignals.io/post/raydium-q3-2025-defi-growth-analysis
