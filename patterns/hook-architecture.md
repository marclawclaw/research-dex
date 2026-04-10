---
pattern: hook-architecture
aka: ["pool hooks", "lifecycle hooks", "AMM plugins"]
used_by: ["Uniswap V4", "Balancer V3"]
tags: [hooks, extensibility, custom-logic, permissions, security]
sources:
  - https://docs.uniswap.org/contracts/v4/concepts/hooks
  - https://blog.uniswap.org/uniswap-v4
  - https://hacken.io/discover/auditing-uniswap-v4-hooks/
  - https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive
  - https://www.certik.com/resources/blog/uniswap-v4-hooks-security-considerations
  - https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped
---

# Pattern: Hook Architecture

## Problem

AMMs with fixed swap curves and static fee logic cannot accommodate all market structures. Protocols needing dynamic fees, limit orders, MEV mitigation, or alternative pricing must fork the core AMM code, fragmenting liquidity. There is no standard way to extend pool behaviour without redeploying the core system.

## Solution

Allow each pool to attach one "hook" contract. The hook is invoked at defined points in the pool lifecycle (before/after swap, before/after liquidity change, etc.). The hook can observe pool state, modify token amounts via delta returns (custom accounting), and run arbitrary Solidity logic. Permissions are encoded in the hook contract's address rather than requiring a registry lookup.

## Uniswap V4 hook lifecycle

### Hook points

| Hook function | Trigger | Can modify amounts? |
|---------------|---------|-------------------|
| `beforeInitialize` | Before pool is created | No |
| `afterInitialize` | After pool is created | No |
| `beforeAddLiquidity` | Before LP adds liquidity | No |
| `afterAddLiquidity` | After LP adds liquidity | Yes (delta) |
| `beforeRemoveLiquidity` | Before LP removes liquidity | No |
| `afterRemoveLiquidity` | After LP removes liquidity | Yes (delta) |
| `beforeSwap` | Before swap executes | Yes (BeforeSwapDelta) |
| `afterSwap` | After swap executes | Yes (AfterSwapDelta) |
| `beforeDonate` | Before donation | No |
| `afterDonate` | After donation | No |

Source: [Uniswap V4 docs](https://docs.uniswap.org/contracts/v4/concepts/hooks).

### Permission encoding in address

Hook permissions are encoded as bit flags in the hook contract's deployment address. The `PoolManager` reads these bits to determine which callbacks to invoke, without any on-chain registry. This is enforced at `initialize()` time: if a hook's address lacks the required bits for a callback its code implements, the pool creation reverts.

Consequence: hooks must be deployed using address mining (e.g., with a `CREATE2` salt) to obtain an address with the correct permission bits. This is an intentional design forcing developers to commit to a hook's capabilities at deployment.

Source: [Uniswap V4 docs](https://docs.uniswap.org/contracts/v4/concepts/hooks); [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

### Custom accounting via delta returns

`beforeSwap` and `afterSwap` hooks can return delta values to modify the amounts involved in a swap:

- `BeforeSwapDelta`: Adjusts the amount the pool processes before the swap curve is applied.
- `AfterSwapDelta`: Adjusts amounts after the swap curve computation.

This enables:
- Surgically redirecting part of swap proceeds to an external contract (e.g., MEV fee capture).
- Implementing an entirely custom pricing curve while bypassing the default concentrated liquidity math.
- Dynamic fee adjustments (changing the effective fee charged per swap).

## Ecosystem adoption (2025)

| Metric | Value | Source |
|--------|-------|--------|
| Hooks initialised on mainnet | 5,000+ | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) |
| Pools using Atrium alumni hooks | 313,533+ | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) |
| Volume through alumni hook pools | $697M+ | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) |
| Hook incubator cohorts 4-7 graduates | 260 | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) |

### Notable hook categories (Atrium Hook Incubator 2025)

| Category | Share of projects | Example hooks |
|----------|------------------|---------------|
| LP optimisation and dynamic fees | 57% | AdaptiveSwap, Prisma |
| Incentives and yield | 37% | YOLO Protocol |
| Cross-chain and routing | 30% | RampHook, Liquidity Oracle |
| MEV and execution | 29% | Detoxer, FrontrunThis |

Source: [Atrium Academy Hook Incubator 2025 Wrapped](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped).

## Security risks

The hook architecture substantially expands the attack surface relative to V3's fixed-curve pools. The Uniswap Foundation does not audit or certify hook implementations.

### 1. Authorization bypass

Hooks that do not verify `msg.sender == address(poolManager)` can be called by any external address. In the Cork Protocol, a missing access control check allowed attackers to call hook functions directly, bypassing the PoolManager entirely.

Mitigation: Enforce `onlyPoolManager` modifier on all callback functions. Use `beforeInitialize()` to restrict the hook to a single designated pool.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

### 2. Delta handling errors

Incorrect `BeforeSwapDelta` parameter ordering or sign handling leaves unsettled deltas, causing the `unlock()` call to revert or silently miscount balances.

Example: A hook that adds +5 tokens in `beforeSwap()` but deducts only -3 in `afterSwap()` leaves a +2 unresolved delta, reverting the entire transaction.

Mitigation: Verify deltas sum to zero across all hook callbacks for every token. Use integration tests with exact balance assertions.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

### 3. Async hook fund theft

Async hooks that take full custody of swapped assets by reversing `amountToSwap` in their delta calculations can redirect tokens to any address. A malicious hook is indistinguishable from a legitimate one at the interface level.

Mitigation: Users must independently audit hook contracts before interacting with hook-attached pools. There is no protocol-level protection.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

### 4. Upgradeable hook centralization

UUPS-upgradeable hooks can change fee logic, pause withdrawals, or redirect liquidity post-deployment. An owner-controlled hook with upgrade rights can impose 100% swap fees after users have deposited liquidity.

Mitigation: Prefer immutable hook logic. If upgradeable, use timelocks and multi-sig governance.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

### 5. Denial of service via gas exhaustion

Hooks with unbounded loops (e.g., iterating an uncapped `authorizedUsers` array in `beforeSwap`) can exceed block gas limits, permanently blocking all swaps in the pool.

Mitigation: Cap array sizes; replace arrays with mappings; profile gas usage on mainnet fork tests.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

### 6. MEV and front-running via hook logic

Price-dependent hook logic (dynamic fees based on oracle data) is observable in the mempool. MEV bots front-run swaps to capture favourable fee states or trigger fee adjustments before the victim transaction lands.

Mitigation: Use private mempools (e.g., MEV Blocker, Flashbots Protect) for sensitive operations; avoid hook logic that depends on easily predictable market states.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/); [academy.exmon.pro](https://academy.exmon.pro/future-of-liquidity-uniswap-v4-hooks-vs-jit-mev-attacks).

### 7. Configuration mismatch (permission bit errors)

If a hook contract is deployed without the address bit corresponding to a callback it implements, the `PoolManager` will never call that callback. This is a silent failure: the hook is live but incomplete.

Mitigation: Use `CREATE2` address mining to derive an address with the correct permission bits. Test against the `PoolManager`'s bit-check logic on a fork before mainnet deployment.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

### 8. Pool exclusivity issues

Hooks that do not validate which pool they are attached to can be reused across multiple pools. If the hook maintains internal state (e.g., accumulated fees, user allowlists), a second pool attaching the same hook can read or corrupt that state, leading to cross-pool manipulation.

Mitigation: Use `beforeInitialize()` to record and enforce a single authorised `PoolId`. Reject initialisation attempts from any other pool.

Source: [Hacken](https://hacken.io/discover/auditing-uniswap-v4-hooks/).

## Comparison with Balancer V3 hooks

Both Uniswap V4 and [[projects/balancer-v3]] implement hook architectures, but with different designs:

| Dimension | Uniswap V4 | Balancer V3 |
|-----------|------------|-------------|
| Hook registration | Encoded in hook address bits | Registered via Vault |
| Permissioning | Address mining at deploy time | Registered permissions |
| Lifecycle points | 10 (before/after × 5 events) | Similar lifecycle; pool-type-specific |
| Custom accounting | `BeforeSwapDelta` / `AfterSwapDelta` | `onComputeDynamicSwapFee`, pool math hooks |
| Audit requirement | Not enforced by protocol | Not enforced by protocol |

## Related notes

- [[patterns/flash-accounting]] (hooks interact with delta system)
- [[patterns/singleton-pool-manager]] (hooks registered per pool in PoolManager)
- [[projects/uniswap-v4]] (full protocol note with hook ecosystem metrics)
- [[projects/balancer-v3]] (alternative hook implementation)
