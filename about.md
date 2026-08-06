---
layout: default
title: About the Competition
---

<div class="page-content">
    <h1>About the Competition</h1>

    <h2>Task</h2>
    <p>Each team builds and deploys an autonomous trading algorithm that trades a fixed universe of perpetual futures on the Hyperliquid testnet, via its public API. Your algorithm will manage the full trading lifecycle: signal generation, order placement on a live order book, position and margin management, funding-rate dynamics, and risk control across assets with different trading regimes. The tradable universe will span multiple asset classes. Further details on the assets and oracles will be shared soon.</p>

    <h2>The Trading Platform</h2>
    <p>Hyperliquid is a layer-1 blockchain whose core state is a fully on-chain central limit order book supporting perpetual futures with cross and isolated margin, funding, and liquidation. Its HIP-3 standard allows an approved deployer to launch and operate independent perpetual markets — choosing the underlying, the oracle, contract specifications, margin parameters, and fee schedule — on the shared matching engine. HIP-3 is available on the Hyperliquid testnet, which mirrors mainnet functionality with valueless test tokens distributed via a faucet.</p>
    <p>You'll trade on a dedicated HIP-3 perpetuals exchange on the testnet. The tradable universe will comprise multiple asset classes, requiring your algorithm to handle several phenomena such as regime heterogeneity, calendar effects, oracle-update discontinuities, and cross-asset risk.</p>
    <p>Market-making bots will quote continuously in all markets to guarantee baseline liquidity, and per-market position limits will prevent any single team from cornering a book. Your team will receive one whitelisted API wallet, funded with a testnet balance identical to every other team — top-ups aren't permitted. Technical instructions and a backtesting dataset will be provided soon, so you can test your strategy ahead of the competition.</p>

    <h2>Eligible Teams</h2>
    <p>Eligible Teams: Teams must be made up of 1 to 4 members. At least one team member must be available to attend ICAIF 2026 in Milan in person.</p>

    <h2>Evaluation Protocol</h2>
    <p>Teams are evaluated according to three criteria:</p>
    <ul>
        <li><strong>Total net return</strong> on initial capital, marked to oracle prices at the end of the Live Trading phase, min–max normalized across teams.</li>
        <li><strong>Negative</strong> of the <strong>standard deviation</strong> of daily equity returns (lower volatility scores higher), min–max normalized.</li>
        <li><strong>Jury score</strong> on a 2–4 page technical report: methodology and depth of AI/ML techniques, originality, rigor and reproducibility, and clarity. Each report is scored independently by at least three jurors.</li>
    </ul>

    <h2>Fairness and Anti-Gaming Rules</h2>
    <p>One trading account per team; accounts are whitelisted and monitored on-chain. To be ranked, a team must trade on at least 10 distinct days and reach a minimum cumulative notional volume. Wash trading, self-crossing, collusion between teams, and attacks on the exchange or oracle infrastructure are prohibited. All trading activity is recorded on-chain and can be reviewed for violations, which lead to disqualification. Fully manual trading is not allowed: entries must be algorithmic, as verified through the technical report and a code walkthrough with the jury.</p>

    <h2>Live Leaderboard</h2>
    <p>A public web leaderboard, updated in near real time from on-chain data, will track team performance throughout the Live Trading phase.</p>

    <h2>Awards</h2>
    <p>The top teams will be invited to present at the ICAIF 2026 Competition Track in Milan. Prize details will be announced soon.</p>

</div>
