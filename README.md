# x402 on Stellar

> **HTTP-native micropayments on the Stellar blockchain — for APIs, AI agents, and the open web.**

[![Live Demo](https://img.shields.io/badge/demo-x402stellar.vercel.app-blueviolet?style=flat-square)](https://x402stellar.vercel.app)
[![Network](https://img.shields.io/badge/network-Stellar-00c0c7?style=flat-square)](https://stellar.org)
[![Protocol](https://img.shields.io/badge/protocol-x402-black?style=flat-square)](https://x402.org)
[![Facilitator](https://img.shields.io/badge/facilitator-OpenZeppelin-4E5EE4?style=flat-square)](https://openzeppelin.com)
[![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)

---

## What is x402?

[x402](https://x402.org) is an open payment protocol that revives the long-reserved **HTTP 402 Payment Required** status code and turns it into a working standard. Originally incubated by Coinbase and now governed by the [x402 Foundation](https://github.com/x402-foundation/x402) (Linux Foundation member since April 2026), the protocol lets any API or web service charge **per request** — with no accounts, no billing systems, and no API keys required.

The flow is simple:

```
Client → GET /resource
Server ← 402 Payment Required  (PAYMENT-REQUIRED header with price/network/recipient)
Client → GET /resource + PAYMENT-SIGNATURE header  (signed stablecoin transfer)
Server → 200 OK  (resource delivered)
```

One HTTP round-trip. No pre-existing relationship between buyer and seller.

---

## Why Stellar?

Stellar joined the x402 ecosystem in **March 2026** with a production-ready facilitator built by the Stellar Development Foundation (SDF) in partnership with OpenZeppelin. Stellar's properties make it uniquely suited for x402 micropayments:

| Property | Stellar |
|---|---|
| **Transaction fee** | ~$0.00001 (fee never exceeds the payment) |
| **Finality** | ~5 seconds |
| **Uptime** | 99.99% across 20.6 billion total operations |
| **Native stablecoins** | USDC, PYUSD, USDY — first-class, not bridged |
| **Smart contracts** | Soroban — programmable spending limits & policies |
| **Privacy controls** | Configurable per-asset authorization flags |

On most other networks, the gas fee would exceed the value of a micropayment. On Stellar, it is essentially free.

---

## Live Demo

**[x402stellar.vercel.app](https://x402stellar.vercel.app)**

Send an x402 payment on Stellar **Testnet or Mainnet** and watch the full payment flow in action — from the 402 response to on-chain settlement.

---

## Architecture

```
┌─────────────────────┐        HTTP 402         ┌──────────────────────┐
│                     │ ──────────────────────► │                      │
│     Client          │                          │   Resource Server    │
│  (AI agent / user)  │ ◄────────────────────── │  (Express + x402     │
│                     │   200 OK + resource      │   middleware)        │
└──────────┬──────────┘                          └──────────┬───────────┘
           │  PAYMENT-SIGNATURE header                      │  /verify + /settle
           │  (Soroban auth entry)                          ▼
           │                                    ┌──────────────────────┐
           │                                    │  Built on Stellar    │
           └───────────────────────────────────►│  x402 Facilitator   │
                                                │  (OpenZeppelin       │
                                                │   Relayer)           │
                                                └──────────┬───────────┘
                                                           │
                                                           ▼
                                                  Stellar Blockchain
                                                  (USDC settlement)
```

The **Built on Stellar x402 Facilitator** (powered by OpenZeppelin Relayer and Channels) handles:
- Payment verification (signature validity, balance check, replay attack prevention)
- On-chain settlement to the seller's Stellar address
- Network fee coverage — sellers and buyers pay zero gas

---

## Quick Start

### Prerequisites

- Node.js (LTS recommended)
- A Stellar testnet wallet with XLM and USDC ([fund here](https://lab.stellar.org/account/fund), [USDC faucet](https://faucet.circle.com))
- An OpenZeppelin Facilitator API key ([generate testnet key — no auth required](https://channels.openzeppelin.com/testnet/gen))

### Install

```bash
npm install express dotenv @stellar/stellar-sdk \
  @x402/core @x402/express @x402/fetch @x402/stellar
```

### Server (Seller)

```js
import express from "express";
import { paymentMiddleware, x402ResourceServer } from "@x402/express";
import { ExactStellarScheme } from "@x402/stellar/exact/server";
import { HTTPFacilitatorClient } from "@x402/core/server";

const facilitatorClient = new HTTPFacilitatorClient({
  url: "https://channels.openzeppelin.com/x402/testnet",
  createAuthHeaders: async () => ({
    verify: { Authorization: `Bearer YOUR_API_KEY` },
    settle: { Authorization: `Bearer YOUR_API_KEY` },
    supported: { Authorization: `Bearer YOUR_API_KEY` },
  }),
});

const app = express();

app.use(
  paymentMiddleware(
    {
      "GET /weather": {
        accepts: [{
          scheme: "exact",
          price: "$0.001",           // Dollar-string — SDK converts to on-chain units
          network: "stellar:testnet",
          payTo: "YOUR_STELLAR_ADDRESS",
        }],
        description: "Real-time weather data",
        mimeType: "application/json",
      },
    },
    new x402ResourceServer(facilitatorClient).register(
      "stellar:testnet",
      new ExactStellarScheme(),
    ),
  ),
);

app.get("/weather", (req, res) => {
  res.json({ weather: "sunny", temperature: 70 });
});

app.listen(4021, () => console.log("x402 server running on :4021"));
```

### Client (Buyer / Agent)

```js
import dotenv from "dotenv";
import { x402Client } from "@x402/fetch";
import { createEd25519Signer } from "@x402/stellar";
import { ExactStellarScheme } from "@x402/stellar/exact/client";

dotenv.config();

const signer = createEd25519Signer(
  process.env.STELLAR_PRIVATE_KEY,
  "stellar:testnet",
);

const client = x402Client({
  signer,
  schemes: [new ExactStellarScheme()],
  rpcUrl: "https://soroban-testnet.stellar.org",
});

const res = await client.fetch("http://localhost:4021/weather");
console.log(await res.json());
// { weather: 'sunny', temperature: 70 }
```

---

## Payment Schemes

| Scheme | Description |
|---|---|
| `exact` | Transfer a fixed amount per request (e.g., `$0.001`) |
| `upto` | Authorize up to a maximum; seller settles actual usage |
| `batch-settlement` | Off-chain vouchers redeemed on-chain in batches (EVM) |

This demo uses the **`exact`** scheme with USDC on Stellar.

---

## Supported Assets

Stellar's x402 facilitator supports SEP-41 compatible assets. Default assets for dollar-string pricing:

| Asset | Contract Address |
|---|---|
| USDC (testnet) | `CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA` |

For other assets, specify `price` as an explicit `{ asset, amount }` object using the token's SEP-41 contract address and amount in base units.

---

## Facilitator Endpoints

| Environment | Base URL |
|---|---|
| Testnet | `https://channels.openzeppelin.com/x402/testnet` |
| Mainnet | `https://channels.openzeppelin.com/x402` |

Standard x402 facilitator endpoints: `/verify`, `/settle`, `/supported`.

---

## Compatible Wallets

To sign Soroban auth entries (required for x402 on Stellar), use a wallet with **auth-entry signing** support. Check the [Stellar x402 docs](https://developers.stellar.org/docs/build/apps/x402) for the current list of compatible wallets.

---

## What's Next

The roadmap for x402 on Stellar includes:

- **MCP Integration** — `x402-MCP` server so AI agents can discover paid resources, authorize payments via smart wallets, and chain multiple paid API calls within user-defined spending policies
- **Embedded Smart Wallets** — OpenZeppelin smart account contracts with spending limits and programmable policies, combined with wallet providers like Crossmint
- **Multi-chain cost optimization** — x402 V2 lets clients select the optimal settlement chain per payment; Stellar's near-zero fees make it the natural default for micropayments

---

## Resources

- [x402 on Stellar — Stellar Docs](https://developers.stellar.org/docs/build/apps/x402)
- [Built on Stellar x402 Facilitator — Stellar Docs](https://developers.stellar.org/docs/build/apps/x402/built-on-stellar)
- [x402 Quickstart Guide](https://developers.stellar.org/docs/build/agentic-payments/x402/quickstart-guide)
- [x402 Protocol Specification](https://github.com/x402-foundation/x402)
- [OpenZeppelin x402 Facilitator Plugin](https://github.com/OpenZeppelin/openzeppelin-relayer)
- [Stellar Development Foundation — x402 announcement](https://stellar.org/blog/foundation-news/x402-on-stellar)
- [x402 Foundation](https://x402.org)

---

## License

[MIT](LICENSE)

---

> x402 is an open-source payment protocol. It is not a regulated financial service. Neither the Stellar Development Foundation nor the Stellar network operates, controls, or intermediates x402 transactions.
