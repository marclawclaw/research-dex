---
pattern: flash-accounting
aka: ["transient accounting", "delta accounting"]
used_by: ["Uniswap V4", "Balancer V3"]
eip: EIP-1153
tags: [accounting, gas-optimisation, transient-storage, eip-1153]
sources:
  - https://docs.uniswap.org/contracts/v4/concepts/flash-accounting
  - https://hacken.io/discover/uniswap-v4-transient-storage-security/
  - https://www.cyfrin.io/blog/uniswap-v4-swap-deep-dive-into-execution-and-accounting
  - https://app.uniswap.org/whitepaper-v4.pdf
---

# Pattern: Flash Accounting (Transient Accounting)

## Problem

In Uniswap V3 and earlier AMMs, each pool is a separate contract and each swap step requires an immediate token transfer between pools. A multi-hop swap through N pools requires N+1 token transfers. This is gas-inefficient and prevents atomic multi-pool compositions without expensive intermediate custody.

Additionally, the per-pool ERC-20 `transferFrom` calls dominate gas costs in complex operations.

## Solution

Flash accounting defers all token transfers until the end of a logical "unlock" session. The protocol tracks net balance changes (credits and debits) per token per caller in transient storage (EIP-1153). Only the final net amount requires an actual on-chain token transfer.

The pattern requires:

1. **Transient storage** to track delta values cheaply within a transaction.
2. **An unlock/callback gate** to enforce atomic settlement.
3. **A zero-delta invariant** enforced at session close.

## How EIP-1153 transient storage works

EIP-1153 introduces two opcodes:

| Opcode | Gas cost | Behaviour |
|--------|----------|-----------|
| `TSTORE(key, value)` | 100 gas | Writes to transient storage; cleared at transaction end |
| `TLOAD(key)` | 100 gas | Reads from transient storage |

Contrast with persistent storage:

| Operation | Gas cost |
|-----------|----------|
| `SSTORE` (cold write) | 20,000 gas |
| `SLOAD` (cold read) | 2,100 gas |
| Reset with refund | ~5,000 gas (net ~14,200) |

For delta tracking, the full round-trip (write delta, read delta, reset) costs approximately 300 gas with transient storage versus ~24,400 gas with persistent storage: a 98% reduction.

Source: [Hacken](https://hacken.io/discover/uniswap-v4-transient-storage-security/), 2024.

## Uniswap V4 implementation

### Key data structures (transient)

| Contract | Purpose |
|----------|---------|
| `CurrencyDelta.sol` | Stores net balance change per (currency, address) pair |
| `NonzeroDeltaCount.sol` | Counter of outstanding non-zero deltas across all currencies |
| `Lock.sol` | Boolean flag tracking whether the PoolManager is currently unlocked |

### The unlock/callback pattern

```
caller.unlock()
  → PoolManager.unlock()
    → caller.unlockCallback()
      [caller performs swaps, liquidity ops, donations]
      [each op: updates CurrencyDelta via TSTORE]
      [caller settles: sync() then settle(), or take()]
    ← callback returns
  ← PoolManager verifies NonzeroDeltaCount == 0
  → revert if any delta unsettled
← returns to caller
```

### Delta sign convention

- **Positive delta:** PoolManager owes tokens to the caller (caller should call `take()`).
- **Negative delta:** Caller owes tokens to PoolManager (caller should call `sync()` then `settle()`).

### Settlement functions

| Function | Purpose |
|----------|---------|
| `sync(currency)` | Snapshots current PoolManager token balance before a transfer; must precede `settle()` |
| `settle()` | Marks that caller has paid; resolves a negative delta |
| `take(currency, recipient, amount)` | Withdraws tokens owed to caller; resolves a positive delta |

Critical ordering: `sync()` must be called before `settle()`. Without it, `CurrencyReserves.getSyncedCurrency()` returns `address(0)` and the settlement reverts.

### Multi-hop gas comparison

| Scenario | V3 token transfers | V4 token transfers |
|----------|-------------------|-------------------|
| Single-hop swap | 2 | 2 |
| 3-hop swap (ETH → USDC → DAI → WBTC) | 4 | 2 |
| N-hop swap | N+1 | 2 (always) |

The USDC and DAI intermediate legs are resolved as internal delta credits; only the initial ETH debit and final WBTC credit require ERC-20 calls.

Source: [Uniswap V4 docs](https://docs.uniswap.org/contracts/v4/concepts/flash-accounting).

## Security considerations

### Transient storage scope

Transient storage clears at transaction end, not at each internal call. In batched transactions processed by relayers or smart-contract wallets, stale delta values can persist across logically separate operations if not explicitly cleared. The Uniswap V4 codebase mitigates this with explicit resets even where automatic clearing would suffice.

### Unsettled delta enforcement

The `NonzeroDeltaCount` invariant ensures that if a caller forgets to settle any delta, the entire transaction reverts. This provides strong composability guarantees: no partial settlement is possible.

### Hook-induced delta errors

Hook contracts that return incorrect `BeforeSwapDelta` or `AfterSwapDelta` values can create unsettled balances. Common errors: wrong sign convention, mismatched `specifiedAmount` vs. `unspecifiedAmount` fields, or off-by-one in fee calculations. See [[patterns/hook-architecture]] for hook delta risks.

## Adopters

| Protocol | Implementation | Notes |
|----------|----------------|-------|
| Uniswap V4 | Core flash accounting | First production use of EIP-1153 at scale |
| Balancer V3 | Transient accounting in Vault | Enables secure re-entrancy; similar unlock/callback pattern |

See [[projects/uniswap-v4]] and [[projects/balancer-v3]] for protocol-specific detail.

## Contrast with alternative patterns

| Pattern | Protocol example | Token custody model |
|---------|-----------------|-------------------|
| Flash accounting | Uniswap V4, Balancer V3 | Centralised; deltas resolved at session end |
| `sync()`/`skim()` | Uniswap V2 | Per-pool cached reserves; surplus claimable |
| Internal balance accounting | Balancer V2 | Single Vault; virtual shares; no ERC-20 per hop |
| Solana account model | Raydium, Orca | Per-pool token accounts; no global delta tracking |

See [[metrics/vault-accounting-patterns]] for a cross-protocol comparison.
