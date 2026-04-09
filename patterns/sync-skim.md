---
tags: [pattern, vault-accounting, uniswap-v2, surplus-recovery, rebasing-tokens]
origin: Uniswap V2
---

# Pattern: sync() / skim()

## Problem

AMM pair contracts cache token reserves in storage (e.g., `reserve0`, `reserve1`) for gas efficiency and to provide a manipulation-resistant snapshot for the invariant check. However, the actual ERC-20 balances of the pair contract can drift from these cached reserves through several channels:

1. **Direct token transfer:** A user sends tokens directly to the pair address (e.g., mistakenly, or deliberately to seed a flash swap).
2. **Positive rebase:** A rebasing token (e.g., AMPL, stETH before wrapping) increases the balance of all holders, including the pair contract, without triggering any transfer event or pair-side code.
3. **Interest accrual:** Interest-bearing tokens (e.g., Aave's aTokens) increase the balance of the pair silently.
4. **Fee-on-transfer or reflection:** Transfers leave dust or extra amounts inside the pair.
5. **`uint112` overflow:** If actual token balance reaches or exceeds `2^112`, the pair cannot store it in `reserve0`/`reserve1` and all operations revert.

Without handling mechanisms, these discrepancies either:
- Trap funds permanently in the contract (surplus above reserves), or
- Cause all swaps to revert (deficit below reserves from negative rebase).

## The sync() function

```solidity
function sync() external lock {
    _update(
        IERC20(token0).balanceOf(address(this)),
        IERC20(token1).balanceOf(address(this)),
        reserve0,
        reserve1
    );
}
```

**What it does:** Reads the actual ERC-20 balances and writes them into `reserve0` and `reserve1`. This makes the reserves match reality, regardless of how the discrepancy arose.

**When to use:**
- After a negative rebase has reduced the actual balance below the cached reserve. Without `sync()`, the next swap would revert because the invariant check uses the (inflated) cached reserve.
- After any event that reduces the pair's actual token balance below what the contract believes it holds.
- When deliberate liquidity is sent directly to the pair before a `mint()` call (the normal liquidity provision flow already calls `_update()` via `mint()`).

**Effect on LPs:** If called after a negative rebase, LPs absorb the loss: the lower reserve means lower redemption value for LP tokens. This is the expected outcome: the pair cannot fabricate missing tokens.

**Effect on invariant:** `sync()` resets `k` to match the new (lower) reality. Subsequent swaps use the new lower `k`.

## The skim() function

```solidity
function skim(address to) external lock {
    address _token0 = token0;
    address _token1 = token1;
    _safeTransfer(_token0, to, IERC20(_token0).balanceOf(address(this)) - reserve0);
    _safeTransfer(_token1, to, IERC20(_token1).balanceOf(address(this)) - reserve1);
}
```

**What it does:** Computes `balance − reserve` for each token and transfers the surplus to `to`. The reserves remain unchanged; only the excess is extracted.

**When to use:**
- After a positive rebase has pushed the actual balance above the cached reserve. The surplus is orphaned (it is above the reserve and thus not part of the invariant). Anyone can extract it.
- Before the balance exceeds `uint112` (2^112 - 1), which would cause the `_update()` call inside `swap()` to revert due to overflow. `skim()` removes the excess and allows the pair to remain operational.

**Effect on LPs:** LPs do not benefit from positive rebases. The rebase surplus accumulates above the reserve line and is publicly extractable by any caller. This is a known limitation of the V2 design.

**Who can call it:** Anyone. The function is permissionless. In practice, MEV bots and arbitrageurs monitor pairs for skimmable surpluses. Callers should use an intermediary contract rather than an EOA to avoid being front-run.

## The skim / sync decision tree

```
Is actual balance < cached reserve?
  YES → call sync()  (absorbs deficit; LPs bear the loss)
  NO  →
    Is actual balance > cached reserve?
      YES → call skim(to)  (extracts surplus to caller; LPs get nothing)
      NO  → balances are in sync; no action needed
```

## Contrast with sync() alternative

Rather than skim-and-sync, a caller could:
- Call `sync()` on a pair with a positive balance surplus. This would write the higher balance into the reserve, effectively gifting the surplus to all LPs proportionally.
- This is the correct action if the sender deliberately sent tokens to add to pool liquidity without minting LP shares (unusual but valid).

## Implications for rebasing tokens

| Scenario | Recommended action | Effect |
|----------|-------------------|--------|
| Positive rebase (balance increases) | Call `skim()` or `sync()` | `skim()`: caller gets surplus. `sync()`: surplus added to LP pool, grows `k`. |
| Negative rebase (balance decreases) | Call `sync()` | Reserves written down; subsequent swaps succeed; LPs absorb loss. |
| Interest-bearing token accrual | Call `skim()` or `sync()` | Same as positive rebase. |
| Direct token donation | Call `sync()` before next swap | Surplus credited to all LPs. |
| Balance approaching `uint112` max | Call `skim()` | Removes overflow risk; pair stays operational. |

## Vulnerability: publicly skimmable surplus

Because `skim()` is permissionless, positive rebases and direct transfers create **a publicly extractable surplus**. The economic value of the rebase is not retained by LPs: it is raced for by MEV bots. This is a fundamental limitation of the V2 design for rebasing assets.

Later protocols address this differently:
- [[patterns/constant-product-amm]]: Curve StableSwapNG stores admin fees separately and reads live balances via `_balance()`, preventing fee corruption and giving LPs the full rebase benefit.
- Balancer V3 uses transient accounting and asset managers to handle yield-bearing tokens while keeping the vault the single source of truth.

## Portability

The sync/skim pattern is highly portable to any new chain:
- It requires only the ability to read actual token balances and compare them to a cached value.
- No external dependencies; the pair is self-healing.
- Any new DEX on a new chain must decide: cached reserves (V2 pattern) or live-balance reads (Curve StableSwapNG pattern) or centralised vault (Balancer pattern).

## References

- Uniswap V2 whitepaper (section 2.2): https://app.uniswap.org/whitepaper.pdf
- V2 pair contract reference: https://docs.uniswap.org/contracts/v2/reference/smart-contracts/pair
- Rebasing token support: https://support.uniswap.org/hc/en-us/articles/34258911235469-Rebasing-reflection-and-debasing-tokens
- Uniswap-skim scanner (nicholashc): https://github.com/nicholashc/uniswap-skim
- V2 pair source: https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol

## Related notes

- [[projects/uniswap-v2]]
- [[patterns/constant-product-amm]]
