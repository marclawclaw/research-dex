# Raydium (AMM + CLMM)

**Ecosystem:** Solana
**Category:** Decentralised Exchange (DEX) / AMM
**Status:** Live (production)
**Launch date:** February 2021
**Website:** https://raydium.io
**Docs:** https://docs.raydium.io
**Source (CLMM):** https://github.com/raydium-io/raydium-clmm
**Source (AMM):** https://github.com/raydium-io/raydium-amm

---

## Overview

Raydium is the largest decentralised exchange by liquidity depth on Solana, self-described as "the most widely integrated liquidity infrastructure on Solana." It combines three pool architectures (standard AMM v4, CLMM, and CPMM) with an optional integration into OpenBook's central limit order book (CLOB). A token launchpad (LaunchLab) was added in April 2025.

The protocol was built by a pseudonymous team with algorithmic-trading backgrounds in traditional finance. The founders identified inefficiencies in Ethereum DeFi (high fees, slow finality) during 2020 and began development in Q4 2020 in collaboration with the Serum team.

**Founders (pseudonymous):**
- AlphaRay — strategy and operations; formerly algorithmic trading in commodities.
- XRay — CTO and dev team lead.
- GammaRay — marketing and communications.
- Timon Peng — co-founder and senior full-stack engineer (public identity), 11+ years crypto experience.

---

## Pool Architecture

Raydium exposes three pool types to liquidity providers and token issuers:

### 1. Standard AMM (v4)
- Constant-product formula: `x * y = k`, adjusted for fees.
- Integrated with OpenBook's CLOB via a Fibonacci-spaced order grid (see [[hybrid-amm-orderbook]]).
- Fee: 0.25% per swap (0.22% to LPs; 0.03% to protocol RAY buyback).
- Described as "the most distributed AMM program on Solana."
- Only pool type affected by the December 2022 exploit.

### 2. CLMM (Concentrated Liquidity Market Maker)
- Also known as v6 internally; corresponds to Uniswap v3's tick-based model.
- LPs choose a price range; liquidity is only active within that range.
- Positions are represented as NFTs (non-fungible tokens) rather than fungible LP tokens.
- Fee tiers tied to tick spacing (see [[raydium-clmm-metrics]]):
  - 1 tick spacing (0.01% fee) — stable pairs (e.g. USDC/USDT).
  - 10 tick spacing (0.05% fee) — correlated assets.
  - 60 tick spacing (0.25% fee) — standard pairs.
  - 120 tick spacing (1% fee) — volatile/exotic pairs.
- Additional fee tiers (0.02%, 0.03%, 0.04%, 2%) were introduced in August 2024, expanding CLMM to eight tiers total.
- LPs receive 84% of fees; 12% to RAY buybacks; 4% to treasury.
- Fees accumulate in-position; not auto-compounded. Must be claimed via withdraw or a zero-liquidity decrease.
- Impermanent loss is amplified: if price exits the chosen range the position becomes entirely single-sided and stops earning fees, with no automatic closure.
- Source: https://docs.raydium.io/raydium/for-liquidity-providers/pool-types/clmm-concentrated

### 3. CPMM (Constant Product Market Maker)
- Simpler, permissionless constant-product pool without OpenBook integration.
- Four fee tiers: 0.25%, 1%, 2%, 4%.
- Fee split: 84% to LPs, 12% to RAY buybacks, 4% to treasury.
- CPMM pool count grew 82% QoQ in Q3 2025.
- Source: https://docs.raydium.io/raydium/for-liquidity-providers/pool-types/cpmm-constant-product

---

## OpenBook Integration

The standard AMM v4 integrates with OpenBook (formerly Serum), Solana's on-chain CLOB. See [[hybrid-amm-orderbook]] for the full pattern note. In brief:

- Idle AMM liquidity is deployed as limit orders on OpenBook at Fibonacci-derived price levels (23.6%, 38.2%, 61.8% from market price).
- The `MonitorStep` instruction manages the order grid, continuously cycling through Plan → Place → Purge states.
- `withdrawPnl` collects PnL accrued from the difference between constant-product pricing and actual executed orderbook prices.
- This integration was the mechanism exploited in December 2022 (see Security section below).

---

## LaunchLab

Launched in April 2025. A bonding-curve token launchpad similar to pump.fun, with third-party integration (BONKfun, LetsBonk). Allows permissionless token creation; tokens graduate to a Raydium pool at a market-cap threshold.

- Revenue model: Raydium earns 25 bps on all bonding-curve swaps, including on platforms integrated with LaunchLab.
- Q2 2025: contributed $4.0M (21.7% of $18.4M total net revenue). Source: https://messari.io/report/state-of-raydium-q2-2025
- Q3 2025: grew 220% QoQ to $12.8M (52% of $24.3M total net revenue). Source: https://altsignals.io/post/raydium-q3-2025-defi-growth-analysis
- Q3 2025 LaunchLab volume concentration: 98% from LetsBonk ecosystem launches.

---

## Adoption Metrics

See also [[raydium-adoption-metrics]].

### TVL
| Period | TVL | Source |
|--------|-----|--------|
| Q3 2025 | $2.46B (+35% QoQ) | Messari / Blockworks |
| Q4 2025 | $1.4B (-38.8% QoQ) | Messari State of Solana Q4 2025 |

Source: https://messari.io/report/state-of-solana-q4-2025

### Trading Volume
| Period | Volume | Notes |
|--------|--------|-------|
| Jan 2025 (peak) | $195.8B monthly | $16B single-day peak on Jan 19 (TRUMP meme launch) |
| Q1 2025 avg daily | $3.6B | Messari Q1 2025 |
| Q2 2025 avg daily | $1.1B (-69.2% QoQ) | Messari Q2 2025 |
| Q3 2025 total | $51.9B (+31% QoQ) | Blockworks Q3 2025 report |
| Q4 2025 avg daily | $545.9M (-33.5% QoQ) | Messari Q4 2025 |

Source: https://messari.io/report/state-of-raydium-q2-2025 ; https://blockworks.com/news/raydium-token-holder-report

### Market Share (Solana DEX)
| Period | Share | Notes |
|--------|-------|-------|
| Jan 2025 | 27.1% | Surpassed Uniswap globally for one month |
| Jul 2025 | 23.5% | Intra-Q3 peak |
| Sep 2025 | 10.5% | Intra-Q3 trough |
| Q3 2025 avg | 15.9% | Blockworks Q3 2025 |
| Q4 2025 | 13.8% | Messari Q4 2025 |

Source: https://www.kucoin.com/news/articles/raydium-surpasses-uniswap-in-monthly-dex-volume-by-25-signaling-shift-in-defi-market-dynamics

### Revenue
| Period | Net Revenue | Notes |
|--------|-------------|-------|
| Q2 2025 | $18.4M | LaunchLab contributed 21.7% |
| Q3 2025 | $24.3M (+69% QoQ) | LaunchLab = 52% of total |

Source: https://altsignals.io/post/raydium-q3-2025-defi-growth-analysis

### Cumulative Volume
- All-time DEX volume: $695.7B (DefiLlama, as at time of research). Source: https://defillama.com/protocol/raydium

---

## RAY Token

- **Max supply:** 555,000,000 RAY.
- **Circulating supply (Q2 2025 end):** ~275.2M RAY (49.6% of max).
- **Circulating supply (adjusted, ex-buybacks):** ~207.1M RAY at Q2-end.
- **Annual emissions from mining reserve:** ~1.9M RAY/year.
- **Buyback mechanism:** A portion of all swap fees is used to buy RAY on-market. CLMM/CPMM: 12% of fees. Standard AMM: 0.03% of 0.25% fee.
- **Q2 2025 buyback allocation:** $10.7M annualised → $0.21 earnings per token; 9.7% yield on adjusted market cap of $441.7M.
- Source: https://messari.io/report/state-of-raydium-q2-2025 ; https://tokenomist.ai/raydium

**Token allocation:**
| Category | Share |
|----------|-------|
| Mining Reserve | 34% |
| Partnership & Ecosystem | 30% |
| Team | 20% |
| Liquidity | 8% |
| Community & Seed | 6% |
| Advisors | 2% |

---

## Security

### Audit History
| Firm | Date | Scope |
|------|------|-------|
| Kudelski Security | Q2 2021 | Order-book AMM |
| OtterSec | Q3 2022 | CLMM, updated Order-book AMM, Staking |
| MadShield | Q2 2023 | Order-book AMM and OpenBook migration |
| MadShield | Q1 2024 | Constant Product AMM (CPMM) |
| Halborn | Q4 2024 | Burn & Earn liquidity locker |
| Halborn | Q2 2025 | LaunchLab |
| Sec3 | Q3 2025 | CPMM update |

Source: https://docs.raydium.io/raydium/protocol/security

Additional review: Neodyme team via bug bounty agreement; ImmuneFi bug bounty programme active.

**Governance:** All program authority held under a Squads multi-sig. Programs remain upgradeable; larger updates engage external security professionals. As of July 2024, Raydium does not employ a timelock program function for updates (source: [Raydium access controls docs](https://docs.raydium.io/raydium/protocol/security/access-controls)); no public announcement of timelock deployment has been found as of Q1 2026.

---

### December 2022 Exploit

**Date:** 16 December 2022, 10:12 UTC.
**Amount stolen:** ~$4.4M across eight constant-product liquidity pools.
**Attack vector:** Private key compromise via Trojan malware on the pool manager's machine. The attacker obtained the key for the account with admin authority over AMM v4 pools.

**Mechanism (two steps):**
1. Used `SetParams` with `AmmParams::SyncNeedTake` to inflate the `need_take_pc` and `need_take_coin` fee parameters without actual trading volume.
2. Called `withdrawPNL` (designed for protocol fee collection) to drain the inflated "fees" from pools.

**Response timeline:**
- 14:16 UTC: hot patch deployed; compromised authority revoked and moved to hardware wallet.
- Dec 17, 10:27 UTC: AMM v4 program upgraded via Squads multisig.
- Dec 17, ~15:00 UTC: remaining admin parameters transferred to multisig.

**Attacker's exit:** ~$2.7M bridged to Ethereum (1,774.5 ETH over 42 transactions), routed through Tornado Cash. Source: https://www.theblock.co/post/203732/raydium-hacker-funnels-2-7-million-through-tornado-cash-mixer

**Remediation:** Removed `AmmParams::MinSize`, `SyncLp`, `SetLpSupply`, `SyncK`, `SyncNeedTake`. All admin parameters for Stable Pools, Acceleraytor, and DropZone also removed. Admin authority moved to Squads multi-sig governance.

Source: https://raydium.medium.com/detailed-post-mortem-and-next-steps-d6d6dd461c3e ; https://www.certik.com/resources/blog/raydium-protocol-exploit-incident-analysis

---

### January 2024 Tick Manipulation Bug (ImmuneFi)

**Discovered:** 10 January 2024 by whitehat @riproprip.
**Bounty paid:** $505,000 in RAY tokens.
**Vulnerability:** In `increase_liquidity.rs`, the `tickarray_bitmap_extension` account passed via `remaining_accounts[0]` was not validated against the pool's actual `TickArrayBitmapExtension` PDA. An attacker could:
1. Create a secondary pool.
2. Swap to shift price to a target range.
3. Pass a fake `TickArrayBitmapExtension` account to flip tick states.
4. Accumulate unearned liquidity credit and then extract disproportionate tokens via swaps.
**Fix:** Added a key-equality check: `remaining_accounts[0]` must match the canonical `TickArrayBitmapExtension` PDA for the pool before processing.

Source: https://immunefi.com/blog/bug-fix-reviews/raydium-tick-manipulation-bugfix-review/

---

## MEV and Bot Activity

Raydium is the primary MEV target on Solana. Key data:

- **72% of Solana sandwich attacks target Raydium** (vs. 18% Orca, 10% pump.fun). Source: https://medium.com/@faatihmudathir/unmasking-solanas-sandwich-attacks-building-an-open-source-mev-dashboard-how-to-quantify-and-b9d1121a22db
- On 15 May 2024, attackers spent 12,000 SOL in fees to extract 47,000 SOL (~$6.2M) from Raydium swaps in a single day. Source: https://blockworks.co/news/lightspeed-newsletter-sandwich-attacks
- The B91 bot targeted 78,800 victims over a 30-day period; peak day: 4,925 sandwich attacks on 21 April 2025. Source: https://medium.com/@joel_28760/breaking-down-mev-sandwich-attacks-on-solana-the-b91-bot-case-study-3e1c1ba35556
- Vpe bot: 1.55M sandwich transactions over 30 days (Dec 2024 to Jan 2025); 88.9% success rate; 65,880 SOL ($13.43M) profit. Source: https://www.helius.dev/blog/solana-mev-report
- In October 2025, >18,000 SOL extracted from >202,000 victims via sandwich attacks across Solana DEXes. Source: https://finance.yahoo.com/news/sandwich-attacks-spiraling-control-solana-112345764.html
- Total sandwich extraction (Solana, 16-month window up to mid-2025): $370M to $500M. Source: https://solanacompass.com/learn/accelerate-25/scale-or-die-at-accelerate-2025-the-state-of-solana-mev
- MEV is possible because Solana's known leader rotation schedule allows bots to predict block producers and submit Jito bundles to guarantee transaction ordering.

**Why Raydium is disproportionately targeted:**
- Largest liquidity; highest volume; widest range of memecoin pools with high slippage tolerance.
- Memecoin traders frequently set high slippage (sometimes 99%), creating large sandwich windows.
- 65% of attacks occur during US market hours (12:00–20:00 UTC); memecoin attacks spike 3x on weekends.

---

## Limitations and Risks

1. **MEV exposure:** As noted above, Raydium's dominance makes it the primary sandwich attack target. 72% of Solana MEV hits Raydium pools.
2. **Impermanent loss amplification in CLMM:** Tight ranges earn more fees but go out-of-range easily, resulting in single-sided exposure and fee cessation. No automated rebalancing.
3. **CLMM position NFT risk:** If a position NFT is lost, sold, or burned, the liquidity it represents is permanently inaccessible.
4. **Admin key risk (historical):** The 2022 exploit demonstrated the danger of a single admin key. Mitigated post-exploit by multisig governance.
5. **OpenBook dependency:** Standard AMM v4 pools depend on an active OpenBook market; if OpenBook liquidity dries up, the hybrid benefit is reduced.
6. **Volume concentration risk:** January 2025's record $195.8B monthly volume was largely driven by TRUMP memecoin speculation. When meme cycles end, volumes contract sharply (Q2 2025: -69.2% QoQ).
7. **LaunchLab concentration:** In Q3 2025, 98% of LaunchLab revenue came from LetsBonk ecosystem launches, creating platform concentration risk.
8. **Upgradeability without timelock:** Programs remain upgradeable by the team multisig with no timelock. Raydium docs (July 2024) confirm no timelock function is currently employed; no public confirmation of deployment as of Q1 2026.

---

## Differentiators

- Only Solana DEX with a hybrid AMM/CLOB architecture (via OpenBook).
- Fibonacci-spaced liquidity grid means the AMM participates in price discovery on the order book side, not just passive rebalancing.
- CLMM is on-chain, fully non-custodial, with position NFTs enabling composability.
- LaunchLab provides a built-in token launch funnel that feeds liquidity directly into Raydium pools at graduation.
- Largest Solana DEX by TVL and historical volume, giving it a network-effects advantage for integrators.

---

## Cross-References

- [[solana-account-model-pools]] — how Solana's account architecture underpins Raydium's pool design.
- [[hybrid-amm-orderbook]] — pattern note on the AMM/CLOB integration.
- [[raydium-adoption-metrics]] — quantitative metrics table.
- [[raydium-clmm-metrics]] — CLMM-specific fee tier and position data.
