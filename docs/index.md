---
layout: default
title: Participant Guide
permalink: /docs/
---
{% assign v = site.data.venue %}
<div class="page-content docs" markdown="1">

# Participant Guide
{: .no_toc}

For markets, order types, limits and errors, refer to the **[Reference](reference/)**.

* TOC
{:toc}

## 1. Venue

| Item | Value |
|---|---|
| Network | Hyperliquid **testnet** |
| REST | `{{ v.api_url }}` |
| WebSocket | `{{ v.ws_url }}` |
| Dex | `{{ v.dex }}` |
| Coins | The dex name, a colon and the coin: `{{ v.dex }}:BTC`, `{{ v.dex }}:ETH`, … |
| Collateral | `{{ v.collateral }}` |

<div class="callout warn" markdown="1">
**WARNING:** Set the `dex` parameter to `{{ v.dex }}` in market-wide and position queries. Use the dex prefix on coin names. Without this selection, raw API queries default to the native Hyperliquid markets and may return data without an error.
</div>

## 2. Setup

Python users need Python 3.10 or later and the official [Hyperliquid Python SDK](https://github.com/hyperliquid-dex/hyperliquid-python-sdk), version **{{ v.sdk_version }}**. The SDK handles asset IDs, serialization and request signing. Configure it for the testnet endpoint listed above.

For other languages, refer to [Other languages](#other-languages).

## 3. Team wallet

The team wallet is your account. The organizers fund it. Use its address for all account queries; these queries do not require a private key.

Create and manage the wallet with your own wallet tooling. Give only its public address to the organizers. Keep the team wallet's private key outside the bot and never share it with organizers or support.

## 4. Set up the account

Do this step one time, after the organizers fund the team wallet.

Your bot signs trading actions with an **API wallet**, a separate key authorized by the team wallet. It can trade, modify and cancel orders, and set leverage. It cannot withdraw or send funds to another address. A compromised API wallet can still damage the account through trading.

| Operation | API action or query | Python SDK | Signing wallet |
|---|---|---|---|
| Approve an API wallet | `approveAgent` | `approve_agent` | Team wallet |
| Set unified mode | `userSetAbstraction`, account mode `unifiedAccount` | `user_set_abstraction` | Team wallet |
| Check approved API wallets | `extraAgents` | `Info.post` | None |
| Check account mode | `userAbstraction` | `query_user_abstraction_state` | None |
| Check collateral balance | `spotClearinghouseState` | `spot_user_state` | None |

The approval identifies the API wallet's public address, name and expiry. The SDK's `approve_agent` generates a new API wallet key and returns it with the approval response. Handle private keys through your own credential management; exclude them from source code, logs and shared files. Check the approval response and `extraAgents` before trading. Account administration is signed by the team wallet; bot trading is signed by the approved API wallet.

In unified mode, one `{{ v.collateral }}` balance backs cross-margin positions and is shared with spot. You do not move funds between spot and the dex. If the balance is zero, ask in `#general` on Discord using only the public account address.

- Use a separate API wallet for each process to avoid nonce collisions.
- An account supports up to 3 named API wallets. Approving a new wallet under an existing name replaces the previous approval.
- Set an expiry for the approval. Replace compromised or expired wallets with fresh keys; never reuse a removed API wallet's key because its nonce history may have been pruned.
- Use the team wallet address for queries. The API wallet address has no balance, positions or orders for your account.
- Hyperliquid may remove an API wallet when it expires or the account no longer has funds. See [API wallet lifecycle](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets).

## 5. Client

The Python SDK separates reads and signed actions:

| Client | Purpose | Configuration |
|---|---|---|
| `Info` | Market data and account queries | Testnet API URL; select `{{ v.dex }}` through `perp_dexs`. Enable its WebSocket manager if using subscriptions. |
| `Exchange` | Signed trading actions | Testnet API URL; approved API wallet signer; team wallet address as `account_address`; select `{{ v.dex }}` through `perp_dexs`. |

Selecting only this dex means plain native coin names are absent from the SDK's market mapping. Use the prefixed coin names listed in the reference.

Inspect both the request status and the result of every order. A request-level status of `ok` can contain rejected orders in `response.data.statuses`. Batch results correspond to individual operations; some may succeed while others fail.

## 6. Trade

Each order must have:

- A size that is a multiple of the lot.
- A price with a maximum of 5 significant figures and a maximum of `6 − szDecimals` decimals. Integer prices are allowed regardless of the number of significant figures.
- A value of $10 or more.

Refer to the [price and size rules](reference/#price-and-size-rules).

Read the market specifications before constructing an order. An order identifies the market, side, quantity, limit price, time in force and whether it is reduce-only. Set leverage for the market explicitly. Track the returned order ID or your client order ID to reconcile updates, fills and cancellations.

| Operation | Python SDK method | Required selection or inputs |
|---|---|---|
| Market specifications | `meta` | Dex |
| Mark, oracle, funding and open interest | `Info.post`, query type `metaAndAssetCtxs` | Dex |
| Mid prices | `all_mids` | Dex |
| Order book | `l2_snapshot` | Coin |
| Candles | `candles_snapshot` | Coin, interval and time range |
| Collateral balance | `spot_user_state` | Team wallet address |
| Positions | `user_state` | Team wallet address and dex; positions are in `assetPositions` |
| Open orders | `open_orders` | Team wallet address and dex |
| Fills | `user_fills` | Team wallet address |
| Leverage | `update_leverage` | Coin, leverage and margin mode |
| Limit or post-only order | `order` | Coin, side, size, price and time in force (`Gtc` or `Alo`) |
| Market execution | `order` or `market_open` | IOC time in force and a price limit; the helper uses a slippage limit |
| Modify | `modify_order` | Order ID and replacement order parameters |
| Cancel | `cancel` | Coin and order ID |
| Close a position | `market_close` | Coin; optional size and slippage limit |

For the underlying instruments, oracles and mark calculation, refer to [Asset references and pricing](reference/#asset-references-and-pricing). For other order types, batch orders and `cloid`, refer to the [Reference](reference/#orders-and-leverage). For fees, funding and margin, refer to [Venue mechanics](reference/#venue-mechanics).

## 7. Streams

Connect to the WebSocket endpoint and subscribe to the channels your algorithm needs. Market channels identify a prefixed coin; account channels identify the team wallet address. The SDK's `subscribe` method associates a subscription with your message handler.

- In the raw protocol, use the `subscribe` method with a subscription specifying the channel type and its parameters.
- Send the `ping` method at intervals of 60 seconds or less. The Python SDK's WebSocket manager sends these heartbeats automatically.
- If the connection stops, do these steps:
  1. Connect again.
  2. Subscribe again.
  3. Read the account state again through REST.
- The first `userFills` and `trades` messages contain recent history. Reconcile that history with events already processed. For public trades, identify duplicates by block time (`time`), `coin` and `tid` together; `tid` alone is not globally unique.

For all channels, refer to the [Reference](reference/#websocket).

## 8. Other languages

- **TypeScript:** [`@nktkas/hyperliquid`](https://github.com/nktkas/hyperliquid)
- **Rust:** [`hyperliquid_rust_sdk`](https://github.com/hyperliquid-dex/hyperliquid-rust-sdk)
- **Raw API:** [Hyperliquid API documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api), [signatures](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/signing)

Use the **testnet** URLs and the dex `{{ v.dex }}`.

When you sign actions without an SDK:

- Set unified mode through `userSetAbstraction`, using the account mode `unifiedAccount` and a team-wallet signature. The agent-signed equivalent is `agentSetAbstraction`, using mode `u`.
- Send `POST {{ v.api_url }}/exchange` with a JSON object containing `action` (object), `nonce` (integer timestamp in ms), and `signature` (object with hex strings `r` and `s`, and integer `v`).
- The asset ID `a` is `100000 + 10000 × {{ v.dex_index }} + the index in meta`. For `{{ v.dex }}:BTC`, the asset ID is {{ v.dex_index | times: 10000 | plus: 100000 }}.
- Send prices and sizes as decimal strings. Remove trailing zeros only from the fractional part and remove an empty decimal point. Preserve integer zeros. The Python SDK handles this conversion.
- Use the current time in ms as the `nonce`. Do not use the same nonce two times with one API wallet.

## 9. Questions

Ask in `#general` on Discord. Do not post a private key.

</div>
