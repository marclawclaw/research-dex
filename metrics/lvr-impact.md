# Metrics: LVR (Loss-Versus-Rebalancing) Impact

**Related pattern:** [[patterns/mev-capturing-amm]]
**Related project:** [[projects/cow-protocol]]

## What is LVR?

Loss-Versus-Rebalancing is the value systematically extracted from AMM liquidity providers by arbitrageurs exploiting stale pool prices. It is distinct from impermanent loss: LVR is a continuous, directional transfer of value from LPs to arbitrageurs, and it does not reverse when prices return to their prior levels.

Academic definition: LVR at each instant equals the difference between the value of the LP position and the value of an equivalent position that rebalances optimally on liquid external markets.

**Mathematical formula:** For a constant-function AMM, instantaneous LVR = σ²/8, where σ is the asset's instantaneous price volatility.

## Scale of LVR in DeFi

| Metric | Value | Source | Date / Notes |
|---|---|---|---|
| Estimated annual LVR across DeFi | $500M+ | https://cow.fi/learn/what-is-loss-versus-rebalancing-lvr | Industry estimate; described as "more than all trading-related forms of MEV combined" |
| Uniswap v3 daily volume needed to offset LVR (at 5% asset volatility, 30 bps fee) | 10% of total pool TVL per day | https://cow.fi/learn/what-is-loss-versus-rebalancing-lvr | Illustrative calculation |
| Cumulative LVR on DeFi AMMs | [NOT FOUND] | — | — |

## CoW AMM LVR Mitigation Performance

| Metric | Value | Source | Date / Notes |
|---|---|---|---|
| LP surplus captured by CoW AMM since launch | $1.2M+ (beta phase) | https://cow.fi/cow-amm | 2024-2025 |
| LP liquidity protected from LVR by CoW AMM | $18M+ | https://cow.fi/cow-amm | 2024-2025 |
| LP return vs traditional CFAM (backtesting) | Equal or better for 10 of 11 most liquid non-stablecoin pairs | https://cow.fi/cow-amm | 6-month 2023 backtesting period |
| TVL uplift in CoW AMM pools vs reference pools | ~4.75% (beta phase) | https://cow.fi/cow-amm | 2024-2025 |

## Academic and Research Sources

| Source | Description | URL |
|---|---|---|
| Milionis, Moallemi, Roughgarden, Zhang (2022) | Original LVR paper; formal mathematical definition and proofs | https://anthonyleezhang.github.io/pdfs/lvr.pdf |
| Moallemi (2023) | Sequel paper on LVR in presence of fees | https://moallemi.com/ciamac/papers/lvr-fee-model-2023.pdf |
| a16z crypto (2022) | Accessible analysis quantifying LVR cost | https://a16zcrypto.com/posts/article/lvr-quantifying-the-cost-of-providing-liquidity-to-automated-market-makers/ |
| Fenbushi Capital (2024) | Survey of LVR solutions | https://fenbushi.vc/2024/01/20/ending-lps-losing-game-exploring-the-loss-versus-rebalancing-lvr-problem-and-its-solutions/ |
| arXiv (2024) | Measuring arbitrage losses and AMM LP profitability | https://arxiv.org/html/2404.05803v1 |

## Notes on Data Quality

The $500M annual LVR estimate is widely cited but derived from modelling rather than direct on-chain measurement. Actual LVR depends on realised volatility and trading volume, which vary significantly over time. The academic papers provide the methodological basis for estimation but do not provide a continuously updated empirical figure.

CoW AMM's reported LP surplus ($100,000+) is cumulative since launch and represents a lower bound on value redirected from arbitrageurs to LPs; the total is expected to grow as CoW AMM TVL scales.
