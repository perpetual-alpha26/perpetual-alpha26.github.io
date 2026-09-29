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

Lookup material for the [Participant Guide](../). Values below are for the **{{ v.phase }}** venue.

* TOC
{:toc}

## Venue

| Setting | Value |
|---|---|
| Network | Hyperliquid **testnet** |
| REST | `{{ v.api_url }}/info` (queries), `{{ v.api_url }}/exchange` (signed actions) |
| WebSocket | `{{ v.ws_url }}` |
| Dex | `{{ v.dex }}` (dex index {{ v.dex_index }}) |
| Collateral | `{{ v.collateral }}`. Full id for transfers: `{{ v.collateral_wire }}`. Token names aren't unique; the id is. |
| Margin | Cross or isolated, your choice per market |
| Explorer | `{{ v.explorer }}<address>` |

## Markets

Specs come from the chain. Read them at runtime with `{"type": "meta", "dex": "{{ v.dex }}"}` rather than hard-coding them.

Each market copies a live Hyperliquid **mainnet** market: its lot size, max leverage and funding parameters. That mainnet market's history (candles, funding, trades) is a close proxy for backtesting.

| Coin | Underlying | Price is quoted as | Lot | Max leverage | Copies (mainnet) | Asset id |
|---|---|---|---|---|---|---|
| `{{ v.dex }}:BTC` | Bitcoin | USD per BTC | 0.0001 | 40x | `BTC` | {{ base_id }} |
| `{{ v.dex }}:ETH` | Ether | USD per ETH | 0.0001 | 25x | `ETH` | {{ base_id | plus: 1 }} |
| `{{ v.dex }}:HYPE` | Hyperliquid token | USD per HYPE | 0.01 | 10x | `HYPE` | {{ base_id | plus: 2 }} |
| `{{ v.dex }}:SP500` | S&P 500 index | index points | 0.001 | 50x | `xyz:SP500` | {{ base_id | plus: 3 }} |
| `{{ v.dex }}:EWY` | iShares MSCI South Korea ETF | USD per share | 0.001 | 20x | `xyz:EWY` | {{ base_id | plus: 4 }} |
| `{{ v.dex }}:EWJ` | iShares MSCI Japan ETF | USD per share | 0.001 | 20x | `xyz:EWJ` | {{ base_id | plus: 5 }} |
| `{{ v.dex }}:BRENT` | Brent crude oil | USD per barrel | 0.01 | 20x | `xyz:BRENTOIL` | {{ base_id | plus: 6 }} |
| `{{ v.dex }}:GOLD` | Gold | USD per troy ounce | 0.0001 | 25x | `xyz:GOLD` | {{ base_id | plus: 7 }} |
| `{{ v.dex }}:USDEUR` | Euro vs US dollar | **USD per 1 EUR** (≈ 1.1, the EUR/USD rate) | 0.1 | 50x | `xyz:EUR` | {{ base_id | plus: 8 }} |
| `{{ v.dex }}:USDJPY` | US dollar vs yen | **JPY per 1 USD** (≈ 150) | 0.01 | 50x | `xyz:JPY` | {{ base_id | plus: 9 }} |

Frontends show a few of these under display names: `S&P500`, `BRENTOIL`, `EURUSD`. The API always uses the coin names above.

On the scored venue, BTC's lot will be 0.00001, matching mainnet. The practice lot can't be changed after registration.

<div class="callout warn" markdown="1">
**The two FX markets are quoted in opposite directions.** Despite its name, `USDEUR` is priced as EUR/USD, in dollars per euro. `USDJPY` is priced in yen per dollar. Check the price, not the name, before you build a cross-asset signal.
</div>

**Trading hours.** Every market trades 24/7, and so does every oracle, including the stock, index, FX and commodity markets. There is no closed-market session. See [oracle and mark price](#oracle-and-mark-price).

## Price and size rules

| Rule | Detail |
|---|---|
| Size | A multiple of the lot, `10^-szDecimals`. Round **down**. |
| Price | At most 5 significant figures, **and** at most `6 − szDecimals` decimals. Integer prices are always valid. |
| Minimum value | $10 per order (price × size) |
| Price band | No more than 80% away from the reference price |
| Wire format | Prices and sizes are sent as strings with no trailing zeros: `"2587.3"`, not `"2587.30"` |

Examples with `{{ v.dex }}:ETH` (szDecimals 4, so at most 2 price decimals):

- `2587.3` ✓
- `2587.35` ✗ (6 significant figures)
- `2587` ✓

## Orders

| Feature | How |
|---|---|
| Limit, good-till-cancel | `{"limit": {"tif": "Gtc"}}` |
| Immediate-or-cancel | `{"limit": {"tif": "Ioc"}}`. This is also how you send a "market" order: IOC with a price cap. `exchange.market_open` / `market_close` do it for you with 5% slippage. |
| Post-only | `{"limit": {"tif": "Alo"}}`. Rejected if it would cross the book. |
| Stop / take-profit | `{"trigger": {"triggerPx": 2500, "isMarket": true, "tpsl": "sl"}}` (or `"tp"`) |
| Reduce-only | `reduce_only=True`. Rejected if it would increase your position. |
| Client order id | `cloid=Cloid.from_str("0x…32 hex chars…")`. Cancel with `cancel_by_cloid`. |
| Modify | `exchange.modify_order(oid, coin, is_buy, sz, px, order_type)`. Cheaper on rate limits than cancel + new. |
| Batch | `exchange.bulk_orders([...])`, `bulk_cancel([...])`. One HTTP request, but each order counts toward the account budget. |
| Dead man's switch | `exchange.schedule_cancel(ms)` cancels all orders at `ms` (at least 5 s ahead). `None` clears it. At most 10 triggers per day (reset at 00:00 UTC). **Only available once your account has traded $1,000,000.** |

### Responses

`POST /exchange` replies `{"status": "ok", ...}` even when an individual order is rejected. Always inspect `statuses`:

```json
{"status": "ok", "response": {"type": "order", "data": {"statuses": [
  {"resting": {"oid": 123456}},
  {"filled": {"totalSz": "0.0057", "avgPx": "2640.1", "oid": 123457}},
  {"error": "Order must have minimum value of $10."}
]}}}
```

A request that fails as a whole returns `{"status": "err", "response": "<reason>"}`.

## Margin, funding, fees

**Margin.**

- Cross or isolated, chosen per market with `update_leverage(n, coin, is_cross=True|False)`, up to that market's max leverage. The SDK defaults to cross.
- Until you set it, each market starts at **cross, 20x**, or the market's max leverage if that is lower (10x on HYPE). Set leverage explicitly before you trade. Check the current setting with `{"type": "activeAssetData", "user": "0x…", "coin": "{{ v.dex }}:BTC"}`.
- **Cross:** your positions on `{{ v.dex }}` share your dex balance as margin. A loss on one eats into the margin of the others.
- **Isolated:** each position has its own margin, which you can add to or remove with `update_isolated_margin`.
- A position is liquidated when its margin falls below maintenance margin, which is half the initial margin at max leverage. For BTC at 40x that is 1.25% of notional.
- `liquidationPx` is in `clearinghouseState.assetPositions`.

**Open interest caps.** Each market's total open interest, across all participants, is capped at **$100,000** notional. Orders that would increase open interest past the cap are rejected. Live values: `{"type": "perpDexLimits", "dex": "{{ v.dex }}"}`.

**Funding.**

- Paid **every hour**, between longs and shorts. A positive rate means longs pay shorts.
- Hyperliquid's builder-dex formula, per 8 hours: `multiplier × (P + clamp(interest − P, ±clamp))`. Each hour pays one eighth.
- `P` is the premium: the mid of the book's impact prices against the oracle. It is measured against the oracle, not the mark.
- Payment = position size × oracle price × hourly rate.
- Parameters copy each market's mainnet counterpart:

| Markets | Multiplier | Interest (per 8 h) | Clamp (per 8 h) |
|---|---|---|---|
| BTC, ETH, HYPE | 1 | 0.01% | ±0.05% |
| SP500, EWY, EWJ, BRENT, GOLD | 0.5 | 0.01% | ±0.03% |
| USDEUR, USDJPY | 0.5 | 0 | ±0.03% |

- Current rate: `funding` in `metaAndAssetCtxs`. Parameters: `perpDexs`. History: `fundingHistory`. Your own payments: `userFunding`.

**Fees.** Charged in `{{ v.collateral }}` on every fill, at Hyperliquid's default rates for a builder dex: twice the base tier, so **0.09% taker and 0.03% maker**. `userFees` reports the base tier (0.045% / 0.015%); the fee actually charged is in the `fee` field of each fill.

## Oracle and mark price

**Oracle price: 24/7, never from our book.**

- Every market's oracle is a [SEDA](https://seda.xyz) composite: a median across external venues.
- Those venues trade around the clock, for stocks, indexes, FX and commodities as well as crypto. There is no closed-market session.
- Our own order book never sets the oracle: it is thin and driven by the competition.
- It is pushed on-chain about every 3.5 seconds.

**Mark price: used for PnL, margin and liquidations.** It is the median of three inputs:

```
mark = median( oracle,
               oracle + 150-second average of (book mid − oracle),   capped at oracle ± 1/maxLeverage
               median(best bid, best ask, last trade) )
```

- Each input moves at most 0.5% per update.
- The mark only follows the book when both the live book and its last 150 seconds agree. A single order can't move it.
- It never leaves oracle ± 1/maxLeverage: ±2% on a 50x market, ±5% on a 20x one.
- With no two-sided book, the average decays to zero and the mark equals the oracle.
- If oracle updates ever stop for more than 10 seconds, the mark falls back to the order book.

**Final scoring** marks positions to the **oracle** price at the end of the Live Trading phase.

## Rate limits

**Per IP: 1200 weight per minute**, shared by everyone behind the same IP, such as a university network.

| Request | Weight |
|---|---|
| Any exchange action | 1 + floor(orders in batch / 40) |
| `l2Book`, `allMids`, `clearinghouseState`, `orderStatus`, `spotClearinghouseState` | 2 |
| `userRole` | 60 |
| Most other info requests | 20 |

**Per account: an action budget.** This one catches people out.

- Every order, cancel and modify spends one request from a budget that starts at **10,000**.
- The budget grows by **1 per dollar of volume you trade**, cumulatively. Volume in `{{ v.collateral }}` on this dex counts.
- When it's spent, you get **one request every 10 seconds**.
- Cancels have extra headroom: `min(budget + 100,000, 2 × budget)`.
- Check where you stand with `{"type": "userRateLimit", "user": "0x…"}`, which returns `nRequestsUsed` and `nRequestsCap`.

Quoting 10 markets on both sides and replacing every second uses about 20 requests per second, which spends 10,000 in under 10 minutes if nothing fills. To stay inside the budget:

- modify instead of cancel-and-replace;
- requote only when your price actually changes;
- trade.

**Other limits:**

- 1000 open orders;
- WebSocket: 10 connections per IP, 1000 subscriptions, 2000 messages per minute sent.

**Nonces.**

- Each action carries a millisecond timestamp nonce, unique per signing key.
- It must be within 2 days behind to 1 day ahead of server time.
- Hyperliquid keeps each key's 100 highest nonces. A new nonce must beat the lowest of them and never repeat.
- In practice: **one API wallet per process** and a synced clock.

## Info queries

`POST {{ v.api_url }}/info` with a JSON body. Every `user` must be the **team wallet** address.

| `type` | Parameters | Returns |
|---|---|---|
| `meta` | `dex` | Markets: name, szDecimals, maxLeverage |
| `metaAndAssetCtxs` | `dex` | The above, plus mark, oracle, mid, funding, open interest, 24h volume |
| `allMids` | `dex` | Mid price per coin |
| `l2Book` | `coin` | Order book, up to 20 levels per side |
| `recentTrades` | `coin` | Latest public trades |
| `candleSnapshot` | `req: {coin, interval, startTime, endTime}` | OHLCV, up to 5000 candles. Intervals `1m` … `1M`. |
| `fundingHistory` | `coin, startTime, endTime?` | Hourly funding rates |
| `perpDexLimits` | `dex` | Open interest caps |
| `clearinghouseState` | `user, dex` | Account value, margin, positions |
| `spotClearinghouseState` | `user` | Spot balances |
| `openOrders` | `user, dex` | Your resting orders |
| `orderStatus` | `user, oid` (or cloid) | One order's status |
| `userFills` / `userFillsByTime` | `user` (`startTime`, `endTime`) | Your fills |
| `userFunding` | `user, startTime, endTime?` | Your funding payments |
| `activeAssetData` | `user, coin` | Your leverage and margin mode on that market, max order sizes |
| `userFees` | `user` | Your base-tier fee rates (this dex charges twice these) |
| `userRateLimit` | `user` | Your action budget |
| `extraAgents` | `user` | Your approved API wallets |

## WebSocket

Send `{"method": "subscribe", "subscription": {...}}` to `{{ v.ws_url }}`, and `{"method": "ping"}` at least every 60 s.

| `type` | Parameters | Streams |
|---|---|---|
| `l2Book` | `coin` | Order book snapshots |
| `bbo` | `coin` | Best bid and offer |
| `trades` | `coin` | Public trades |
| `candle` | `coin, interval` | Candles |
| `activeAssetCtx` | `coin` | Mark, oracle, funding, open interest |
| `allMids` | `dex` | Mids for all our markets |
| `userFills` | `user` | Your fills |
| `orderUpdates` | `user` | Your order status changes |
| `userEvents` | `user` | Fills, funding, liquidations |
| `userFundings` | `user` | Your funding payments |

`coin` is always prefixed (`{{ v.dex }}:BTC`) and `user` is always the team wallet. Two notes:

- `webData2`, which appears in some examples, isn't available on testnet.
- Tested through the Python SDK: `l2Book`, `bbo`, `trades`, `userFills`, `orderUpdates`. The SDK doesn't deliver every channel; if another subscription stays silent there, use a raw WebSocket client.
- The first `userFills` and `trades` messages are snapshots of recent history. De-duplicate by `tid`.

## Errors

These are the exact messages, captured on `{{ v.dex }}`. Order errors end with `asset=<asset id>`, which tells you which market.

| You see | Cause | Fix |
|---|---|---|
| `User or API Wallet 0x… does not exist.` | API wallet not approved, expired, replaced or removed; or you're signing for mainnet | Re-run `approve_api_wallet.py`; check you use the testnet URL |
| `KeyError: 'BTC'` (Python SDK) | Plain coin name | Use `{{ v.dex }}:BTC` |
| `Order must have minimum value of $10.` | price × size < 10 | Increase size |
| `Order has invalid price.` | Too many decimals in the price | Round the price (see [rules](#price-and-size-rules)) |
| `Price must be divisible by tick size.` | More than 5 significant figures | Round the price to 5 significant figures |
| `Order has invalid size.` | Size not a multiple of the lot | Round the size down to `szDecimals` |
| `Order price cannot be more than 80% away from the reference price` | Price far from the market | Price closer to the oracle |
| `Post only order would have immediately matched, bbo was <bid>@<ask>.` | Your `Alo` price crosses the book | Move the price away from the other side |
| `Order could not immediately match against any resting orders.` | `Ioc` with nothing to fill at your price | Check the book, widen your cap |
| `Insufficient margin to place order.` | Not enough free balance for size ÷ leverage | Smaller size, higher leverage, or add margin |
| `Reduce only order would increase position.` | Reduce-only with no position, on the wrong side, or bigger than the position | Check side and size |
| `Invalid leverage value` | Leverage above the market's max | See max leverage in the [markets](#markets) table |
| `Cannot set scheduled cancel time until enough volume traded. Required: $1000000.` | `schedule_cancel` before $1M of volume | Handle cleanup in your own code (see the guide, step 9) |
| `Too many cumulative requests sent` | Account action budget spent | Slow down; see [rate limits](#rate-limits) |
| Error mentioning `nonce` | Two processes share an API wallet, or your clock is off | One API wallet per process; sync your clock |
| Queries return empty / zero | Querying the API wallet's address, or forgot `dex` | Use the team wallet address and `dex: "{{ v.dex }}"` |
| Prices look nothing like expected | Reading Hyperliquid's native market | Prefix the coin, pass `dex` |

## Practice vs scored

| | Practice | Scored |
|---|---|---|
| Dates | 5–18 October | 19 October – 6 November |
| Dex | `par` | To be announced |
| Collateral | `USDPR` | To be announced |
| Counts toward ranking | No | Yes |
| Team wallet | Same | Same |
| API wallets | Same (they belong to your account, not to a dex) | Same |
| What to change in your code | | `DEX`, and the token id if you move funds |

## Glossary

**API wallet** (also *agent*)
: A key pair your team wallet authorizes to trade on its behalf. It can't withdraw or transfer.

**Builder dex / HIP-3**
: An independent perpetuals exchange deployed on Hyperliquid's engine, with its own markets, collateral and oracle. `{{ v.dex }}` is one.

**cloid**
: Client order id, 16 bytes of hex that you choose.

**Cross margin**
: All your positions on the dex share your dex balance as margin.

**Funding**
: Hourly payment between longs and shorts that keeps the perp price close to the oracle.

**HyperCore**
: Hyperliquid's on-chain exchange engine: order books, margin, the API you trade on.

**Isolated margin**
: Each position has its own margin; a loss on one can't eat another's.

**Mark price**
: The price used for PnL, margin and liquidations.

**Nonce**
: Millisecond timestamp on every signed action, unique per signing key.

**Open interest (OI)**
: Total size of open positions in a market.

**Oracle price**
: The external reference price (from SEDA) that the market is anchored to.

**szDecimals**
: Number of decimals allowed in an order size; the lot is `10^-szDecimals`.

**Team wallet**
: The address you registered: your account, your balance, your score.

**TIF**
: Time in force: `Gtc`, `Ioc` or `Alo` (post-only).

</div>
