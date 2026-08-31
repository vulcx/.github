# Vulcx

**Swap routing for [Fogo](https://fogo.io).** Given a token pair and an amount, Vulcx
finds the best execution across every pool on the chain and hands back a transaction
you can sign.

You don't pay us for calls. Vulcx charges a protocol fee on each swap; you can add
your own fee on top and **keep 100% of it**. The two rates are independent and they
add — yours is not a share of ours, and ours does not shrink when you raise yours.
Both are collected on-chain by the aggregator program and reported separately on every
response, because one blended number cannot tell you what you earned.

```bash
curl -H "Authorization: Bearer $VULCX_KEY" \
  "https://api.vulcx.xyz/api/v1/quote?inputMint=So11111111111111111111111111111111111111112\
&outputMint=uSd2czE61Evaf76RNbq4KPpXnkiL3irdzgLFUMe3NoG&amount=1000000000&swapMode=ExactIn"
```

Everything below is detail.

---

## What's here

| Repo | Stack | What it does |
|---|---|---|
| **route-engine** | Go | The routing engine and REST API. Pathfinding, quoting, transaction building. One binary. |
| **aggregator_contract** | Rust / Anchor | The on-chain program that executes a routed swap atomically. `Vu1cUxynmbPUsFqVz51FJJ2y69vX2yrkTS13ajomd9D` |
| **sdk** | TypeScript | `@vulcx/sdk` — a thin typed client over the HTTP API. |
| **vulcan** | Docker | Self-hosting: run the whole engine yourself against your own RPC. **Not released yet.** |

Dependency direction is one way: the contract defines the instruction layout, the
engine targets it, the SDK wraps the engine.

---

## How it works

Three endpoints carry all the traffic:

```
GET  /api/v1/quote          price discovery — what will I get, and what will it cost
POST /api/v1/swap           a signed-ready transaction, built server-side
POST /api/v1/instructions   raw instructions, so you compose the transaction yourself
```

Use `/swap` when you want the fast path. Use `/instructions` when you need to add
your own instructions around the swap — creating token accounts, attaching a memo,
wrapping the swap inside something larger.

**Liquidity sources on Fogo:** Valiant (concentrated liquidity), Fluxbeam
(constant product), and Moonit (bonding curves). Routes go up to 5 hops
and may split across several pools when splitting beats any single path.

**Pool state is live, not polled.** The engine consumes account updates as the chain
produces them and keeps pool state in memory, so a quote reads state that is current
to the slot rather than to the last refresh. Nothing in the quote path touches RPC.

**The watched set is complete.** It covers every pool and the accounts behind it —
tick arrays, vaults — so a quote is never priced off a partial book. No state is
silently missed.

### Firm quotes

Every quote carries a `quoteId`. Hand it back to `/swap` or `/instructions` and the
engine replays that exact route with min-out anchored to the price you were quoted,
rather than re-routing against whatever the book looks like a second later.

Add `"firm": true` inside the firm window and slippage collapses to a fixed margin:
price-or-fail. If the market moved past it, you get a `409` before a transaction is
ever built — not a signed transaction that reverts on chain.

Quotes are Ed25519-signed. The verification key is public at
`/.well-known/vulcx-quote-signer`, so a quote can be verified by someone who does not
trust the server that issued it.

### Streaming

`wss://api.vulcx.xyz/api/v1/stream` — subscribe to pairs, receive a quote push when
the underlying pools change rather than on a timer. The stream also emits
`{"type":"invalidate","quoteId"}` when a firm quote drifts out of its margin, so a UI
can grey out a stale price instead of letting someone click it.

### Calling from another program

`POST /api/v1/cpi/route-accounts` returns the account list a program needs to CPI into
the aggregator. If you are building something that swaps as part of a larger
instruction — a liquidation, a rebalance — this is the entry point.

---

## Fees

Two independent rates on every swap:

| | Who sets it | Who keeps it |
|---|---|---|
| `platformFeeBps` | Vulcx | Vulcx |
| `integratorFeeBps` | you, per request | **you, in full** |

They add. Their **sum is capped at 100 bps (1%)**, enforced both on-chain and in the
API — so an over-cap request comes back as a `400` rather than as a transaction that
fails after your user has signed it.

Every quote, swap, instruction set and CPI account list reports both rates
separately. Partner agreements can lower Vulcx's side; nothing lowers yours.

---

## Rate limits

**There is one rate limit and every caller gets the same one: 100 cost units per
second, burst 200.**

Buckets are debited by what a request costs us, not by how many requests you made:

| Request | Cost |
|---|---|
| `/quote`, `/price` | 1 — an in-memory read |
| `/pools/*`, `/cpi/route-accounts` | 3 — a corpus walk, or an account-set build |
| `/swap`, `/instructions` | 5 — routing plus transaction construction |

So the same budget is 100 quotes per second, or 20 transaction builds per second, or
any mix. One number, seen through the cost table, instead of two numbers that drift
apart.

Deliberately flat: a rate ceiling derived from trailing volume cannot be
capacity-planned against, and it throttles a new integration hardest on day one —
exactly when its quote-to-swap ratio is at its worst and it can least afford to be
told to slow down.

---

## Getting started

**TypeScript**

```bash
npm install @vulcx/sdk
```

```ts
import { VulcxSDK } from '@vulcx/sdk'

const vulcx = new VulcxSDK({ apiKey: process.env.VULCX_KEY })

const quote = await vulcx.quote({
  inputMint: 'So11111111111111111111111111111111111111112',
  outputMint: 'uSd2czE61Evaf76RNbq4KPpXnkiL3irdzgLFUMe3NoG',
  amount: '1000000000',
  swapMode: 'ExactIn',
})
```

**Anything else** — the API is plain HTTP and JSON. Any language with a socket works.

Get a key at **[vulcx.xyz](https://vulcx.xyz)**. Docs:
**[docs.vulcx.xyz](https://docs.vulcx.xyz)** · Machine-readable index at
[`/llms.txt`](https://docs.vulcx.xyz/llms.txt).

---

## Self-hosting — not released yet

There is nothing to download today. Ask on [Telegram](https://t.me/vulcxsupport) to
be told when it opens.

When it does: the engine is one binary with no CGO and no external database in the
request path, and `vulcan` has the Docker setup. You supply an RPC endpoint with
Yellowstone gRPC access; everything else is in the image.

Self-hosting will change who runs the engine. It will not change the on-chain fee
model: the protocol fee is collected by the program, not by the API.

---

## Chains

Fogo today, and the engine is Fogo-specific in exactly one place that matters: how
pool state is ingested. Solana support is planned; the blocker is ingestion and
per-DEX dispatch, not the routing itself.

Anything claiming Solana support today is early. `CHAIN=solana` refuses to start on
purpose, rather than running a build that would quote against a chain it cannot
execute on.

---

## License and contributions

Each repo carries its own license and contribution guide. Issues and pull requests are
welcome on all of them; the routing engine and the on-chain program are where the
interesting problems live.

If you find a way to get a wrong quote out of the engine, that is the bug report we
most want to receive.
