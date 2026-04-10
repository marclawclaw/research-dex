---
metric: tvl-comparison
updated: 2026-04-09
sources:
  - https://defillama.com/protocol/uniswap-v4
  - https://defillama.com/protocol/uniswap
  - https://coinlaw.io/uniswap-statistics/
  - https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped
tags: [tvl, metrics, comparison]
---

# Metric: TVL Comparison

Cross-protocol TVL figures for DEX research. All values sourced from DeFiLlama or protocol-official sources with dates.

## Uniswap ecosystem TVL (April 2026)

| Version | TVL (USD) | Source | Date |
|---------|-----------|--------|------|
| Uniswap (all versions) | ~$6.8B | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2026 |
| Uniswap V3 | ~$1.59B | [DeFiLlama](https://defillama.com/protocol/uniswap) | ~2026 |
| Uniswap V2 | ~$840M | [DeFiLlama](https://defillama.com/protocol/uniswap) | ~2026 |
| Uniswap V4 | >$1B (milestone 2025-07-27) | [DeFiLlama](https://defillama.com/protocol/uniswap-v4) | 2025-07-27 |
| Uniswap V4 share of Uniswap total | ~14% | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025/2026 |

## Uniswap V4 adoption timeline

| Milestone | Date | Source |
|-----------|------|--------|
| Mainnet launch | 2025-01-31 | [Uniswap blog](https://blog.uniswap.org/uniswap-v4-is-here) |
| $1B TVL reached | 2025-07-27 (177 days post-launch) | [DeFiLlama](https://defillama.com/protocol/uniswap-v4) |
| 5,000+ hooks on mainnet | End of 2025 | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) |
| 313,533+ pools with Atrium alumni hooks | End of 2025 | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) |
| >$190B cumulative volume | End of 2025 | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) |
| Deployed on 17 mainnets | April 2026 | [Uniswap docs](https://docs.uniswap.org/contracts/v4/deployments) |

## Cross-protocol TVL (from research index)

| Protocol | TVL (USD) | Ecosystem | Date |
|----------|-----------|-----------|------|
| Uniswap (all) | ~$6.8B | Ethereum | 2026 |
| Curve V1/V2 | ~$2.25B | Ethereum | ~2025 |
| Aerodrome | ~$1.24B | Base | Dec 2025 |
| Uniswap V3 | ~$1.59B | Ethereum | ~2026 |
| Raydium | ~$2.5B | Solana | Q3 2025 |
| Uniswap V2 | ~$840M | Ethereum | ~2026 |
| Orca Whirlpools | ~$247–500M | Solana | ~2025 |
| Balancer V2/V3 | ~$500M | Ethereum | ~2025 |

Sources: DeFiLlama protocol pages; [[_index]] research index.

## Volume metrics (Uniswap V4)

| Metric | Value | Source | Date |
|--------|-------|--------|------|
| Cumulative trading volume | >$190B | [Atrium Academy](https://blog.atrium.academy/uniswap-hook-incubator-2025-wrapped) | End 2025 |
| Unichain share of V4 volume | ~50% | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025/2026 |
| L2 share of V4 daily volume | 67.5% | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025/2026 |
| Uniswap V2+V3 historical volume | $2.75T | [Uniswap blog](https://blog.uniswap.org/uniswap-v4-is-here) | Pre-V4 launch |

## Notes for RFP writer

- Uniswap V4 reached $1B TVL in 177 days. For comparison, V3 reached $1B TVL in approximately 45 days after its May 2021 launch, so V4's path to $1B was slower in absolute terms (though market conditions differed significantly). This is a useful data point for adoption curve projections.
- The $190B cumulative volume figure (end 2025) comes from Atrium Academy's hook incubator report, which covers hooks built by their graduates; total V4 volume is higher.
- L2 dominance (67.5% of daily volume) reflects the chain deployment strategy: Unichain, Base, Arbitrum, and Optimism are primary activity centres.
- For precise current TVL, check [DeFiLlama Uniswap V4](https://defillama.com/protocol/uniswap-v4) directly; the DeFiLlama page was returning 403 at time of research (April 2026).

## Related notes

- [[projects/uniswap-v4]]
- [[metrics/vault-accounting-patterns]]
