# Raydium CLMM: Technical and Fee Metrics

**Project:** [[raydium]]
**Component:** Concentrated Liquidity Market Maker (CLMM / v6 pools)
**Last updated:** April 2026

---

## Fee Tiers and Tick Spacing

| Tick Spacing | Fee Rate | Best For | Notes |
|-------------|----------|----------|-------|
| 1 | 0.01% (1 bp) | Stable / pegged pairs (e.g. USDC/USDT) | Finest granularity; highest capital efficiency for stable assets |
| 10 | 0.05% (5 bp) | Correlated assets | |
| 60 | 0.25% (25 bp) | Standard pairs | Most common general-purpose tier |
| 120 | 1.00% (100 bp) | Volatile / exotic pairs | Coarse granularity; less frequent tick crossings |

**Note:** In August 2024 Raydium expanded CLMM to eight fee tiers by adding 0.02%, 0.03%, 0.04%, and 2%. Source: https://x.com/RaydiumProtocol/status/1819053525264973854

Source: https://deepwiki.com/raydium-io/raydium-sdk-V2/4.1.1-clmm-pool-creation-and-management ; https://docs.raydium.io/raydium/for-liquidity-providers/pool-types/clmm-concentrated

**Tick spacing and capital efficiency:** Smaller tick spacing allows LPs to place liquidity in tighter price bands, amplifying fee yield per unit of capital when the price stays in-range. The trade-off is greater out-of-range risk and more frequent tick-crossing overhead during swaps.

---

## Fee Distribution (CLMM and CPMM)

| Recipient | Share |
|-----------|-------|
| Liquidity providers | 84% |
| RAY token buyback | 12% |
| Protocol treasury | 4% |

Source: https://docs.raydium.io/raydium/for-liquidity-providers/pool-types/clmm-concentrated

---

## Position Structure

- **Position representation:** NFT (non-fungible token). Each CLMM position mints a unique NFT that controls both the liquidity and uncollected fees/rewards for that range.
- **Loss risk:** If the position NFT is lost, sold, or burned, the liquidity it represents is permanently inaccessible. No recovery mechanism.
- **Multiple positions:** Wallets may hold an unlimited number of positions; overlapping price ranges are permitted.
- **Fee collection:** Fees accumulate within the position but are not auto-compounded. They transfer to the wallet only when the LP removes liquidity or executes a zero-liquidity decrease (`decreaseLiquidity` with `liquidity = 0`).

Source: https://docs.raydium.io/raydium/for-liquidity-providers/pool-types/clmm-concentrated

---

## Price Representation (On-Chain)

| Field | Format | Notes |
|-------|--------|-------|
| `sqrt_price_x64` | u128, Q64.64 fixed-point | `sqrt(price) × 2^64` |
| Minimum price (tick −443,636) | `MIN_SQRT_PRICE_X64` | |
| Maximum price (tick +443,636) | `MAX_SQRT_PRICE_X64` | |
| Fee growth accumulators | u128, Q64.64 fixed-point | Per-unit-of-liquidity fee since inception |

Rationale: Solana programs cannot use floating-point arithmetic in on-chain code (no FPU in BPF VM). All price and fee calculations use Q64.64 fixed-point integers to preserve precision while remaining deterministic.

Source: https://github.com/raydium-io/raydium-clmm/blob/master/programs/amm/src/states/pool.rs

---

## Tick Array Structure

- Each `TickArray` account holds 60 tick structs.
- Each tick stores: `liquidity_net` (liquidity change at that tick), `fee_growth_outside_0/1` (fee accumulators for the range), and reward tracking fields.
- The pool's `tick_array_bitmap` (inline in `PoolState`, 16 × u64 = 1,024 bits) tracks which tick arrays are initialised across the standard price range.
- For price ranges beyond the standard bitmap capacity, a `TickArrayBitmapExtension` PDA account is used.

**PDA seeds:**
- `TickArrayBitmapExtension`: `["pool_tick_array_bitmap_extension", pool_state_pubkey]`
- Pool vault: `["pool_vault", pool_state_pubkey, token_mint_pubkey]`

Source: https://github.com/raydium-io/raydium-clmm/blob/master/programs/amm/src/libraries/tick_array_bit_map.rs ; https://deepwiki.com/raydium-io/raydium-sdk-V2/4.1.1-clmm-pool-creation-and-management

---

## Token-2022 Support

CLMM pools support Token-2022 mint extensions, including transfer fees. When creating a pool with Token-2022 mints, the SDK automatically includes extension accounts in the transaction to handle transfer fee deduction correctly.

Source: https://deepwiki.com/raydium-io/raydium-sdk-V2/4.1.1-clmm-pool-creation-and-management

---

## Reward Tokens

- Each CLMM pool can have up to 3 concurrent reward tokens (`REWARD_NUM = 3`).
- Rewards are emitted pro-rata to in-range liquidity; out-of-range positions earn no rewards.
- Reward info is stored inline in `PoolState` as `[RewardInfo; 3]`.

Source: https://github.com/raydium-io/raydium-clmm/blob/master/programs/amm/src/states/pool.rs

---

## Known Bug: Tick Manipulation (January 2024)

| Field | Detail |
|-------|--------|
| Discovered | 10 January 2024 |
| Discoverer | Whitehat @riproprip (ImmuneFi) |
| Bounty | $505,000 in RAY tokens |
| Affected instruction | `increase_liquidity` in `increase_liquidity.rs` |
| Root cause | `remaining_accounts[0]` (the `TickArrayBitmapExtension` account) was not validated against the pool's canonical PDA; attacker could substitute a fake account |
| Impact | Arbitrary tick state flipping; accumulation of unearned liquidity; extraction of disproportionate token amounts via swaps |
| Fix | Added key-equality validation: `remaining_accounts[0]` must match `TickArrayBitmapExtension` PDA for the pool |

Source: https://immunefi.com/blog/bug-fix-reviews/raydium-tick-manipulation-bugfix-review/

---

## CLMM Volume Share (Q3 2025)

- CPMM contributed $5B in volume (4.8% of total $51.9B Raydium Q3 2025 volume); CLMM and CPMM together anchored stable swap revenue of $10.5M.
- AMM v4 (OpenBook-integrated) remained the dominant volume driver due to memecoin activity ($27.5B meme volume = 53% of total).

Source: https://altsignals.io/post/raydium-q3-2025-defi-growth-analysis
