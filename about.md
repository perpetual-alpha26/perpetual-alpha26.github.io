---
layout: default
title: About the Competition
---

<div class="page-content">
    <h1>About the Competition</h1>

    <h2>Task</h2>
    <p>Each team deploys an autonomous trading algorithm that trades a fixed universe of perpetual futures on the Hyperliquid testnet via its public API. Agents must manage the full trading lifecycle: signal generation, order placement on a live order book, position and margin management, funding-rate dynamics, and risk control across assets with heterogeneous trading regimes. The panel of tradable assets will be made up of about 10 between cryptos, stock indexes, exchange rates, and commodities. Further details on the assets and oracles will be provided soon.</p>

    <h2>Competition Platform and Market Design</h2>
    <p>Hyperliquid is a layer-1 blockchain whose core state is a fully on-chain central limit order book supporting perpetual futures with cross and isolated margin, funding, and liquidation. Its HIP-3 standard allows an approved deployer to launch and operate independent perpetual markets — choosing the underlying, the oracle, contract specifications, margin parameters, and fee schedule — on the shared matching engine. HIP-3 is available on the Hyperliquid testnet, which mirrors mainnet functionality with valueless test tokens distributed via faucet.</p>
    <p>The teams will trade on a dedicated HIP-3 perp DEX on the testnet containing a mixed universe of perpetuals, with oracle prices streamed from liquid reference markets during relevant market-hours. The assets universe comprises cryptos, stock indexes, exchange rates, and commodities. The tradable perimeter will be announced soon. This mixed universe is a deliberate design choice: agents must handle regime heterogeneity, calendar effects, oracle-update discontinuities, and cross-asset risk — a materially harder and more instructive problem than trading a single 24/7 crypto pair.</p>
    <p>Market-making bots will quote continuously in all markets to guarantee baseline liquidity, and per-market position limits will prevent any single team from cornering a book. Each team receives one whitelisted API wallet funded with an identical testnet balance; top-ups are not permitted. Technical instructions and a dataset for backtesting the strategies before the competition will be provided via email to the enrolled teams.</p>

    <h2>Eligible Teams</h2>
    <p>Teams must be made up of 1 to 4 people. At least one team member should be available to present at ICAIF '26 Milan.</p>

    <h2>Evaluation Protocol</h2>
    <p>Teams are evaluated according three criterion:</p>
    <ul>
        <li><strong>Total net return</strong> on initial capital, marked to oracle prices at the end of the live phase, min–max normalized across teams.</li>
        <li><strong>Negative</strong> of the <strong>standard deviation</strong> of daily equity returns (lower volatility scores higher), min–max normalized.</li>
        <li><strong>Jury score</strong> on a 2–4 page technical report: methodology and depth of AI/ML techniques, originality, rigor and reproducibility, and clarity. Each report is scored independently by at least three jurors.</li>
    </ul>

    <h2>Fairness and Anti-Gaming Rules</h2>
    <p>One trading account per team; accounts are whitelisted and monitored on-chain. To be ranked, a team must trade on at least 10 distinct days and reach a minimum cumulative notional volume. Wash trading, self-crossing, collusion between teams, and attacks on the exchange or oracle infrastructure are prohibited and detectable from the public on-chain record; violations lead to disqualification. Fully manual trading is not allowed: entries must be algorithmic, verified through the technical report and a code walkthrough with the jury.</p>

    <h2>Live Leaderboard</h2>
    <p>A public web leaderboard, updated in near real time from on-chain data, will display equity curves, PnL, and volatility for all teams throughout the live phase (with a short delay to avoid strategy leakage).</p>

    <h2>Awards</h2>
    <p>The top teams will be invited to present at ICAIF '26 Milan. The prizes are to be defined.</p>

</div>
