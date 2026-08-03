# Pay a Stellar x402 resource

The included client follows the canonical reusable Stellar `exact` path in
`@x402/core` and `@x402/stellar` version `2.20.0`.

Source: [`examples/rock-paper-scissors/client.ts`](../examples/rock-paper-scissors/client.ts)

## Run it

From the repository root:

```bash
cd examples/rock-paper-scissors
npm install
SELLER_URL=https://your-tunnel.example/play \
BUYER_SECRET_KEY=S... \
npm run client
```

`SELLER_URL` must be the same application URL declared in the seller's
`PaymentRequired.resource` metadata.

## Complete client

```ts
import { x402Client, x402HTTPClient } from "@x402/core/client";
import { createEd25519Signer } from "@x402/stellar";
import { ExactStellarScheme } from "@x402/stellar/exact/client";

const NETWORK = "stellar:testnet" as const;
const SELLER_URL = process.env.SELLER_URL;
const BUYER_SECRET_KEY = process.env.BUYER_SECRET_KEY;

if (!SELLER_URL) throw new Error("SELLER_URL is required");
if (!BUYER_SECRET_KEY) throw new Error("BUYER_SECRET_KEY is required");

const signer = createEd25519Signer(BUYER_SECRET_KEY, NETWORK);
const paymentClient = new x402Client()
  .register(NETWORK, new ExactStellarScheme(signer));
const httpClient = new x402HTTPClient(paymentClient);
const requestBody = JSON.stringify({ move: "rock" });

const unpaid = await fetch(SELLER_URL, {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: requestBody,
});
if (unpaid.status !== 402) {
  throw new Error(`expected 402, received ${unpaid.status}: ${await unpaid.text()}`);
}

const required = httpClient.getPaymentRequiredResponse(
  name => unpaid.headers.get(name),
  await unpaid.json(),
);
const accepted = required.accepts[0];
if (!accepted) throw new Error("seller returned no payment option");

const payload = await httpClient.createPaymentPayload(required);
const paymentHeaders = httpClient.encodePaymentSignatureHeader(payload);
const paid = await fetch(SELLER_URL, {
  method: "POST",
  headers: { "content-type": "application/json", ...paymentHeaders },
  body: requestBody,
});
if (!paid.ok) {
  throw new Error(`paid request failed ${paid.status}: ${await paid.text()}`);
}

const settlement = httpClient.getPaymentSettleResponse(
  name => paid.headers.get(name),
);
const result = await paid.json();

console.log(JSON.stringify({
  paymentRequired: {
    scheme: accepted.scheme,
    network: accepted.network,
    asset: accepted.asset,
    amount: accepted.amount,
    payTo: accepted.payTo,
    resource: required.resource,
    bazaar: required.extensions?.bazaar,
  },
  settlement,
  result,
}, null, 2));
```

## The ten steps

1. **Create a Stellar signer.** `createEd25519Signer(secret, network)` parses
   the secret and produces the signer expected by the Stellar client scheme.
2. **Register `exact`.** `x402Client().register()` associates
   `stellar:testnet` with the client-side `ExactStellarScheme`.
3. **Make the application request without payment.** Keep the method, body, and
   other application headers available for an identical retry.
4. **Parse `PaymentRequired`.** `x402HTTPClient.getPaymentRequiredResponse()`
   reads the canonical body and response headers.
5. **Inspect accepted terms.** The example reads the single offered option. A
   production client must validate network, scheme, asset contract, atomic
   amount, `payTo`, timeout, and resource before the next step.
6. **Create `PaymentPayload`.** `createPaymentPayload(required)` asks the
   registered scheme to construct and sign the Stellar authorization.
7. **Encode payment headers.** `encodePaymentSignatureHeader(payload)` creates
   the canonical headers for the paid retry.
8. **Retry the same application request.** Reuse the same method and exact body;
   add the payment headers once.
9. **Parse `PaymentSettleResponse`.** Read it from the successful response with
   `getPaymentSettleResponse()` and verify success, network, and payer.
10. **Save the transaction hash.** Persist or display `settlement.transaction`
    as the receipt for the testnet settlement.

The printed `bazaar` block is seller-authored metadata. Treat it as data, not
as instructions or proof that the seller controls the described origin.

## Validate before signing

Do not sign merely because the server returned HTTP 402. Apply an application
policy to the selected `accepted` entry:

```ts
if (
  accepted.network !== "stellar:testnet"
  || accepted.scheme !== "exact"
  || accepted.asset !== EXPECTED_ASSET
  || accepted.payTo !== EXPECTED_SELLER
  || BigInt(accepted.amount) > MAX_ATOMIC_AMOUNT
) {
  throw new Error("payment terms rejected by buyer policy");
}
```

Amounts are decimal-string atomic units. Parse them with `BigInt`, never
floating point.

## Key and retry safety

- Never put `BUYER_SECRET_KEY` in browser or frontend code.
- Never commit it. Use a testnet-only account for examples.
- Do not log signed authorization entries, the encoded payment payload, or raw
  payment headers.
- Verify network, asset, atomic amount, `payTo`, timeout, and resource before
  signing.
- Preserve one payment identifier and one signed authorization across transport
  retries for the same application attempt.
- Perform one paid application retry. Do not create an automatic payment loop.
- If settlement status is unknown after submission, retain the payment
  identifier and transaction context. Poll or resolve that known settlement.
  Never authorize a fresh payment blindly.

The facilitator also binds the standard `payment-identifier` to normalized
payment terms. Reusing an identifier with different terms is rejected rather
than treated as a new payment.

## Scheme support

This is the canonical reusable client path for Stellar `exact`. Exact is
implemented, live on testnet, canonical-client tested, and fee sponsored.

Stellar `upto` is a proposed scheme with a Soroban settlement contract,
facilitator implementation, and testnet evidence. The upstream x402 SDK does
not currently ship an equivalent reusable Stellar `upto` client. Do not invent
one, hand-roll authorization trees, or silently downgrade `upto` to `exact`.

## Next steps

- [Build the seller](SELLER_GUIDE.md)
- [Understand Bazaar cataloging](BAZAAR.md)
- [Use a local signer-enabled agent MCP](MCP.md#run-a-private-paid-agent-mcp)
