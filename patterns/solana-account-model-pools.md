# Pattern: Solana Account Model for Pool Token Custody

**Applies to:** All Raydium pool types (AMM v4, CLMM, CPMM); also used by Orca, Meteora, and other Solana DEXes.
**Key example:** [[raydium]]
**Related patterns:** [[hybrid-amm-orderbook]]

---

## Overview

On Ethereum, a DEX pool contract owns its own token balances directly via the ERC-20 `balanceOf` mapping. On Solana, every piece of state — including token balances — lives in a separate account. This creates a distinct multi-account architecture for liquidity pools that has significant implications for security, composability, and gas (compute unit) efficiency.

---

## Solana Account Primitives

**Account:** The atomic unit of Solana state. Every account has:
- A `pubkey` address.
- A `lamport` balance (rent).
- A `data` field (arbitrary bytes, up to ~10 MB).
- An `owner` field: the program that can write to `data`.
- A `rent_exempt_reserve`.

**Program-Derived Address (PDA):** A public key derived deterministically from a seed array and a program ID. PDAs have no corresponding private key; only the owning program can sign for them. This makes PDAs the canonical way to give a program authority over accounts without exposing a private key.

**SPL Token Account:** A standardised account layout (owned by the SPL Token program) that holds a balance of a specific mint for a specific authority. Authority can be a wallet or a PDA.

---

## Pool Account Cluster (AMM v4 / CPMM)

A Raydium constant-product pool requires the following accounts to exist before trading can begin:

| Account | Type | Purpose |
|---------|------|---------|
| `AmmInfo` (pool state) | Program data account | Stores `k`, reserves, fee parameters, OpenBook market reference, PnL tracking |
| `token_vault_0` | SPL Token Account | Holds custody of token A (e.g. SOL). Authority: pool PDA |
| `token_vault_1` | SPL Token Account | Holds custody of token B (e.g. USDC). Authority: pool PDA |
| `lp_mint` | SPL Mint Account | Mint for LP tokens; mint authority: pool PDA |
| `amm_authority` (PDA) | PDA | Signs all vault transfers; derived from seed `"AUTHORITY_AMM"` + nonce stored in `AmmInfo` |
| `open_orders` | OpenBook account | Holds the AMM's active orderbook positions and unsettled funds |
| `target_orders` | Raydium data account | Tracks target bid/ask orders and PnL baseline values (`calc_pnl_x`, `calc_pnl_y`) |

**Key principle:** The pool program never holds tokens in its own account. Tokens sit in SPL Token Accounts whose `authority` is the pool PDA. The program can move tokens only by invoking the SPL Token program with a PDA signature derived via `invoke_signed`.

---

## Pool Account Cluster (CLMM)

The CLMM adds tick-based price accounting. The account set is larger:

| Account | Type | Purpose |
|---------|------|---------|
| `PoolState` | Zero-copy data account | Central state: current price (`sqrt_price_x64`), active liquidity, tick, fee growth globals, swap tracking |
| `token_vault_0` | SPL Token Account | Custody of token A; authority: pool PDA |
| `token_vault_1` | SPL Token Account | Custody of token B; authority: pool PDA |
| `observation_key` | Oracle account | Circular buffer of price observations for TWAP |
| `tick_array_bitmap` (inline in PoolState) | 16× u64 | Bitmap of which tick arrays are initialised (covers standard range) |
| `TickArrayBitmapExtension` (PDA) | Extension account | Extended bitmap for price ranges beyond default tick array capacity |
| `TickArray` accounts | Data accounts | Arrays of 60 tick structs; each tick stores `liquidity_net`, `fee_growth_outside`, reward tracking |
| `PositionState` (NFT-linked PDA) | Data account | Per-position state: tick range, liquidity amount, fee growth checkpoints |

**PDA seeds used in CLMM:**
- Pool: `["pool", token_mint_0, token_mint_1, tick_spacing]`
- Vault: `["pool_vault", pool_state, token_mint]`
- Bitmap extension: `["pool_tick_array_bitmap_extension", pool_state]`

Source: https://github.com/raydium-io/raydium-clmm/blob/master/programs/amm/src/states/pool.rs

---

## `PoolState` On-Chain Layout (CLMM)

The `PoolState` struct is declared `#[repr(C, packed)]` and `#[account(zero_copy(unsafe))]`, allowing the Anchor framework to deserialise it as a direct memory mapping with no heap allocation:

```rust
pub struct PoolState {
    pub bump: [u8; 1],
    pub amm_config: Pubkey,        // fee config reference
    pub owner: Pubkey,
    pub token_mint_0: Pubkey,
    pub token_mint_1: Pubkey,
    pub token_vault_0: Pubkey,
    pub token_vault_1: Pubkey,
    pub observation_key: Pubkey,
    pub mint_decimals_0: u8,
    pub mint_decimals_1: u8,
    pub tick_spacing: u16,
    pub liquidity: u128,           // total active liquidity at current tick
    pub sqrt_price_x64: u128,      // Q64.64 fixed-point square root of price
    pub tick_current: i32,
    pub fee_growth_global_0_x64: u128,  // global fee accumulator, token 0
    pub fee_growth_global_1_x64: u128,  // global fee accumulator, token 1
    pub protocol_fees_token_0: u64,
    pub protocol_fees_token_1: u64,
    pub swap_in_amount_token_0: u128,   // lifetime swap tracking
    pub swap_out_amount_token_1: u128,
    pub swap_in_amount_token_1: u128,
    pub swap_out_amount_token_0: u128,
    pub status: u8,                // bitfield: positions enabled, swaps enabled, etc.
    pub reward_infos: [RewardInfo; 3],
    pub tick_array_bitmap: [u64; 16],
    pub total_fees_token_0: u64,
    pub total_fees_claimed_token_0: u64,
    pub total_fees_token_1: u64,
    pub total_fees_claimed_token_1: u64,
    pub fund_fees_token_0: u64,
    pub fund_fees_token_1: u64,
    pub open_time: u64,
    pub recent_epoch: u64,
}
```

**Price representation:** `sqrt_price_x64` stores `sqrt(price) × 2^64` as a `u128`. This avoids floating-point arithmetic entirely. Price bounds span tick −443,636 to +443,636.

**Fee growth globals:** `fee_growth_global_0_x64` accumulates total fees per unit of liquidity since pool inception, in Q64.64 format. Per-position fee entitlement is calculated as `(fee_growth_global - fee_growth_outside_checkpoints_at_range_boundaries) × position_liquidity`.

---

## How Token Custody Works: Swap Flow

For a swap in a constant-product pool:

1. User submits transaction calling `swap_base_in` (or `swap_base_out`).
2. Program reads `AmmInfo` to get vault addresses and fee parameters.
3. Applies `x' × y' >= x × y × (1 + fee)` to compute output amount.
4. Invokes SPL Token's `transfer` via `invoke_signed`, using the `amm_authority` PDA to sign.
5. Tokens move: user's token account → `token_vault_1`, `token_vault_0` → user's token account.
6. Updates reserve amounts in `AmmInfo`.

The program never takes custody of tokens in its own data; the vault SPL accounts are the escrow.

---

## Security Properties

**No private-key custody:** Vault authority is a PDA; no individual can sign for it. The only attack surface is a program bug or an admin key compromise (as occurred in the 2022 Raydium exploit, where admin authority over fee parameters was held by a single keypair rather than a PDA).

**Account validation:** Every instruction must pass the correct vault, mint, and authority accounts, and the program verifies ownership and PDA derivation. The January 2024 tick manipulation bug (see [[raydium]]) arose precisely because one account — the `TickArrayBitmapExtension` — was not validated against its expected PDA derivation, allowing an attacker to pass a substitute.

**Rent exemption:** Every data account must maintain a minimum lamport balance to be rent-exempt; otherwise the runtime may deallocate it. Pool accounts are initialised with sufficient lamports at creation and funded by the pool creator.

---

## Implications for DEX Design

| Consideration | Ethereum model | Solana model |
|--------------|---------------|--------------|
| Token custody | Pool contract owns ERC-20 balances | SPL Token accounts owned by pool PDA |
| State location | In the contract's storage slots | Multiple separate data accounts |
| Parallel execution | Single-threaded per contract | Multiple accounts can be processed in parallel if non-overlapping |
| Account pre-declaration | Not required | All accounts that will be read or written must be declared upfront in the transaction |
| LP position | Fungible ERC-20 LP tokens | Fungible SPL LP tokens (AMM/CPMM) or NFT positions (CLMM) |
| Tick initialisation cost | Lazy (first-use pays gas) | Explicit: `TickArray` accounts must be pre-initialised; cost borne by the initialising party |

**Account pre-declaration** is a particularly important constraint for CLMM: a swap that crosses multiple tick array boundaries must include all relevant `TickArray` and `TickArrayBitmapExtension` accounts in the transaction's account list before signing. This adds complexity for swap routers.

---

## Sources

- https://extremelysunnyyk.medium.com/solana-amm-under-the-hood-raydium-insights-for-solana-builders-218ac339fde1
- https://deepwiki.com/raydium-io/raydium-amm/3.3-orderbook-integration
- https://github.com/raydium-io/raydium-clmm/blob/master/programs/amm/src/states/pool.rs
- https://deepwiki.com/raydium-io/raydium-sdk-V2/4.1.1-clmm-pool-creation-and-management
- https://immunefi.com/blog/bug-fix-reviews/raydium-tick-manipulation-bugfix-review/
- https://docs.raydium.io/raydium/protocol/security
