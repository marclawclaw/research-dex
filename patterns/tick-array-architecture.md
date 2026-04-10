# Pattern: Tick Array Architecture (Solana CLMM)

**Origin in this research:** [[projects/orca-whirlpools]]
**Primary sources:**
- [dev.orca.so — Understanding Tick Arrays](https://dev.orca.so/Architecture%20Overview/Understanding%20Tick%20Arrays/)
- [GitHub — Tick Arrays doc](https://github.com/orca-so/whirlpools/blob/main/docs/whirlpool/docs/02-Architecture%20Overview/03-Understanding%20Tick%20Arrays.md)
- [Orca Medium — Dynamic Tick Arrays](https://orca-so.medium.com/create-pools-for-less-with-orcas-dynamic-tick-arrays-13e8c5dbcc8c)
- [Orca Medium — SparseSwap](https://orca-so.medium.com/introducing-sparseswap-a-new-era-in-trading-on-orca-c0eefa067980)

---

## Problem: Storing Tick Data on Solana

On Ethereum, a Uniswap v3 pool stores all tick data in a single contract's storage mapping. This is possible because Ethereum's account model allows a contract to have arbitrarily large internal storage that grows lazily.

On Solana, **every on-chain account has a fixed size that must be allocated and rent-paid at creation**. There is no native concept of a dynamically growing storage mapping within a single account. This means a CLMM on Solana cannot simply replicate the Uniswap v3 storage layout.

---

## Solution: Tick Arrays as Separate PDA Accounts

Orca Whirlpools stores tick data in dedicated on-chain accounts called **tick arrays**. Each tick array is a PDA (Program Derived Address) account storing 88 tick slots.

### Account Structure

| Property | Value |
|----------|-------|
| Ticks per array | 88 |
| Account size | 10 KB |
| Total price range per array | `88 × tick_spacing` |
| Account type | PDA |
| Closeable | No (once initialised, permanent) |
| Requires reinit | Never |

### PDA Derivation

Each tick array PDA is derived from two seeds:
1. The Whirlpool pool's public key
2. The **start tick index** of the tick array (the lowest tick index the array covers)

The start tick index is computed as the nearest multiple of `(tick_spacing × 88)` that does not exceed the target tick index. This ensures each array covers a non-overlapping contiguous range.

### Coverage

Because a pool's price range spans from the minimum to maximum tick index (set by the Whirlpool program), a fully initialised pool would require many tick array accounts. In practice, only the arrays relevant to the current price and near-by ranges need to be initialised.

---

## Swap Mechanics

### Basic Swap

When a user executes a swap, they must specify the tick arrays the swap will traverse. The program accepts up to 3 tick arrays and automatically orders them. The first array typically contains the pool's current tick index, and subsequent arrays extend in the swap direction.

### SparseSwap (August 2024)

Before SparseSwap, swaps would fail if they encountered an uninitialized tick array. This was a significant UX problem: any gap in initialised tick arrays could halt a trade even when liquidity existed beyond the gap.

SparseSwap resolves this by treating **uninitialized tick arrays as initialised but with zero liquidity**. The swap traverses the gap without error, and without affecting fee calculations or liquidity accounting.

Additional SparseSwap improvements:
- The swap instruction (`swap_v2`, `two_hop_swap_v2`) now accepts **up to 6 tick arrays** total: 3 standard plus 3 supplemental via `remaining_accounts`.
- The program **automatically reorders** received tick arrays based on the current pool price, removing the requirement that `tick_array_0` always contain the current tick.
- SDK version: 0.13.4 (backward compatible with 0.13.0+).

---

## Dynamic Tick Arrays (July 2025)

### Problem with Original Fixed-Size Arrays

In the original design, tick array accounts were allocated at full 10 KB even if only one or two ticks within that range were actually used. The initialising party paid the full rent-exempt deposit (~0.07 SOL per array, ~$10 USD at the time). For a new pool with 5 price ranges, this cost $20-30, which was 10x higher than competing CLMMs.

The cost disproportionately burdened early liquidity providers opening positions in previously unused price ranges.

### Dynamic Array Mechanism

The July 2025 upgrade implements a **grow-as-you-go** model:

1. **Minimal initialisation:** A tick array account starts at the smallest possible size, covering only the currently needed ticks.
2. **Dynamic resizing:** The account expands automatically as new positions open in previously unused tick slots within that array's range.
3. **Refundable deposits:** When an LP pays to expand an existing tick array (by opening a position in a new tick slot), that expansion deposit is **fully refundable** when the position is closed.

### Cost Comparison

| Operation | Before (Jul 2025) | After (Jul 2025) |
|-----------|-----------|---------|
| Single tick array init | ~0.07 SOL (~$10 USD) | ~0.002 SOL (~$0.30 USD) |
| New pool (5 arrays) | $20-30 USD | <$5 USD |
| Savings vs. other CLMMs | | >10x |

Source: [Orca Medium — Dynamic Tick Arrays](https://orca-so.medium.com/create-pools-for-less-with-orcas-dynamic-tick-arrays-13e8c5dbcc8c)

---

## Implications for Pool Owners and LPs

### Preemptive Initialisation

Pool owners may choose to preemptively initialise tick array ranges on behalf of users to prevent unexpected costs when a user opens a position in an uninitialised range. After Dynamic Tick Arrays, this is far cheaper and the responsibility shifted to the protocol.

### Position NFTs

Each liquidity position in a Whirlpool is represented as an NFT. In November 2024, Orca integrated Token Extensions to make position NFT rent **fully refundable** upon position closure, further reducing LP cost basis.

---

## Why This Matters for SVM-Native DEX Design

The tick array pattern is a direct consequence of Solana's account model constraints. Key lessons:

1. **Data partitioning:** On Solana, large datasets must be split across multiple accounts; CLMM tick data is a natural fit for this pattern.
2. **PDA predictability:** PDA derivation from pool key + start index makes tick arrays discoverable without an index.
3. **Cost transparency:** Separate accounts make rent costs explicit and attributable to specific price ranges.
4. **SparseSwap pattern:** Treating absent accounts as zero-value (rather than errors) is a Solana-native resilience pattern applicable beyond CLMMs.
5. **Grow-as-you-go:** Dynamic resizing accounts are a general Solana pattern for reducing upfront costs while maintaining correctness.

---

## Related Notes

- [[projects/orca-whirlpools]]
- [[patterns/adaptive-fees]]
