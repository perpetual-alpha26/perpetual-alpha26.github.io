---
layout: default
title: Participant Reference
permalink: /docs/reference/
---
{% assign v = site.data.venue %}
{% assign base_id = v.dex_index | times: 10000 | plus: 100000 %}
<div class="page-content docs" markdown="1">

# Reference
{: .no_toc}

This page is the API reference for the [Participant Guide](../).

* TOC
{:toc}

## Venue

| Item | Value |
|---|---|
| Network | Hyperliquid **testnet** |
| REST | `{{ v.api_url }}/info` for queries. `{{ v.api_url }}/exchange` for signed actions. |
| WebSocket | `{{ v.ws_url }}` |
| Dex | `{{ v.dex }}` (dex index {{ v.dex_index }}) |
| Collateral | `{{ v.collateral }}` |
| Account mode | Unified (`unifiedAccount`). One `{{ v.collateral }}` balance is the margin for all positions. |

## Markets

Use the `meta` info query with the `dex` parameter set to `{{ v.dex }}`. Read `szDecimals` and `maxLeverage` at runtime; the table describes the current `{{ v.dex }}` markets.

| Coin | Price units | Lot | Max leverage | Asset ID |
|---|---|---|---|---|
| `{{ v.dex }}:BTC` | USD per BTC | 0.0001 | 40x | {{ base_id }} |
| `{{ v.dex }}:ETH` | USD per ETH | 0.0001 | 25x | {{ base_id | plus: 1 }} |
| `{{ v.dex }}:HYPE` | USD per HYPE | 0.01 | 10x | {{ base_id | plus: 2 }} |
| `{{ v.dex }}:SP500` | S&P 500 index points | 0.001 | 50x | {{ base_id | plus: 3 }} |
| `{{ v.dex }}:EWY` | USD per share of the iShares MSCI South Korea ETF | 0.001 | 20x | {{ base_id | plus: 4 }} |
| `{{ v.dex }}:EWJ` | USD per share of the iShares MSCI Japan ETF | 0.001 | 20x | {{ base_id | plus: 5 }} |
| `{{ v.dex }}:BRENT` | USD per barrel of Brent crude | 0.01 | 20x | {{ base_id | plus: 6 }} |
| `{{ v.dex }}:GOLD` | USD per troy ounce of gold | 0.0001 | 25x | {{ base_id | plus: 7 }} |
| `{{ v.dex }}:USDEUR` | **USD per EUR (EUR/USD)** | 0.1 | 50x | {{ base_id | plus: 8 }} |
| `{{ v.dex }}:USDJPY` | **JPY per USD (USD/JPY)** | 0.01 | 50x | {{ base_id | plus: 9 }} |

Despite its ticker, `USDEUR` tracks EUR/USD. All markets trade 24/7 and settle in `{{ v.collateral }}`, including the FX contracts.

## Asset references and pricing

The table combines the venue's oracle and mark specification with live `{{ v.dex }}` asset annotations (`perpAnnotation`) and market metadata, checked on 30 September 2026. Market specifications come from `meta`; oracle and mark prices come from `metaAndAssetCtxs`.

All ten assets use **SEDA Signal**, delivered through SEDA Fast. The oracle is the median of quotes returned by the configured sources: Binance, Binance Futures, Lighter and Hydromancer. **EWJ excludes Lighter.** Only sources returning a quote participate; the feed requires at least one source.

| Asset | Reference instrument and mainnet counterpart | Oracle computation | Mark methodology |
|---|---|---|---|
| `{{ v.dex }}:BTC` | Bitcoin; Hyperliquid `BTC` | `BTC/USD` feed, venue median | Median method below; EMA basis bound ±2.5% of oracle |
| `{{ v.dex }}:ETH` | Ethereum; Hyperliquid `ETH` | `ETH/USD` feed, venue median | Median method below; EMA basis bound ±4% of oracle |
| `{{ v.dex }}:HYPE` | Hyperliquid's HYPE token; Hyperliquid `HYPE` | `HYPE/USD` feed, venue median | Median method below; EMA basis bound ±10% of oracle |
| `{{ v.dex }}:SP500` | S&P 500 index; `xyz:SP500` | `SP500/USD` feed, venue median | Median method below; EMA basis bound ±2% of oracle |
| `{{ v.dex }}:EWY` | iShares MSCI South Korea ETF; `xyz:EWY` | `EWY/USD` feed, venue median | Median method below; EMA basis bound ±5% of oracle |
| `{{ v.dex }}:EWJ` | iShares MSCI Japan ETF; `xyz:EWJ` | `EWJ/USD` feed, venue median excluding Lighter | Median method below; EMA basis bound ±5% of oracle |
| `{{ v.dex }}:BRENT` | Brent crude; `xyz:BRENTOIL` | `BRENT/USD` feed, venue median | Median method below; EMA basis bound ±5% of oracle |
| `{{ v.dex }}:GOLD` | Gold (XAU); `xyz:GOLD` | `XAU/USD` feed, venue median | Median method below; EMA basis bound ±4% of oracle |
| `{{ v.dex }}:USDEUR` | EUR/USD: USD per EUR; `xyz:EUR` | `EUR/USD` feed, venue median | Median method below; EMA basis bound ±2% of oracle |
| `{{ v.dex }}:USDJPY` | USD/JPY: JPY per USD; `xyz:JPY` | `USD/JPY` feed, venue median | Median method below; EMA basis bound ±2% of oracle |

### Oracle and mark prices

The **oracle** is the SEDA composite submitted by the venue's authorized oracle relayers, rounded to the market's price precision. The upstream feed computation is part of the venue's operator specification. The public API returns the resulting `oraclePx`; it does not return the individual upstream quotes.

The **mark** is the price used for unrealized PnL, margin and liquidations. The venue computes it as follows:

1. The relayer maintains a **150-second exponential moving average (EMA)** of the local book mid price minus the oracle price, using the mid and oracle from the same on-chain snapshot. This is the book basis. The EMA starts at zero and decays toward zero when there is no two-sided book.
2. The basis adjustment is capped at **±oracle price / maximum leverage**. The table expresses this bound as a percentage of the oracle.
3. The relayer prepares two mark inputs: the oracle, and the oracle plus the capped basis. Each input is limited to a **±0.5% move from the previous on-chain mark** and rounded to the market's price precision.
4. HyperCore takes the **median of those two submitted inputs and the local book mark**. The local book mark is the median of the best bid, best ask and last trade price. Accepted live `{{ v.dex }}` oracle-update transactions confirm the two submitted input lists.

HyperCore also applies its own price clamps, including a 1% limit on each mark update. After 10 seconds without a mark update, the protocol falls back to the local book mark. See the [HIP-3 mark calculation](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/hip-3-deployer-actions).

Read `oraclePx` and `markPx` separately from `metaAndAssetCtxs`. A reference instrument identifies what the contract tracks; the published oracle, the local execution price and the mark can differ.

## Venue mechanics

| Item | Detail |
|---|---|
| Fees | At the undiscounted base tier, **0.09% taker and 0.03% maker**, charged in `{{ v.collateral }}` on each fill. This venue has `deployerFeeScale=1` and no growth mode, so positive base rates from `userFees` are doubled. The actual charge is in each fill's `fee` and `feeToken` fields. See the [fee formula](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/fees#fee-formula-for-developers) for tiers and discounts. |
| Funding | Paid every hour. A positive rate means longs pay shorts; a negative rate means shorts pay longs. `funding` in `metaAndAssetCtxs` is the hourly rate as a decimal. Payment magnitude is `abs(position size) × oracle price × abs(hourly rate)`. Get rates from `fundingHistory` and account payments from `userFunding`. |
| Oracle price | The SEDA Signal venue median described in [Asset references and pricing](#asset-references-and-pricing). Available as `oraclePx` in `metaAndAssetCtxs`. |
| Mark price | The median calculation and basis limits described in [Asset references and pricing](#asset-references-and-pricing). It can differ from the oracle. Available as `markPx` in `metaAndAssetCtxs`. |
| Margin | Cross positions share the unified collateral balance. Losses on one position reduce the margin available to others. Set leverage with `update_leverage`; check leverage and available buying power with `activeAssetData`. The spot balance's `total` is not free margin. |
| Liquidation | Triggered when collateral no longer covers maintenance margin. Check `liquidationPx` in each position and the account's margin state. See [unified account risk](https://hyperliquid.gitbook.io/hyperliquid-docs/trading/account-abstraction-modes#unified-account-ratio) for the maintenance-margin ratio. |

## Price and size rules

| Rule | Detail |
|---|---|
| Size | A multiple of the lot (`10^-szDecimals`). If necessary, decrease the size to a multiple of the lot. |
| Price | A maximum of 5 significant figures **and** a maximum of `6 − szDecimals` decimals. Integer prices are always correct. |
| Minimum value | $10 for each order (price × size) |
| Price band | Not more than 80% from the reference price |
| Format | Decimal strings. Remove trailing zeros from the fractional part and an empty decimal point; preserve integer zeros. The SDK handles conversion. |
| Open interest limit | $100,000 notional for each market, for all participants together. The API rejects orders that increase open interest above this limit. Get the values with `perpDexLimits`. |

## Orders and leverage

| Function | How to use |
|---|---|
| Good-till-cancel | Limit order with time in force `Gtc`; the unfilled remainder stays on the book. |
| Immediate-or-cancel | Limit order with time in force `Ioc`; available liquidity fills within the price limit and the remainder is cancelled. `market_open` and `market_close` use a default slippage limit of 5%. |
| Post-only | Limit order with time in force `Alo`. The API rejects it if it would immediately cross the book. |
| Stop or take-profit | Trigger order with a trigger price (`triggerPx`), market-or-limit execution (`isMarket`) and stop-loss or take-profit designation (`tpsl`: `sl` or `tp`). TP/SL orders must be reduce-only. Raw JSON prices are decimal strings; the SDK accepts numeric prices and serializes them. |
| Reduce-only | Set the reduce-only flag. The order may reduce an existing position but must not increase exposure. |
| Client order ID | A user-assigned 16-byte hexadecimal ID (`cloid`). The SDK represents it with `Cloid` from `hyperliquid.utils.types`; cancellation by this ID uses `cancel_by_cloid`. |
| Change an order | `modify_order` replaces an order's parameters, identifying it by order ID or client order ID. |
| Batch | `bulk_orders` and `bulk_cancel` submit several operations together. Each operation counts against the account budget. |
| Leverage | `update_leverage` selects leverage and cross or isolated margin for a coin. Leverage must not exceed the market's maximum. |
| Dead man's switch | `schedule_cancel` schedules cancellation of all open orders at a future timestamp, at least 5 seconds ahead. Up to 10 triggers per day, resetting at 00:00 UTC. Requires $1,000,000 of account trade volume. |

The request-level `status` can be `ok` even when an order is rejected. Inspect each result in `response.data.statuses`:

- `resting.oid`: the order ID of the remaining order on the book.
- `filled.totalSz`, `filled.avgPx`, `filled.oid`: the executed size, average fill price and order ID. An IOC can fill partially; `totalSz` may be smaller than the requested size. Its unfilled remainder is cancelled.
- `error`: the reason the API rejected the order.

If the full request fails, `status` is `err` and `response` contains the reason.

The SDK's `market_close` submits a reduce-only IOC. It can partially fill or be rejected; read the position again to confirm the remaining size. It returns no result if it finds no position for the coin.

## Rate limits

**Each IP address:** a maximum weight of 1200 each minute, shared by all clients using that IP.

| Request | Weight |
|---|---|
| Exchange action | 1 + floor(orders in the batch / 40) |
| `l2Book`, `allMids`, `clearinghouseState`, `orderStatus`, `spotClearinghouseState`, `exchangeStatus` | 2 |
| `userRole` | 60 |
| Other info requests | 20 |

History queries also have response-size charges: `recentTrades`, `userFills`, `userFillsByTime`, `fundingHistory` and `userFunding` add weight per 20 returned items; `candleSnapshot` adds weight per 60 returned items. See the [full rate-limit specification](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits) for all endpoints.

**Each account:**

- Each order, cancel and change uses one request from the account budget.
- The budget starts at **10,000**. It increases by **1 for each $1 that you trade**.
- When the budget is empty, you can send one request each 10 seconds.
- The cancel limit is `min(budget + 100,000, 2 × budget)`.
- Get the status with `userRateLimit` (`nRequestsUsed`, `nRequestsCap`).
- In unified mode, the account can send a maximum of 50,000 actions each day.

**Other limits:**

- 1000 open orders.
- WebSocket: 10 connections for each IP address, 1000 subscriptions and 2000 sent messages each minute.

**Nonces:**

- Use the current time in ms.
- Do not use the same nonce two times with one key.
- The nonce must be between 2 days before and 1 day after the server time.
- Hyperliquid keeps the 100 highest nonces for each key. A new nonce must be higher than the lowest of these.
- Use one API wallet for each process.
- Keep your clock correct (NTP).

## Info queries

Send `POST {{ v.api_url }}/info` with a JSON body. The `user` is always the **team wallet** address.

| `type` | Parameters | Result |
|---|---|---|
| `meta` | `dex` | Markets: name, szDecimals, maxLeverage |
| `metaAndAssetCtxs` | `dex` | Markets, mark, oracle, mid, funding, open interest, 24h volume |
| `allMids` | `dex` | Mid price for each coin |
| `l2Book` | `coin` | Order book, maximum 20 levels on each side |
| `recentTrades` | `coin` | Latest public trades |
| `candleSnapshot` | `req` object containing `coin`, `interval`, `startTime`, `endTime` | OHLCV, maximum 5000 candles, `1m` to `1M` |
| `fundingHistory` | `coin, startTime, endTime?` | Funding rates for each hour |
| `perpDexs` | | Dex list and funding parameters |
| `perpDexLimits` | `dex` | Open interest limits |
| `spotClearinghouseState` | `user` | Balance. In unified mode, this is the balance for all positions. |
| `clearinghouseState` | `user, dex` | Positions on the dex |
| `openOrders` | `user, dex` | Open orders |
| `orderStatus` | `user, oid` (or cloid) | Status of one order |
| `userFills` / `userFillsByTime` | `user` (`startTime`, `endTime`) | Fills |
| `userFunding` | `user, startTime, endTime?` | Funding payments |
| `activeAssetData` | `user, coin` | Leverage, margin mode, maximum order sizes |
| `userFees` | `user` | Base fee rates |
| `userRateLimit` | `user` | Account budget |
| `userAbstraction` | `user` | Account mode |
| `extraAgents` | `user` | Approved API wallets |

## WebSocket

Connect to `{{ v.ws_url }}`. Use the `subscribe` method with a `subscription` object identifying the channel type and its parameters. Send the `ping` method at intervals of 60 seconds or less.

| `type` | Parameters | Data |
|---|---|---|
| `l2Book` | `coin` | Order book |
| `bbo` | `coin` | Best bid and offer |
| `trades` | `coin` | Public trades |
| `candle` | `coin, interval` | Candles |
| `activeAssetCtx` | `coin` | Mark, oracle, funding, open interest |
| `allMids` | `dex` | Mid prices |
| `userFills` | `user` | Your fills |
| `orderUpdates` | `user` | Changes to the status of your orders |
| `userEvents` | `user` | Fills, funding, liquidations |
| `userFundings` | `user` | Your funding payments |

- The `coin` has the dex prefix (`{{ v.dex }}:BTC`). The `user` is the team wallet address.
- The first `userFills` and `trades` messages contain recent history. Reconcile that history with events already processed. For public trades, identify duplicates by block time (`time`), `coin` and `tid` together; `tid` alone is not globally unique.
- The Python SDK does not send data for all channels. If a channel sends no data, use a raw WebSocket client.

## Errors

Order errors may end with `asset=<asset ID>`. This ID shows the market.

| Error | Action |
|---|---|
| `Must deposit before performing actions.` | The team wallet has no funds. Ask in `#general` on Discord. |
| `User or API Wallet 0x… does not exist.` | Check the approval with `extraAgents` and use the testnet URL. If the approval expired or was removed, authorize a fresh API wallet through the team wallet. |
| `KeyError: 'BTC'` (Python SDK) | Use `{{ v.dex }}:BTC`. |
| `Order must have minimum value of $10.` | Increase the size. |
| `Order has invalid price.` | Decrease the number of decimals. Refer to the [rules](#price-and-size-rules). |
| `Price must be divisible by tick size.` | Use a maximum of 5 significant figures. |
| `Order has invalid size.` | Decrease the size to a multiple of the lot. |
| `Order price cannot be more than 80% away from the reference price` | Move the price nearer to the market price. |
| `Post only order would have immediately matched, bbo was <bid>@<ask>.` | The `Alo` price crosses the book. Move the price away from the other side. |
| `Order could not immediately match against any resting orders.` | No order is available at your `Ioc` price. Move the price limit further into the book. |
| `Insufficient margin to place order.` | Decrease the size, or increase the leverage. |
| `Reduce only order would increase position.` | Make sure that the side and the size are correct. |
| `Invalid leverage value` | Use a leverage that is not more than the [max leverage](#markets). |
| `Cannot set scheduled cancel time until enough volume traded. Required: $1000000.` | `schedule_cancel` is available after a trade volume of $1,000,000. |
| `Too many cumulative requests sent` | The account budget is empty. Refer to [Rate limits](#rate-limits). |
| An error about the `nonce` | Use one API wallet for each process. Make sure that your clock is correct. |
| The balance is zero | Get the balance with `spotClearinghouseState`. Use the team wallet address. |
| No positions or orders | Use the team wallet address and set the `dex` parameter to `{{ v.dex }}`. |
| The prices are not correct | You read a native market. Add the dex prefix to the coin, and add `dex`. |

</div>
