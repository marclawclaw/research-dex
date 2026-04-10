---
metric: vault-accounting-patterns
updated: 2026-04-09
tags: [accounting, vault, comparison, patterns, rebasing]
sources:
  - https://docs.uniswap.org/contracts/v4/concepts/flash-accounting
  - https://docs.uniswap.org/contracts/v4/overview
  - https://app.uniswap.org/whitepaper-v4.pdf
  - https://mixbytes.io/blog/modern-dex-es-how-they-re-made-balancer-v3
  - https://docs-v2.balancer.fi/reference/contracts/flash-loans.html
---

# Metric: Vault Accounting Patterns (Cross-Protocol)

Comparison of how each researched DEX handles token custody, balance reconciliation, surplus/deficit, and rebasing tokens.

## Summary table

| Protocol | Custody model | Balance reconciliation | Surplus recovery | Rebasing support | Flash/transient accounting |
|----------|--------------|----------------------|-----------------|-----------------|--------------------------|
| Uniswap V2 | Per-pool contract | Cached reserves; `sync()` updates to live balance | `skim()` sends surplus to caller | Partial (positive rebase creates skimmable surplus; negative requires `sync()`) | No |
| Uniswap V3 | Per-pool contract | `slot0` price + `positions`; no reserve cache | None (`collect()` for fees only) | Limited (fee-on-transfer partially supported) | No |
| Uniswap V4 | Singleton PoolManager | Flash accounting: transient delta tracking; `sync()` + `settle()` at session end | Not applicable (no cached reserve separate from live balance) | Hook-extensible; not native | Yes (EIP-1153) |
| Balancer V2 | Single external Vault | Internal balance accounting; virtual shares | Asset managers can reclaim yield | Limited | No (internal accounting, not transient; supports flash loans via Vault, but flash loans are a different mechanism from EIP-1153 transient accounting) |
| Balancer V3 | Single external Vault | Transient accounting (EIP-1153); before/after hooks | Native yield-bearing token integration | Yes (ERC4626 native) | Yes (EIP-1153) |
| Curve StableSwapNG | Per-pool contract | `_balance()` reads live balance; admin fees in separate array | Not applicable (no skim; admin fees isolated) | Yes (rebasing, rate-oraclised, ERC4626) | No |
| Raydium | Per-pool token accounts (Solana) | Solana account data; no cached reserve | Not applicable | Not documented | No |
| Orca Whirlpools | Per-pool token accounts (Solana) | Solana account data; tick-based CLMM | Not applicable | Not documented | No |

## Uniswap V4 vault accounting detail

Uniswap V4 represents the most novel accounting model in this research cohort:

- **No per-pool token custody.** All tokens reside in the singleton `PoolManager`.
- **No cached reserve separate from live balance.** The flash accounting delta system tracks net obligations in transient storage; the PoolManager's token balance is always the ground truth.
- **No `sync()`/`skim()` equivalents** in the sense of Uniswap V2. The `sync()` function in V4 serves a different purpose: it snapshots the PoolManager's current token balance before a `settle()` call to detect how many tokens were transferred in.
- **Custom accounting via hooks.** Hook contracts can intercept and modify delta values, enabling alternative fee mechanisms or surplus redistribution without modifying core contracts. This is V4's extension point for handling non-standard tokens.

### V4 `sync()` vs V2 `sync()`

| Function | Protocol | Purpose |
|----------|---------|---------|
| `sync(currency)` | Uniswap V4 | Snapshots PoolManager token balance before `settle()`; detects inbound transfer amount |
| `sync()` | Uniswap V2 | Updates cached reserves to match live ERC-20 balances; handles negative rebase |

These functions share a name but serve different roles. V2's `sync()` reconciles a stale reserve cache. V4's `sync()` is part of the settlement protocol.

## Key design trade-offs

| Design choice | Advantage | Disadvantage |
|---------------|-----------|-------------|
| Singleton custody (V4) | Cross-pool delta netting; no per-pool transfer overhead | Single contract failure affects all pools |
| Per-pool custody (V2/V3) | Pool isolation; failure does not propagate | Inefficient for multi-hop; each pool holds its own tokens |
| Single external vault (Balancer) | Same cross-pool netting benefit; pool math contracts are isolated from custody | External vault is a high-value target |
| Transient accounting (V4, Balancer V3) | 98% gas reduction for state tracking; arbitrary complexity for fixed cost | Requires EIP-1153 support (post-Cancun Ethereum only) |
| Per-pool Solana accounts (Raydium, Orca) | Solana's account model enforces ownership at runtime | No shared accounting; cross-pool routing requires multiple CPI calls |

## Rebasing token handling comparison

| Protocol | Approach | Risk |
|----------|---------|------|
| Uniswap V2 | `sync()` / `skim()`; positive rebase creates skimmable surplus | Sandwich around rebase event to extract surplus |
| Uniswap V3 | Not natively supported; fee-on-transfer partially ok | Rebase mid-position corrupts LP share math |
| Uniswap V4 | Hook-extensible; no native handling in core | Hook must explicitly handle rebase; bugs are pool-specific |
| Curve StableSwapNG | First-class support via `_balance()` and rate oracles | Admin fee isolation required; `exchange_received` disabled for rebasing pools |
| Balancer V3 | ERC4626 native yield-bearing token support | Rate provider manipulation risk |

## Related notes

- [[patterns/flash-accounting]]
- [[patterns/singleton-pool-manager]]
- [[projects/uniswap-v4]]
- [[projects/uniswap-v2]]
- [[projects/balancer-v3]]
- [[projects/curve-stableswapng]]
