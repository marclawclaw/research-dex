---
tags: [pattern, accounting, rebasing, fees, ethereum]
applies-to: [curve-stableswapng]
---

# Pattern: Rebase-Safe Accounting

## Problem

AMM pools that hold rebasing tokens (tokens whose `balanceOf` values change without a transfer, such as Lido's stETH) face a fundamental accounting problem: the pool cannot distinguish between a balance change caused by a swap or liquidity event and one caused by a rebase event. If the protocol infers the admin (protocol) fee by computing the difference between the current token balance and what the pool "should" hold given all swaps and liquidity additions, a positive rebase will silently inflate what the protocol claims as its fee. A negative rebase (slashing) will silently deflate what LPs can withdraw.

Earlier Curve V1 pools had no native rebasing support. Protocols that naively admitted stETH into a pool without special handling saw rebase accrual attributed to whichever address happened to call a fee-collection function first.

## Solution: `_balance()` + explicit `admin_balances[]`

Curve StableSwapNG (source: [[projects/curve-stableswapng]]) addresses this with two complementary mechanisms:

### 1. `_balance(i)` reads live on-chain balances

On every swap, add-liquidity, or remove-liquidity call, the pool calls an internal `_balance(i)` function for each coin `i`. This function issues a live `balanceOf(address(this))` call (or the equivalent for ERC4626 tokens) rather than reading a cached reserve. The returned value is the true current balance including any rebase that has occurred since the last interaction.

This means the pool always operates on actual balances. There is no "reserve" that lags behind reality.

### 2. Admin fees tracked in an explicit `admin_balances[N_COINS]` array

When a swap is executed and a fee is earned, the admin's portion of that fee is added to `admin_balances[i]` (a storage array, not derived from balance arithmetic). The `_balances()` function (plural, returning LP-attributed balances for each coin) computes:

```
lp_balance[i] = _balance(i) - admin_balances[i]
```

This subtraction isolates the LP portion. Because `admin_balances[i]` is only ever incremented by actual fee events and is never touched by rebase mechanics, the following properties hold:

- **Positive rebase:** `_balance(i)` increases; `admin_balances[i]` does not change; therefore `lp_balance[i]` increases. LPs receive the full upside of the rebase.
- **Negative rebase (slashing):** `_balance(i)` decreases; `admin_balances[i]` does not change; therefore `lp_balance[i]` decreases. LPs bear the full downside of the slashing. Admin fees are insulated from the slash because they were explicitly credited at fee time.

### 3. `exchange_received` disabled for rebasing pools

The `exchange_received` function (used by routers for gas-efficient single-call swaps) infers the amount received by computing `_balance(i) - _stored_balance[i]`. If a rebase occurs between the pre-transfer state and the call, this delta would be misattributed as "tokens sent by the user." To prevent this exploit, the function reverts unconditionally if any pool coin has `asset_type == 2` (rebasing).

## Trade-offs and known edge cases

### Admin fee immunity creates slashing asymmetry

The `admin_balances[]` array stores nominal fee amounts that were correct at the time of accumulation. If a slashing event reduces the actual balance below the sum `lp_balance + admin_balances`, withdrawing admin fees first will consume tokens that LP holders are owed. The MixBytes audit (Oct 2023) flagged this as a known design limitation rather than a bug: the protocol mitigates it operationally by ensuring admin fees are not withdrawn during or shortly after a known slashing event.

### `stored_balances` critical bug (fixed)

A critical finding in the MixBytes audit identified that the `stored_balances` tracking variable (used for `exchange_received` optimisation) did not account for rebase accrual, meaning deposited tokens with accumulated rebase rewards could become stranded in the contract. This was fixed before mainnet deployment.

### ERC4626 variant

For ERC4626 tokens (asset type 3), `_balance(i)` calls `convertToAssets(balanceOf(address(this)))` to convert share-denominated balances to underlying asset quantities before feeding into the invariant. The admin fee is tracked in the same underlying units, preserving the same isolation property.

## Comparison to alternative approaches

| Approach | Protocol | Rebase goes to | Admin fee immune to slash? | `exchange_received` usable? |
|---|---|---|---|---|
| `_balance()` + `admin_balances[]` | Curve StableSwapNG | LPs | Yes | No (disabled for type 2) |
| `sync()` + `skim()` | Uniswap V2 | Anyone who calls `skim()` | N/A (no admin fee) | N/A |
| Transient flash accounting | Uniswap V4 | Hook-defined | Depends on hook | Yes |
| Single vault + asset managers | Balancer V2/V3 | LPs (via asset manager) | Partial | N/A |

## Sources

- [Curve StableSwapNG docs: Overview](https://docs.curve.finance/stableswap-exchange/stableswap-ng/pools/overview/) — accessed 2026-04-09
- [MixBytes: Modern DEXes — Curve StableSwapNG](https://mixbytes.io/blog/modern-dex-es-how-they-re-made-curve-stable-swap-ng) — accessed 2026-04-09
- [MixBytes StableSwapNG audit findings](https://github.com/mixbytes/audits_public/blob/master/Curve%20Finance/StableSwapNG/README.md) — accessed 2026-04-09
