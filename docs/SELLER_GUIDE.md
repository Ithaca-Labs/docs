# Build a paid HTTP seller

Protect an Express route with Stellar `exact`, advertise official Bazaar
metadata, and settle through an openx402 facilitator.

## Install

From your seller project:

```bash
npm install @openx402/bazaar-sdk \
  @x402/core@2.20.0 \
  @x402/express@2.20.0 \
  @x402/stellar@2.20.0 \
  express
```

Use Node.js 22 or newer. The repository's complete runnable version is
[`examples/rock-paper-scissors/index.ts`](../examples/rock-paper-scissors/index.ts).

## Complete POST seller

The following is the same payment integration used by the included example;
only the application handler is shortened.

```ts
import express from "express";
import { bazaar } from "@openx402/bazaar-sdk";
import { HTTPFacilitatorClient } from "@x402/core/server";
import { paymentMiddleware, x402ResourceServer } from "@x402/express";
import { ExactStellarScheme } from "@x402/stellar/exact/server";

const PORT = Number(process.env.PORT ?? 4788);
const NETWORK = "stellar:testnet" as const;
const FACILITATOR_URL = process.env.FACILITATOR_URL
  ?? "https://facilitator-production-8430.up.railway.app";
const PUBLIC_URL = process.env.SELLER_PUBLIC_URL?.replace(/\/$/, "");
const PAY_TO = process.env.SELLER_PAY_TO;

// Stellar testnet native XLM Stellar Asset Contract.
const XLM_SAC = "CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC";
const PRICE_ATOMIC = "1000";

if (!PUBLIC_URL?.startsWith("https://")) {
  throw new Error("SELLER_PUBLIC_URL must be the public HTTPS tunnel origin");
}
if (!PAY_TO) throw new Error("SELLER_PAY_TO is required");

const metadata = bazaar.http({
  description: "Plays one round of rock, paper, scissors against the server.",
  serviceName: "Rock Paper Scissors",
  tags: ["game", "rps", "random"],
  method: "POST",
  body: {
    move: {
      type: "string",
      description: "Player move: rock, paper, or scissors.",
      enum: ["rock", "paper", "scissors"],
      required: true,
      example: "rock",
    },
  },
  output: {
    type: "json",
    description: "The player move, server move, and round result.",
    example: { player: "rock", server: "scissors", result: "win" },
  },
});

const facilitator = new HTTPFacilitatorClient({ url: FACILITATOR_URL });
const resourceServer = new x402ResourceServer(facilitator)
  .register(NETWORK, new ExactStellarScheme());

const app = express();
app.use(express.json({ limit: "16kb" }));

app.use(paymentMiddleware({
  "POST /play": {
    accepts: [{
      scheme: "exact",
      network: NETWORK,
      price: { asset: XLM_SAC, amount: PRICE_ATOMIC },
      payTo: PAY_TO,
      maxTimeoutSeconds: 60,
      extra: { areFeesSponsored: true },
    }],
    resource: `${PUBLIC_URL}/play`,
    description: metadata.resource.description,
    serviceName: metadata.resource.serviceName,
    tags: metadata.resource.tags,
    mimeType: "application/json",
    extensions: metadata.extensions,
  },
}, resourceServer));

app.post("/play", (req, res) => {
  const move = req.body?.move;
  if (!["rock", "paper", "scissors"].includes(move)) {
    res.status(400).json({ error: "move must be rock, paper, or scissors" });
    return;
  }
  res.json({ player: move, server: "scissors", result: move === "rock" ? "win" : "lose" });
});

app.listen(PORT, "127.0.0.1");
```

From the seller project:

```bash
SELLER_PUBLIC_URL=https://your-tunnel.example \
SELLER_PAY_TO=G... \
FACILITATOR_URL=https://facilitator-production-8430.up.railway.app \
npx tsx index.ts
```

## How the integration fits together

| API | Responsibility |
| --- | --- |
| `HTTPFacilitatorClient` | Calls the configured facilitator's x402 HTTP endpoints. It does not hold the seller or buyer key. |
| `x402ResourceServer` | Coordinates server-side verification and settlement and holds the registered network schemes. |
| `ExactStellarScheme` from `@x402/stellar/exact/server` | Interprets Stellar `exact` payment payloads on the resource-server side. |
| `paymentMiddleware` | Returns the first 402 challenge, reads payment headers on the retry, invokes the resource server, and exposes settlement headers. |
| `bazaar.http()` | Compiles readable endpoint metadata and HTTP parameters to official Bazaar metadata and JSON Schema. |

Do not hand-write Bazaar JSON. Use `metadata.resource` and
`metadata.extensions` so the official extension shape stays aligned with the
x402 SDK.

## Payment and resource fields

### `accepts`

Each entry is one payment option. This example offers one option: Stellar
testnet, `exact`, 1,000 atomic units of native XLM SAC, paid to `SELLER_PAY_TO`.

Amounts are decimal strings in the token's atomic units. Do not use floating
point and do not assume every Stellar token has seven decimals. Read decimals
from the asset configuration or token interface.

`maxTimeoutSeconds` limits how long the signed authorization may remain valid.
The facilitator also applies an operator maximum; the seller cannot extend it
indefinitely.

`extra.areFeesSponsored: true` tells the buyer that the facilitator pays the
Stellar transaction fee. It does not make the facilitator the payer. The buyer
still owns and supplies the payment asset.

### `payTo` and the four addresses

| Address | Meaning |
| --- | --- |
| Seller address (`G...` or compatible account) | Receives the payment asset through `payTo`. |
| Buyer payer address | Owns the token balance and signs the payment authorization. |
| Facilitator sponsor address | Pays Stellar transaction fees and signs the fee-bump envelope. |
| Token contract (`C...`) | Identifies the SEP-41 token or Stellar Asset Contract being transferred. |

The seller receives the asset. The facilitator is never the recipient, payer,
or custodian of seller funds.

### `resource`

`resource` is the externally reachable identity of the paid route. It must be
the URL the buyer calls, including the actual path. For hosted discovery use a
public HTTPS URL. Do not advertise the local listener, a different TLS origin,
or a logical name that clients cannot reach.

The hosted catalog rejects loopback/private hosts and plain-HTTP public
origins. `SELLER_PUBLIC_URL=https://your-tunnel.example` and
`resource=https://your-tunnel.example/play` keep the challenge, buyer request,
and catalog identity consistent.

### `extensions`

Pass `metadata.extensions` unchanged. The Bazaar SDK delegates to
`@x402/extensions/bazaar`; it does not create a proprietary extension. Invalid
metadata is reported through `EXTENSION-RESPONSES` and soft-fails without
changing a valid payment result.

## GET and query endpoints

Use `query` for a GET endpoint. The helper compiles parameter examples and
descriptions into both Bazaar `info.input.queryParams` and JSON Schema.

```ts
const weather = bazaar.http({
  description: "Returns the current weather for a city.",
  serviceName: "Weather API",
  tags: ["weather", "forecast"],
  method: "GET",
  query: {
    city: {
      type: "string",
      description: "The city to look up, such as Mumbai or London.",
      required: true,
      example: "Mumbai",
    },
    units: {
      type: "string",
      description: "Temperature units.",
      enum: ["celsius", "fahrenheit"],
      example: "celsius",
    },
  },
  output: {
    type: "json",
    example: { city: "Mumbai", temperature: 29, condition: "Sunny" },
  },
});
```

Place `weather.resource` fields and `weather.extensions` in the route's x402
configuration exactly as in the POST example. The canonical standalone helper
is [`packages/bazaar-sdk/examples/weather-http.ts`](../packages/bazaar-sdk/examples/weather-http.ts).

## Next steps

- [Run and understand the buyer client](BUYER_CLIENT.md)
- [Understand catalog identity, trust, and discovery](BAZAAR.md)
- [Troubleshoot repeated 402s and rejected metadata](TROUBLESHOOTING.md)
