# Pattern: Adaptive Fees (Dynamic Fee Adjustment)

**Also known as:** Volatility-responsive fees, dynamic fee tiers
**Origin in this research:** [[projects/orca-whirlpools]] (Orca Whirlpools, June 2025)
**Source:** [dev.orca.so — Whirlpool Fees](https://dev.orca.so/Architecture%20Overview/Whirlpool%20Fees/)

---

## Problem

Fixed fee tiers create a mismatch between LP compensation and market conditions:

- During **high volatility**, LPs face elevated impermanent loss but collect fees at the same static rate as calm periods. This under-compensates them for the increased risk.
- During **low volatility**, a static fee may be higher than necessary, reducing trade competitiveness versus rivals with lower fees.
- **Competing protocols** (e.g., Uniswap v4 hooks, Meteora DLMM) have introduced dynamic fee mechanisms, creating pressure for CLMMs to respond.

---

## Solution

Adaptive fee pools add a **variable fee component** layered on top of a static base fee. The variable component tracks price volatility in real time using a volatility accumulator and scales the fee non-linearly with observed price movement.

---

## How It Works (Orca Whirlpools Implementation)

### Total Fee Formula

```
total_fee = base_fee (f_b) + variable_fee (f_v)
f_v = A × (v_a × s)²
```

Where:
- `f_b` is the static base fee (equivalent to the standard `fee_rate`)
- `A` is the `adaptiveFeeControlFactor` (controls aggressiveness of fee scaling)
- `v_a` is the **volatility accumulator** (dimensionless measure of recent tick-group crossings)
- `s` is the tick group size (equals tick spacing for standard pools; 128 for Splash Pools)
- The squared relationship produces **non-linear** (exponential-like) fee increases during high volatility periods
- **Hard cap:** Maximum total fee is 10%, regardless of volatility

### Volatility Accumulator (`v_a`)

The accumulator measures market turbulence by counting tick-group crossings within transactions:

1. **Tick groups** partition the price range into segments of size `s`. Each crossing of a tick group boundary increments the accumulator.
2. After each swap, the accumulator decays based on elapsed time using three parameters:
   - `filterPeriod`: minimum time between volatility reference updates (defines the high-frequency trading window)
   - `decayPeriod`: time threshold after which volatility fully resets if no major swaps occur
   - `reductionFactor`: rate at which volatility decays (represented as a fraction of 10,000)
3. References are **only updated** on "major swaps" exceeding `majorSwapThresholdTicks` to prevent manipulation through small repeated transactions.
4. If `MAX_REFERENCE_AGE` (1 hour) elapses without activity, fees reset to the base rate regardless of prior volatility.

### The Skip Optimisation

Adaptive fee computation is bypassed in three cases to save compute units:
1. Zero-liquidity areas (no active positions in that range)
2. When `adaptiveFeeControlFactor` equals zero (effectively disabling the variable component)
3. When price movement exceeds the core range around the reference tick

---

## New Account Types

Adaptive fee pools require two additional on-chain accounts that do not exist in standard Whirlpools:

| Account | Scope | Contents |
|---------|-------|----------|
| **Adaptive Fee Tier Account** | Shared across multiple pools | `filterPeriod`, `decayPeriod`, `reductionFactor`, `adaptiveFeeControlFactor`, `majorSwapThresholdTicks` |
| **Pool-Specific Oracle Account** | Per pool | Volatility references, timestamps, current accumulator values |

---

## Launch Context

Orca introduced adaptive fee pools in **June 2025** as an upgrade to the existing Whirlpools program. The mechanism targets pools with volatile trading pairs where static fees create LP under-compensation during price discovery events.

---

## Tradeoffs and Limitations

| Tradeoff | Detail |
|----------|--------|
| Complexity | Adds two new account types and on-chain math per swap |
| Front-running risk (reduced) | `majorSwapThresholdTicks` guard prevents fee manipulation via micro-transactions |
| Predictability | Traders cannot know exact fee before swap if price movement is volatile; quotes become probabilistic |
| Reset lag | 1-hour `MAX_REFERENCE_AGE` means fees can remain elevated for up to an hour after volatility subsides |

---

## Comparable Implementations

- **Uniswap v4 hooks:** Fee hooks allow custom dynamic fee logic but require external hook contracts; more flexible but more complex to deploy.
- **Meteora DLMM:** Bin-based model with variable fee tiers per bin; different mechanism but similar goal.
- **Ambient/CrocSwap:** Pool-wide dynamic fees based on maker/taker model.

Orca's approach is notable for being **built into the core Whirlpools program** rather than requiring external hook contracts, preserving composability with the existing CPI interface.

---

## Related Notes

- [[projects/orca-whirlpools]]
- [[patterns/tick-array-architecture]]
