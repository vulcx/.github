<div align="center">

# Vulcx

### The best-price swap router for Fogo Chain.

Give it a token pair and an amount. Vulcx searches every pool on Fogo — up to five hops, splitting size across paths when splitting wins — and hands back a transaction ready to sign.

[![Try a swap](https://img.shields.io/badge/TRY_A_SWAP-0B0B0F?style=for-the-badge)](https://vulcx.xyz)
[![Free during beta](https://img.shields.io/badge/FREE_DURING_BETA-FF3D01?style=for-the-badge&labelColor=0B0B0F)](https://docs.vulcx.xyz/get-started/authentication)
[![Read the docs](https://img.shields.io/badge/READ_THE_DOCS-0B0B0F?style=for-the-badge)](https://docs.vulcx.xyz)
[![Status](https://img.shields.io/badge/STATUS-0B0B0F?style=for-the-badge)](https://vulcx.xyz/status/)

</div>

---

## Routing built for Fogo

Vulcx gives builders production-ready swap routing on Fogo without running any infrastructure of their own.

- **Multi-hop, split routing** — up to five hops per path, with size split across multiple routes when that beats a single path.
- **Start with no key** — `/quote`, `/swap`, `/instructions` and `/price` answer anonymous requests, so the first call costs you nothing to set up. A free key raises your budget from 1 to 100 cost units per second; the WebSocket stream is the one endpoint that always needs one.
- **Firm quotes** — every quote carries a `quoteId`. Redeem it and you commit at the price you were shown, or the request fails cleanly — no signed transaction that reverts on chain.
- **Transaction builder** — get back a ready-to-sign transaction, or raw instructions if you're composing your own.
- **On-chain settlement** — a single Solana program executes the routed swap atomically, including split routes, in one transaction.

Revenue comes from the on-chain protocol fee, not from gating API access — which is why the API stays usable without an account.

## Start building

```bash
npm install @vulcx/sdk
```

```typescript
import { VulcxSDK } from "@vulcx/sdk";

// The key is optional — omit it to try things out anonymously.
const vulcx = new VulcxSDK({ apiKey: process.env.VULCX_KEY });

const quote = await vulcx.quote({
  inputMint: "So11111111111111111111111111111111111111112",
  outputMint: "uSd2czE61Evaf76RNbq4KPpXnkiL3irdzgLFUMe3NoG",
  amount: "1000000000", // 1 FOGO (9 decimals)
  swapMode: "ExactIn",
  slippageBps: 50,
});

console.log(`Output: ${quote.amountOut}, Impact: ${quote.priceImpactPercent}`);
```

Keys are issued by hand during beta and cost nothing — [ask on Telegram](https://t.me/vulcxsupport).

## Repositories

| Repo | What it is |
| --- | --- |
| [`sdk`](https://github.com/vulcx/sdk) | TypeScript client for the route engine's HTTP API |
| [`vulcx-cpi`](https://github.com/vulcx/vulcx-cpi) | Rust crate for calling the aggregator's `route` instruction from your own program via CPI |
| [`vulcx-docs`](https://github.com/vulcx/vulcx-docs) | Integration guides and API reference |
| [`landing_page`](https://github.com/vulcx/landing_page) | vulcx.xyz, including the live quote panel and the status page |

<sub>The route engine and on-chain settlement program aren't open source. Self-hosting the engine is planned and not released yet.</sub>

---

<div align="center">

[vulcx.xyz](https://vulcx.xyz) · [docs.vulcx.xyz](https://docs.vulcx.xyz) · [status](https://vulcx.xyz/status/) · [@vulcx_xyz](https://x.com/vulcx_xyz)

</div>
