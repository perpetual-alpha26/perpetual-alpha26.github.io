---
layout: default
title: Participant Guide
permalink: /docs/
---
{% assign v = site.data.venue %}
<div class="page-content docs" markdown="1">

# Participant Guide
{: .no_toc}

This guide takes you from *"we have a wallet and it's funded"* to *"our code is trading"*, in about 30 minutes. Everything happens in a terminal. You never need to open a website or connect your wallet to anything.

<div class="callout" markdown="1">
**Current venue: {{ v.phase }} ({{ v.phase_dates }}).** Dex `{{ v.dex }}`, collateral `{{ v.collateral }}`. The scored phase starts on 19 October on a separate venue. Its values will be announced on Discord and updated on this page. Your wallet and API wallet carry over.
</div>

Looking something up? See the **[Reference](reference/)**: markets, order types, limits, errors.

* TOC
{:toc}

## 1. Five-minute orientation

### What Hyperliquid is

Hyperliquid is an exchange that runs as its own blockchain. From your code's point of view it behaves like any exchange API:

- a central limit order book;
- REST for requests and WebSocket for streams;
- orders confirmed in well under a second.

There is no gas to pay per order and no smart contract to call.

The competition runs on the **Hyperliquid testnet**. It uses the same software and API as the real exchange, but its tokens have no value.

### What you trade on

On that testnet we run our own venue, a **dex** called `{{ v.dex }}`. It shares Hyperliquid's matching engine but has its own markets, its own collateral token (`{{ v.collateral }}`) and its own margin accounts.

The books are quoted by our market-making partner and traded by all the teams.

Hyperliquid's own markets (plain `BTC`, `ETH`, …) sit next to it on the same API. Ignore them: they don't count, and your funds can't reach them.

```
Hyperliquid testnet API  ({{ v.api_url }})
├── native markets   BTC, ETH, SOL, …                  ← not the competition, ignore
└── dex "{{ v.dex }}"        {{ v.dex }}:BTC, {{ v.dex }}:ETH, … {{ v.dex }}:USDJPY    ← the competition (10 markets)
      collateral: {{ v.collateral }}
```

### Translation table

If you have run bots on a centralized exchange, this is what the familiar pieces are called here:

| You know | On Hyperliquid |
|---|---|
| Account, login | Your **team wallet** address. No sign-up; it exists once it has received funds. |
| API key + secret | An **API wallet**: a second key pair that your team wallet authorizes once. Every request is signed with its private key. Nothing secret is sent to the exchange. |
| API key permissions | An API wallet can place, modify and cancel orders and set leverage. It **cannot** withdraw or send funds to another address. |
| Exchange / venue | The dex `{{ v.dex }}` |
| Symbol `BTCUSDT` | `{{ v.dex }}:BTC` by name. Orders carry a numeric **asset id** that the SDK looks up for you. |
| Margin currency | `{{ v.collateral }}`, a test token. 1 `{{ v.collateral }}` counts as $1. |
| Spot wallet / futures wallet | **Spot balance** and **dex balance**. Only the dex balance can be traded. |
| `timestamp` / `recvWindow` | `nonce`: the current time in milliseconds, unique per API wallet |
| Funding every 8 hours | Funding **every hour** |
| Cross / isolated margin | Both, chosen per market. Cross shares your dex balance across all your positions. |
| Market order | An IOC limit order with a price cap |
| Post-only | Time-in-force `Alo` ("add liquidity only") |
| Client order id | `cloid`, a 16-byte hex string |
| Rate limit per IP | Per IP **and** a per-account action budget. See [limits](reference/#rate-limits). |

### What you don't need

- no sign-up or KYC;
- no deposit or bridge;
- no ETH for gas, and no faucet;
- no browser wallet connection.

We fund your wallet directly. If anyone asks you to do any of the above, it isn't us.

## 2. Before you start

You need:

1. **The wallet address you registered, and its private key.** You use the private key only on your own machine: in step 4 to authorize a separate key for your bot, and in step 3 if your funds need moving.
   - In MetaMask: open the account menu, then *Account details → Show private key*. The wording varies a little between versions; other wallets have an equivalent *Export private key* option.
   - Keep a backup. Without this key you can't create a new API wallet or move funds, and nobody can recover it for you.
   - Never paste it into a website, a chat or a support request.
2. **Python 3.10 or newer**, and a terminal: macOS or Linux, or WSL on Windows. The `curl` commands below use bash quoting.

Set up a project folder with the official Hyperliquid Python SDK. Save every script in this guide in that folder and run it with `python <name>.py`:

```bash
mkdir pa-bot && cd pa-bot
python3 -m venv .venv && source .venv/bin/activate
pip install hyperliquid-python-sdk=={{ v.sdk_version }} python-dotenv
```

<div class="callout" markdown="1">
**Using another language?** The API is plain JSON over HTTPS and WebSocket. The steps and concepts below are identical. Only the signing needs a library; see [other languages](#other-languages).
</div>

## 3. Check your funding

These are read-only queries. They need no key, only your address.

```bash
export TEAM_WALLET=0xYourRegisteredAddress

# balance inside the dex (what you trade with)
curl -s {{ v.api_url }}/info -H 'Content-Type: application/json' \
  -d '{"type":"clearinghouseState","user":"'$TEAM_WALLET'","dex":"{{ v.dex }}"}'

# spot balance
curl -s {{ v.api_url }}/info -H 'Content-Type: application/json' \
  -d '{"type":"spotClearinghouseState","user":"'$TEAM_WALLET'"}'
```

Read the result:

- **`marginSummary.accountValue` is above 0.** You're ready. Go to [step 4](#create-your-api-wallet).
- **`balances` lists `{{ v.collateral }}` but `accountValue` is `"0.0"`.** Your funds are on spot. Move them into the dex (below).
- **Both are empty.** You haven't been funded yet, or this isn't the address you registered. Ask in your team's Discord channel.

Prefer a browser? The explorer shows the same balances: {{ v.explorer }}0xYourRegisteredAddress

### Move funds from spot into the dex (only if needed)

Transfers must be signed by the team wallet itself. This script asks for the key, doesn't store it, and moves your whole `{{ v.collateral }}` spot balance into `{{ v.dex }}`. At the prompt, paste the key and press Enter. Nothing shows while you paste, and it works with or without the `0x` prefix.

```python
# move_to_dex.py
import getpass

import eth_account
from hyperliquid.exchange import Exchange
from hyperliquid.info import Info
from hyperliquid.utils import constants

DEX = "{{ v.dex }}"
TOKEN = "{{ v.collateral_wire }}"  # name:tokenId; token names are not unique, the id is

team = eth_account.Account.from_key(getpass.getpass("Team wallet private key (not stored): ").strip())
info = Info(constants.TESTNET_API_URL, skip_ws=True)
spot = info.spot_user_state(team.address)["balances"]
amount = next((float(b["total"]) for b in spot if b["coin"] == TOKEN.split(":")[0]), 0.0)
if amount == 0:
    raise SystemExit("nothing on spot to move")

res = Exchange(team, constants.TESTNET_API_URL).send_asset(team.address, "spot", DEX, TOKEN, amount)
print(res)  # {'status': 'ok', 'response': {'type': 'default'}}
```

Run the dex-balance `curl` again. `accountValue` should now show the amount.

## 4. Create your API wallet

Your **team wallet** is the account: it holds the funds, it is what we score, and it is what every query uses. Your bot should never hold its key.

Instead, the team wallet signs **one** authorization for an **API wallet**, a fresh key pair. From then on your bot signs everything with the API wallet key. If that key leaks, the worst that can happen is bad trades; it can't move funds out.

This script:

1. generates the API wallet;
2. has your team wallet approve it, named and valid until after the competition;
3. writes your team wallet address and the API wallet key to `.env`, readable only by you. If you use git, add `.env` to `.gitignore`.

```python
# approve_api_wallet.py
import getpass
import os

import eth_account
from hyperliquid.exchange import Exchange
from hyperliquid.utils import constants

NAME = "bot1"                    # up to 3 named API wallets per account
VALID_UNTIL = 1_796_083_200_000  # 2026-12-01 00:00 UTC in ms, after the competition ends

team = eth_account.Account.from_key(getpass.getpass("Team wallet private key (not stored): ").strip())
exchange = Exchange(team, constants.TESTNET_API_URL)
result, api_key = exchange.approve_agent(f"{NAME} valid_until {VALID_UNTIL}")
if result.get("status") != "ok":
    raise SystemExit(f"approval failed: {result}")

api = eth_account.Account.from_key(api_key)
fd = os.open(".env", os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
with os.fdopen(fd, "w") as f:
    f.write(f"TEAM_WALLET_ADDRESS={team.address}\n")
    f.write(f"API_WALLET_PRIVATE_KEY={api_key}\n")
print(f"API wallet {api.address} ({NAME}) approved for {team.address}")
```

Confirm it's live:

```bash
curl -s {{ v.api_url }}/info -H 'Content-Type: application/json' \
  -d '{"type":"extraAgents","user":"'$TEAM_WALLET'"}'
# [{"name":"bot1","address":"0x…","validUntil":1796083200000}]
```

**Now put the team wallet key away.** You only need it again to move funds or to replace an API wallet.

Rules for API wallets:

- **One API wallet per running process.** Nonces are tracked per signing key, so two processes sharing one key will reject each other's requests. For a second bot, run the script again with `NAME = "bot2"` from that bot's own folder: the script overwrites `.env`.
- **Replacing one:** run the script again with the same name. The old key stops working immediately. Never reuse an old key.
- **Query with the team wallet address, never the API wallet's.** The API wallet has no balance, positions or orders of its own, so queries with its address come back empty.
- **An API wallet is removed if your account balance ever reaches zero,** or when it expires.
- **Don't change your account's "abstraction mode"** (unified or portfolio margin). It changes how balances across dexs are reported and isn't supported in this competition.

## 5. Connect from Python

Every snippet from here on imports this file:

```python
# client.py
import os

import eth_account
from dotenv import load_dotenv
from hyperliquid.exchange import Exchange
from hyperliquid.info import Info
from hyperliquid.utils import constants

load_dotenv()
DEX = "{{ v.dex }}"
ACCOUNT = os.environ["TEAM_WALLET_ADDRESS"]  # query with this
api_wallet = eth_account.Account.from_key(os.environ["API_WALLET_PRIVATE_KEY"])  # sign with this

info = Info(constants.TESTNET_API_URL, skip_ws=True, perp_dexs=[DEX])
exchange = Exchange(api_wallet, constants.TESTNET_API_URL, account_address=ACCOUNT, perp_dexs=[DEX])


def check(res):
    """Raise on any failure. Exchange replies are HTTP 200 even when an order was rejected."""
    if res.get("status") != "ok":
        raise RuntimeError(res.get("response"))
    data = res["response"].get("data", {}) if isinstance(res["response"], dict) else {}
    statuses = data.get("statuses", [])
    for s in statuses:
        if isinstance(s, dict) and "error" in s:
            raise RuntimeError(s["error"])
    return statuses
```

Two details in this file matter:

- `perp_dexs=[DEX]` loads only our markets. A stray `"BTC"` then raises an error instead of quietly using Hyperliquid's native BTC.
- `check()` exists because a rejected order still comes back with `"status": "ok"`. The rejection is in the per-order `statuses`. Always check them.

Test it:

```bash
python -c "from client import *; print(info.user_state(ACCOUNT, DEX)['marginSummary'])"
# {'accountValue': '10000.0', 'totalNtlPos': '0.0', 'totalRawUsd': '10000.0', 'totalMarginUsed': '0.0'}   (your balance)
```

## 6. Read the market

```python
# market.py
import time

from client import DEX, info

# Contract specs: lot size and max leverage per market
for m in info.meta(dex=DEX)["universe"]:
    print(f'{m["name"]:12} lot {10 ** -m["szDecimals"]:<8g} max leverage {m["maxLeverage"]}x')

# Mark, oracle, funding, open interest per market
meta, ctxs = info.post("/info", {"type": "metaAndAssetCtxs", "dex": DEX})
for m, c in zip(meta["universe"], ctxs):
    print(m["name"], "mark", c["markPx"], "oracle", c["oraclePx"], "funding/h", c["funding"], "OI", c["openInterest"])

print(info.all_mids(DEX))                  # {'{{ v.dex }}:BTC': '83369.0', ...}
print(info.l2_snapshot("{{ v.dex }}:BTC"))           # {'levels': [[bids...], [asks...]]}, level = {'px', 'sz', 'n'}

now = int(time.time() * 1000)
print(info.candles_snapshot("{{ v.dex }}:BTC", "1m", now - 3_600_000, now))  # [] if nothing traded in that hour
```

For backtesting: each market copies a live Hyperliquid mainnet market (`BTC`, `xyz:SP500`, …), so that market's history is a close proxy. The mapping is in the [reference](reference/#markets). For the organizers' own dataset, see the [FAQ](/faq/).

<div class="callout warn" markdown="1">
**Always name our dex.** Pass `"dex": "{{ v.dex }}"` to account and market-wide queries, and use the prefixed coin (`{{ v.dex }}:BTC`) everywhere else. With plain `BTC`, or without `dex`, the API returns Hyperliquid's native markets **with no error**. A strategy built on those prices is trading the wrong thing.
</div>

## 7. Your first trade

Three rules every order must satisfy (details in the [reference](reference/#price-and-size-rules)):

- **Size** is a multiple of the market's lot size (`szDecimals` decimals).
- **Price** has at most 5 significant figures and at most `6 − szDecimals` decimals.
- **Value** is at least $10.

This round trip places a post-only bid, cancels it, buys a little, then closes:

```python
# first_trade.py
from client import ACCOUNT, DEX, check, exchange, info

COIN = "{{ v.dex }}:ETH"
sz_decimals = next(m["szDecimals"] for m in info.meta(dex=DEX)["universe"] if m["name"] == COIN)


def px(p):  # 5 significant figures, at most 6 - szDecimals decimals
    return round(float(f"{p:.5g}"), 6 - sz_decimals)


def sz(s):  # round down to the lot size
    return int(s * 10**sz_decimals) / 10**sz_decimals


# 1. Leverage for this market: 3x cross (pass is_cross=False for isolated)
check(exchange.update_leverage(3, COIN))

# 2. Post-only bid 2% under mid, about $15
mid = float(info.all_mids(DEX)[COIN])
size = sz(15 / mid)
status = check(exchange.order(COIN, True, size, px(mid * 0.98), {"limit": {"tif": "Alo"}}))[0]
oid = status["resting"]["oid"]
print("resting", oid, info.open_orders(ACCOUNT, DEX))

# 3. Cancel it
check(exchange.cancel(COIN, oid))

# 4. Buy now: IOC, willing to pay up to 1% over mid
print(check(exchange.order(COIN, True, size, px(mid * 1.01), {"limit": {"tif": "Ioc"}})))
# [{'filled': {'totalSz': '0.0055', 'avgPx': '2684.8', 'oid': 61402157574}}]
print(info.user_state(ACCOUNT, DEX)["assetPositions"])

# 5. Close: reduce-only IOC
print(check(exchange.market_close(COIN)))
```

If step 4 stops with `Order could not immediately match against any resting orders`, nobody is selling within 1% of mid right now. Look at the order book (step 6) and run it again.

<div class="callout" markdown="1">
**What the SDK sends.** If you write your own client, step 2 goes to `POST {{ v.api_url }}/exchange` as:

```json
{
  "action": {
    "type": "order",
    "orders": [{"a": {{ v.dex_index | times: 10000 | plus: 100001 }}, "b": true, "p": "2628.5", "s": "0.0055", "r": false,
                "t": {"limit": {"tif": "Alo"}}}],
    "grouping": "na"
  },
  "nonce": 1790694813422,
  "signature": {"r": "0x…", "s": "0x…", "v": 27}
}
```

`a` is the asset id: `100000 + 10000 × dex index + position in meta`. `{{ v.dex }}` is dex index {{ v.dex_index }}, so `{{ v.dex }}:BTC` is {{ v.dex_index | times: 10000 | plus: 100000 }} and `{{ v.dex }}:ETH` is {{ v.dex_index | times: 10000 | plus: 100001 }}. Prices and sizes are strings with no trailing zeros. The signing scheme is in [Hyperliquid's docs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/signing).
</div>

## 8. Stream data

For anything live, use the WebSocket (`{{ v.ws_url }}`) instead of polling:

```python
# stream.py
import time

from hyperliquid.info import Info
from hyperliquid.utils import constants

from client import ACCOUNT, DEX

ws = Info(constants.TESTNET_API_URL, skip_ws=False, perp_dexs=[DEX])
ws.subscribe({"type": "l2Book", "coin": "{{ v.dex }}:BTC"}, print)     # book snapshots
ws.subscribe({"type": "trades", "coin": "{{ v.dex }}:BTC"}, print)     # public trades
ws.subscribe({"type": "userFills", "user": ACCOUNT}, print)   # your fills (team wallet address)
ws.subscribe({"type": "orderUpdates", "user": ACCOUNT}, print)  # your order status changes
while True:
    time.sleep(60)
```

The first `userFills` and `trades` messages after subscribing are **snapshots** of recent history (`"isSnapshot": true` on fills). Skip them, or de-duplicate by `tid`, so you don't count a fill twice.

The raw protocol:

- **Subscribe:** send `{"method": "subscribe", "subscription": {"type": "l2Book", "coin": "{{ v.dex }}:BTC"}}`.
- **Keep alive:** send `{"method": "ping"}` at least every 60 seconds.
- **Plan for disconnects:** reconnect, resubscribe, and re-read state over REST, since you may have missed messages.

The full list of streams is in the [reference](reference/#websocket).

## 9. From script to bot

You have every building block. A bot that survives three weeks also needs the following:

- **Start from reality.** On startup, read positions and open orders. Never assume you are flat.
- **Tag your orders.** Set a `cloid` on each order so you can match fills and updates to your own bookkeeping.
- **Check every response** with `check()` or equivalent. A rejected order doesn't raise by itself.
- **Clean up when your bot dies.** Cancel everything in your shutdown and exception handlers. Also run a separate watchdog, such as a cron job, that runs `kill.py` (below) when your bot stops updating a heartbeat file.
  - Hyperliquid's built-in dead man's switch, `schedule_cancel`, only unlocks once your account has traded **$1,000,000** in volume. Until then it is rejected, so don't rely on it.
- **Budget your requests.** Every order, cancel and modify spends from a per-account allowance that only grows as you trade. A bot that cancels and replaces quotes every second can drain it in minutes, then gets one request every 10 seconds. To avoid that:
  - prefer `modify` over cancel-and-replace;
  - batch orders;
  - watch `{"type": "userRateLimit", "user": "0x…"}`.

  See the [rate limits](reference/#rate-limits).
- **Keep the clock synced** (NTP). Nonces are millisecond timestamps, and requests too far from server time are rejected.
- **Log every request and response.** You will submit your code, and logs make the jury walkthrough and debugging much easier.
- **Run it where it stays up.** Use a small cloud server, not a laptop. To be ranked you must trade on at least 10 distinct days.
- **Have a kill switch**, and run it whenever things look wrong:

```python
# kill.py: cancel everything and close every position
from client import ACCOUNT, DEX, exchange, info

for o in info.open_orders(ACCOUNT, DEX):
    print(exchange.cancel(o["coin"], o["oid"]))
for p in info.user_state(ACCOUNT, DEX)["assetPositions"]:
    print(exchange.market_close(p["position"]["coin"]))
```

### Other languages

- **TypeScript:** [`@nktkas/hyperliquid`](https://github.com/nktkas/hyperliquid). It supports builder dexs through its `dex` parameters and `SymbolConverter` for asset ids.
- **Rust:** [`hyperliquid_rust_sdk`](https://github.com/hyperliquid-dex/hyperliquid-rust-sdk), the official one.
- **Anything else:** the [API docs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api), which include the signing scheme.

Whatever you use, point it at **testnet** and at the dex `{{ v.dex }}`.

## 10. Rules your code must respect

The full rules are on the [About](/about) page. These are the ones that shape code:

- **Trading must be algorithmic.** Your report and a code walkthrough with the jury will check this.
- **One account per team:** the wallet you registered. All trading goes through it, including via API wallets.
- **No self-crossing, wash trading or collusion.** If your own order would hit your own resting order, Hyperliquid cancels the resting one. Design your bot so this doesn't happen at all.
- **To be ranked:** trade on at least 10 distinct days and reach the minimum cumulative volume.
- **Don't attack or flood the exchange or the oracle.**
- **Keep your code:** the code and a 2–4 page report are due on 9 November.

## 11. Getting help

- **Where:** ask in `#tech-support` on Discord. For anything involving your address or positions, use your team's private channel.
- **What to include:**
  - your team wallet address;
  - the time (UTC);
  - the coin, and the `oid` or `cloid`;
  - the exact error text;
  - the snippet you ran.
- **Never post a private key**, not even the API wallet's. Organizers will never DM you first and will never ask for a key or a seed phrase.

</div>
