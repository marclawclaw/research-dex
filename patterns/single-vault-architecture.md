# Pattern: Single-Vault Architecture

**First introduced:** Balancer V2 (2021)
**Refined in:** [[balancer-v3]] (2024)
**Related:** [[yield-bearing-token-support]]

---

## Problem

In pool-per-contract AMM designs (e.g. Uniswap V2, V3), each pool holds its own token reserves. This creates several inefficiencies and risks:

1. **Multi-hop gas cost:** A route through three pools requires six token transfers (two per pool), each costing ~20,000+ gas for ERC-20 `transfer` calls.
2. **Accounting duplication:** Each pool must implement its own token balance tracking, decimal scaling, fee accounting, and reentrancy protection. Code is duplicated across pools, multiplying audit surface.
3. **No internal balance netting:** Tokens must physically move between pools even when a protocol wants to batch multiple operations internally.

---

## Solution

All tokens across all pools are held in a single Vault contract. Individual pool contracts are pure mathematics: they receive scaled token amounts, compute invariants and swap outputs, and return results. They do not custody tokens.

The Vault handles:
- Token custody (all ERC-20 `balanceOf` queries and transfers go to the Vault)
- Decimal scaling (uniform 18-decimal precision normalisation before pool math)
- Fee collection and distribution
- BPT (Balancer Pool Token) minting and burning
- Internal balance netting across multi-pool routes

---

## Balancer V2 implementation

In V2, the Vault held all tokens. Pools were registered with the Vault and only computed math. Internal balances could be maintained for users, allowing off-chain balance tracking without actual token transfers. Multi-pool batch swaps netted intermediate amounts, reducing transfers.

However, V2 still required persistent storage reads/writes for every intermediate step of a batch operation, and composable pools (which held BPT tokens internally) introduced recursive accounting complexity that ultimately led to the November 2025 exploit.

Source: https://medium.com/balancer-protocol/balancer-v2-a-one-stop-shop-6af1678003f7

---

## Balancer V3 refinements

V3 makes three significant improvements to the single-vault architecture:

### 1. Transient accounting (EIP-1153 "Till" pattern)

V3 moves intermediate accounting from persistent storage to transient storage (EIP-1153 TLOAD/TSTORE opcodes). When the Vault is "unlocked" by a Router callback:

- Each operation calls `_takeDebt()` or `_supplyCredit()` to record a token delta in transient storage.
- A counter tracks the number of tokens with non-zero outstanding deltas.
- At the end of the callback, all deltas must net to zero (checked via `_nonZeroDeltaCount == 0`).
- Actual ERC-20 transfers occur only for the net amounts via `settle()` and `sendTo()`.

Result: a three-pool multi-hop swap requires only two ERC-20 transfers (input token in, output token out), regardless of intermediate pool count.

### 2. Vault-level decimal and rate scaling

In V2, each pool was responsible for scaling its tokens to 18-decimal precision. In V3, the Vault applies this uniformly before calling any pool's math. This eliminates per-pool scaling bugs and ensures that yield-bearing tokens are correctly converted to their underlying value before the pool invariant is computed.

### 3. ERC20MultiToken: atomic BPT and balance updates

The V3 Vault uses an `ERC20MultiToken` pattern that atomically updates both pool token balances and BPT supply in a single operation. In V2, a window existed between the BPT mint and the balance update, enabling read-only reentrancy attacks. V3 closes this window by design.

Source: https://medium.com/balancer-protocol/balancer-v3-the-future-of-amm-innovation-f8f856040122; https://mixbytes.io/blog/modern-dex-es-how-they-re-made-balancer-v3

---

## Comparison with Uniswap V4 (PoolManager)

Uniswap V4 independently arrived at a structurally similar design: a singleton `PoolManager` holds all tokens and uses flash accounting via transient storage (EIP-1153) to net deltas across multi-pool operations. The key differences:

| Dimension | Balancer V3 Vault | Uniswap V4 PoolManager |
|-----------|------------------|----------------------|
| Pool math location | External pool contracts (registered with Vault) | Internal to PoolManager (by pool ID) |
| Hooks | Standalone hook contracts (one per pool; 10 lifecycle points) | Hook contracts (one per pool; address-encoded flags) |
| Token types explicitly supported | ERC-20, ERC-4626 (yield-bearing) | ERC-20, native ETH |
| Yield-bearing token native support | Yes (Rate Providers + liquidity buffers) | No (handled by hooks if at all) |
| Pool creator fee | Yes (ERC-5313 ownership) | No (protocol-level fee only) |
| Multi-token pools | Up to 8 tokens | Two tokens per pool (multi-hop via routing) |

---

## Risks of single-vault architecture

1. **Systemic single point of failure:** A critical Vault vulnerability affects all pools simultaneously. The November 2025 Balancer V2 exploit drained ~$128M from all V2 Composable Stable Pools because they shared one Vault contract.
2. **Vault upgrade difficulty:** Upgrading the Vault requires migrating all pool liquidity, an extremely disruptive operation. V3 mitigated this by auditing more thoroughly and building in more guardrails pre-launch, but the upgrade path remains complex.
3. **Hook trust surface:** Hooks registered against pools can call back into the Vault via the transient mechanism. Malicious hooks can modify swap output amounts (via return deltas) to extract value from LPs, and the Vault does not audit hook logic.

Sources: https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025; https://docs.balancer.fi/concepts/core-concepts/hooks.html

---

## Portability assessment for a new chain

The single-vault architecture is highly portable. The core pattern requires:

1. A Vault contract with token custody, internal balance tracking, and a callback-based unlock mechanism.
2. Per-pool math contracts that receive scaled amounts and return results.
3. An atomic settlement mechanism (transient storage is ideal; a reentrancy-locked mutex with in-memory delta tracking is a fallback for chains without EIP-1153).

The main portability challenge is audit depth: because all pools share one Vault, a Vault bug is a protocol-wide bug. The Vault must be exceptionally well audited before any pools are deployed.

---

## See also

- [[balancer-v3]]: main project note
- [[yield-bearing-token-support]]: how the vault handles ERC-4626 tokens natively
