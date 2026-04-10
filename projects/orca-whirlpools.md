# Orca Whirlpools

**Ecosystem:** Solana (also deployed on Eclipse)
**Category:** Concentrated Liquidity AMM (CLMM)
**Program ID:** `whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc`
**Website:** https://orca.so
**Developer Docs:** https://dev.orca.so
**Source:** https://github.com/orca-so/whirlpools

---

## Summary

Orca Whirlpools is the concentrated liquidity market maker (CLMM) layer of Orca DEX, one of Solana's oldest and most established decentralised exchanges. Launched in February 2021, Orca was among the first protocols on Solana. The Whirlpools CLMM was built from scratch for Solana's account model rather than ported from Ethereum, giving it architectural advantages such as tick arrays stored as separate PDAs, a full-featured Rust SDK, and a June 2025 adaptive fee upgrade.

By February 2025, Orca had processed over $300 billion in cumulative trading volume. It is the second-largest AMM on Solana by TVL, competing principally with Raydium CLMM, Jupiter-aggregated pools, and Meteora DLMM.

---

## Key Metrics

See [[metrics/orca-whirlpools-metrics]] for sourced figures.

| Metric | Value | Source | Date |
|--------|-------|--------|------|
| TVL | $246.8 million | [DefiLlama](https://defillama.com/protocol/orca) | ~2025 |
| Cumulative volume | >$300 billion | [Orca Medium](https://orca-so.medium.com/orca-solanas-liquidity-layer-3855b5985aa3) | Feb 2025 |
| SOL/USDC pool 30-day volume | $22.8 billion | [PANews (Chinese edition)](https://www.panewslab.com/en/articles/5zrwdqh9) [source unverified in English] | May 2025 |
| Solana DEX market share (volume) | ~16% | [bitcoinethereumnews.com](https://bitcoinethereumnews.com/finance/solanas-dex-wars-intensify-as-pumpfun-and-orca-challenge-raydiums-dominance/) | Apr 2025 |
| ORCA max supply | 100 million (hard cap) | [April 2025 Updates, Orca Medium](https://orca-so.medium.com/april-2025-updates-e670b5d86d17) | Apr 2025 |
| ORCA circulating supply | ~60.8 million (post-burn) | [Bitrue](https://www.bitrue.com/blog/orca-token-burn-how-this-will-impact-orcas-price) | May 2025 |
| Token pairs | 480+ tokens, 1,295+ pairs | [Orca Medium](https://orca-so.medium.com/orca-solanas-liquidity-layer-3855b5985aa3) | Feb 2025 |
| GitHub stars | 525 | [GitHub](https://github.com/orca-so/whirlpools) | 2025 |

---

## History and Funding

- **Founded:** September 2020
- **Mainnet launch:** February 2021
- **Whirlpools CLMM launch:** 2022 (initial), continually upgraded
- **Series A:** $18 million raised September 2021. Token sale (ORCA from treasury). Led by Polychain Capital, Three Arrows Capital, and Placeholder VC. Participants included Coinbase Ventures, Jump Capital, Sino Global Capital, Collab+Currency, DeFiance Capital, Zee Prime Capital, and Solana Capital. ([The Block](https://www.theblock.co/post/118279/solana-dex-orca-funding-18-million-three-arrows-capital-others), Sep 2021)
- Prior to Series A, Orca received a Solana Foundation grant.

---

## Architecture

### Pool Model

Whirlpools implements a tick-based CLMM modelled conceptually after Uniswap v3 but rewritten from scratch for Solana's account model. Key differences from Ethereum CLMMs:

- Tick data is stored in separate on-chain PDA accounts called **tick arrays** rather than in a single pool contract storage slot. See [[patterns/tick-array-architecture]].
- Liquidity positions are represented as NFTs (position NFTs) rather than fungible LP tokens.
- The program is deployed as a single Solana program (smart contract) identified by its fixed program ID.

### Pool Types

| Type | Description | Use Case |
|------|-------------|----------|
| Concentrated Liquidity Whirlpool | Custom price ranges, tick-based | Active LPs, professional market makers |
| Splash Pool | Full-range only, simplified UI, no extra tick array init | New token launches, passive LPs, memecoins |

**Splash Pools** are built on top of the same Whirlpool program but restrict positions to full-range, eliminating the need to pre-initialise tick arrays. ([Orca Docs](https://docs.orca.so) [LINK MOVED])

### Fee Tiers

Standard fee tiers available on Solana mainnet-beta:

| Tick Spacing | Fee Rate |
|---|---|
| 1 | 0.01% |
| 2 | 0.02% |
| 32 | (common stable) |
| 64 | (common volatile) |
| 128 | (wide range) |
| 256 | 2.00% |

Fee distribution (from protocol docs): 87% to liquidity providers, 12% to DAO treasury, 1% to Climate Fund.

### Adaptive Fee Pools

Introduced June 2025. See [[patterns/adaptive-fees]] for full detail.

In summary: adaptive fee pools layer a **variable fee** on top of a static base fee. The variable component scales quadratically with a volatility accumulator that tracks tick-group crossings per transaction. Two new account types are added per pool: an Adaptive Fee Tier Account (shared config) and an Oracle Account (per-pool dynamic state).

**Formula:** `f_v = A × (v_a × s)²`

Where `A` is the `variableFeeControl` parameter, `v_a` is the volatility accumulator, and `s` is tick group size. Maximum total fee is capped at 10%.

Source: [dev.orca.so Whirlpool Fees](https://dev.orca.so/Architecture%20Overview/Whirlpool%20Fees/)

---

## Key Upgrades Timeline

| Date | Feature | Impact |
|------|---------|--------|
| Aug 2024 | **SparseSwap** (SDK v0.13.4) | Swaps traverse uninitialized tick arrays treating them as zero-liquidity; automatic tick array reordering; up to 6 tick arrays via `remaining_accounts` | [Orca Medium](https://orca-so.medium.com/introducing-sparseswap-a-new-era-in-trading-on-orca-c0eefa067980) |
| Nov 2024 | Token Extensions support; fully refundable position NFT rent | Lower cost to close positions | [April 2025 Updates](https://orca-so.medium.com/april-2025-updates-e670b5d86d17) |
| Apr 2025 | 25M ORCA token burn, 20% fee buyback activated | Deflationary supply; xORCA staking in development | [Orca Medium](https://orca-so.medium.com/april-2025-updates-e670b5d86d17) |
| Apr 2025 | CPI Integration Repository released | Simplified on-chain composability for builders | [Orca Medium](https://orca-so.medium.com/april-2025-updates-e670b5d86d17) |
| Jun 2025 | **Adaptive Fee Pools** | Dynamic fees based on volatility accumulator | [dev.orca.so](https://dev.orca.so/Architecture%20Overview/Whirlpool%20Fees/) |
| Jul 2025 | **Dynamic Tick Arrays** | 97% reduction in tick array init cost; grow-as-you-go model | [Orca Medium](https://orca-so.medium.com/create-pools-for-less-with-orcas-dynamic-tick-arrays-13e8c5dbcc8c) |
| Jul 2025 | **Wavebreak** token launchpad | Anti-bot token launch with bonding curve; graduates to Whirlpool | [Blockworks](https://blockworks.co/news/orca-solana-token-launchpad) |
| Aug 2025 | 24-month ORCA buyback programme (55k SOL + $400k USDC) | Sustained buy pressure; governance-approved | [OKX](https://www.okx.com/en-ae/learn/orca-buybacks-staking-solana-tokenomics) |

---

## SDK and Developer Ecosystem

The Whirlpools program is open source and provides multi-language SDKs:

- **TypeScript:** `@orca-so/whirlpools` (recommended, v1.0+); legacy `@orca-so/whirlpools-sdk`
- **Rust:** `orca_whirlpools` (crates.io)
- **Lower-level:** `whirlpools-client` (auto-generated), `whirlpools-core` (math/quoting)
- **CPI:** `whirlpool-cpi` crate for composable on-chain integration

The monorepo uses NX and requires Anchor v0.32.1 and Solana v2.1.0.

The program uses **verifiable builds**, allowing on-chain code to be matched against the repository source. ([GitHub](https://github.com/orca-so/whirlpools))

### Supported Networks

- Solana Mainnet-Beta
- Solana Devnet
- Eclipse Mainnet
- Eclipse Testnet

Orca deployed on Eclipse (Ethereum-settlement SVM chain) as its first multichain expansion. ([Orca Medium](https://orca-so.medium.com/orca-set-to-launch-on-eclipse-a6ab997db109))

---

## Security

| Auditor | Date | Notes |
|---------|------|-------|
| Kudelski Security | Jan 2022 | Initial audit |
| Neodyme AG | May 2022 | [Public report](https://reports.neodyme.io/reports/orca_report.pdf) |
| OtterSec | Aug 2024 | |
| Sec3 | Feb 2025 | |
| Sec3 | Jun 2025 | |
| Sec3 | Aug 2025 | |

Source: [GitHub repository](https://github.com/orca-so/whirlpools); [Orca support docs](https://support.withorca.com/security--legal-fine-print/aZYfzuthvijeBDSZN7kfdA/orca%E2%80%99s-external-audits/vC2w6cNWp7vFuFnGvaKAoG)

**No hacks or exploits to date** (as of the research date). ([Orca Medium](https://orca-so.medium.com/orca-solanas-liquidity-layer-3855b5985aa3))

---

## ORCA Token and Governance

- **Token:** ORCA (SPL token, governance token)
- **Launch:** September 2021
- **Max supply:** 100 million (hard cap)
- **Post-burn circulating supply:** ~60.8 million (after 25M burn, May 2025)
- **Original distribution:** Treasury 50%, team 22%, future team 12%, investors 10%, advisors 6%
- **Governance:** Orca DAO; on-chain voting for major protocol changes

### Value Accrual

1. **Fee buyback:** 20% of Whirlpools protocol fees used to buy ORCA on open market (governance proposal, Apr 2025). Separate from the August 2025 treasury-funded programme.
2. **Treasury buyback:** 55,000 SOL + $400,000 USDC from DAO treasury allocated over 24 months; capped at 2% of daily ORCA trading volume per day. ([OKX](https://www.okx.com/en-ae/learn/orca-buybacks-staking-solana-tokenomics), Aug 2025)
3. **xORCA staking:** In development as of April 2025; purchased tokens seed initial staking rewards.
4. **Validator node:** 55,000 SOL staked in Orca's own Solana validator; staking yield reinvested into ecosystem.

---

## Wavebreak Launchpad

Wavebreak is Orca's "human-first" token launchpad, launched July 2025. It targets the memecoin and new-token market dominated by Pump.fun.

**Key mechanics:**
- Transparent bonding curve sale; price rises smoothly with demand.
- Anti-bot system: CAPTCHA plus on-chain permissions prevent sniping.
- Traders earn points proportional to bonding-phase volume; points convert to daily rewards.
- **Graduation threshold:** When the bonding curve is fulfilled (market cap ~85 SOL), the token graduates to an Orca Whirlpool and becomes tradeable on all aggregators.

**Connection to Whirlpools:** Graduated tokens feed directly into Orca's CLMM, deepening Whirlpool liquidity and driving protocol fee revenue (which in turn feeds ORCA buybacks).

Sources: [Blockworks](https://blockworks.co/news/orca-solana-token-launchpad); [Ainvest](https://www.ainvest.com/news/solana-news-today-orca-launches-wavebreak-challenge-launchpad-dominance-anti-bot-mechanisms-2507/)

---

## Competitive Position

### vs. Raydium CLMM

Raydium is the largest Solana DEX by volume, holding ~31% DEX market share vs. Orca's ~16% (week of Apr 14-20, 2025). ([bitcoinethereumnews.com](https://bitcoinethereumnews.com/finance/solanas-dex-wars-intensify-as-pumpfun-and-orca-challenge-raydiums-dominance/))

Key differences:

| Dimension | Orca Whirlpools | Raydium CLMM |
|-----------|----------------|--------------|
| LP token model | NFT positions | NFT positions |
| Launchpad | Wavebreak (Jul 2025) | LaunchLab |
| Developer tools | Extensive SDKs, CPI repo, verifiable builds | Raydium SDK |
| Adaptive fees | Yes (Jun 2025) | No (as of research date) |
| Eclipse deployment | Yes | No |
| Climate Fund | 1% of fees | No |
| Pool size / depth | Smaller vs. Raydium | Deeper; lower slippage for large trades |
| Aggregator routing | ~16% of Jupiter volume | ~55% of Jupiter volume |

Orca leads in LST (Liquid Staking Token) swaps and project token markets (e.g., JLP, JTO). Raydium's deeper liquidity makes it preferred for large-trade routing. ([PANews](https://www.panewslab.com/en/articles/5zrwdqh9))

### vs. Meteora DLMM

Meteora's Dynamic Liquidity Market Maker (DLMM) captured over 22% of Solana DEX market share by volume as of February 2025 ([CoinGecko](https://www.coingecko.com/learn/what-is-meteora-dex-solana-crypto)), posing a significant challenge. Meteora uses a different bin-based liquidity model rather than ticks.

---

## MEV and Limitations

### MEV Exposure

Orca Whirlpools pools are targeted by sandwich MEV bots. Jito-based MEV bots explicitly support Orca Whirlpool pools as targets. ([DL News](https://www.dlnews.com/articles/defi/solana-users-use-jito-to-stop-sandwich-attacks-and-mev/))

**Mitigations available:**
- Jito's `DontFront` feature: adding a public key starting with `jitodontfront` to a transaction instruction forces the transaction to index 0 in any Jito bundle. ([Solana Docs](https://solana.com/developers/guides/advanced/mev-protection))
- Slippage tolerance limits: narrow slippage settings reduce sandwich profitability.
- Orca does not currently implement native MEV protection at the protocol level; it relies on infrastructure-layer solutions (Jito).

### Capital Inefficiency vs. Active Management

Concentrated positions go out-of-range when price moves beyond the selected band. Out-of-range positions earn no fees and are exposed to single-asset risk. A 10% price move outside range produces IL comparable to a 30% move in a full-range pool. This requires active position management or vault strategies. ([Orca Docs](https://docs.orca.so) [LINK MOVED])

### Large-Trade Slippage

Orca's overall liquidity depth is smaller than Raydium's, leading to higher slippage for large trades. This limits its share of high-volume institutional or aggregator-routed flow. ([ChainCatcher](https://www.chaincatcher.com/en/article/2168403))

### Regulatory Initiative

April 2025: Orca partnered with the Solana Policy Institute and Superstate to submit a proposal to the SEC for compliant issuance, trading, and settlement of securities on public blockchains via AMMs. ([Orca April 2025 Updates](https://orca-so.medium.com/april-2025-updates-e670b5d86d17))

---

## Related Notes

- [[patterns/adaptive-fees]]
- [[patterns/tick-array-architecture]]
- [[metrics/orca-whirlpools-metrics]]
