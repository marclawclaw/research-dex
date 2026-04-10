---
tags: [metrics, volume, comparison, dex, ethereum, solana]
last-updated: 2026-04-09
---

# Volume Comparison: DEX Ecosystem

Cross-project trading volume comparison for selected DEXs. All figures sourced and dated.

## Uniswap V2 volume metrics

| Metric | Value | Source | Date |
|--------|-------|--------|------|
| 24h volume | ~$15.1M | [CoinMarketCap](https://coinmarketcap.com/exchanges/uniswap-v2/) | April 2026 |
| 30-day volume | ~$3.1B | [DeFiLlama](https://defillama.com/protocol/uniswap-v2) | Early 2026 |
| 30-day volume trend | Down ~13% vs prior 30 days | [DeFiLlama](https://defillama.com/protocol/uniswap-v2) | Early 2026 |
| Cumulative all-time DEX volume | ~$604B | [DeFiLlama](https://defillama.com/protocol/uniswap-v2) | Early 2026 |

## Uniswap protocol total (all versions)

| Metric | Value | Source | Date |
|--------|-------|--------|------|
| Total historical DEX volume (all versions) | >$3.45T | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2026 |
| Annualised volume (all versions) | ~$1T+ | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025 |
| 30-day protocol-wide volume | >$148B | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025 |
| V2 share of total Uniswap volume | Minority; V3 ~60%, V4 ~30%, V2 remainder | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025 |

## Cross-protocol volume comparison

| Protocol | Volume metric | Value | Source | Date |
|----------|--------------|-------|--------|------|
| Uniswap V2 | 30-day | ~$3.1B | [DeFiLlama](https://defillama.com/protocol/uniswap-v2) | Early 2026 |
| Uniswap (all) | 30-day | >$148B | [CoinLaw](https://coinlaw.io/uniswap-statistics/) | 2025 |
| Jupiter (Solana aggregator) | Cumulative | >$1T | [Medium/Scoper](https://medium.com/@Scoper/solana-defi-deep-dives-jupiter-ultra-v3-next-gen-dex-aggregator-late-2025-2cef75c97301) | 2025 |
| Jupiter | Solana aggregator market share | 93.6% | [Medium/Scoper](https://medium.com/@Scoper/solana-defi-deep-dives-jupiter-ultra-v3-next-gen-dex-aggregator-late-2025-2cef75c97301) | 2025 |
| Raydium | Q3 2025 volume | [DATA NEEDED] | — | — |
| CoW Protocol | Daily RFQ (JupiterZ equivalent: JupiterZ ~$100M/day) | N/A (different model) | — | — |

## Notes on Uniswap V2 volume decline

Uniswap V2 volume has declined relative to V3 and V4 as liquidity has migrated to concentrated liquidity pools for better capital efficiency. However, V2 remains relevant because:
1. It is still the deepest constant-product AMM on Ethereum mainnet for long-tail tokens not supported by V3/V4 positions.
2. Many forks on other chains aggregate significant volume (not captured in V2-specific figures).
3. The protocol fee switch activated December 2025 demonstrates that V2 still generates material revenue (estimated ~$26M annualised protocol fees across V2 and V3 Ethereum mainnet pools in the initial 12-day post-activation period, per [Talos State of the Network](https://www.talos.com/insights/state-of-the-network-346)).

## Fee revenue context

From the December 2025 UNIfication fee switch activation:
- Protocol fee: 0.05% (1/6 of the 0.3% LP fee) redirected from V2 and V3 Ethereum pools
- Flow: trade fees → protocol portion → TokenJar contract → UNI burn via Firepit contract
- Annualised projected burn rate: ~4–5M UNI per year based on early data
- 100M UNI burned on 28 December 2025 from Community Treasury as one-time event

Source: [Uniswap Blog (UNIfication)](https://blog.uniswap.org/unification); [Talos State of the Network](https://www.talos.com/insights/state-of-the-network-346)

## Related notes

- [[projects/uniswap-v2]]
- [[metrics/tvl-comparison]]
